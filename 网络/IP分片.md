## 深入理解TCP/IP协议的实现之ip分片重组

我们都知道数据链路层有mtu的限制，如果我们上层发的包太大，那就要分片，那么对端就需要重组分片，组装好再通知上层。



ipq结构体是代表一个完整的传输层包，他被ip层分成了多个分片。ipfrag结构体是代表一个ip分片。他是传输层包的一个部分。



我们开始分析组片之前，先看一下一些基础函数。
1 创建一个用于重组传输层数据包的结构体ipq。这个是第一个ip分片到达时调用的。**他维护了属于同一个分片组（同一个传输层数据包）的多个分片**：

```
/ 创建一个队列用于重组分片，iphdr 即ip报文头
static struct ipq *ip_create(struct sk_buff *skb, struct iphdr *iph, struct device *dev)
{
    struct ipq *qp;
    int maclen;
    int ihlen;
    // 申请一个新ipq的表示分片队列
    qp = (struct ipq *) kmalloc(sizeof(struct ipq), GFP_ATOMIC);
    // 初始化内存
    memset(qp, 0, sizeof(struct ipq));

    // skb->data指向mac头首地址，mac头长度等于ip头减去mac头首地址
    maclen = ((unsigned long) iph) - ((unsigned long) skb->data);
    // 分配内存保存mac头
    qp->mac = (unsigned char *) kmalloc(maclen, GFP_ATOMIC);

    // ip头长度由ip头字段*4得出，多分配8个字节给icmp
    ihlen = (iph->ihl * sizeof(unsigned long));
    qp->iph = (struct iphdr *) kmalloc(ihlen + 8, GFP_ATOMIC);

    // 把mac头内容复制到mac字段
    memcpy(qp->mac, skb->data, maclen);
    // 把ip头和传输层的8个字节复制到iph字段，8个字段的内容用于发送icmp报文时
    memcpy(qp->iph, iph, ihlen + 8);

    // 未分片的ip报文的总长度，未知，收到所有分片后重新赋值
    qp->len = 0;
    // 当前分片的ip头和mac头长度
    qp->ihlen = ihlen;
    qp->maclen = maclen;
    qp->fragments = NULL;
    // 关联的设备
    qp->dev = dev;
    // 开始计时，一定时间内还没收到所有分片则重组失败，发送icmp报文
    qp->timer.expires = IP_FRAG_TIME;       
    qp->timer.data = (unsigned long) qp;        
    qp->timer.function = ip_expire;     
    // 开始计时，如果超时没有收到新的ip分片，则重组失败，如果收到一个新的，重置计时器
    add_timer(&qp->timer);

    qp->prev = NULL;
    cli();
    // 头插法插入分片重组的队列
    qp->next = ipqueue;
    /*
        如果当前新增的节点不是第一个节点则把当前第一个
        节点的prev指针指向新增的节点
    */
    if (qp->next != NULL)
        qp->next->prev = qp;
    //更新ipqueue指向新增的节点，新增节点是首节点 
    ipqueue = qp;

    sti();
    return(qp);
}
```

上面的代码不难理解。主要是申请一个ipq结构体。然后初始化各个字段，最后插入ipq队列。并且开始计算分片重组的超时时间和超时回调。



2 通过ip头查找对应的ipq队列。

```
static struct ipq *ip_find(struct iphdr *iph)
{
    struct ipq *qp;
    struct ipq *qplast;

    cli();
    qplast = NULL;
    // ipqueue类似一个全局头指针
    for(qp = ipqueue; qp != NULL; qplast = qp, qp = qp->next)
    {   // 对比ip头里的几个字段，id是标记报文属于同一个传输层原始包的
        if (iph->id== qp->iph->id && 
            iph->saddr == qp->iph->saddr &&
            iph->daddr == qp->iph->daddr && 
            iph->protocol == qp->iph->protocol
        )
        {   // 找到后重置计时器，重新计算重组的超时时间，即每次新分片到来，重新计算超时时间
            del_timer(&qp->timer);  
            sti();
            return(qp);
        }
    }
    sti();
    return(NULL);
}
```

3 创建一个表示单个ip分片的结构体ipfrag 。他表示其中一个分片。

