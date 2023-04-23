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

### 网络子系统初始化

在网络子系统初始化过程中，会为每个`CPU`初始化`softnet_data`，也会为`RX_SOFTIRQ`和`TX_SOFTIRQ`注册处理函数，流程图如下

![sub_initial](zixitongchushihua.png)

内核通过调用`subsys_initcall`来初始化各个子系统，在源代码目录里可以用`grep`命令搜索出许多对这个函数的调用，这里要说的是网络子系统的初始化，会执行`net_dev_init`函数

```C
// file: net/core/dev.c

static int __init net_dev_init(void) 
{
    for_each_possible_cpu(i) {
        struct softnet_data *sd = &per_cpu(softnet_data, i);
        memset(sd, 0, sizeof(*sd));
        skb_queue_head_init(&sd->input_pkt_queue);
        skb_queue_head_init(&sd->process_queue);
        sd->completion_queue = NULL;
        INIT_LIST_HEAD(&sd->poll_list);
        ...
    }
    open_softirq(NET_TX_SOFTIRQ, net_tx_action);
    open_softirq(NET_RX_SOFTIRQ, net_rx_action);

}

subsys_initcall(net_dev_init);
```

在这个函数中，会为每个`CPU`都申请一个`softnet_data`数据结构，这个数据结构里的`poll_list`用于等待驱动程序将其`poll`函数注册进来
稍后网卡驱动程序初始化的时候可以看到这一过程
另外，`open_softirq`会为每一种软中断都注册一个处理函数，`NET_TX_SOFTIRQ`的处理函数为`net_tx_action`，`NET_RX_SOFTIRQ`的处理函数为`net_rx_action`
继续跟踪`open_softirq`后会发现这个注册的方式记录在`softirq_vec`变量中的。后面`ksoftirqd`线程收到软中断的时候，也会使用这个变量来找到每个软中断对应的处理函数

```C
// file: kernel/softirq.c
void open_softirq(int nr, void(*action)(struct softirq_action *)) 
{
    softirq_vec[nr].action = action;
}
```