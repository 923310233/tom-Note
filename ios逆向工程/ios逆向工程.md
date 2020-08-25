􏰫􏰬􏿼􏼿􏰕􏰀􏼰􏰭􏻈􏰮􏰯􏹞􏼫􏰰􏽁􏾎􏰀􏰱􏰲􏰓􏰔 􏻈􏰳􏼴􏷶􏰴􏿺􏰲􏰘􏿍􏼰􏾏􏽇􏻈􏰵􏰶􏰷􏰸􏰲􏰆􏰇􏰹􏰺􏰻 􏰼􏼬􏰫􏰬􏿼􏼿􏰕􏰀􏼰􏰭􏻈􏰮􏰯􏹞􏼫􏰰􏽁􏾎􏰀􏰱􏰲􏰓􏰔 􏻈􏰳􏼴􏷶􏰴􏿺􏰲􏰘􏿍􏼰􏾏􏽇􏻈􏰵􏰶􏰷􏰸􏰲􏰆􏰇􏰹􏰺􏰻 􏰼􏼬􏰽􏻈􏹐􏹑􏷎􏷏􏷮􏷒􏷓􏹒􏰤􏹓􏹔􏴣􏹕􏹖􏹗􏷑􏷒􏹘􏹙􏷩􏷪 􏰤􏹚􏷔􏹛􏹜􏷌􏹙􏲈􏹝􏷓􏹞􏹟􏰤􏹠􏹡􏹢􏹣􏹙􏷛􏷜􏹤􏹥􏹦 􏹧􏹨􏹩􏹐􏹑􏷎􏷏􏷮􏷒􏷓􏹒􏰤􏹓􏹔􏴣􏹕􏹖􏹗􏷑􏷒􏹘􏹙􏷩􏷪 􏰤􏹚􏷔􏹛􏹜􏷌􏹙􏲈􏹝􏷓􏹞􏹟􏰤􏹠􏹡􏹢􏹣􏹙􏷛􏷜􏹤􏹥􏹦 􏹧􏹨􏰟􏰠􏰐􏰔

## 1.4 iOS应用逆向工程的工具

相对于正向开发，逆向工程的工具并不那么智能，很多工作需要我们手动完成



iOS逆向工程的工具可以分为四类：

监测工具，反汇编工具，调试工具，开发工具



### 1.4.1 监测工具

这类工具通常可以记录目标程序的某些操作，如UI变化，网络活动，文件访问

iOS逆向常用的检测工具有reveal, snoop-it, introspy



reveal 可以用来实时地监测目标APP的UI布局变化



### 1.4.2 反汇编工具

从UI层面切入代码层面后，就要用反汇编工具来梳理代码



反汇编工具把二进制文件作为输入，经过处理后输出这个文件的汇编

常用的反汇编工具是IDA和Hopper



阅读生成的汇编代码是最具挑战的部分



### 1.4.3 调试工具

在iOS逆向中，用LLDB进行调试



### 1.4.4 开发工具

对于APP开发者来说，Xcode是最常用的开发工具，但是我们一旦把战场从APP store转到越狱的iOS，开发工具就得到了扩充，不但有基于Xcode的iOSOpenDev，还有偏命令行的TheOS

TheOS似乎非常牛逼



# 2 越狱iOS平台

未越狱的iOS是个封闭的黑盒子。对于未越狱的iOS，苹果官方开放的第三方直接访问iOS文件系统的接口非常有限，开发者只需要根据参考文档就行了。



因为权限太低，来自APP store的普通APP不能访问自身文件目录以外的绝大多数文件。而越狱后可以访问全文件系统。

来自Cydia的iFile是一个优秀的第三方文件管理APP



也可以通过iFunBox等PC端软件来访问全iOS文件系统



### 2.1.1 iOS目录简介

iOS是由OS X演化而来的，而OSX是基于Unix的

/ 根目录

/bin 用户级基础功能二进制

/boot 使系统成功启动的所有文件

/dev 存放BSD设备文件 每个文件代表系统的一个块设备或者字符设备

/sbin 系统级基础功能的二进制

/etc 系统脚本及配置文件





## 2.2 iOS二进制文件类型

三类二进制文件：

Application

Dynamic Library 活着dylib

Daemon



### 2.2.1 Application

即我们最熟悉的APP



**1 bundle**

Bundle是一个含有可执行的代码及代码所需资源，以特定标准的层次结构组合起来的文件夹。这里的可执行是指编译过后可直接运行的代码程序。



