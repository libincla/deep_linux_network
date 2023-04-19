## linux网络收包总览


![tcpip](/Users/wecash/Desktop/codes/deep_into_network/kernel_recv_transmit_package/tcpip.png)

如同上图，`TCP/IP`网络分层模型中，整个协议栈被分成为了
- 物理层
- 数据链路层
- 网络层
- 传输层
- 应用层

应用层主要是（比如`Nginx`，`FTP`等应用），`linux`内核已经网卡驱动主要是由链路层、网络层以及传输层这三层的功能
内核为更上面的应用层提供`socket`接口来支持用户的进程访问


