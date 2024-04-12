# Linux网络编程

## 基本概念

### 字节序

由于不同主机通信时字节序可能不同, 因此发送数据前需要将数据先转换为网络字节序(大端存储). x86和arm都默认是小端. 大端意味着0x1234在内存中的存储次序从低地址到高地址也是0x1234. 小端则是0x3412

```c
#include <arpa/inet.h>

//将主机字节序转化为网络字节序
uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);

//将网络字节序，转化为主机字节序
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);
```

这会将不同大小的数据进行字节序的转换. 需要注意的是, 大小端只是针对word等这样的**大于一个字节的数据单位**来说的, 两个word之间的顺序不会因为大小端而影响. 

### IP地址的转化

IPv4的IP地址长度为4字节, 用点分十进制表示. 以下两个函数可以将字符串形式的IP地址和网络字节序的IP地址进行相互转换. 

```c
#include <arpa/inet.h>　
// convert IPv4 and IPv6 addresses from text to binary form
int inet_pton(int af, const char *src, void *dst);
// convert IPv4 and IPv6 addresses from binary to text form
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size); 
```

### 计网知识

一个计算机上可以有多个网卡, 每个网卡都有一个IP地址. 

## socket编程

网络中的进程是通过socket来通信的, socket类似与Unix系统中的文件, 通过open, read/write, close等类似的方法进行读写即可进行网络通信. 其基本的几个接口函数如下:

#### socket()函数

```c
int socket(int domain, int type, int protocol);
```

类似于打开文件的操作, 返回一个socket描述符. 

* 参数:

domain：地址族

1. AF_UNIX   本地通信
2. AF_INET   ipv4网络协议
3. AF_INET6  ipv6网络协议
4. AF_PACKET 底层协议通信

type：套接字的类型

1. SOCK_STREAM 流式套接字  --TCP
2. SOCK_DGRAM  数据报套接字  --UDP
3. SOCK_RAW    原始套接字

protocol：指定协议, 如果为0表示和type一致

#### bind()函数

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

bind()函数把一个地址族中的特定地址赋给socket。例如对应AF_INET、AF_INET6就是把一个ipv4或ipv6地址和端口号组合赋给socket。通常服务器在启动时会使用bind函数绑定一个地址和端口用于提供服务, 而客户端则不需要, 由系统在connect函数中自动分配. 

* 参数:

  sockfd：要绑定的socket描述符

  addr: 指向要绑定给sockfd的协议地址, 地址结构根据地址创建socket时的地址协议族的不同而不同:

  * ipv4的地址结构:

    ```c
    struct sockaddr_in {
        sa_family_t    sin_family; /* address family: AF_INET */
        in_port_t      sin_port;   /* port in network byte order */
        struct in_addr sin_addr;   /* internet address */
    };
    
    /* Internet address. */
    struct in_addr {
        uint32_t       s_addr;     /* address in network byte order */
    };
    ```

  * ipv6的地址结构:

    ```c
    struct sockaddr_in6 { 
        sa_family_t     sin6_family;   /* AF_INET6 */ 
        in_port_t       sin6_port;     /* port number */ 
        uint32_t        sin6_flowinfo; /* IPv6 flow information */ 
        struct in6_addr sin6_addr;     /* IPv6 address */ 
        uint32_t        sin6_scope_id; /* Scope ID (new in 2.4) */ 
    };
    
    struct in6_addr { 
        unsigned char   s6_addr[16];   /* IPv6 address */ 
    };
    ```

  * Unix域:

    ```c
    #define UNIX_PATH_MAX    108
    
    struct sockaddr_un { 
        sa_family_t sun_family;               /* AF_UNIX */ 
        char        sun_path[UNIX_PATH_MAX];  /* pathname */ 
    };
    ```

  addrlen：地址结构的长度, 例如对于Ipv4的地址则为`sizeof(struct sockaddr_in)`

#### listen()和connect()函数

如果作为一个服务器，在调用socket()、bind()之后就会调用listen()来监听这个socket，如果客户端这时调用connect()发出连接请求，服务器端就会接收到这个请求。

```c
int listen(int sockfd, int backlog);
```

监听sockfd对应的socket, 等待客户的连接. backlog表示可以排队的最大连接个数. 

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

sockfd为客户端的socket描述符, addr为服务器的addr地址, addrlen为该地址的长度.

#### accept()函数

服务器调用listen后, 在收到客户端的请求后, 就会通过accept函数取接收请求, 这样就建立了连接. 

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

sockfd为服务器端的socket描述符, addr用于返回客户端的协议地址, addrlen返回客户端协议地址的长度. 如果accpet成功，那么该函数返回值是由内核自动生成的一个全新的描述符，代表与返回客户的TCP连接。

#### read(), write()等函数

当完成accept建立连接后, 就可以通过网络IO进行读写操作了. 网络I/O操作有下面几组：

- read()/write() 
- recv()/send() 
- readv()/writev() 
- recvmsg()/sendmsg() 
- recvfrom()/sendto()

它们的声明如下:

```c
   #include <unistd.h>

   ssize_t read(int fd, void *buf, size_t count);
   ssize_t write(int fd, const void *buf, size_t count);

   #include <sys/types.h>
   #include <sys/socket.h>

   ssize_t send(int sockfd, const void *buf, size_t len, int flags);
   ssize_t recv(int sockfd, void *buf, size_t len, int flags);

   ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
                  const struct sockaddr *dest_addr, socklen_t addrlen);
   ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                    struct sockaddr *src_addr, socklen_t *addrlen);

   ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
   ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```

* read/write

函数返回值: 若大于0, 则为读到/写入的字节数；read返回值若为0, 则表示读到文件的结束, 也就是对方关闭了连接；小于0则出现了错误.

推荐使用recvmsg()/sendmsg()函数, 因为最通用, 也可以把其他函数替换掉.

#### close()函数

```c
#include <unistd.h>
int close(int fd);
```

close一个TCP socket的缺省行为时把该socket标记为以关闭，然后立即返回到调用进程。该描述字不能再由调用进程使用，也就是说不能再作为read或write的第一个参数。

注意：close操作只是使相应socket描述字的引用计数-1，只有当引用计数为0的时候，才会触发TCP客户端向服务器发送终止连接请求。

### TCP中的socket

* 三次握手建立连接

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240110155225027.png" alt="image-20240110155225027" style="zoom: 67%;" />

* 四次握手释放连接

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240110155417220.png" alt="image-20240110155417220" style="zoom:67%;" />



## 虚拟网络设备

Bridge, tap/tun都是虚拟网络设备, 都可以配置IP, MAC. 

## 参考资料

1. [Linux网络编程基本概念](https://juejin.cn/post/7011505595387740196)

2. [Linux socket编程](https://www.cnblogs.com/skynet/archive/2010/12/12/1903949.html)

3. [Linux 网络编程系列](https://juejin.cn/column/7011506439080558600)

   