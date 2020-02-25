## API Attacks

## 21 Network Attack and Defense

```
认为可以使用加密技术解决问题的人，不了解他的问题，也不了​​解加密技术。
——Roger Needham
```
Google hacking: the bad guys use search engines to find web servers that are running vulnerable applications.

他们以不到一美元的价格将残余感染的机器卖给了一个僵尸网络牧民，后者经营着一个大型的受感染机器网络，并出租给垃圾邮件制造者，网络钓鱼者和敲诈者。

### 21.2 Vulnerabilities in Network Protocols
LAN 局域网（local area network）

 IPv6使用128位地址。大多数现代工具包都准备好使用IPv6



Internet的核心路由协议是**边界网关协议Border Gateway Protocol (BGP)**,
互联网由大量的自治系统（AS）组成，例如ISP，电信公司和大型公司. 每个自治系统控制一系列IP地址。

本地网络通常使用以太网ethernet，其中设备具有唯一的以太网地址（也称为MAC地址），这些地址使用地址解析协议（ARP）映射到IP地址。由于IP地址的日益短缺，大多数组织和ISP现在都使用动态主机配置协议（DHCP）根据需要将IP地址分配给计算机，并确保每个IP地址都是唯一的。因此，如果您想追踪一台做坏事的机器，通常将必须获取将MAC地址映射到IP地址的日志。

#### 21.2.1 Attacks on Local Networks
Suppose the attacker controls one of your PCs. There are several possibilities open to him.


1. 他可以安装数据包嗅探器软件以获取密码，获取root密码，从而接管合适的帐户。如果您使用challenge-response password generators或ssh之类的协议来确保明文密码不会通过LAN，则可以阻止密码嗅探攻击。


2. 另一种方法是伪装成目标用户（例如sysadmin）已经登录的计算机。通常，攻击者只需将其MAC地址和IP地址设置为目标服务器的MAC地址和IP地址即可。

3. 有大量技术地址劫持攻击，可以很好地应对老式LAN。例子有ARP欺骗，还有bogus DHCP messages

4. 如果目标公司使用Linux或Unix服务器，则他们很可能会使用Sun的网络文件系统（NFS）进行文件共享。This allows workstations to use a network disk drive as if it were a local disk, and has a number of well-known vulnerabilities to attackers on the same LAN.


这就提出了一个更广泛的问题：网络边界在哪里？在过去，许多公司只有一个内部网络，该网络通过某种类型的防火墙连接到Internet。但是，如我在第9章中讨论的那样，分区通常是有意义的：**每个部门使用单独的网络可以限制受损机器可能造成的损害**。
近来，移动性和虚拟网络使清晰的网络边界的定义变得更加困难。


有时，人们会在公共场所（例如机场）中发现恶意部署的WiFi接入点。如果您使用它，他将能够嗅探您输入的任何明文密码，例如到您的Webmail或Amazon帐户；


#### 21.2.2.5 DNS Security and Pharming
到目前为止，我已经给出了两个攻击示例，这些攻击都是通过**将用户定向到恶意DNS服务器来欺骗的**，结果是，当他尝试访问其银行网站时，他最终将密码输入了一个伪造的密码。这通常称为Pharming。
我提到过drive-by骗术，**即通过下载的网页中的恶意JavaScript重新配置人们的家庭路由器**，以及流氓访问点，攻击者在流氓访问点中为受害者提供WiFi服务，然后完全控制其所有不受保护的流量。


DNS污染
DNS污染是指一些刻意制造或无意中制造出来的域名服务器数据包，把域名指往不正确的IP地址。一般来说，在互联网上都有可信赖的网域服务器，但为减低网络上的流量压力，**一般的域名服务器都会把从上游的域名服务器获得的解析记录暂存起来**，待下次有其他机器要求解析域名时，可以立即提供服务。

对于运营商，网络管理员等人，尤其是在办公室等地方，管理员希望网络使用者无法浏览某些与工作无关的网站，通常采用DNS抢答机制。机器查询DNS时，采用的是UDP协议进行通讯，队列的每个查询有一个id进行标识。
```
192.168.2.2    8.8.8.8    Standard query 0x0003 AAAA facebook.com
8.8.8.8    192.168.2.2    Standard query response 0x0003 A 93.46.8.89
8.8.8.8    192.168.2.2    Standard query response 0x0002 A 31.13.90.2
8.8.8.8    192.168.2.2    Standard query response 0x0003 AAAA 2a03:2880:f01a:1:face:b00c:0:1
```
我们假设A为用户端，B为DNS服务器，C为A到B链路的一个节点的网络设备（路由器，交换机，网关等等）。然后我们来模拟一次被污染的DNS请求过程。
A向B构建UDP连接，然后，A向B发送查询请求，查询请求内容通常是：“A baidu.com”，这一个数据包经过节点设备C继续前往DNS服务器B；然而在这个过程中，**C通过对数据包进行特征分析（远程通讯端口为DNS服务器端口，激发内容关键字检查，检查特定的域名如上述的“baidu.com",以及查询的记录类型"A记录"），从而立刻返回一个错误的解析结果**（如返回了"A 123.123.123.123"），众所周知，作为链路上的一个节点，**C机器的这个结果必定会先于真正的域名服务器的返回结果到达用户机器A**，而我们的DNS解析机制有一个重要的原则，就是只认第一，因此C节点所返回的查询结果就被A机器当作了最终返回结果，用于构建链接。


**对于DNS污染，一般除了使用代理服务器和VPN之类的软件之外，并没有什么其它办法**。但是利用我们对DNS污染的了解，还是可以做到不用代理服务器和VPN之类的软件就能解决DNS污染的问题，从而在不使用代理服务器或VPN的情况下访问原本访问不了的一些网站。当然这无法解决所有问题，当一些无法访问的网站本身并不是由DNS污染问题导致的时候，还是需要使用代理服务器或VPN才能访问的。



DNS抢答
所谓DNS抢答，顾名思义，就是在用户查询的DNS服务器还没有回应的情况下，由别的服务器提前将结果发给用户，一般这个解析结果是错误的。
比如用户设置的是8.8.8.8 Google的DNS服务器。正常上网过程中，用户发起的DNS请求，都要发送到8.8.8.8来解析。但是，如果中间有一些特殊的设备，监测到了这些DNS请求数据包，然后提前将一个错误的结果发给用户，一般用户电脑都会已第一个收到的应答包的内容为准。

要破解DNS抢答，就不能让中间设备监测到DNS查询，现在有个技术，叫DNS OVER HTTPS，将DNS查询封装在HTTPS的加密通道中，就可以解决这个问题。


## 21.3 木马，病毒，蠕虫和Rootkit
**增长最快的问题是rootkit**，这是一种软件，一旦安装在计算机上，就会秘密地将其置于远程控制之下。 Rootkit可以用于有针对性的攻击（执法机构使用它们将嫌疑人的笔记本电脑变成侦听设备）或用于财务欺诈（它们可能与捕获密码的键盘记录器一起提供）。如今，rootkit的最显着特征之一就是隐身性。他们试图从操作系统中隐藏起来，以便无法使用标准工具来定位和删除它们。
一方面，大多数以这种方式感染的PC最终会成为僵尸网络。


### 21.3.2 How Viruses and Worms Work
malware 恶意软件
worms 蠕虫
trojons 木马

![1.png](./images/1.png)


