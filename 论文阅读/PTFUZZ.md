At the level of [machine language](https://en.wikipedia.org/wiki/Machine_language) or [assembly language](https://en.wikipedia.org/wiki/Assembly_language), control flow instructions usually work by altering the [program counter](https://en.wikipedia.org/wiki/Program_counter). For some [central processing units](https://en.wikipedia.org/wiki/Central_processing_unit) (CPUs), the only control flow instructions available are conditional or unconditional [branch](https://en.wikipedia.org/wiki/Branch_(computer_science)) instructions, also termed jumps.



当前的灰盒模糊测试通常使用两种方法来收集分支信息：编译时检测（AFL）和仿真（QAFL）。



Compile-time instrumentation is efficient, but it does not support binary programs. 

In this paper, we propose a greybox fuzzing approach named PTfuzz, which leverages hardware mechanism (Intel Processor Trace) to collect branch information. 

Our experiments show that PTfuzz can fuzz the original binary programs without any modification



## I.补充

现有的灰盒测试技术的局限性：

1. No binary-only fuzzing support：诸如AFL，AFLFast和VUzzer之类的灰盒测试模糊器**都依赖于目标程序的源代码**
2. **Slow feedback mechanism.**：几种反馈机制，例如动态二进制插桩（Intel PIN [10]），静态重写（AFL-dyninst ）和仿真（QEMU）被引入到模糊测试中。QEMU通常的性能成本是2-5倍。**这种费用的原因超出了本文的研究范围**。毫无疑问，在我们的模糊实践中，由于缓慢的反馈机制而导致的这种性能开销是无法忍受的。迫切需要改进这些缓慢的机制。
3. Inaccurate coverage feedback：AFL and AFLFast **use bitmap to trace basic block transitions and measure code coverage**. Each byte of the bitmap represents hit count of a specific edge(e.g.from A to B).



英特尔PT 是英特尔处理器的新功能。它可以显示准确而详细的程序控制流信息轨迹，例如条件跳转和无条件跳转。特别是，PT可以跟踪每个基本块的准确地址。因此，在PTfuzz中，遵循AFL的思想，我们使用PT的此功能来测量基本块之间的转换，并为模糊循环提供准确的覆盖范围反馈信息。

PTfuzz能够模糊任何仅二进制的软件，因为我们直接从处理器获取执行信息，并且完全不依赖任何源代码。



## Backround

 If a test case exercises a new program path, it will be saved as a seed to generate more test cases。但是，如何确定测试用例是否走上了新道路？有几种选择：

1. Total block (TBL)
2.  new block (NBL)
3.  total branch (TBR) 
4. new branch (NBR) coverage.

Total indicates the whole number of hit blocks or branches, and *new* indicates that new blocks or branches are desired. 

For example, in TBL, if test case T1 hits basic block A, and T2 hits blocks A and B. Then we can decide that *T* 2 exercises a new path. 



AFL的Zalewski [6]对这4种方法进行了实验，发现NBR具有最佳性能

every hit branch is recorded in a specific memory space called **bitmap**. 



###  WHAT DO VALUES IN BITMAP INDICATE?

the number of hit branches recorded in bitmap is a useful indicator指标 to compare code coverage of different fuzzing 

bitmap的记录是衡量代码覆盖率的一个指标



### FEEDBACK IN GREYBOX FUZZING

feedback mechanism adopted in each fuzzer：

1. Compile-time instrumentation：预处理和编译两个阶段之间，对源文件进行修改。 AFL，AFLFast之类的模糊器采用编译时工具来利用其反馈机制。They provide code coverage feedback to the main fuzzing loop **by recording transitions between basic blocks**. 但是，基本块的地址是随机分配的值，而不是运行时的确切地址。
2. Dynamic binary instrumentation: PIN和DynamoRIO是最著名的作品。动态二进制插桩的最大问题是相当大的开销。
3. QEMU这样的仿真通过动态二进制转换来仿真CPU
4. Intel Processor Trace is also a hardware feature of processors, but it is much more advanced than BTS. PT is capable of accurately tracing program control flow information and we can record basic block transitions based on it. 



Intel PT：

先前的硬件功能（例如Intel Last Branch Record）也执行程序跟踪，但是其输出存储在特殊的寄存器中，而不是主存储器中。

**Intel PT利用内存空间来存储跟踪数据，因此连续PT跟踪仅限于主内存的大小**。



具体来说，PT的输出以**数据包**的格式收集。

数据包根据其功能可以分为两种：基本执行信息分组和控制流信息分组 （basic execution information packet and control flow information packet.）：

1. basic execution information packet： Packet Stream Boundary (PSB), Time-Stamp Counter (TSC) and other relevant packets, 它们说明了程序的一般运行状态
2. control flow information packet： 这种packet是去给解码器使用的。control flow information includes **time, program flow and other information** during run time.



基本块是一个连续的代码段，没有跳转或分支。在本文中，为了捕获基本块之间的转换，我们需要专注于程序流信息，





Intel PT specifies instructions that can change program flow as Change of Flow Instructions (COFI). 

Three types of COFI instructions are introduced: Direct transfer COFI, Indirect transfer COFI and Far transfer COFI.

Moreover, Intel PT introduces 4 specific packets **to trace COFI instructions**:

• Taken Not-Taken (TNT) packet. A specific bit in TNT packet can indicate whether a branch is taken or not in conditional jumps. So it is used to trace direction of conditional branches.

• Target IP (TIP) packet. TIP records the **target instruction pointer**(IP) of **jump or transfer instructions**. IP value is stored in specific bits in this packet. In detail, TIP can be classified into TIP, TIP.PGE, TIP.PGD and TIP.FUP according to different application scenarios.

• Flow Update Packet (FUP). When asynchronous events such as interrupts or traps happen, we need FUP to pro- vide source IP addresses, because TIP is out of function in these events.

• MODE packet. MODE provides important program execution information and it has a wide range of formats to indicate the execution mode.



## 关于符号执行

Symbolic execution and fuzzing are the major two parts of software testing and debugging techniques.

路径爆炸问题



模糊测试是一种有效的测试技术，可以揭示错误和漏洞。**给定一个初始输入种子**，可以通过简单的突变来生成一堆新的测试用例，以便尽可能地行使并覆盖尽可能多的程序路径。如今，大多数漏洞是由特别轻量级的模糊测试程序暴露的，这些测试程序没有利用密集的程序分析[6]。

## 3.Model

PTfuzz mainly contains two relevant parts: the main fuzzing loop and the PT infrastructure.

- Pre-build COFI map and write MSR registers. The target binary will be loaded and instructions are dumped to construct COFI map mentioned in Section II-C for decoding purpose. And specific MSR registers are written to set up ip filtering for PT process.

具体的解释一下：**Target binary is loaded into memory and instructions in text section are dumped to build COFI map**.

- Fork. At the beginning of execution, a child thread is called up to perform as Processor Trace infrastructure by *fork*(). And this child thread is reaped after execution is finished, and **decoded tracing data will be sent to the parent thread**.



As mentioned above, the Processor Trace infrastructure,is created by the main fuzzing loop through fork(). And it mainly completes these tasks:

- Enable PT. In order to perform continuous tracing of program execution, Processor Trace needs to be enabled. After fork() operation, the PT infrastructure enables PT at the beginning of program execution.

- Record tracing data. After PT is enabled, it will captures program execution information non-stop. And the **PT infrastructure specifies a certain memory space to store tracing data after enabling PT**. Thus PT can **write tracing data in this specific space**.
- **Decode tracing data**. Our purpose is to find basic block transitions in program execution, so the PT infrastructure decodes the raw tracing data and captures TNT, TIP and FUP packets as described in Section II-C and other relevant information. And basic block transitions are written into the bitmap so the main fuzzing loop can access it and make decisions.  子线程来解码



## 实现的两个主要部分

Processor Trace output. Previous works about PT suchas Simple-PT, cannot perform continuous tracing因为它们将跟踪信息存储在文件中。写入文件可能很慢，并且无法与主模糊循环进行实时交互。.

So in PTfuzz we use *mmap*_*page* feature in perf_event [36]. This feature configures a specific memory space to store tracing data. And PTfuzz records tracing information in this memory. Accessing memory can be much faster than files and interaction with the main fuzzing loop is timely.



Decoding PT information. 英特尔已经提出了自己的PT解码器库[37]。但是，它是通用解码器，不能很好地满足我们的需求，因为它不提供用于跟踪基本块转换的API。

