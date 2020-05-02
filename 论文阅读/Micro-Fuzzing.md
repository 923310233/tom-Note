150亿个运行Java的设备，其中许多连接到Internet。发现这些设备面临的任何未知的安全威胁仍然是一项重要任务。

Contemporary当代 fuzz testing techniques focus on identifying memory corruption vulnerabilities that allow adversaries to achieve either remote code execution or information disclosure. （应该就是crash的）

Meanwhile, Algorithmic Complexity (AC) vulnerabilities, which are a common attack vector for denial-of- service attacks, remain an understudied threat.



In this paper, we present HotFuzz, a framework for automatically discovering AC vulnerabilities in Java libraries. 



## I. INTRODUCTION

算法复杂度（AC）漏洞就是这样一种形式，where a small adversarial **input induces worst-case behavior in the processing of that input**, resulting in a **denial of service**.

尽管 these particular vulnerabilities involved unintended **CPU time complexity**, AC vulnerabilities can also manifest in the spatial domain for resources such as **memory, storage, or network bandwidth**.



模糊测试（在测试过程中，模糊器将随机输入馈送到被测程序，直到该程序崩溃或超时）

Recent work has adapted existing state-of-the-art fuzz testers such as AFL [64] and libFuzzer [7] to automatically **slow down programs** with known performance problems. These approaches include **favoring inputs** that **maximize** the length of an input’s **execution** in a program’s Control Flow Graph (CFG) 



现代的模糊器可以自动降低程序的速度，例如排序例程，哈希表操作



在测试中的程序上实现高代码覆盖率是一项众所周知的艰巨任务，



Observe that注意 this approach is analogous to类似于 micro-execution , which execute arbitrary任意的 machine code by using a virtual machine

micro-fuzzing constructs test harnesses represented as function inputs, directly **invokes functions** on those inputs, and **measures the amount of resources each input consumes** using model specific registers available on the host machine. 

这就减少了手动定义测试用例的需要，supports **fuzzing whole programs** and libraries by considering **every function within them** as a possible entry- point.

一旦观察到的运行时超过了配置的阈值，我们就终止，并将该函数highlight 为 易受攻击的。



JVM’s support for introspection**内省 allows HotFuzz to automatically generate test harnesses**, represented as **valid Java objects**, for individual methods dynamically at runtime. 



HotFuzz leverages利用 the EyeVM, an instrumented JVM that provides run-time measurements at method-level granularity. 方法级别的粒度 



## 补充： JAVA的内省Introspector

```
package com.peidasoft.Introspector;
 
public class UserInfo {
   
  private long userId;
  private String userName;
  private int age;
  private String emailAddress;
   
  public long getUserId() {
    return userId;
  }
  public void setUserId(long userId) {
    this.userId = userId;
  }
  public String getUserName() {
    return userName;
  }
  public void setUserName(String userName) {
    this.userName = userName;
  }
  public int getAge() {
    return age;
  }
  public void setAge(int age) {
    this.age = age;
  }
  public String getEmailAddress() {
    return emailAddress;
  }
  public void setEmailAddress(String emailAddress) {
    this.emailAddress = emailAddress;
  }
}
```

在类UserInfo中有属性 userName, 那我们可以通过 getUserName,setUserName来得到其值或者设置新的值。通过 getUserName/setUserName来访问 userName属性，这就是默认的规则。

**Java JDK中提供了一套 API 用来访问某个属性的 getter/setter 方法**，这就是内省。

```
 BeanInfo beanInfo=Introspector.getBeanInfo(UserInfo.class);
```



## II. BACKGROUND AND THREAT MODEL

HotFuzz does not require an analyst to manually define a test harness in order to **fuzz individual methods** contained in a library.

one can fuzz an individual method with AFL by defining a test harness that transforms a **bitmap** read from stdin into **function inputs**



Understanding what techniques work best to slow down code is neces- sary to understand how to design a fuzzer to detect AC vulnerabilities.

PerfFuzz went a step further更进一步 and showed how **incorporating合并** a performance map that **tracks the most visited edges** in a program’s CFG can help a fuzzer further slow down programs.



但是 这些工具都缺少三个重要的特性来检测未知的AC漏洞。

1. they require manually defined test harnesses in order to fuzz individual functions.

2. these fuzzing engines only consider flat bitmaps as input to the programs under test, and miss the opportunity to evolve the high level classes of the function’s domain in the fuzzer’s genetic algorithm.

3. not provide a method for sanitizing method execution for AC vulnerabilities and presenting these results to a human analyst.

   

## III. HOTFUZZ OVERVIEW

HotFuzz adopts a dynamic testing approach to detecting AC vulnerabilities, where the testing procedure consists of two phases:

1. micro-fuzzing
2. witness synthesis综合 and validation. 

In the first phase, a Java library under test is submitted for *micro-fuzzing

In this process, **the library is decomposed分解 into individual methods**, where each method is considered a distinct entrypoint for testing by a μ*Fuzz* instance. 

As opposed to traditional fuzzing, **where the goal is to provide inputs that crash a program under test**, here each μFuzz instance attempts to **maximize the resource consumption** of individual methods under test using genetic optimization遗传优化 over the method’s inputs.

To that end, seed inputs for each method under test are generated using one of two instantiation strategies: *Identity Value Instantiation* (IVI) and *Small Recursive Instantiation* (SRI).

**Method-level resource consumption when executed on these inputs is measured using a specially-instrumented Java virtual machine we call the EyeVM**. 



如果优化最终产生的执行被测量为超过预定义的阈值，则该测试用例将转发到测试过程的第二阶段 second phase。



Differences between the micro-fuzzing and realistic execution environments can lead to false positives. micro-fuzzing 和 实际执行直接的差异可能会导致误报

第二阶段的目的是验证在micro-fuzzing期间发现的测试用例是否在实际的Java运行时环境中执行时是否表示实际的漏洞，从而减少最终结果中的误报次数。

This validation is achieved through *witness synthesis* where, for each test case discovered by the first phase, **a program is generated that invokes the method** under test with the associated inputs that produce abnormal resource usage. 

### 3.2 micro-fuzzing

Micro-fuzzing represents a drastically different approach to vulnerability detection than traditional automated whole- program fuzzing. 

In the latter case, **inputs are generated for an entire program** either randomly, through mutation of seed inputs, or incorporating feedback from introspection on execution. 

**whole-program fuzzing also has the well-known drawback that full coverage of the test artifact is difficult to achieve**. 因此，衡量传统模糊器功效的一项重要指标是其有效覆盖测试工件中路径的能力。



micro-fuzzing constructs realistic intermediate program states, **defined as Java objects**, and directly executes individual methods on these states. 

Thus, we can cover all methods by simply **enumerating枚举 all the methods that comprise a test artifact**, while the difficulty lies instead in ensuring that constructed states used as method inputs are feasible in practice.

In our problem setting, where we aim to preemptively warn developers against insecure usage of AC-vulnerable methods or conservatively defend against powerful adversaries, we believe micro-fuzzing represents an interesting and useful point in the design space that complements whole program fuzzing approaches. In this work, we consider the program’s state as the inputs given to the methods we micro-fuzz. Modeling implicit parameters, such as files, static variables, or environment variables are outside the scope of this work.

