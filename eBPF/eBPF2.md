eBPF Verifier

before executing your code, the kernel will actually verify that your code is safe by restricting you from reading memory outside a given range, stopping any jumps either outside well defined space, as well as any jump backwards, and, if the flags are set properly, it will also prohibit any pointer arithmetic.

*The eBPF in-kernel verifier*, but the gist of this is that it's extremely cautious, and with the particularly large fallout that you cannot create any loop in your code (lest it becomes an infinite loop and crashes your kernel). 

因为C文件中的code被编译后要注入到内核去,所以eBPF Verifier会做详细的检查. 代码中不能出现循环





## nodejs USDT

必须要确保Node.js has built-in USDT (user statically-defined tracing) probes for performance analysis and debugging



先装一个Dtrace

```
sudo apt-get install systemtap-sdt-dev 
```

直接从官网下载下来的nodejs好像不能用

手动 built Node.js :

http://nodejs.org/dist/v8.15.0/

```
 ./configure --with-dtrace
  make -j8
```

make -j8 因为我是8核CPU

 最重要的是必须加上--with-dtrace, 让他支持Dtrace, 即为用户态的tracepoint





## 

