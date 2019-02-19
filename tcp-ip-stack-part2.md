# Linux内核中的 TCP/IP
前面一章介绍了Socket族相关的接口, 那么接下来我们来看看Linux内核是怎么实现 `TCP/IP协议` 的吧.

## 数据传输
我们先不考虑内核是怎么实现 `TCP/IP协议` 的, 而是先来了解下当网卡接收到数据时, 内核有什么相应的处理. 

当网卡接收到数据时, 会向CPU发送 `中断信号`. 当CPU接收到 `中断信号` 后会调用相应的 `中断处理程序` 来处理中断. 因为不同的网卡处理数据的方式不一样, 所以不同的网卡的 `中断处理程序` 也不一样. 譬如 `NS8390芯片` 网卡的中断处理程序是 `ei_interrupt()`, 下面我们也主要使用这个中断处理程序作为例子, 起代码如下:
```cpp
void ei_interrupt(int reg_ptr)
{
    int irq = -(((struct pt_regs *)reg_ptr)->orig_eax+2);
    struct device *dev = (struct device *)(irq2dev_map[irq]);
    int e8390_base;
    int interrupts, boguscount = 0;
    struct ei_device *ei_local;

    if (dev == NULL) {
        printk ("net_interrupt(): irq %d for unknown device.\n", irq);
        return;
    }
    e8390_base = dev->base_addr;
    ei_local = (struct ei_device *) dev->priv;
    if (dev->interrupt || ei_local->irqlock) {
        /* The "irqlock" check is only for testing. */
        sti();
        printk(ei_local->irqlock
               ? "%s: Interrupted while interrupts are masked! isr=%#2x imr=%#2x.\n"
               : "%s: Reentering the interrupt handler! isr=%#2x imr=%#2x.\n",
               dev->name, inb_p(e8390_base + EN0_ISR),
               inb_p(e8390_base + EN0_IMR));
        return;
    }

    dev->interrupt = 1;
    sti(); /* Allow other interrupts. */

    /* Change to page 0 and read the intr status reg. */
    outb_p(E8390_NODMA+E8390_PAGE0, e8390_base + E8390_CMD);
    if (ei_debug > 3)
        printk("%s: interrupt(isr=%#2.2x).\n", dev->name,
               inb_p(e8390_base + EN0_ISR));

    /* !!Assumption!! -- we stay in page 0.     Don't break this. */
    while ((interrupts = inb_p(e8390_base + EN0_ISR)) != 0
           && ++boguscount < 5) {
        if (interrupts & ENISR_RDC) {
            /* Ack meaningless DMA complete. */
            outb_p(ENISR_RDC, e8390_base + EN0_ISR);
        }
        if (interrupts & ENISR_OVER) {
            ei_rx_overrun(dev);
        } else if (interrupts & (ENISR_RX+ENISR_RX_ERR)) {
            /* Got a good (?) packet. */
            ei_receive(dev);
        }
        /* Push the next to-transmit packet through. */
        if (interrupts & ENISR_TX) {
            ei_tx_intr(dev);
        } else if (interrupts & ENISR_COUNTERS) {
            struct ei_device *ei_local = (struct ei_device *) dev->priv;
            ei_local->stat.rx_frame_errors += inb_p(e8390_base + EN0_COUNTER0);
            ei_local->stat.rx_crc_errors   += inb_p(e8390_base + EN0_COUNTER1);
            ei_local->stat.rx_missed_errors+= inb_p(e8390_base + EN0_COUNTER2);
            outb_p(ENISR_COUNTERS, e8390_base + EN0_ISR); /* Ack intr. */
        }

        /* Ignore the transmit errs and reset intr for now. */
        if (interrupts & ENISR_TX_ERR) {
            outb_p(ENISR_TX_ERR, e8390_base + EN0_ISR); /* Ack intr. */
        }
        outb_p(E8390_NODMA+E8390_PAGE0+E8390_START, e8390_base + E8390_CMD);
    }

    if (interrupts && ei_debug) {
        printk("%s: unknown interrupt %#2x\n", dev->name, interrupts);
        outb_p(E8390_NODMA+E8390_PAGE0+E8390_START, e8390_base + E8390_CMD);
        outb_p(0xff, e8390_base + EN0_ISR); /* Ack. all intrs. */
    }
    dev->interrupt = 0;
    return;
}
```
