# PSCI学习

PSCI，全称Power State Coordination Interface，是一个统一协调各个EL级上软件对PE电源管理的接口规范。电源管理主要用于改变CPU核的工作状态，可以用来平衡负载。电源管理的例子有：

Idle Management：当一个CPU核上没有任何线程调度时，该CPU核会被置为clock-gated, retention, or even fully power-gated state

Hotplug：当该CPU核计算负载很低时，会从物理上被关闭变为offline状态，直到计算任务多时才会处于online

## 一些术语

OSPM：Operating System-directed Power Management. 指OS包含的一个组件，该组件进行电源的管理。

power states：CPU核可以处于以下任意一个power state：

Run：该核已通电并可运行

Standby：该核已通电，但处于减小能源消耗的状态。一般通过wfi或wfe指令进入该状态，当wakeup event发生时退出该状态。处于该状态CPU核的所有状态不变，包括上下文，并且可以在唤醒时直接访问。

Retention：从OS的视角来看，Retention和Standby没有区别。

Powerdown：核已断电，需要将以前的状态保存起来，当重新加电时需要reset cpu

## PSCI使用示例

当软件想要通过PSCI管理core power state时，需要调用SMC指令，

### Idle management

当一个核是空闲状态时，OSPF会将其置为low power state，但仍然视该核为可用的。有许多low power states，state X 比state Y更深的意思是，state X下的CPU耗能比state Y更少，但一般要恢复到running state所花的时间更长。

当一个核被置为low power state时，可以通过wakeup event恢复到running state，例如中断。

### CPU hotplug

Hotplug可以动态启动和关闭core。和powerdown不同，hotplug的特点如下：

如果一个核处于unplugged，该核会变为不可用状态offline，如果要将一个核变为online，则软件需要发出指令。并且wakeup event不会对hotplug产生作用。
