# IO多路复用

I/O 多路复用是一种通过一组系统调用，同时监视多个文件描述符的技术。这允许一个单独的进程或线程来管理多个 I/O 操作，而无需为每个 I/O 操作创建一个单独的线程或进程。常见的 I/O 多路复用机制包括 `select`、`poll`、`epoll`, 这些机制允许程序等待多个文件描述符中的任何一个（或多个）变为可读、可写或出现异常，从而有效地处理多个并发的 I/O 操作。

## poll

poll函数的本质与select相同, 都是监视并等待一组文件描述符的状态发生变化. 调用Poll函数后, 进程会阻塞, 直到有文件描述符的要监测的状态变化或者超时. 

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

其中pollfd为:

```c
struct pollfd{
	int fd;			//文件描述符
	short events;	//等待的事件
	short revents;	//实际发生的事件
}
```

**events：**指定要监测fd的事件（输入、输出、错误），每一个事件有多个取值，如下：

![img](https://img-blog.csdn.net/20180917154157899?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NreXBlbmc1Nw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**revents：**真实发生的事件，内核在调用返回时设置这个域。

**返回值**: 成功时，poll() 返回结构体中 revents 域不为 0 的文件描述符个数；如果在超时前没有任何事件发生，poll()返回 0；

## epoll

epoll相比前两者, 用法更加灵活. epoll机制的核心是epoll instance, 可以看作是两个列表的容器:

* interest list: epoll进行监测的所有文件描述符集合
* ready list: interest list中被检测到可用的文件描述符集合

相关的函数如下:

### epoll_create

```c
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_create1(int flags);
```

这两个函数都会创建一个epoll instance, 并返回一个指向该instance的fd.

参数size必须大于0, 没有什么用. flags如果为0, 那么create1和create函数一样.

### epoll_wait

```c
       #include <sys/epoll.h>

       int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
```

阻塞监测epfd指向的epoll instance, timeout指定最长等待时间, 如果为-1则表示无限期阻塞等待. 

参数:

* events: 表示ready list, `epoll_event`的定义如下:

  ```c
  typedef union epoll_data {
     void    *ptr;
     int      fd;
     uint32_t u32;
     uint64_t u64;
  } epoll_data_t;
  
  struct epoll_event {
     uint32_t     events;    /* Epoll events */
     epoll_data_t data;      /* User data variable */
  };
  ```

  `events`是一个位掩码, 表示监测到的event. `data`则是`epoll_ctl`增加fd时指定的data. 

* maxevents: events中最多元素的个数

返回值: 监测到可用IO的描述符个数. 如果为-1则表示出错

注意, 如果先调用了epoll_wait, 然后通过epoll_ctl向epfd中增加了监听的fd, 那么会立刻生效. 当fd可用时, 先前的epoll_wait会返回. 

另外, 如果epoll_wait等待的epfd中监听描述符个数为0, 那么还是会阻塞, 直到后面epoll_ctl加进来的描述符可用.

