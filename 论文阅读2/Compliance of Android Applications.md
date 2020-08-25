# Cardpliance: PCI DSS Compliance of Android Applications

要求用户输入信用卡号码的应用在很大程度上被之前的研究所忽视，这些研究经常报告在一般的移动应用生态系统中普遍存在安全和隐私问题。Such applications are particularly security-sensitive, and they are subject to the Payment Card Industry Data Security Standard (PCI DSS). 

在本文中，我们设计了一个名为Cardpliance的工具，which bridges the semantics of the graphical user interface with static program analysis to capture relevant requirements from PCI DSS.

我们使用Cardpliance研究了358个来自Google Play的流行应用，这些应用要求用户输入信用卡号码。总的来说，我们发现这358个应用程序中，有1.67%的应用程序不符合PCI DSS的要求，存在的漏洞包括不正确地存储信用卡号码和卡验证码。



## 1 Introduction

最近的工作报告称，在随机抽样的50162个Google Play应用中，有4433个应用要求用户通过应用UI中的文本字段输入信用卡信息。出现这种情况的原因有很多。例如，如果用户不想使用谷歌或苹果的支付系统，应用开发者可能希望提供一个替代方案。或者，应用开发者可能希望避免从Google和苹果公司收取管理费用[37, 38]。不管原因是什么，事实是：应用程序要求用户输入信用卡信息。

PCI DSS[6]金融行业标准要求软件系统以特定方式保护支付信息。



Our work is motivated by the research question: do mobile applications mishandle payment information？

Answering this question introduces several technical research challenges.

 First, which PCI DSS requirements apply to mobile applications? PCI DSS v3.2.1 (May 2018) is 139 pages and applies to a broad variety of payment systems. 

Second, how can those requirements be translated into static program analysis tasks?



In this paper, we design a static program analysis tool called *Cardpliance* that captures key requirements from PCI DSS that are applicable to mobile applications. 

我们使用Cardpliance来研究一组17,500个流行的免费应用程序，这些应用程序是在Google Play的所有类别中选择的。Using the UI semantic inference of UiRef [8], Cardpliance re- duces this sample to 358 applications known to ask for credit card information from the user.  然后，Cardpliance识别出40个可能违反PCI DSS的应用程序。经过人工反编译和源代码审查，我们确认了6个不合规的应用程序。

然而，我们发现有6个应用程序在Google Play上总共有近150万次下载，违反了PCI DSS的要求，它们以明文方式存储或记录信用卡号码（5/6），persisting credit card verification codes （3/6），以及在显示时不屏蔽信用卡号码（2/6）。



## 2 PCI Data Security Standard

We found that PCI DSS distinguishes between cardholder data (CHD) and sensitive account data (SAD), which impacts software processing, as shown in Table 1.



要求1（限制CHD存储和保留时间）。
PCI DSS第3.1条规定："将持卡人数据的存储和保留时间限制在业务、法律和/或监管目的所需的范围内。
将持卡人数据的存储和保留时间限制在业务、法律和/或监管目的所需的范围内，如数据保留政策所规定。至少每季度清理一次不必要的存储数据。
因此，移动应用程序应尽量减少将信用卡号码和其他CHD值写入持久存储的情况。理想情况下，CHD永远不会被写入，但如果被写入，应用程序需要一种方法来删除它。CHD也永远不应该被写入共享存储位置，例如Android中的SD卡，因为它可能会被其他应用程序读取。



 要求2（限制SAD存储）。PCI DSS第3.2条规定：
不要在自动识别后存储敏感的认证数据（即使是加密的）。如果收到敏感的认证数据，在授权过程完成后，使所有数据无法恢复。

Therefore, SAD values such as full track data (magnetic- stripe data or equivalent on a chip), card security codes (e.g., CAV2/CVC2/CVV2/CID), PINs and PIN blocks should never **be written to persistent storage**, even if it is encrypted or in a location only accessible to the application.



要求3（显示时屏蔽PAN）。PCI DSS第3.3条规定：
当显示PAN时，要对PAN进行屏蔽（前六位和后四位是你可以显示的最大数字），这样只有具有合法业务需求的授权人员才能看到超过前六位/后四位的PAN。

该标准警告说，在电脑屏幕、移动用户界面、支付卡收据、传真或纸质报告上显示完整的PAN可能会帮助未经授权的个人进行不受欢迎的活动。因此，在用户输入信用卡号码后，应用程序应在显示前对其进行屏蔽（例如，在随后的用户界面屏幕上）。





要求4（存储时保护PAN）。PCI DSS Section 3.4规定："在任何地方都不能读取PAN。
使PAN在存储的任何地方都无法读取--包括便携式数字媒体、备份媒体、日志以及从无线网络接收或存储的数据。 Technology solutions for this requirement may include strong one-way hash functions of the entire PAN, truncation截断, index tokens with securely stored pads, or strong cryptography.
本要求是对要求1的补充，特别是对信用卡号码（PAN）的限制。如果要写，则需要某种保护。



要求5（使用安全通信）。PCI DSS第4.1条规定："使用强大的加密技术和安全协议，在开放的公共网络(如因特网、无线技术、手机技术)上传输敏感的持卡人数据时，安全地保护这些数据。

从移动应用的角度来看，所有的网络连接都应该使用TLS/SSL。此外，应用程序不应取消服务器认证检查，之前的工作[17]已经发现这是移动应用程序中常见的漏洞。





