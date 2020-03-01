eBPF Verifier

before executing your code, the kernel will actually verify that your code is safe by restricting you from reading memory outside a given range, stopping any jumps either outside well defined space, as well as any jump backwards, and, if the flags are set properly, it will also prohibit any pointer arithmetic.

*The eBPF in-kernel verifier*, but the gist of this is that it's extremely cautious, and with the particularly large fallout that you cannot create any loop in your code (lest it becomes an infinite loop and crashes your kernel). 

因为C文件中的code被编译后要注入到内核去,所以eBPF Verifier会做详细的检查. 代码中不能出现循环



bcc提供的接口在这个文档里,可以去查

https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md

![801.png](./images/801.png)



#### What exactly am I tracing with BPF?



从广义上讲，可以将程序附加到并跟踪以下4类

- kprobes

- uprobes

- tracepoints

- USDT (User Statically-Defined Tracing)

  

  使用eBPF，不仅可以跟踪内核功能和入口/出口点，而且还可以跟踪用户空间代码。

kprobes的最大缺点是它们没有定义的接口，因此，每当出现新版本的内核时，所有kprobes的子集都可能完全改变并且变得不兼容。

为了解决这个问题，kernel developers decided to create a more formalized tracing API, which makes actual guarantees on consistency and compatibility. **They peppered these tracepoints all around the kernel**, at useful-to-trace points, like syscall entries and exits, interrupts, TCP events, and signals.

内核开发人员决定创建一个更加形式化的跟踪API，从而对一致性和兼容性做出实际保证。他们在有用的跟踪点（例如系统调用入口和出口，中断，TCP事件和信号）处将这些跟踪点**遍及整个内核**。您可以在/ sys / kernel / debug / tracing / events中找到大多数此类文件。



## hello

```
from __future__ import print_function
from bcc import BPF

prog = """
int hello(void *ctx) {
  bpf_trace_printk("Hello, World!\\n");
  return 0;
}
"""
b = BPF(text=prog)
b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hello")
print("PID MESSAGE")
try:
    b.trace_print(fmt="{1} {5}")
except KeyboardInterrupt:
    exit()
```

每次在系统调用clone发生的时候都会执行 hello();

![802.png](./images/802.png)

**If you need to define some helper function that will not be executed on a probe, they need to be defined as `static inline` in order to be inlined by the compiler. **

`b.trace_fields()`: Returns a fixed set of fields from trace_pipe. Similar to trace_print()





##

```
from __future__ import print_function
from bcc import BPF
from bcc.utils import printb

REQ_WRITE = 1		# from include/linux/blk_types.h

# load BPF program
b = BPF(text="""
#include <uapi/linux/ptrace.h>
#include <linux/blkdev.h>
BPF_HASH(start, struct request *);
void trace_start(struct pt_regs *ctx, struct request *req) {
	// stash start timestamp by request ptr
	u64 ts = bpf_ktime_get_ns();
	start.update(&req, &ts);
}
void trace_completion(struct pt_regs *ctx, struct request *req) {
	u64 *tsp, delta;
	tsp = start.lookup(&req);
	if (tsp != 0) {
		delta = bpf_ktime_get_ns() - *tsp;
		bpf_trace_printk("%d %x %d\\n", req->__data_len,
		    req->cmd_flags, delta / 1000);
		start.delete(&req);
	}
}
""")

if BPF.get_kprobe_functions(b'blk_start_request'):
        b.attach_kprobe(event="blk_start_request", fn_name="trace_start")
b.attach_kprobe(event="blk_mq_start_request", fn_name="trace_start")
b.attach_kprobe(event="blk_account_io_completion", fn_name="trace_completion")

# header
print("%-18s %-2s %-7s %8s" % ("TIME(s)", "T", "BYTES", "LAT(ms)"))

# format output
while 1:
	try:
		(task, pid, cpu, flags, ts, msg) = b.trace_fields()
		(bytes_s, bflags_s, us_s) = msg.split()

		if int(bflags_s, 16) & REQ_WRITE:
			type_s = b"W"
		elif bytes_s == "0":	# see blk_fill_rwbs() for logic
			type_s = b"M"
		else:
			type_s = b"R"
		ms = float(int(us_s, 10)) / 1000

		printb(b"%-18.9f %-2s %-7s %8.2f" % (ts, type_s, bytes_s, ms))
	except KeyboardInterrupt:
		exit()
```



```
BPF_HASH(start, struct request *);
```

This creates a hash named `start` where the key is a `struct request *`, and the value defaults to u64. This hash is used by the disksnoop.py example for saving timestamps for each I/O request, where the key is the pointer to struct request, and the value is the timestamp.



```trace_start（struct pt_regs * ctx，struct request * req）```：此函数稍后将附加到kprobes。 kprobe函数的参数是struct pt_regs * ctx（用于寄存器和BPF上下文），然后是该函数的实际参数。我们将其附加到blk_start_request（），其中第一个参数是struct request *。



```start.update（＆req，＆ts）```：我们使用指向请求结构的指针作为哈希中的键。这在跟踪中很常见。指向结构的指针非常有用，因为它们是唯一的：两个结构不能具有相同的指针地址。 

















































else

kprobe: kprobe是一种内核调试方式，可以在内核执行代码位置中断，并执行我们插入的代码**。

注意,这是Linux中的kprobe, 与上文所说的kprobe并没有关系.