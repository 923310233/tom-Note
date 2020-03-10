## 从使用sk_buff发一个UDP包开始

### 使用LKM:Loadable Kernel Module

可加载内核模块（Loadable Kernel Module，LKM），是一段运行在内核空间的代码，可以访问操作系统最核心的部分。

LKM是Linux内核为了扩展其功能所使用的可加载内核模块。

LKM的优点是动态加载，在不重编译内核和重启系统的条件下对类Unix系统的系统内核进行修改和扩展。否则的话，对Kernel代码的任何修改，都需要重新编译Kernel，大大浪费了时间和效率。

基于此特性，LKM常被用作特殊设备的驱动程序（或文件系统），如声卡的驱动程序等等.

**其实就是Linux中的insmod, 把自己写的ko文件挂载上去**



所有的LKM包含两个最基本的函数：

```text
int init_module(void) /*用于初始化所有成员*/ 
{ 
... 
} 
void cleanup_module(void) /*用于退出清理*/ 
{ 
... 
} 
```



编译的话，LKM也和应用层代码使用的gcc或者g++不同，Kernel Module使用Makefile，kbuild。

典型的LKM的Makefile如下所示：

```text
obj-m += hello-world.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```



```
加载内核模块：insmod
卸载内核模块：rmmod
查看内核模块：lsmod
```

