# Security Analysis of Unified Payments Interface and Payment Apps in India



自2016年以来，在印度政府的大力推动下，基于智能手机的支付应用已经成为主流. Many of these apps use a common infrastructure introduced by the Indian government, called the **Unified Payments Interface** (UPI), but there has been no security analysis of this critical piece of infrastructure that supports money transfers.

本文采用原则性的方法，通过7个流行的UPI应用，对UPI协议的设计进行逆向工程，对UPI协议进行详细的安全分析。





## 1 Introduction

印度银行联盟印度国家支付公司（NPCI）推出了统一支付接口（UPI），实现了不同用户银行账户之间的免费即时转账。Currently, there are about 88 UPI payment apps and over 140 banks that enable transactions with those apps via UPI [40, 41]. 

 This paper focuses on vulnerabilities in the design of UPI and UPI’s usage by payment apps.

一个关键的挑战是，虽然印度有数以百万计的用户使用该协议，但协议细节却无法获得。我们也无法访问UPI服务器。因此，我们不得不通过使用UPI协议的UPI应用进行逆向工程



我们的威胁模型假设用户在非root的Android手机上小心翼翼地使用授权支付应用，但安装了一个具有常用权限的攻击者控制的应用。我们在UPI 1.0协议中发现了几个设计选择，导致了以下类型攻击的可能性：

- 攻击方式1：未经授权注册，给定用户手机号。未经授权的注册，给用户的手机号码。这种攻击会泄露用户的隐私数据，比如用户的银行账户和银行账户号。

- 攻击#2：给定用户的手机号和部分借记卡号，在银行账户上进行未经授权的交易。在印度使用借记卡购物，无论是在商店还是在网上，都需要用户通过输入一个秘密的PIN码来授权付款。在这种攻击中，攻击者通过知道用户的手机号码和印在卡上的借记卡信息（最后六位数字和到期日，不含PIN码），就可以在从未使用过UPI应用进行支付的用户的银行账户上进行交易。

- 攻击3：没有借记卡号的未经授权的交易。这个攻击显示了攻击者如何在不知道用户认证因素的情况下，学习所有的因素，在该用户的银行账户上进行未经授权的交易。

  

  ## 2 Background

图1b显示了UPI汇款系统与图1a中的传统网银系统的比较。

The UPI payment system requires Alice to register her primary cellphone (or cell) number with her bank account(s)  out-of-band to send or receive money. 

UPI uses the cell number 

(i) as a proxy for a user’s digital identity with the bank to look up a bank account given a cell number;

(ii) as a factor in authentication via SMS one-time passcodes (OTP); 

 (iii) to alert users on transactions.



The Government of India requires cellphone providers to get copies of government-issued IDs, manually verify the IDs, and do biometric verification 并进行生物识别验证。





