# TAILQ学习

TAILQ，是标准C库提供的一种双向链表的实现，头文件为`<sys/queue.h>`。双向链表可以直接删除一个指定的元素，不需要遍历整个链表。

其使用方法如下：

## TAILQ的创建

1. 首先需要设计一个包含TAILQ_ENTRY的结构体：

```c
typedef struct Node {
    /*some attr*/
    ...
    /*link*/
    TAILQ_ENTRY(Node) link;
} Node;
```

TAILQ_ENTRY会定义一个结构体字段，这个字段包含了两个指向该节点前面和后面节点的指针，用于链接双向链表。

2. 需要创建一个链表头，这个头节点并不在链表中，只是指向了这个链表的第一个元素和最后一个元素。

```c
TAILQ_HEAD(head, Node) head;
TAILQ_INIT(&head); // 初始化头节点
```

这个宏的效果如下：

```c
struct head { 
    Node * tqh_first; 
    Node ** tqh_last; 
} head;
```

## TAILQ的使用

可通过man tailq来获取更多内容。

* 反向遍历队列的实现原理

TAILQ_ENTRY的定义为：

```c
#define TAILQ_ENTRY(type)                                                   \
struct {                                                                    \
    struct type *tqe_next;      /* next element */                          \
    struct type **tqe_prev;     /* address of previous next element */      \
}
```

其中prev指向的是前一个节点的next指针，因此next指针偏移量加4就是前一个节点的prev了，很自然地就知道前一个节点的地址了。

## 其他类型的Queue

可以通过man队列名来查看其他类型的链表队列。

![image-20240326153947220](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240326153947220.png)

![image-20240326153955869](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240326153955869.png)

## 参考资料

1. https://www.man7.org/linux/man-pages/man3/tailq.3.html
2. https://www.cnblogs.com/fuzidage/p/14482501.html