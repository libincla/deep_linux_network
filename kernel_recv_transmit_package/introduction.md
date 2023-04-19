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