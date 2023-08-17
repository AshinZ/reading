### 中断和中断处理



#### 什么是中断？

来自硬件的特殊信号，告知内核有一个操作来了。每个中断对应一个中断值，来帮助判断这是什么中断。这些值可以被称为中断请求值`IRQ`。例如经典`PC`机`IRQ0`是时钟中断。但是在`PCI`总线上的设备，`IRQ`是动态分配的。



#### 中断如何处理？

内核通过中断处理程序，或者中断处理例程来处理中断。每个中断对应一个中断处理程序，每个设备的中断处理程序是设备驱动的一部分。

对于内核而言，中断处理程序就是一段普通的代码，当中断来了以后执行它去处理中断。其特点在于中断处理程序运行在中断上下文中，是属于不可打断的。

简单来说，中断处理程序还负责告诉硬件中断已被接收。还需要做其他的事情，取决于设备类型。



#### 更快的进行中断处理？

中断处理的速度快和处理的内容多是不可兼得的。因而人们尝试将中断分成两部分，第一部分追求速度快，需要在明确的时间内完成，称为上半部(`top half`)，收到中断后就立刻执行，例如对中断做应答等；其他的事情则被推迟到下半部(`bottom half`)去做。

例如对于网卡来说，当网卡发出一个中断的时候，请求的数据已经到了网卡上，因此内核需要快速的做响应，把数据移走从而避免延迟的出现；对于数据包的处理则可以放到后续来做，因为这些部分并不追求速度。



#### 如何注册中断处理程序？

驱动可以通过`request_irq()`注册中断处理程序：

```c
int request_irq(unsigned int irq,
                irq_handler_t handler,
                unsigned long flags,
                const char *name,
                void *dev)
```

第一个参数的含义就是`irq`的编号，指明这个中断处理程序对应的`irq`。

第二个参数是一个指针，指向了中断的实际中断处理程序。因此当操作系统接收到值为`irq`的中断时，就会调用：

```c
typedef irqreturn_t (*irq_hanlder_t)(int, void*)
```

第三个参数是一个标志位，可以设置成多种属性，例如：

- IRQF_DISABLED：设置以后该中断处理程序进行时会禁止其他中断；
- IRQF_SAMPLE_RANDOM：设置后标明该中断的中断对内核熵池有效果。熵池负责从随机事件中导出真正随机数。因此只有这个中断是随机触发时，才设置该信号；
- IRQF_TIMER：专为系统定时器设计；
- IRQF_SHARED：表明多个中断处理程序之间共享`IRQ`。在同一个给定线上的处理程序必须要设置这个，不然会阻塞其他中断。

第四个参数`name`是表名这个中断的设备的名字。

第五个参数`dev`用于共享`IRQ`，当中断处理程序需要释放的时候，`dev`可以提供标志信息，从而让内核知道应该删除哪个中断处理程序。如果中断不共享`IRQ`，则无须设置。

`request_irq`成功为返回`0`，错误情况下不会注册中断处理程序。

`request_irq`会睡眠，因此不能在中断上下文或者其他不允许阻塞的代码中调用。

例如，我们可以通过如下的代码设置一个中断处理函数：

```c
request_irq();
if (request_irq(irqn, my_interrupt, IRQF_SHARED, "my_device", my_dev)) {
    printk(KERN_ERR "cannnot register");
    return -EIO;
}
```



#### 释放中断处理程序

卸载驱动时，我们需要注销对应的中断处理程序并释放`IRQ`：

```c
void free_irq(unsigned int irq, void *dev)
```

如果`IRQ`是共享的，那我们需要只删除`dev`指向的中断处理程序，而如果不共享则直接删除。当中断线没有中断处理程序时就会被禁用。





#### 编写中断处理程序

一个中断处理程序声明：

```c
static irqreturn_c intr_handler(int irq, void *dev)
```

第一个函数是`irq`，不提。

第二个参数是一个通用指针，也就是前文提到在共享`irq`时的`dev`，也即`cookie`，用来区分同一个中断处理程序的多个设备。

中断处理程序的返回值是`irqreturn_t`。有两个特殊的返回值：

- `IRQ_NONE`：表示中断和对应设备不一致
- `IRQ_HANDLED`：表示被正确调用了返回的应答



共享的处理程序和非共享的处理程序有以下差异：

- `request_irq`必须设置`IRQF_SHARED`；
- `dev`参数必须唯一，否则无法分别；
- 中断程序能够区分其设备是否真的发生了中断；

内核接收到了中断，会依次调用该`IRQ`对应的处理程序。

实际的注册中断程序代码：

```c
if (request(rtc_irq, rtc_interrupt, IRQF_SHARED, "rtc", (void *)&rtc_port)){
	printk(KERN_ERR "cannnot register");
    return -EIO;
}
```

处理程序代码：

```c
static irqreturn_t rtc_interrupt(int irq, void *dev){
    spin_lock (&rtc_lock);
    
    rtc_irq_data += 0x100;
    rtc_irq_data &= ~ 0xff;
    rtc_irq_data |= (CMOS_READ(RTC_INTR_FLAGS) & 0xF0);
    
    if (rtc_status & RTC_TIME_ON)
        mod_timer(&trc_irq_timer, jiffies + HZ/rtc_freq + 2 * HZ/100);
    
    spin_unlock(&rtc_lock);
    
    spin_lock(&rtc_task_lock);
    if (rtc_callback())
        	rtc_callback->func(rtc_callback->private_data);
    spin_unlock(&rtc_task_lock);
    wake_up_interruptible(&rtc_wait);
    
    kill_fasync(&rtc_async_queue, SIGIO, POLL_IN);
    
    return IRQ_HANDLED;
}
```

