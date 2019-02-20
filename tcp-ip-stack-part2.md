# Linux内核中的 TCP/IP
前面一章介绍了Socket族相关的接口, 那么接下来我们来看看Linux内核是怎么实现 `TCP/IP协议` 的吧.

## 数据接收
我们先不考虑内核是怎么实现 `TCP/IP协议` 的, 而是先来了解下当网卡接收到数据时, 内核有什么相应的处理. 

当网卡接收到数据时, 会向CPU发送 `中断信号`. 当CPU接收到 `中断信号` 后会调用相应的 `中断处理程序` 来处理中断. 因为不同的网卡处理数据的方式不一样, 所以不同的网卡的 `中断处理程序` 也不一样. 譬如 `NS8390芯片` 网卡的中断处理程序是 `ei_interrupt()`, 下面我们也主要使用这个中断处理程序作为例子, 起代码如下:
```cpp
void ei_interrupt(int reg_ptr)
{
    ...
    while ((interrupts = inb_p(e8390_base + EN0_ISR)) != 0
           && ++boguscount < 5) {
        ...
        if (interrupts & ENISR_OVER) {
            ei_rx_overrun(dev);
        } else if (interrupts & (ENISR_RX+ENISR_RX_ERR)) {
            /* Got a good (?) packet. */
            ei_receive(dev);
        }
        ...
    }
    ...
    dev->interrupt = 0;
    return;
}
```
我们忽略了硬件相关的操作, 直接看软件是怎么处理的, 在网卡接收到数据时会触发调用 `ei_interrupt()` 中断处理函数, 而 `ei_interrupt()` 会调用 `ei_receive()` 函数读取数据, `ei_receive()` 函数代码如下:
```cpp
void
netif_rx(struct sk_buff *skb)
{
  /* Set any necessary flags. */
  skb->sk = NULL;
  skb->free = 1;

  /* and add it to the "backlog" queue. */
  IS_SKB(skb);
  skb_queue_tail(&backlog, skb);

  /* If any packet arrived, mark it for processing. */
  if (backlog != NULL) mark_bh(INET_BH);

  return;
}

static void ei_receive(struct device *dev)
{
    ...
        if ((rx_frame.status & 0x0F) == ENRSR_RXOK) {
            int sksize = sizeof(struct sk_buff) + pkt_len;
            struct sk_buff *skb;

            skb = alloc_skb(sksize, GFP_ATOMIC);
            if (skb == NULL) {
                if (ei_debug)
                    printk("%s: Couldn't allocate a sk_buff of size %d.\n",
                           dev->name, sksize);
                ei_local->stat.rx_dropped++;
                break;
            } else {
                skb->mem_len = sksize;
                skb->mem_addr = skb;
                skb->len = pkt_len;
                skb->dev = dev;

                ei_block_input(dev, pkt_len, (char *) skb->data,
                               current_offset + sizeof(rx_frame));
                netif_rx(skb);
                ei_local->stat.rx_packets++;
            }
        }
    ...
    return;
}
```
`ei_receive()` 函数首先调用 `alloc_skb()` 函数创建一个 `sk_buff` 对象, 这个对象主要是用来保存接收到的数据和要发送的数据. 然后调用 `ei_block_input()` 函数从网卡中读取包数据, 并通过调用 `netif_rx()` 函数把这个 `sk_buff` 对象添加到 `blacklog` 链表中. 最后调用 `mark_bh(INET_BH)` 标记中断下半部分需要触发 `inet_bh()` 函数.