To transfer money to Bob, Alice first logs into a UPI app using the passcode she set during user registration. Then, out- of-band, Alice requests Bob to provide his UPI ID, which is often Bob’s cell number. Alice chooses one of the bank accounts she previously added to the app (Figure 2, screen- shot #7), initiates the transaction to Bob, and authorizes it by providing her UPI PIN. Internally, the UPI payment interface directly transfers money from Alice’s chosen bank account to Bob’s bank account linked with his UPI ID.



### 2.2 UPI Specs for User Registration

1. Set up a UPI user profile: 一旦UPI应用获得了用户的手机号码，应用就必须从Alice的手机向UPI服务器发送一条出站加密短信。这个过程是自动进行的，不需要用户参与，以保证用户的手机和她的设备之间有很强的关联。根据UPI的说法，这是协议中 "最关键的安全要求"，因为从用户设备上进行的所有资金交易都会首先基于这种关联进行验证。UPI将用户的设备（由设备ID、应用ID和IMEI号等参数识别）与她的手机号的这种关联称为设备硬绑定。
2. UPI spec mandates that all transactions must at least be 2FA using a cell phone (the device fingerprint) as one factor and the UPI PIN as the second. 
3. 

### 2.3 Threat Model

我们假设一个正常的用户Alice，从Google Play等官方渠道安装支付应用，支付应用都不包含无关的恶意代码。爱丽丝拥有一部配置正确的带互联网设施的手机，并防止不受信任的人对其进行物理访问。
另一方面，攻击者Eve使用的是经过root的手机。Eve可以使用她所掌握的任何工具来对支付应用进行逆向工程。我们假设Eve发布了一个名为Mally的明显可用的非特权应用，它请求以下两个权限-android.permission.INTERNET和android.permission.RECEIVE_SMS。爱丽丝发现这个应用很有用，于是安装了它，并授予它必要的权限。



## 3 Security Analysis 

### 3.1 Methodology

App Reversing-Engineering. One approach to capture the protocol data sent and received by an app is to run it in a sand- box. 。沙盒工具如CuckooDroid[14]使用模拟器进行动态分析。因此，为了测试UPI应用是否能在沙盒中运行，我们在Linux主机上手动运行Android SDK内置的模拟器中的每个应用。然而，我们发现这些应用在没有物理SIM卡的情况下无法运行，而SIM卡在模拟器上是无法使用的。这些应用程序还使用了反仿真技术，使它们无法在模拟器中运行。

除了反仿真技术，我们发现这些支付应用程序还使用了其他一些防御措施。例如，所有的应用程序都会检测到手机是否被root，并阻止用户在root的手机上运行应用程序。Some apps also look for the presence of hooking libraries such as Xposed [28] that typically require root access to modify system files. 



Our security assessments show that some apps, such as BHIM, allow repackaging. We leverage this to instrument an app’s code statically to learn specifics of the authentication handshake, such as the name of the activity and method that generated network traffic. 

 To instrument the app, ，我们首先使用APKTool[4]对其进行反汇编，插入调试语句，然后用我们的签名对其进行重新打包。



One question that arises is where to instrument in an app’s code as this requires knowledge of the methods of the app we want to instrument. Since we do not know this a priori, we manually reverse-engineer the apps using the JEB [30] disassembler and decompiler. Some times, JEB fails to decompile certain classes that are control-flow obfuscated. In such cases, we use JDK’s javap command to read bytecode. We augment our analysis with results from the static components of two hybrid analyzers MobSF [21] and Drozer [26].

我们无法对某些应用进行重新包装，比如Google Pay。在这种情况下，我们使用名为mitmproxy[36]的TLS中间人代理拦截应用程序的网络流量。 We install the OpenVPN app on our Android phone and an OpenVPN service on a Linux host and configure the host’s firewall rules to route traffic to the mitmproxy. 

这个设置还需要我们在手机上安装mitmproxy的证书。但我们发现，启动Android Nougat后，Android并不信任用户安装的证书，而设置系统证书需要root权限，这是一个阻碍。因此，我们在Android Marshmallow和Lollipop设备上进行分析。





Alternate Workflow1. In the default workflow described above, BHIM sends the device registration token to the UPI server as an SMS message for device hard-binding (Step 3). In case the UPI server does not receive the SMS, thus failing to hard-bind, BHIM provides an alternate workflow for hard- binding, as shown in Figure 4a. BHIM提示Alice键入她的手机号；BHIM将键入的手机号和设备注册令牌token以HTTPS消息的形式发送给UPI服务器。UPI服务器发送一个OTP给Alice，她必须输入这个OTP才能完成设备绑定。





Alternate Workflow2. If Alice, an already registered user, changes her cell phone, then the UPI server has to re-bind her cell number with the new cell phone. At the time of device binding, the UPI server finds that an account for Alice already exists and notifies BHIM of the same (*accountExists* flag in Step 6). The UPI server prompts Alice for her passcode, and once Alice is verified (Step 7), the server sends back Alice’s bank account information that she previously added to BHIM (Step 10). This workflow makes it convenient for Alice to transfer her bank accounts to another phone, without going through the hassle of adding all her bank accounts again.





1. 潜在安全漏洞1：攻击者Eve要想接管Alice的账户，首先要克服的障碍之一是UPI的设备绑定机制，该机制将Alice的手机号码与她的手机绑定。对于Eve来说，要想打破绑定，Eve必须能够将她的手机与Al- ice的手机号进行绑定。虽然默认的工作流程使这一点变得很困难，但备用工作流程1提供了一个潜在的后备方案，允许Eve将Alice的手机号码作为HTTPS消息从Eve的手机发送。



2. 潜在的安全漏洞2：备用工作流1使用OTP验证进行设备绑定。比如说，如果Alice在手机上输入朋友Bob的手机号，UPI服务器会将OTP发送到Bob的手机上。如果Bob与Alice共享该OTP，那么Alice就可以向UPI服务器确认OTP，UPI服务器就会将Alice的手机与Bob的手机号进行硬绑定。因此，Bob将收到UPI服务器今后向Alice发送的所有短信。



#### 3.3.2 Attack #1: Unauthorized registration, given a victim’s cell number

在这次攻击中，我们展示了远程攻击者Eve如何在给定受害者手机号码的情况下，建立一个UPI账户。要想攻击成功，Eve只需要一件事：受害者的手机安装了Mally应用。

Eve on her phone has a repackaged version of BHIM that has client-side security checks disabled. 

Eve将Mally作为一个潜在的有用的应用在各个应用商店中放出，并等待不知情的用户安装Mally。

为了让攻击发生，Eve必须有办法发现受害者的手机号码。为了简化攻击描述，我们假设Mally也有READ_PHONE_STATE权限，它用这个权限从受害者的手机中获取手机号码（几乎35%的应用都使用这个权限[60]）。



下面我们将展示Eve如何在Alice无意中安装Mally到手机上后，以Alice的身份向UPI服务器注册。

1. Mally一旦安装在Alice的手机上，就会通过互联网向Eve的C&C服务器报告（Android自动授予INTERNET权限）。Mally将Alice的手机号报告给Eve
2.  Eve exploits Potential Security Hole #1 in BHIM’s workflow to bind her device to Alice’s cell number as shown in Figure 4b.  Eve首先将手机置于飞行模式，同时通过Wi- Fi保持与互联网的连接。Eve手机上的BHIM应用通过发送Eve的设备详细信息开始握手。The UPI server responds with a device registration token for Eve.  理想情况下，Eve的BHIM必须通过短信将token传回UPI服务器。然而，由于Eve已关闭SMS消息，包含令牌的SMS无法发送。BHIM提示Eve键入一个手机号码，Eve键入Alice的手机号码。BHIM现在将Eve的设备注册登录令牌和Alice的手机号码作为HTTPS消息发送给UPI服务器进行硬绑定。然后UPI服务器向Alice发送OTP。
3. Intercept the OTP. On Alice’s phone, Mally in- tercepts the incoming OTP message because its RE- CEIVE_SMS permission allows it. 然后，Mally将OTP以HTTPS的方式发送到攻击者的C&C服务器
4. C&C服务器会发送一条包含OTP的短信到攻击者的手机上。请注意，BHIM应用程序通常会检查它收到的OTP消息的ori- gin，并且只接受来自已知UPI服务器的OTP。然而，Eve在攻击之前，在她手机上的BHIM的重新包装版本中取消了这一保障措施，从而利用了潜在的安全漏洞#2。
5. New BHIM user? Create BHIM’s Passcode: BHIM on Eve’s phone will ask for BHIM’s 4-digit passcode. Now Eve does not know if Alice is a new user of BHIM or a registered user. However, Eve can determine this from Step 6 of the handshake where the UPI server sets a flag called accountExists to false for a new user. Eve可以继续为新用户Alice设置一个新的密码。



#### 3.3.4 Attack #2: Unauthorized transactions on bank ac- counts given cell number and partial debit card number

For the attack to succeed, Eve requires additional knowledge about Alice: the last six digits of Alice’s debit card number and expiry date. 在印度的商店和餐馆里，借记卡在结账时被粗心地交给不明身份的人（通常带有手机号码，因为收银员经常收集手机号码来发送折扣优惠或给予奖励积分）。

To reset the UPI PIN, Eve requires the last six digits of the debit card number, expiry date, and an OTP, all of which she has.



