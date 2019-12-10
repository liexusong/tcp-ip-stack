## 什么是TCP/IP协议
`TCP/IP协议` 不是一个协议而是一个协议族的统称，里面包括了IP协议，IMCP协议，TCP协议等等。同时是Internet最基本的协议、Internet国际互联网络的基础，由网络层的IP协议和传输层的TCP协议组成。 `TCP/IP协议` 定义了电子设备如何连入因特网，以及数据如何在它们之间传输的标准。协议采用了4层的层级结构，每一层都通过调用它的下一层所提供的协议来完成自己的需求，如下图：

![tcp-ip-stack-layer](https://raw.githubusercontent.com/liexusong/tcp-ip-stack/master/images/tcp-ip-stack-layer.jpg)

上图左边对应的是OSI模型分层，OSI把网络按照层次分为七层，由下到上分别为： `物理层`、`数据链路层`、`网络层`、`传输层`、`会话层`、`表示层`、`应用层`。而TCP/IP模型分层只分为四层，分别为：`网络接口层`、`网络层`、`传输层`、`应用层`，它们之间的对应关系如上图所示。

本文分析TCP/IP协议栈时，从底层开始向上分析，这样做的好处是能够把一个网络数据包从网络接口层一直向上传输到应用层的过程直观地描述出来。

## 网络接口层
`网络接口层` 在TCP/IP模型是最底部的一层，这一层主要包括网卡驱动程序和对数据包接收和发送，把接收到的数据提交并通知上一层协议进行处理。

### 网络接口中断
本文主要以 `NS8390网卡` 作为分析对象，对于网卡驱动，本文不作详细分析（因为网卡驱动是比较机械化的一些操作，而且对于理解TCP/IP协议并无多大帮助），我们只要知道当网卡接收到数据时会发生什么事情即可。

当网卡接收到数据时会触发网卡中断，对于 `NS8390网卡` 而言，中断的处理函数为 `ei_interrupt()`，我们来看看 `ei_interrupt()` 函数的实现：
```c
void ei_interrupt(int reg_ptr)
{
    ...
    while ((interrupts = inb_p(e8390_base + EN0_ISR)) != 0 && ++boguscount < 5) {
        ...
        if (interrupts & ENISR_OVER) {
            ...
        } else if (interrupts & (ENISR_RX+ENISR_RX_ERR)) {
            /* 接收到数据包 */
            ei_receive(dev);
        }
        /* 数据已经发送成功 */
        if (interrupts & ENISR_TX) {
            ei_tx_intr(dev);
        } else if (interrupts & ENISR_COUNTERS) {
            ...
        }
        ...
    }
    ...
    return;
}
```
当网卡接收到数据包或者成功发送数据包都会触发 `ei_interrupt()` 中断处理函数，而 `ei_interrupt()` 中断处理函数会根据触发中断的类型进行处理，比如接收到数据包便会调用 `ei_receive()` 处理，而成功发生数据包便会调用 `ei_tx_intr()` 函数进行处理。

下面我们先来分析一下怎么处理接收到的数据包。

### 处理接收到的数据包

上面介绍过，当网卡接收到数据包时，会调用 `ei_receive()` 函数进行处理，`ei_receive()` 函数的实现如下：
```c
static void ei_receive(struct device *dev)
{
    ...
    while (++rx_pkt_count < 10) {
        ...
        if (pkt_len < 60  ||  pkt_len > 1518) {
            ...
        } else if ((rx_frame.status & 0x0F) == ENRSR_RXOK) {
            int sksize = sizeof(struct sk_buff) + pkt_len;
            struct sk_buff *skb;
            
            skb = alloc_skb(sksize, GFP_ATOMIC);
            if (skb == NULL) {
                ...
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
        } else {
            ...
        }
        ...
    }
    ...
    return;
}
```
`ei_receive()` 函数主要做了三件事：1) 调用 `alloc_skb()` 申请一个 `sk_buff对象` 用于保存接收到的数据包。2) 调用 `ei_block_input()` 函数从网卡中读取数据包的数据。3) 调用 `netif_rx()` 函数把数据包存储到 `backlog` 队列中，并且调用 `mark_bh(INET_BH)` 来触发网卡中断下半部处理。

### 网卡中断下半部处理
接下来我们分析一下网卡中断下半部相关的处理，当网卡接收到数据包后会触发中断处理，然后中断处理会调用 `mark_bh(INET_BH)` 来触发网卡中断下半部处理。网卡中断下半部处理主要通过 `inet_bh()` 函数实现，代码如下：
```c
void
inet_bh(void *tmp)
{
  ...
  while((skb=skb_dequeue(&backlog))!=NULL) // 从backlog队列中获取一个sk_buff
  {
    ...
    type = skb->dev->type_trans(skb, skb->dev);

    for (ptype = ptype_base; ptype != NULL; ptype = ptype->next) {
        if (ptype->type == type || ptype->type == NET16(ETH_P_ALL)) {
            struct sk_buff *skb2;

            if (ptype->type==NET16(ETH_P_ALL))
                nitcount--;
            if (ptype->copy || nitcount) {    /* copy if we need to */
                skb2 = alloc_skb(skb->mem_len, GFP_ATOMIC);
                if (skb2 == NULL)
                    continue;
                memcpy(skb2, (const void *) skb, skb->mem_len);
                skb2->mem_addr = skb2;
                skb2->h.raw = (unsigned char *)((unsigned long)skb2 + (unsigned long)skb->h.raw - (unsigned long)skb);
                skb2->free = 1;
            } else {
                skb2 = skb;
            }
            ...
            ptype->func(skb2, skb->dev, ptype);
        }
    }
    ...
  }
  ...
}
```
`inet_bh()` 函数首先调用 `skb_dequeue()` 函数从 `backlog` 队列中获取一个数据包，然后调用 `skb->dev->type_trans(skb, skb->dev)` 函数从数据包中获取到上一层协议的类型，对于 `NS8390网卡` 而言调用的是 `eth_type_trans()` 函数，代码如下：
```c
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
`struct ethhdr` 是以太帧头部，定义如下：
```c
struct ethhdr {
  unsigned char     h_dest[ETH_ALEN];   /* destination eth addr */
  unsigned char     h_source[ETH_ALEN]; /* source ether addr    */
  unsigned short    h_proto;            /* packet type ID field */
};
```
网络层的类型保存在以太帧头部的 `h_proto` 字段中，`eth_type_trans()` 函数就是读取以太帧的 `h_proto` 字段内容并且返回。

接着分析 `inet_bh()` 函数，获取到网络层协议类型后，遍历所有系统支持的网络层协议（`ptype_base`），然后找到跟以太帧头部的 `h_proto` 字段一致的协议对象（`struct packet_type`），再通过调用协议对象的 `func` 回调来处理数据包。`ptype_base` 变量定义如下：
```c
...
static struct packet_type arp_packet_type = {
  NET16(ETH_P_ARP),
  0,        /* copy */
  arp_rcv,
  NULL,
#ifdef CONFIG_IPX
#ifndef CONFIG_AX25
  &ipx_packet_type
#else
  &ax25_packet_type
#endif
#else
#ifdef CONFIG_AX25
  &ax25_packet_type
#else
  NULL        /* next */
#endif
#endif
};


static struct packet_type ip_packet_type = {
  NET16(ETH_P_IP),
  0,        /* copy */
  ip_rcv,
  NULL,
  &arp_packet_type
};

struct packet_type *ptype_base = &ip_packet_type;
```
