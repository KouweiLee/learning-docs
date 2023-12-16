# gdb学习

常见指令：

`n`： next, 向下执行源代码的一行指令；不会跳入子函数中

`s`：step，进入函数，转到任何子函数要执行的下一行指令。可以加number 参数，表示执行number行命令

`bt`：backtrace，展示函数调用栈

`p 变量名`：打印指定变量的值；如果要在当前环境下动态插入一个表达式，也可以通过`p 表达式`的方式，执行一个表达式

`l`：list，展示接下来的10行源代码

`break [file:][function|line]`：打断点

`run [arglist]`：启动程序，如果需要还可以加命令行参数

`c`：继续运行程序，直到断点或停止

`x/10i $pc`：  从当前pc值开会，在内存中反汇编10条指令。数据为0时表现为unimp。

## 启动gdb

`gdb 可执行文件`：开始调试该可执行文件。

`gdb -p 进程ID`：调试进程

`!command-string`：在gdb中执行shell命令command-string

`pipe [command]|shell_command`：在gdb中执行指令command（若省略则为上一条指令），通过管道发给一条shell指令

* 设置gdb的输出

如果想要将gdb各个指令的输出保存在文件中，使用命令：

```
set logging enabled [on|off] 
set logging file filename // 默认的logfile是gdb.txt，还可以设置其名称
```

## GDB命令

回车：重复上一个命令

TAB：补全该命令

`指令缩写 help`：查看该指令的全称

## 多线程调试

查看所有线程：info threads

切换线程：thread 线程id