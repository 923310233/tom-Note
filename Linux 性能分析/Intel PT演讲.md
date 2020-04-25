PT decoder 不需要知道where data begins

https://easyperf.net/blog/2019/09/06/Intel-PT-part3

https://easyperf.net/blog/2019/09/13/Intel-PT-part4



# Better debugging experience.

Logs still are very useful because you can print some values in them. Until `PTWRITE` instruction came out there was no way of dumping data in processor traces. Traces were only useful for determining control flow. But in recent CPUs we have `PTWRITE` instruction that allows writing values into the PT packets[7](https://easyperf.net/blog/2019/08/30/Intel-PT-part2#fn:7). According to [Intel SD Manual](https://software.intel.com/en-us/articles/intel-sdm):

> This instruction reads data in the source operand and sends it to the Intel Processor Trace hardware to be encoded in a PTW packet.

您可以使用ptfeature工具（这是simple-pt的一部分）来检查您的CPU是否支持PTWRITE。







#  Analyzing performance glitches.

a.cpp:

```
#include <random>

int goFastPath(int* arr, int n);
int goSlowPath(int* arr, int n);

int main() {
  int arr[1000];
  for (int i = 0; i < 1000; i++) {
    arr[i] = i;
  }

  const int min = 0;
  const int max = 999;
  std::default_random_engine generator;
  std::uniform_int_distribution<int> distribution(min,max);

  // counting sum up to N
  for (int i = 0; i < 100000; i++) {
    int random_int = distribution(generator);
    if (random_int < 999)
      goFastPath(arr, random_int);
    else
      goSlowPath(arr, random_int);
  }
  return 0;
}
```

b.cpp:

```
int goFastPath(int* arr, int n) {
  return (n * (n + 1)) / 2;
}

int goSlowPath(int* arr, int n) {
  int res = 0;
  for (int i = 0; i <= n; i++)
    res += arr[i];
  return res;
}
```

99.9％的时间我们使用的goFastPath。但是在0.1％的时间内，我们将退回到慢速路径

```
 g++ a.cpp -c -O2 -g
 g++ b.cpp -c -O2 -g
 g++ a.o b.o
 
 
 perf record -e intel_pt/cyc=1/u ./a.out
```

注意，我使用cyc = 1来获得最好的粒度。此选项要求CPU在每个周期产生定时包。 

要将跟踪解码为易于阅读的形式，请使用以下命令：

```
perf script --ns --itrace=i1t -F +srcline,+srccode > decoded.dump
```

 Option `--ns` will show the timestamps in nanoseconds. 

Option `-F +srcline,+srccode` will add source line and source code for each decoded assembly instruction.



选项--trace = i1t有点难以解释。此选项指定解码器将采样我们的迹线的时间段。就我而言，我要求对每个时钟信号进行采样，即它将分别输出每条指令并为其添加时间戳。

例如，如果指定--trace = i100ns，则解码器将大致每100ns合成一条指令。它只会输出1个样本，但会在其附近显示省略了多少指令。这是对大轨迹进行初始分析的一种方法，因为增加周期可以使解码器更快地工作。在Linux内核文档中了解更多信息。 



Decoded processor traces won’t be able to show the values of different variables  but you will get precise control flow.

但是……有一个警告。当向输出添加+ srcline或+ srccode时，解码会变得非常慢。对于仅运行7ns且生成的迹线小于1MB的工作负载，解码花费了一天多的时间！ 8我知道这对大多数用户来说都是胡说八道。我不确定为什么会发生这种情况，但我怀疑实施方式可能不是最佳的。



好消息是，有一种解决方法。您不必事先解码整个轨迹，也可以懒惰地完成它。首先，您可以解码而无需在perf脚本命令中添加+ srcline或+ srccode。然后查看您关心的时间范围，然后仅使用--time start，stop选项对该时间范围进行解码。例如：

```
perf script --ns --itrace=i1t -F +srcline,+srccode --time 253.555413140,253.555413520 > time_range.dump
```



#  Better profiling experience.

```
perf record -e intel_pt//u ./a.out
```

This command essentially dumps the processor traces into a file on a disk. Those traces contain encoded history for the entire runtime. 



我在计算机上基于中断的采样所能收集的最大采样率约为100Khz（每秒100K个采样）。即每10微秒大约有1个样本。不要误会我的意思，它的精度仍然很高，适用于大多数情况。但是，开销从1.87s1变为2.22s，现在约为28％，这远远超过了Intel PT：

```
$ echo 999999999 | sudo tee /proc/sys/kernel/perf_event_max_sample_rate
$ time -p perf record -F 999999999 -e cycles:u -o pmi.data ./a.out
[ perf record: Woken up 39 times to write data ]
[ perf record: Captured and wrote 9.712 MB pmi.data (253272 samples) ]
real 2.22
user 2.11
sys 0.05
$ perf report -i pmi.data -n --stdio --no-call-graph --itrace=i100ns
# Samples: 434K of event 'cycles:u'
# Overhead       Samples  Command  Shared Object  Symbol                                                                                                                         
# ........  ............  .......  .............  ..................................................
#
    88.53%        384801  a.out    a.out          [.] std::uniform_int_distribution<int>::operator()
     5.76%         25044  a.out    a.out          [.] goFastPath
     3.64%         15807  a.out    a.out          [.] main
     1.82%          7923  a.out    a.out          [.] goSlowPath
     0.25%          1067  a.out    ld-2.27.so     [.] _start
```



With Intel PT, easily we get 100x more samples:

```
$ perf record -e intel_pt/cyc=1/u -o pt.data ./a.out
$ perf report -i pt.data -n --stdio --no-call-graph --itrace=i100ns
# Samples: 32M of event 'instructions:u'
# Overhead       Samples  Command  Shared Object        Symbol                                                                                                                         
# ........  ............  .......  ...................  ..................................................
#
    76.10%      25030327  a.out    a.out                [.] std::uniform_int_distribution<int>::operator()
    10.57%       3523065  a.out    a.out                [.] main
     8.48%       2811222  a.out    a.out                [.] goFastPath
     4.82%        677681  a.out    a.out                [.] goSlowPath
```



After executing `perf record` we have the traces in `pt.data`.

之后，我们要求perf通过每100ns（--itrace = i100ns）综合一个指令样本来构建报告。

SO, the process basically is very much the same as we do with interrupts: skip 100ns of traces, see where we are in the program, skip 100ns of traces more and record the instruction again, and so on. After that we can identify the hotspots by looking at the samples distribution.



它实际上将开销从运行时转移到了分析时间。您只需收集一次PT跟踪，然后就可以进行您可能需要的各种复杂分析。 With interrupt-based sampling you need to rerun the collection every time you need to modify the parameters.

使用PT进行配置存在一个有趣的限制。使用基于中断的分析，您可以对不同的事件进行采样，例如缓存未命中。使用PT只能合成指令样本，因为有关缓存未命中的信息未在跟踪中进行编码。





For example, to limit collecting traces for only `goSlowPath` function you can use:

```
$ perf record -e intel_pt//u --filter 'filter goSlowPath @ a.out' ./a.out
$ perf script ...
```

To analyze traces only during particular time frame you can use:

```
$ perf record -e intel_pt//u ./a.out
$ perf script --ns --itrace=i1t -F +srcline,+srccode --time 253.555413140,253.555413520 > time_range.dump
```









## 其它

### Linux的Time命令

linux下time命令可以获取到一个程序的执行时间，包括程序的实际运行时间(real time)，以及程序运行在用户态的时间(user time)和内核态的时间(sys time)。

它的使用方法和前面讲过的strace类似，在待执行的命令前加上time即可。

来看一个例子程序test.c

```
#include <stdio.h>
int main()
{
FILE *fp = fopen("/tmp/testfile","w");
int i=0;
for(i=0;i<3;++i)
{
fprintf(fp,"%d\n",i);
}
fclose(fp);
return 0;
}
```

编译后用time命令来统计它的执行时间：

```
[leconte@localhost test]$ time ./test
real    0m0.020s
user    0m0.000s
sys     0m0.018s
```

结果表明，程序实际运行时间0.020s，用户态运行时间接近0s，内核态运行时间0.018s。这是因为我们主要操作是使用文件相关的系统调用，程序大部分时间工作在内核态。

需要注意的是，real并不等于user+sys的总和。real代表的是程序从开始到结束的全部时间，即使程序不占CPU也统计时间。而user+sys是程序占用CPU的总时间，因此real总是大于或者等于user+sys的。