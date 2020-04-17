当前的灰盒模糊测试通常使用两种方法来收集分支信息：编译时检测（AFL）和仿真（QAFL）



Compile-time instrumentation is efficient, but it does not support binary programs. 

In this paper, we propose a greybox fuzzing approach named PTfuzz, which leverages hardware mechanism (Intel Processor Trace) to collect branch information. 

Our experiments show that PTfuzz can fuzz the original binary programs without any modification



## I.补充

CTI(compile-time instrumentation)：对源代码进行修改，以达到控制内存操作功能

动态二进制插桩（dynamic binary instrumentation ,DBI）技术是一种通过注入插桩代码，来分析二进制应用程序在运行时的行为的方法。在程序执行过程中插入特定分析代码，实现对程序动态执行过程的监控与分析。

Pin是由英特尔公司开发的动态二进制插桩框架，它允许我们为Windows，Linux和OSX构建称为Pintools的程序分析工具。我们可以使用这些工具来监控、修改和记录程序运行时的行为。



blocks:



## 2.Model

with the help of TNT, TIP and FUP packets, we are able to write a decoder to capture basic block transitions in program execution. 

More specifically, we need to record addresses of basic blocks.



IDA中每一块代码就代表着一个基本块，就是以指令跳转为作划分界限的。



PTfuzz mainly contains two relevant parts: the main fuzzing loop and the PT infrastructure.



- Fork. At the beginning of execution, a child thread is called up to perform as Processor Trace infrastructure by *fork*(). And this child thread is reaped after execution is finished, and **decoded tracing data will be sent to the parent thread**.



As mentioned above, the Processor Trace infrastructure,is created by the main fuzzing loop through fork(). And it mainly completes these tasks:

- Enable PT. In order to perform continuous tracing of program execution, Processor Trace needs to be enabled. After fork() operation, the PT infrastructure enables PT at the beginning of program execution.

- Record tracing data. After PT is enabled, it will captures program execution information non-stop. And the **PT infrastructure specifies a certain memory space to store tracing data after enabling PT**. Thus PT can **write tracing data in this specific space**.
- **Decode tracing data**. Our purpose is to find basic block transitions in program execution, so the PT infrastructure decodes the raw tracing data and captures TNT, TIP and FUP packets as described in Section II-C and other relevant information. And basic block transitions are written into the bitmap so the main fuzzing loop can access it and make decisions.  子线程来解码



## 实现的两个主要部分

Processor Trace output. Previous works about PT suchas Simple-PT [35], cannot perform continuous tracing因为它们将跟踪信息存储在文件中。写入文件可能很慢，并且无法与主模糊循环进行实时交互。.

So in PTfuzz we use *mmap*_*page* feature in perf_event [36]. This feature configures a specific memory space to store tracing data. And PTfuzz records tracing information in this memory. Accessing memory can be much faster than files and interaction with the main fuzzing loop is timely.



Decoding PT information. 英特尔已经提出了自己的PT解码器库[37]。但是，它是通用解码器，不能很好地满足我们的需求，因为它不提供用于跟踪基本块转换的API。

