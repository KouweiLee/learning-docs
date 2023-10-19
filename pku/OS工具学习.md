# OS工具学习

## makefile

教程：[三、Makefile文件的语法_w3cschool](https://www.w3cschool.cn/mexvtg/dsiguozt.html)

### 函数

#### foreach

```makefile
$(foreach var,list,text)
```

一般用于处理文件列表list中的每个文件，对其执行text操作。

* var：临时变量
* list：以空格隔开的文件列表，循环时每次取一个文件名单词赋值给var
* text：对var变量进行操作

#### wildcard

根据通配符获得所有符合条件的文件名

#### patsubst

```makefile
$(patsubst pattern,replacement,text)
```

将text看做匹配pattern的字符串，再替换成匹配replacement的字符串。例如：

```makefile
ELFS := $(patsubst app/%.rs, target/%, app/a.rs)
```

ELFS最后会等于target/a

## tmux

终端执行tmux指令

```
ctrl+b %    垂直分屏
ctrl+b d    退出会话，回到shell的终端环境
ctrl+b o	切换tmux分屏
```

[Linux 终端复用神器 Tmux 使用详解，看完可以回家躺平了～-51CTO.COM](https://www.51cto.com/article/664989.html)

## git

### 1.如何打patch

**如何快速继承上一章练习题的修改**

从这一章开始，在完成本章习题之前，首先要做的就是将上一章框架的修改继承到本章的框架代码。出于各种原因，实际上通过 `git merge` 并不是很方便，这里给出一种打 patch 的方法，希望能够有所帮助。

1. 切换到上一章的分支，通过 `git log` 找到你在此分支上的第一次 commit 的前一个 commit 的 ID ，复制其前 8 位，记作 `base-commit` 。假设分支上最新的一次 commit ID 是 `last-commit` 。
2. 确保你位于项目根目录 `rCore-Tutorial-v3` 下。通过 `git diff <base-commit> <last-commit> > > <patch-path>` 即可在 `patch-path` 路径位置（比如 `~/Desktop/chx.patch` ）生成一个描述你对于上一章分支进行的全部修改的一个补丁文件。打开看一下，它给出了每个被修改的文件中涉及了哪些块的修改，还附加了块前后的若干行代码。如果想更加灵活进行合并的话，可以通过 `git format-patch <base-commit>` 命令在当前目录下生成一组补丁，它会对于 `base-commit` 后面的每一次 commit 均按照顺序生成一个补丁。
3. 切换到本章分支，通过 `git apply --reject <patch-path>` 来将一个补丁打到当前章节上。它的大概原理是对于补丁中的每个被修改文件中的每个修改块，尝试通过块的前后若干行代码来定位它在当前分支上的位置并进行替换。有一些块可能无法匹配，此时会生成与这些块所在的文件同名的 `*.rej` 文件，描述了哪些块替换失败了。在项目根目录 `rCore-Tutorial-v3` 下，可以通过 `find . -name *.rej` 来找到所有相关的 `*.rej` 文件并手动完成替换。
4. 在处理完所有 `*.rej` 之后，将它们删除并 commit 一下。现在就可以开始本章的实验了。

### 2.如何rebase

[如何使用 git rebase 将多个 commit 合并成一个commit - 掘金](https://juejin.cn/post/7050705771218075684)

要把多个commits合并为commit，则可以使用git rebase命令。

```
git rebase -i 要合并到的第一个commit的前一个commit的hash值
git rebase --continue // 如果要冲突了，先解决冲突，然后git add .，之后执行这个命令
git push --force-with-lease origin 
```

## GDB

教程：[Commands | GDB Tutorial](http://www.gdbtutorial.com/tutorial/commands)

GDB官方手册

[GDB调试指南 | 守望的个人博客 (yanbinghu.com)](https://www.yanbinghu.com/2019/04/20/41283.html)

[介绍 | 100个gdb小技巧 (gitbooks.io)](https://wizardforcel.gitbooks.io/100-gdb-tips/content/index.html)

`x/10i $pc`：  从当前pc值开会，在内存中反汇编10条指令。数据为0时表现为unimp。

`si`：向下执行一条指令，并打印即将要执行的指令的地址

`p/x $t0`：以十六进制打印$t0。十进制则使用`d`

`disas /m 函数名`：将函数代码和汇编指令映射起来

`display 变量名`：打印变量的值

### 打断点

`b *0x80200000`：在特定地址处打断点

`b 真实函数名`：在指定函数名上打断点

`c`：continue的缩写，向下运行直到一个断点

### GDB Files

当使用gdbserver远程调试时，在运行时动态加载一个文件时，需要从该文件中读取额外的符号表信息。故可以使用add-symbol-file指令：

```
add-symbol-file filename [ textaddress ] [
-s section address ... ]
```

filename指定文件名，textaddress指定文件的text段加载到的内存地址（？？？什么地址），使用`-s section address`来指定额外的段基地址信息。

### 技巧信息

切换到源代码视角：layout src

切换到汇编视角：layout asm

查看栈帧：info frame

watch point：

## GNU

[Binutils - GNU Project - Free Software Foundation](https://www.gnu.org/software/binutils/)

### gcc

gcc -E可以预编译宏进行宏展开

### objdump

objdump允许查看目标文件的一些信息。其具体用法见：[objdump (GNU Binary Utilities) (sourceware.org)](https://sourceware.org/binutils/docs/binutils/objdump.html)

参数：

-d：呈现目标文件的汇编代码

### ld

链接器，[LD (sourceware.org)](https://sourceware.org/binutils/docs-2.41/ld.html)

### strace



### find

```
find -name 文件名
```

文件名支持正则表达式，例如`find -name ramfs*`

> 关于 gdb，可以在此下载更新版本的预编译包
>
> https://github.com/sifive/freedom-tools
>
> 使用其中的 riscv64-unknown-elf-gdb-py（上文中旧版本也有）代替 riscv64-unknown-elf-gdb 以获得 python 支持
>
> 然后安装
>
> https://github.com/cyrus-and/gdb-dashboard
>
> 以获得更便捷的调试体验，以 rCore 的 ch1 分支为例，`make debug` 后效果如下图：
>
> https://i.v2ex.co/4mzn264Y.png
>
> ------
>
> 修改 rCore/os/Makefile 文件，可以支持在 Clion 中断点调试
>
> ```
> # 修改 release 模式为 debug
> MODE := debug
> 
> # 修改 cargo build 为debug（去掉后面的 --release）
> 	@cargo build
> 
> # 新增远程调试（相当于服务端）
> debug-remote: build
> 	qemu-system-riscv64 -machine virt -nographic -bios $(BOOTLOADER) -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA) -s -S
> ```
>
> 在 Clion 中新增远程调试（相当于客户端）
>
> https://i.v2ex.co/72BPu4hI.png
>
> 调试器：
> /Users/用户名/工具保存目录/riscv64-unknown-elf-toolchain-10.2.0-2020.12.8-x86_64-apple-darwin/bin/riscv64-unknown-elf-gdb
>
> > 此处使用非 python 版本，减少 GDB 面板输出
>
> ‘target remote’ 实参：
> localhost:1234
>
> 符号文件：
> /Users/用户名/项目保存目录/rCore/os/target/riscv64gc-unknown-none-elf/debug/os
>
> 在 Clion 中添加断点（必须，否则直接运行结束）
>
> 先运行
> `make debug-remote`
>
> 然后运行 Clion 中配置的远程调试，运行效果如下图：
>
> https://i.v2ex.co/fIzlX2z8.png
>
> ------
>
> 如看不到图片，可以下载查看
>
> https://www.aliyundrive.com/s/C9DtRWtjXRs

> ### 补一个vscode远程调试的方法
>
> 1.先在os路径下执行,make debugserver,开启远程调试
>
> 2. 配置launch.json如下
>
> ```
> {
>     "version": "0.2.0",
>     "configurations": [
>         {
>             "type": "cppdbg",
>             "request": "launch",
>             "name": "Attach to gdbserver",
>             "program": "${workspaceFolder}/os/target/riscv64gc-unknown-none-elf/release/os",
>             "miDebuggerServerAddress": "localhost:1234",
> 
>             "miDebuggerPath": "/home/***/riscv64-gcc/bin/riscv64-unknown-elf-gdb",
>             "cwd": "${workspaceRoot}",
>         }
>     ]
> }
> ```
>
> 1. F5开始快乐调试吧
> 2. 补充说明,当前main分支下的gdbserver缺少参数,需要补充
>
> ```
> -drive file=$(FS_IMG),if=none,format=raw,id=x0 \
> -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -s -S
> ```
>
> 才可以正常调试

> [@kidcats](https://github.com/kidcats) 转载你的原始内容，便于在GitHub Issue中搜索，谢谢你的贡献。
> 补一个vscode远程调试的方法
> 1.先在os路径下执行,make debugserver,开启远程调试
>
> 2. 配置launch.json
>
> ```
> {
>     "version": "0.2.0",
>     "configurations": [
>         {
>             "type": "cppdbg",
>             "request": "launch",
>             "name": "Attach to gdbserver",
>             "program": "${workspaceFolder}/os/target/riscv64gc-unknown-none-elf/release/os",
>             "miDebuggerServerAddress": "localhost:1234",
>             "miDebuggerPath": "/home/***/riscv64-gcc/bin/riscv64-unknown-elf-gdb",
>             "cwd": "${workspaceRoot}",
>         }
>     ]
> }
> ```

## 为wsl瘦身

打开powershell执行：

```
diskpart
```

执行：

```
select vdisk file="C:\Users\15035\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_79rhkp1fndgsc\LocalState\ext4.vhdx"
compact vdisk
```