```
// 创建一个表示ip分片的结构体
static struct ipfrag *ip_frag_create(int offset, int end, struct sk_buff *skb, unsigned char *ptr)
{
    struct ipfrag *fp;

    fp = (struct ipfrag *) kmalloc(sizeof(struct ipfrag), GFP_ATOMIC);
    memset(fp, 0, sizeof(struct ipfrag));
    // ip分配的首字节在未分片数据（分片前的数据包）中的偏移
    fp->offset = offset; 
    // 最后一个字节的偏移 + 1，即下一个分片的首字节偏移
    fp->end = end; 
    // 分片长度，即数据长度
    fp->len = end - offset; 
    fp->skb = skb;
    // 指向分片的数据首地址
    fp->ptr = ptr; 
    return(fp);
}
```

4 判断分片是否已经全部到达

```
/ 判断分片是否全部到达
static int ip_done(struct ipq *qp)
{
    struct ipfrag *fp;
    int offset;
    /*
        收到最后分片的时候才会更新len字段，如果没有收到他就是初始化0，
        所以为0说明最后一个分片还没到达，直接返回未完成
    */
    if (qp->len == 0)
        return(0);
    // 走到这里说明全部分片已经到达,fragments指向所有分片队列的首节点
    fp = qp->fragments;
    offset = 0;
    /*
     检查所有分片，每个分片按照偏移从小到大(从链表头到尾)排序的链表
    ，因为每次分片节点到达时会插入相应的位置
    */
    while (fp != NULL)
    {   /*
            如果当前节点的偏移大于期待的偏移(即上一个节点的最后一个
            字节的偏移+1，由end字段表示)，说明有中间节点没到达，直
            接返回未完成
        */
        if (fp->offset > offset)
            return(0);  
        // 下一个期待的偏移
        offset = fp->end;
        fp = fp->next;
    }

    // 分片全部到达并且每个分片的字节连续则重组完成
    return(1);
}
```

5 重组同一队列里的所有ip分片。

重组的大致流程就是申请一块新内存，然后把mac头、ip头复制过去。再遍历分片队列，把每个分片的数据拼起来。最后更新一些字段。





我们熟悉了ip分片中需要用到的各种基本操作函数。接下来我们从整体来看看，整个组包的过程：

首先做一些判断，包括
1 根据ip头找是否已经存在分片队列
2 取得分片的真实偏移（即在没有分片的数据中的偏移）、分片标记。

如果是第一个分片到达。则需要创建一个队列去维护陆续到来的分片。如果不是第一个分片，则重置计时器。



接着看下面代码
1 取得当前ip分片的ip头大小。
2 tot_len是ip报文的长度。减去ip头长度，得到分片中的数据长度。加上偏移得到末位置。
3 skb->data是mac头首地址，加上mac头长度得到ip头首地址
4 判断是不是最后一个分片了，这个根据ip头的标记可以得到。如果是，记录整个ip原始包的总长度。即由最后一个分片的数据计算出来。

```
    // ip头长度
    ihl = (iph->ihl * sizeof(unsigned long));
    // 偏移+数据部分长度等于end，end的值是最后一个字节+1
    end = offset + ntohs(iph->tot_len) - ihl;
    // data指向整个报文首地址，即mac头首地址，ptr指向ip报文的数据部分
    ptr = skb->data + dev->hard_header_len + ihl;
    /*
        是否是最后一个分片，是的话，未分片的ip报文长度为end，
        即最后一个报文的最后一个字节的偏移+1，因为偏移从0算起
    */
    if ((flags & IP_MF) == 0)
        qp->len = end;
```

找到当前分片的位置，从队列的头到尾按偏移从小到大排序。prev指针指向待插入分片的 前一个分片。

处理分片重叠问题。

```
    /*
        处理当前节点和前面节点的重叠问题，
        1 prev 为空，说明当前就一个分片，不存在重叠情况
        2 prev非空，保证了offset >= prev->offset，
        所以只需要比较当前节点的偏移offset和prev->end字段
    */
    if (prev != NULL && offset < prev->end)
    {   
        /*
            说明存在重叠，算出重叠的大小，把当前节点的重叠部分丢弃，
            更新offset和ptr指针往前走,没处理完全重叠的情况
        */
        // 重叠的数据大小
        i = prev->end - offset;
        // 把当前分片的数据丢掉i个字节。指针也向后移动i个字节。
        offset += i;    
        ptr += i;   
    }
```