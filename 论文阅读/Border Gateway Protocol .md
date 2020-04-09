## What Is BGP Hijacking?

BGP hijacking is when attackers maliciously reroute Internet traffic. **Attackers accomplish this by falsely announcing ownership of groups of IP addresses, called IP prefixes**, that they do not actually own, control, or route to.  BGP劫持非常类似于有人要改变一段高速公路上的所有标志并将汽车交通重新定向到错误的出口。

BGP是基于互连网络在说出它们拥有的IP地址的事实而建立的，所以BGP劫持几乎是不可能停止的

Attackers need to control or compromise a BGP-enabled router that bridges between one autonomous system (AS) and another, so not just anyone can carry out a BGP hijack.



粗略地说，如果DNS是Internet的通讯簿，则BGP是Internet的路线图。

Each BGP router **stores a routing table with the best routes between autonomous systems**. These are updated almost continually as each AS – **often an Internet service provider (ISP)** – broadcasts new IP prefixes that they own. BGP always **favors the shortest and most direct path from AS to AS** in order to reach IP addresses via the fewest possible hops across networks.

#### Definition of an autonomous system (AS)

An autonomous system is a large network or group of networks managed by a single organization. An AS may have many subnetworks, but all share the same routing policy. Usually an AS is either an ISP or a very large organization with its own network and multiple upstream connections from that network to ISPs (this is called a 'multihomed network'). Each AS is assigned its own Autonomous System Number, or ASN, to identify them easily.

通常，AS是ISP或非常大型的组织，具有自己的网络，and multiple upstream connections from that network to ISPs 。每个AS都分配有其自己的自治系统编号（ASN），以轻松识别它们。

#### Why is BGP important?

Internet由相互连接的多个大型网络组成。由于它是分散式的，因此没有理事机构或交通警察为数据包到达其预期IP地址目的地的最佳路线定下路线。 BGP担当此角色。如果不使用BGP，则由于路由效率低下，Web流量可能会花费大量时间到达其目的地，或者根本不会到达预期的目的地。

#### How can BGP be hijacked?

When an AS announces a route to IP prefixes that it does not actually control, this announcement, if not filtered, can spread and be added to routing tables in BGP routers across the Internet. From then until somebody notices and corrects the routes, traffic to those IPs will be routed to that AS.



BGP always favors the shortest, most specific path to the desired IP address. 

BGP始终倾向于使用最短的路径到达所需的IP地址。为了使BGP劫持成功，路由通告必须：

1) Offer a more specific route by announcing a smaller range of IP addresses **than other ASes** had previously announced.

2) Offer a shorter route to certain blocks of IP addresses. 



目前全球范围内有80,000多个自治系统，因此有些系统是不可信任的也就不足为奇了。此外，BGP劫持并不总是很明显或很容易检测到。



# PHAS: A Prefix Hijack Alert System 

The Internet relies on the Border Gateway Protocol (BGP) to convey routing information. However, if BGP provides incorrect routing information, packets may never reach the intended destination, and may even be misdirected to malicious destinations. T

 In a common prefix hijacking event, an Autonomous System (AS) originates a route for an address space, termed as prefix, but does not provide data delivery for that prefix.



由于AS的大小和影响，可能会较快地检测到AS错误地劫持了数千个前缀的大规模事件。例如，在上述的AS 9121事件中，来自不同来源的成千上万个前缀突然更改为来源AS 9121，这是前缀劫持的明确指示。但是更小的规模错误和故意攻击可能更难检测。例如，假设恶意AS仅向一条前缀131.179.0.0/16（UCLA）发出错误路径。