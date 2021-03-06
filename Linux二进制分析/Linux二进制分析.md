内容提要

二进制分析属于信息安全业界逆向工程中的一种技术，通过利用可执行 的机器代码(二进制)来分析应用程序的控制结构和运行方式，有助于信 息安全从业人员更好地分析各种漏洞、病毒以及恶意软件，从而找到相应 的解决方案。



逆向工程是发现程序如何运行和发挥作用的行为，进 一步讲，就是使用反编译器和逆向工具进行组合，并依靠我们的专业技能来 控制要进行反编译的目标程序，来理解、解析或者修改程序的行为。

# Linux 环境和相关工具



### 1.1.2  objdump

objdump是一种对代码进行快速反编译的简洁方案，在反编译简单的、未被篡改的二进制文件时非常有用

其最主要的一个缺陷就是需要依赖 ELF 节头，并且不会进行控制流分析， 这极大地降低了 objdump 的健壮性。如果要反编译的文件没有节头，那么使 用 objdump 的后果就是无法正确地反编译二进制文件中的代码，甚至都不能 打开二进制文件。

### 1.1.4 strace

system call trace(strace，系统调用追踪)是**基于 ptrace(2)系统调用的一款工具**，strace 通过在一个循环中使用 PTRACE_SYSCALL 请求来**显示运行中程序的系统调用**(也称为 syscalls)活动相关的信息以及程序执行中捕捉到的信号量。

strace 在调试过程中非常有用，也可以用来收集运行时**系统调用相关**的信息。

### 1.1.5 ltrace

library trace(ltrace，库追踪)是另外一个简洁的小工具，与strace 非常类似。ltrace 会解析共享库，即一个程序的链接信息，并打印出用到的 库函数。

### 1.1.7 ftrace

function trace(ftrace，函数追踪)  ftrace 的功能与 ltrace 类似，但还可以显示出二进制文件本身的函数调用。

### 1.1.8 readelf

readelf 命令是一个非常有用的解析 ELF 二进制文件的工具。在进行 反编译之前，需要收集目标文件相关的信息，该命令能够提供收集信息所需要的特定于 ELF 的所有数据。在本书中，我们将会使用 readelf 命令收集 符号、段、节、重定向入口、数据动态链接等相关信息。readelf 命令是 分析 ELF 二进制文件的利器。



# ELF 二进制格式

ELF 目前已 经成为 UNIX 和类 UNIX 操作系统的标准二进制格式。

一个 ELF 文件可以被标记为以下几种类型之一。

- 􏰄  ET_NONE:未知类型。这个标记表明文件类型不确定，或者还未定义。

- 􏰄  ET_REL:重定位文件。ELF 类型标记为 relocatable 意味着该文件 被标记为了一段可重定位的代码，有时也称为目标文件。可重定位 目标文件通常是还未被链接到可执行程序的一段位置独立的代码

  (position independent code)。在编译完代码之后通常可以看到一 个.o 格式的文件，这种文件包含了创建可执行文件所需要的代码 和数据。

- 􏰄  ET_EXEC:可执行文件。ELF 类型为 executable，表明这个文件被标 记为可执行文件。这种类型的文件也称为程序，是一个进程开始执 行的入口。

- 􏰄 ET_DYN:共享目标文件。ELF 类型为 dynamic，意味着该文件被标记 为了一个动态的可链接的目标文件，也称为共享库。这类共享库会在 程序运行时被装载并链接到程序的进程镜像中。

  􏰄 ET_CORE:核心文件。在程序崩溃或者进程传递了一个 SIGSEGV 信 号(分段违规)时，会在核心文件中记录整个进程的镜像信息。可以 使用 GDB 读取这类文件来辅助调试并查找程序崩溃的原因。



## 2.2 ELF 程序头ElfN_Ehdr

ELF 程序头是对二进制文件中段的描述，是程序装载必需的一部分。