## 3 Overview

Cardpliance使用一系列定制的静态程序分析测试来解决这些挑战。在可能的情况下，我们利用现有的开源项目，这些项目体现了十年来移动应用分析的知识优势。具体来说，我们利用UiRef[8]来推断文本输入的语义，利用Amandroid[19]（也称为Argus-SAF）来进行静态数据流分析。我们的分析还利用MalloDroid[17]的概念来识别SSL漏洞，以及StringDroid[39]的概念来识别用于网络连接的URL字符串。结合这些现有的技术-技术来创建特定的PCI DSS检查，需要精心构建，并代表了一个独特的贡献。





Figure 1 provides a high-level overview of Cardpliance’s approach to identifying PCI DSS violations in mobile applications.

第一步是识别哪些应用程序要求用户输入信用卡信息。虽然我们建立在UiRef的基础上进行用户界面分析，the analysis requires injecting a code executing the repackaged application. 。这个过程对于应用发现来说太过沉重。

因此，我们采用了两阶段的应用过滤，, first using a lightweight keyword-based search of the strings used by the application, ，然后使用UiRef来确认应用是否真的要求用户输入信用卡信息（例如，这些术语可能在其他上下文中使用）。



The next phase is the Data Dependence Graph (DDG) extraction. Amandroid的一个关键特征是生成图形，在此基础上可以执行不同的静态分析任务。 This approach encapsulates traditional static program analysis within the core Amandroid tool and allows users of Amandroid to focus on their goals as graph traversal algorithms. 

 First, we use information from UiRef to annotate UI input widgets as being related to credit card information. Second, we enhance how Amandroid handles OnClickListener callbacks to correctly track data flows from UI input.







### 4.1 DDG Extraction

The Cardpliance tests are graph queries on Amandroid’s Data Dependence Graph (DDG). 

Amandroid performs flow- and context-sensitive static program analysis on .apk files.

 It analyzes each Android component (e.g., Activity component) separately and then combines the per-component analysis to handle inter-component communication (ICC). 



Amandroid is primarily focused on data flow analysis. It calculates points-to information for each instruction in the control flow graph, storing it in a Points-to Analysis Results (PTAResult) hash map. 

It also keeps track of ICC invocations调用 in a summary table (ST). 

Amandroid then produces an Interprocedural Data Flow Graph (IDFG) for each component, which combines the Interprocedural Control Flow Graph (ICFG) with the PTAResult for that component. It then generates an Interprocedural Data Dependency Graph (IDDG), which contains the same nodes as the IDFG, but the edges are the dependencies between each object’s definition to its use. Finally, a DDG for the entire application is created by combining each component’s IDDG and the ST.





Amandroid uses the DDG to perform taint analysis. Given a set of taint sources and taint sinks, Amandroid marks the sources and sinks in the DDG and computes the set of all paths between them. （Amandroid使用DDG来进行污点分析。给定一组污点源和污点汇，Amandroid在DDG中标记源和汇，并计算它们之间所有路径的集合。从源到汇的路径列表存储在污点分析结果（TAR）结构中。Amandroid允许用户在配置文件中通过方法签名的文本字符串来定义源和汇。）



Cardpliance analyzes how applications handle credit card information entered by the user into text fields. Applications access this text via the TextView.getText() method. However, Cardpliance needs to determine which TextView objects correspond to the UI widgets that collect different types of credit card information. 

To acquire a TextView object, the application calls Activity.findViewById(R.id.widget_name), where R.id.widget_name is a unique integer managed by the application’s resource R class. 因此，Cardpliance使用Activity.findViewById(int)作为污点源。分析将对返回的TextView和随后的TextView.getText()的字符串进行污点。

PCI DSS测试可以使用Amandroid的ExplicitValueFinder.findExplicitLiteralForArgs()方法来确定传递给污点源的整数值。然后，它使用UiRef[8]确定的信用卡信息部件的资源ID 来确定流向每个sinks的信息类型。



### 4.2 PCI DSS Tests

在高层次上，Cardpliance使用Amandroid的污点分析结果（TAR）来识别潜在的PCI DSS违规行为。然而，TAR并不考虑源和汇的上下文，或者源和汇之间的所有不同路径。Cardpliance使用DDG根据PTAResult哈希图中的常量值来识别特定的指令作为源和汇。然后，它计算这些特定的源指令和汇指令之间的所有路径，确定是否发生特殊条件（例如，调用混淆方法）。

In contrast to *S* and *K*, *R* places *requirements* on the data flow path. Informally, *R* defines a set of methods that should be called on the data flow path (e.g., a string manipulation method that could mask characters). If no methods from *R* exist on the path, then a potential violation is raised.





每个PCI DSS测试都是根据调用三组方法的指令来定义的：源方法(S)、汇方法(K)和所需方法(R)。S和K是传统的污点分析的源和汇。而Amandroid的源和汇是方法签名，Cardpli- ance的一些源和汇是上下文敏感的。例如，调用Activity.findViewById(int)的指令，只有当参数是UiRef确定的请求信用卡信息的资源ID列表中的一个整数时才是源。
与S和K相比，R对数据流路径提出了要求。在非正式的情况下，R定义了一组应该在数据流路径上被调用的方法（例如，可以屏蔽字符的字符串操作方法）。如果路径上不存在R的方法，那么就会引发潜在的违规行为。