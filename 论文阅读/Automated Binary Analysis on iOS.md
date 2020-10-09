# Automated Binary Analysis on iOS – A Case Study on Cryptographic Misuse in iOS Applications


## 摘要

苹果iOS平台的各种移动应用都会处理敏感数据，因此需要依靠操作系统本身提供的保护机制。 A wrong application of cryptography or security-critical APIs ，会将秘密暴露给不相关的各方，并破坏整体的安全性。



我们介绍了一种在iOS应用中发现密码学误用的方法。我们提出了一种将64位ARM二进制文件反编译为其LLVM中间表示（IR）的方法。Based on the reverse-engineered code, static program slicing is applied to determine the data flow in relevant code segments. 为了使这一分析最为准确，我们提出了Andersen指针分析的改编版本，能够处理从二进制中恢复类型信息的反编译LLVM IR代码。



最后，为了突出加密API的不当使用，我们根据提取的执行路径检查了一组预定义的安全规则。因此，我们不仅能够确认iOS应用中存在的问题语句，而且还能准确定位其来源。



## 介绍

由于开发者通常不提供详细的信息，而且移动应用程序的源代码也不提供，因此只能通过对最终应用程序进行逆向工程来验证加密功能的正确实现。

安卓平台的分析框架已经可以使用，而且研究[12，16]已经证实加密滥用是安卓平台的一个重要问题。虽然人们预期iOS平台上也会存在加密误用的问题，但到目前为止还没有对iOS平台进行分析。



与Android平台[12，20]相比，这样的分析在iOS平台上代表了一项具有挑战性的任务。例如，Android应用是以可逆的字节码格式提供的，而iOS应用则被编译成针对特定CPU架构的机器代码。手动检查iOS二进制的拆解代码可能是一项具有挑战性的工作。



主要的挑战

- 反编译和简化机器代码。针对ARMv8架构编译的64位二进制文件，分析起来容易出错且繁琐。因此，我们将二进制文件转化为更高级别的LLVM中间表示法(IR)，where all low-level CPU instructions have to be modeled appropriately. This allows to re-use existing LLVM-based tools, such as KLEE [10], PAGAI [31], and LLBMC [41].
- iOS应用是用Objective-C和Swift等面向运行时的语言开发的，大部分的控制流决策都是在运行时进行的。 To recover a semantically correct control flow from the binary, we reconstruct the hierarchies of classes, methods, and types from binaries. This information allows to resolve the target of a function call through the dispatch routine.
- Pointer analysis: Computing control flow and data dependencies, as well as the identification of instructions and variables that have an impact on a particular program statement, requires information about where different variables (and CPU registers) point to during execution. Since computing points- to sets is an undecidable problem [3], we propose a solution addressing the trade-off between the accuracy of program slices and the runtime overhead.（计算控制流和数据依赖关系，以及识别对特定程序语句有影响的指令和变量，需要了解执行过程中不同变量(和CPU寄存器)的指向信息。由于计算点到集是一个无法确定的问题[3]，我们提出了一个解决程序切片的精度和运行时开销之间的权衡问题的方案。）



通过解决这些挑战，我们开发了一个框架，允许通过重建相应的控制流和数据流来检查64位iOS二进制文件。



## 2 related work

在过去的几年里，移动平台上的安全方面的分析吸引了很多人的关注。该领域的大多数出版物都集中在Android生态系统上，该平台的开放性促进了程序检查。由於 Android 應用程式中的 Dalvik 字節碼可以反編為 Java 程式碼，因此現有的靜態分析工具很容易適用[5, 6, 54]。同样，动态分析的方法[19, 50]可以在应用程序被Dalvik虚拟机执行时进行实时信息流跟踪。



## 3 BACKGROUND

### 3.1 iOS Applications

A Mach-O file is split into three regions:

1. Header： 标识文件为Mach-O文件，包括目标CPU架构的信息。
2. Load Commands：指定文件的布局和指定的内存。指定文件布局和段的指定内存位置。
3. Data：由不同的区域和部分组成。由加载到内存中的不同区域和段组成。动态加载器信息定义了动态符号在执行过程中必须存储的位置。

The Data region itself contains all infor- mation needed for execution, including the machine instructions.



Method invocations are handled by a dynamic dispatch派遣 function in the Objective-C runtime library, which requires the type of the object and the name of the method to be called. Therefore, the following sections in the binary play an essential role:





During execution, applications can invoke externally defined library functions using indirect symbols. Occurring as lazy or non- lazy symbols, our analysis has to detect and handle these calls correctly as they may create or modify data. 



3.2 Program Slicing







Static slicing can be used to determine all code statements of a program that may affect a value at a specified point of execution (slicing criterion).由此产生的程序切片覆盖了所有可能的执行路径，并允许得出关于程序的功能性的结论。我们采用Weiser[55]的算法来创建LLVM IR代码的切片，并寻找从参数的起源到其使用的路径，例如，在加密函数中。
Weiser presented an intra-procedural method that models the data flow within a function using equations. Relevant variables and statements are determined in an iterative manner. As summarized by Tip [52], the algorithm consists of two steps:







