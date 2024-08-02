# FPGA学习

## 基础知识回顾

### 寄存器

寄存器，当时钟信号出现上升沿时，输出等于输入；否则，输出不变，保持原状。

触发器的意思就是，没有外加信号的时候，保持原来电路状态，有了外加信号，结合原来电路状态，得到新的状态。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405171056700.png" alt="image-20240517105651453" style="zoom:67%;" />

![image-20240517105831679](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405171058838.png)

### 有限状态机

输出的内容与**此时的输入**是否有关，决定了该状态机是 Moore 机还是 Mealy 机。

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405171924260.png" alt="image-20240517192411080" style="zoom:50%;" />

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405172015451.png" alt="image-20240517201525312" style="zoom:50%;" />

### Verilog语法

Verilog 是一种硬件描述语言。低层次内部结构和实现细节等的考虑，以及将其转化为物理电路的过程，都由有关软件自动完成。其中这一转化过程就叫综合 (synthesis)。

结构化建模：module，实例化

行为级描述的方法一般有两种：1. 利用连续赋值语句 `assign` 描述电路。2. 利用 `initial` 结构、`always` 结构和过程控制语句描述电路。assign必须是wire数据

`reg` 类型数据只是一个变量，用途是方便我们的描述，并不一定对应一个真实电路中的寄存器。

为了写出正确、可综合的程序，**在描述时序逻辑时要使用非阻塞式赋值 `<=`**。

* 阻塞赋值与非阻塞赋值

非阻塞赋值：由`<=`表示，处在一个 `always` 块中的非阻塞赋值是**在块结束时**同时**并发**执行的。

阻塞赋值：阻塞赋值语句的执行是具有明确**顺序**关系的，在 `begin` - `end` 的顺序块中，当前一句阻塞赋值完成后（即 `=` 左边的变化为右边的值后），下一条阻塞赋值语句才会被继续执行。

* 代码规范

[Verilog 代码规范 - 计算机组成教程](http://127.0.0.1:8000/P1/verilog-coding-standard/#vc-014-localparam)

### FPGA

FPGA （现场可编程逻辑门阵列）是一种可编程的集成电路器件，可以将 Verilog HDL 描述的电路在硬件上执行。

从 Verilog 编写的代码到最终可在 FPGA 上运行的电路，需要经历两个基本的步骤：**综合**（Synthesis）与**实现**（Implementation）。

**综合**是指将硬件描述语言（如 Verilog）编写的代码转换成描述逻辑门、触发器、多路选择器等元件之间相互连接的电路网表的过程。

* 约束文件

在vivado中，一般为xdc，与综合后的电路图一同送给实现工具，包含物理约束和时序约束等。物理约束包括引脚分配、引脚电平等，引脚分配就是指定 Verilog 顶层模块的每一位 IO 信号与 FPGA 芯片物理引脚之间的对应关系。引脚电平定义了引脚所适用的电平标准，即逻辑高低电平与物理电压的关系。

## ZCU102板子

* 板子序列号：

![image-20240519150424942](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405191504121.png)

用户手册第20页

* 板子启动方式：

![image-20240518144947996](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405181449146.png)

位于[ug1182-zcu102-eval-bd.pdf](file:///E:/学习/pku/fpga/zcu102/ug1182-zcu102-eval-bd.pdf)第19页。

* 板子手册的一些信息

FPGA包含两个部分，PS和PL。Ps与pl通过axi总线通信，pl可以通过硬件编程实现一些软件算法的加速（图像，通信算法，AI算法，CNN等）,把结果回传给PS。

U1在12页，表示FPGA。

![image-20240519170158492](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405191701010.png)

* User IO

![](https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202405201715168.png)

这里的user io，就是指可以用在fpga上用户可定义的。

* PS Bank和PL Bank

可以这样理解，Bank是外设和处理器核之间的一片引脚的集中区域，其中PS Bank是外设和Arm处理器连接的桥梁，PL Bank是外设和FPGA连接的桥梁。

* 处理器的前端和后端

前端和后端的协同工作确保处理器能够高效地执行指令，前端负责准备和组织指令，包含取值、译码、分支预测等，后端负责执行指令和产生结果，包括调度、执行、内存访问和写回。



## TODO

- [x] module中的，input和output一般都是什么变量

需要指出的是，**模块中的语句除了顺序块之外，都是“并行的”**；输入输出端口若不特别说明类型及位宽，**默认为 1 位 `wire` 型**。

- [x] reg和wire型变量的区别

wire型变量需要有输入才有输出，代表组合逻辑信号，一般使用assign进行驱动

reg则是寄存器类型变量，能存储数据，一般在always中使用。但不一定综合成寄存器。