系统如何识别Bundle呢？一般而言一个文件夹如果带着.app，.bundle，.framework，.plugin，.kext等等特定后缀，那么系统就认为是Bundle。

 从上面的后缀我们也可以看出**Bundle主要分为**：

- Appliction。应用程序，包含代码和资源。iOS和macOS的app就是这种。
- Frameworks。框架，包含动态共享库和相应资源。我们常用的系统库和第三方库都属于这种。
- Plug-Ins。插件，macOS很多的系统功能支持插件，一种动态加载代码模块的方式。
- 

Application类型的bundle是很常见的，一般里面会包含一下几中类型的文件：

- Info.plist文件。每个程序中必须有这个文件，因为它包含了程序运行的配置信息，是系统运行程序的依据。
- 可执行代码文件。这是程序的主体，包含了程序的进入点和链接的静态代码。
- 资源文件。程序运行过程中需要的资源，比如图片，音频，视频，多语言的字符串文件，nib文件，数据文件，配置文件等等。这里面的大部分文件都可以根据语言、地区、设备通过特定的结构或命名方式加以区分，程序会自动识别加载。
- 其他支持文件。Mac app可以嵌入高层资源，比如私有库，插件，文件末班，自定义数据资源等等。iOS app可以包含自定义数据资源，但是不能包含私有库和插件。



Framework也是bundle，但是framework的bundle中存放的是一个dylib

通常来说 framework的地位比APP更高，**因为一个APP的绝大多数功能都是通过framework提供的接口来实现的**

将某个 bundle确定为逆向目标后，绝大多数的逆向线索都可以在bundle中找到。

 



**3 系统APP VS storeAPP**

/Application目录放的是系统APP和从Cydia下载的APP

而/var/mobile/Containers/下放的是storeAPP



存在如下的差异：

两种APP的bundle内部目录的结构差别不大，都包含info.plist，可执行文件以及lproj 但数据目录的位置不同

storeAPP的数据目录在/var/mobile/Containers/Data下，

以mobile权限运行的系统APP数据目录在/var/mobile下

以root权限运行的系统APP数据目录在/var/root下



Cydia APP的安装包格式一般是deb

storeAPP的安装包格式一般是ipa



deb从Debian移植过来，能够以root权限运行

而ipa则是苹果味iOS推出的APP安装包格式，只能以mobile权限运行



沙盒 sandbox

iOS中的沙盒是一种访问权限机制，它是iOS最核心的安全组件之一

沙盒会将APP的文件访问范围限制在这个APP内部



### 2.2.2 dylib

Xcode工程中导入的各种framework，链接的各种lib，其本质都是dylib

在iOS中，lib分为static和dynamic两种，其中static在编译阶段成为APP可执行文件的一部分，会增加可执行文件的大小



### 2.2.3 Daemon

1 iOS中并没有真正的后台多任务，对于大多数的storeAPP来说，按下home键，大多数的功能就会被暂停



Daemon为了后台运行而生，给用户提供了各种守护





# OS X工具集

## 3.1 class-dump

class-dump，用来dump目标对象的class信息的工具









### 碰到的问题

Bundle identifier 包名，是应用（application）在手机里的唯一标识符



## Mach-o

 Apple的操作系统只支持三种文件格式:

1. 以`#!`开头的脚本文件
2. 通用二进制文件
3. Mach-O格式文件

但是实际上  以`#!`开头的脚本文件其实是shell解释器找到后面指定的脚本解释器来执行的, 而通用二进制文件其实是多个架构的Mach-O文件的打包体。



Unix标准了一个可移植的二进制格式`ELF`但是苹果并没有实现它而是维护了一套NeXTSTEP的遗物 `Mach-Object`简称`Mach-O`。

官方文档《OS X ABI Mach-O File Format Reference》：



![macho](./images/macho.png)



header的结构：

```
struct mach_header_64 {
    uint32_t magic;           /* mach magic number identifier */
    cpu_type_t cputype;       /* cpu specifier */
    cpu_subtype_t cpusubtype; /* machine specifier */
    uint32_t filetype;        /* type of file */
    uint32_t ncmds;           /* number of load commands */
    uint32_t sizeofcmds;      /* the size of all the load commands */
    uint32_t flags;           /* flags */
    uint32_t reserved;        /* reserved */
};
```



![header](./images/header.png)

filetype，描述了二进制文件的类型，包括了十来个有效值，常见的：

```c
#define MH_OBJECT      0x1    // 中间目标文件，例如.o文件
#define MH_EXECUTE     0x2    // 可执行文件
#define MH_DYLIB       0x6    // 动态链接库
#define MH_DYLINKER    0x7    // 动态链接器
```



使用MachOView工具查看Mach-O文件