我们在来看看中断下半部分 `inet_bh()` 函数做了什么工作:
```cpp
void
inet_bh(void *tmp)
{
    ...
    while((skb=skb_dequeue(&backlog))!=NULL) // 从sk_buff队列中获取一个sk_buff
    {
        ...
        skb->h.raw = skb->data + skb->dev->hard_header_len;
        skb->len -= skb->dev->hard_header_len;

        // 每个缓存对象都来源于一个网卡设备, 而不同的网卡设备有不同的物理帧头部
        // 这里使用网卡设备来解包缓存对象, 获取到的type指定了下一层使用的协议(如IP, ARP等)
        type = skb->dev->type_trans(skb, skb->dev); 

        for (ptype = ptype_base; ptype != NULL; ptype = ptype->next) {
            if (ptype->type == type || ptype->type == NET16(ETH_P_ALL)) {
                struct sk_buff *skb2;

                if (ptype->type==NET16(ETH_P_ALL))
                    nitcount--;
                if (ptype->copy || nitcount) {
                    skb2 = alloc_skb(skb->mem_len, GFP_ATOMIC);
                    if (skb2 == NULL)
                        continue;
                    memcpy(skb2, (const void *) skb, skb->mem_len);
                    skb2->mem_addr = skb2;
                    skb2->h.raw = (unsigned char *)(
                        (unsigned long) skb2 +
                        (unsigned long) skb->h.raw -
                        (unsigned long) skb
                    );
                    skb2->free = 1;
                } else {
                    skb2 = skb;
                }

                flag = 1;

                ptype->func(skb2, skb->dev, ptype);
            }
        }
    }
    ...
}
```
`inet_bh()` 函数首先会从 `blacklog` 队列中拿到一个 `sk_buff` 缓存对象, 由于缓存对象必须来源于某一个特定的网卡设备, 而不同的网卡设备的数据包头部是不一样的, 所以对于不同的网卡设备需要提供不同的解包函数, 所以代码 `skb->dev->type_trans(skb, skb->dev)` 就是根据不同的网卡设备来解包, 获取到下一层所使用的协议(如IP协议或者ARP协议等). 例如对于 `NS8390芯片` 的网卡使用的就是 `eth_type_trans()` 函数, 所以这里我们来看看 `eth_type_trans()` 这个函数的实现:
```cpp
#define ETH_ALEN    6

struct ethhdr {
  unsigned char     h_dest[ETH_ALEN];   /* destination eth addr */
  unsigned char     h_source[ETH_ALEN]; /* source ether addr    */
  unsigned short    h_proto;            /* packet type ID field */
};

unsigned short
eth_type_trans(struct sk_buff *skb, struct device *dev)
{
  struct ethhdr *eth;

  eth = (struct ethhdr *) skb->data;

  if(ntohs(eth->h_proto)<1536)
      return(htons(ETH_P_802_3));
  return(eth->h_proto);
}
```
`struct ethhdr` 结构表示以太网帧头部, 结构如下图:
![eth_frame_header](https://raw.githubusercontent.com/liexusong/tcp-ip-stack/master/images/eth_frame.jpg)

`h_proto` 字段指定下一层使用的协议类型, 所以 `eth_type_trans()` 函数返回的就是下一层使用的协议类型.

获取到下一层使用的协议类型后, 就遍历 `ptype_base` 列表找到对应协议的处理函数. 由于本文只讨论 `IP协议`, 所以找到的处理函数是 `ip_rcv()`, 代码如下:
```cpp
int
ip_rcv(struct sk_buff *skb, struct device *dev, struct packet_type *pt)
{
  struct iphdr *iph = skb->h.iph;
  ...

  skb->ip_hdr = iph;
  skb->h.raw += iph->ihl*4; // 下一个协议的包头开始

  hash = iph->protocol & (MAX_INET_PROTOS -1);

  for (ipprot = (struct inet_protocol *)inet_protos[hash];
       ipprot != NULL;
       ipprot=(struct inet_protocol *)ipprot->next)
  {
    ...
    ipprot->handler(skb2, dev, opts_p ? &opt : 0, iph->daddr,
            (ntohs(iph->tot_len) - (iph->ihl * 4)),
            iph->saddr, 0, ipprot);
  }
  ...
  return(0);
}
```
`ip_rcv()` 函数首先根据 `IP包头` 的 `protocol` 字段来查找下一层协议的处理函数, `IP包头` 结构定义如下:
```cpp
struct iphdr {
  unsigned char   ihl:4,
                  version:4;
  unsigned char   tos;
  unsigned short  tot_len;
  unsigned short  id;
  unsigned short  frag_off;
  unsigned char   ttl;
  unsigned char   protocol;
  unsigned short  check;
  unsigned long   saddr;
  unsigned long   daddr;
  /*The options start here. */
};
```
可以使用下图来直观的表示 `struct iphdr` 结构:
![ip_packet_header](https://raw.githubusercontent.com/liexusong/tcp-ip-stack/master/images/ip_packet.jpg)

由于本文只讨论 `TCP协议`, 所以 `IP包头` 的 `protocol` 字段必须为 `IPPROTO_TCP`, 而其处理函数为 `tcp_rcv()`, 由于 `tcp_rcv()` 函数很长, 所以这里分片来分析:
```cpp

```
