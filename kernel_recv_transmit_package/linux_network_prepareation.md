## linux启动准备工作

`linux`驱动、内核协议栈等模块在能够接收网卡数据包之前，要做很多准备工作才行，包括
1. 提前创建好`ksoftirqd`内核线程
2. 注册好各个协议对应的处理函数
3. 网卡设备子系统要提前初始化好
4. 网卡启动好
以上准备工作做好后，才开始真正的接收数据包


### 创建`ksoftirqd`内核线程

`linux`的软中断都是在专门的内核线程(`ksoftirqd`)中进行的，线程数量等于你服务器的核心数，初始化后，才能在后面更准确了解收包过程

`ksoftirqd`创建： 系统初始化的时候，会执行到`spawn_ksoftirqd`（位于`kernel/softirq.c`)来创建出`softirqd`线程

1. 启动 --> kernel/smpboot.c --> 2. spawn内核线程ksoftirqd --> kernel/softirq.c --> 3.注册线程函数`ksoftirqd_should_run`, `run_ksoftirqd`

代码如下:

```C

//file: kernel/softirq.c
static struct smp_hotplug_thread softirq_threads = {
    .store = &ksoftirqd,
    .thread_should_run = ksoftirqd_should_run,
    .thread_fn = run_ksoftirqd,
    .thread_comm = "ksoftirqd/%u",
};

static __init int spawn_ksoftirqd(void)
{
    register_cpu_notifier(&cpu_nfb);
    BUG_ON(smpboot_register_percpu_thread(&softirq_threads));
    return 0;
}
early_initcall(spawn_ksoftirqd);
```

`ksoftirqd`被创建出来后，它就会进入自己的线程循环函数`ksoftirqd_should_run`和`run_ksoftirqd`了，接下来判断有没有软中断需要处理

`linux`内核在`interrupt.h`中定义了所有的软中断类型，网络软中断只是其中一个，具体如下

```C

// file: include/linux/interrupt.h

enum 
{
    HI_SOFTIRQ = 0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    BLOCK_IOPOLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ,
    RCU_SOFTIRQ,
    NR_SOFTIRQS
};
```