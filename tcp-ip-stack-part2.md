# Linux内核中的 TCP/IP
前面一章介绍了Socket族相关的接口, 那么接下来我们来看看Linux内核是怎么实现 `TCP/IP协议` 的吧.

## 数据传输
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
