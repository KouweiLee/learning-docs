# WorkQueue学习

## 

## 前言---中断处理的Bottom Half

中断处理函数要求必须处理迅速, 如果确实需要较长时间的处理, 可以将其分为两个部分, Top Half和Bottom Half. 其中Top Half是中断处理函数, 很快会执行完. Bottom Half则是在之后的某个时刻会执行. WorkQueue是一种Bottom Half机制.

## Workqueue

工作队列机制在Linux内核中广泛用于延迟执行或将耗时操作从中断上下文或关键代码路径移到专门的处理线程中。这样做可以减少对主处理流程的干扰，提高系统的响应性和性能。

### 创建工作

一般不需要创建一个新的工作队列, 如果需要的话可以参考 [这里](https://embetronicx.com/tutorials/linux/device-drivers/work-queue-in-linux-own-workqueue/). 

创建工作可以通过两个内核宏来实现. 

* INIT_WORK

INIT_WORK宏, 用于初始化一个`work_struct`结构的实例，并指定一个处理函数。

* INIT_DELAYED_WORK

用于初始化一个`delayed_work`结构的工作, 它内部包含`work_struct`. 

### 将工作加入工作队列

创建工作后, 需要将其加入到工作队列中才能调度运行. 加入到工作队列的方式有:

* schedule_work

```c
int schedule_work( struct work_struct *work );
```

将work加入到内核全局工作队列中, 之后work对应的处理函数在某个时间就会被执行了. 

* scheduled_delayed_work

```c
int scheduled_delayed_work( struct delayed_work *dwork, unsigned long delay );
```

等待delay时间后, 将dwork加入到内核全局工作队列中.

## 参考教程

1. https://embetronicx.com/tutorials/linux/device-drivers/workqueue-in-linux-kernel/