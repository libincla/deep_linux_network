## 问题

在现在互联网世界中，几乎天天和网络请求打交道
如果以`C`语言来说，一行`read`函数调用代码就能接受对端的数据

```C
int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    connect(sock, ...);
    read(sock, buffer, sizeof(buffer)-1);
    .....
}
```

只要客户端有对应的数据发送过来，服务器端执行`read`后就能收到

## 思考，`linux`下数据是如何从网卡一步步到达你的进程的

### `RingBuffer`到底是什么，`RingBuffer`为什么会丢包

- `RingBuffer`到底存在哪一块，如何被利用，真的是一个环形的队列
- `RingBuffer`的内存是预先分配好的还是会随着网络包收发而动态分配的？
- `RingBuffer`会丢包，如果丢包应该如何解决？

### 网络的软、硬中断都是什么

- 这两个中断的区别是什么？二者如何协作
- 针对**网卡中断绑定**，究竟是 硬中断号和`CPU`之间的绑定关系，最终效果却是软中断一起调整，软中断开销也被绑定到不同`CPU`

### `linux`中的`ksoftirqd`内核线程都是干什么的

在服务器上执行`ps aux | grep ksoftirqd`，查看一下内核线程

```shell
# 服务器是4核16g

$ sudo ps aux | grep -i ksoftirq
root         6  0.0  0.0      0     0 ?        S     2022  13:35 [ksoftirqd/0]
root        14  0.0  0.0      0     0 ?        S     2022   1:34 [ksoftirqd/1]
root        19  0.0  0.0      0     0 ?        S     2022   1:38 [ksoftirqd/2]
root        24  0.0  0.0      0     0 ?        S     2022   1:35 [ksoftirqd/3]

```

### 为什么网卡开启多队列会提升网络性能

### `tcpdump`到底是如何工作的

### `iptables/netfilter`在哪一层实现的

### `tcpdump`是否抓到被`iptables`封禁的包?

### 网络接收过程中的`CPU`开销如何查看

### `DPDK`究竟是什么

