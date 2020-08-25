# How to Attack PSD2 Internet Banking

2019年9月14日，修订后的《支付服务指令》（PSD2）的监管技术标准将在欧盟和欧洲经济区内生效。该法规强制规定了两个广泛需求的交易安全属性：双因素认证，以及将认证码与交易的受益人和金额动态关联（全交易认证）。即使从安全的角度来看，该规定无疑是一个积极的进展，但它并没有考虑到交易过程中涉及的所有技术和人为的薄弱环节。在本文中，我们探讨了一系列针对网上银行和手机银行的攻击，这些攻击即使在后PSD2时代也有可能发生。尽管这项工作是出于监管的动机，但所提出的问题和解决这些问题的建议很可能对一般的网上银行具有普遍性。



## Introduction

在FC 2013上，Adham等人发表了题为 "如何攻击双因素认证网银 "的作品[1]。他们概述了英国(UK)网上银行交易安全的现状，并指出了它可能被攻击的方式。Although they appreciated the increasing adoption of a second factor for transaction authentication, **they argued that an additional one-time password (OTP) alone** would not sufficiently protect a customer from falling prey to malware. 

 As a consequence, 2FA did only stop adversaries from performing arbitrary transactions at any time, but did not prevent a real-time transaction manipulation attack.

2018年3月，监管技术标准（RTS）生效，并将于2019年9月起适用[8]。RTS是修订后的支付服务指令(PSD2)的一部分，它取代了最初引入单一欧洲支付区(SEPA)的前身PSD。



 To achieve this, the RTS stipulate strong customer authentication (SCA) that requires remote payments to make use of at least two independent and mutually exclusive elements of the categories knowledge, possession and inherence. 

即使无疑是一个积极的发展，RTS也不会排除所有的攻击载体。本着Adham等人的精神，我们的目标是找出完全交易认证和新规都没有解决的薄弱点。





## 背景和相关工作

进行信用转账包括两个步骤：issuing和确认。

首先，客户需要登录网上银行。这一过程通常通过a knowledge authentication element，即密码来保障。登录成功后，客户通过指定自己的账号和金额，向所需受益人发出转账。为了使转账有效，银行还需要对交易进行确认，通常会要求客户提供一个OTP，在网上银行中，这个OTP通常被称为TAN（交易认证号transaction authentication number）。

The method that dynamically links the transaction and yields the TAN is hence called TAN method.

In the following, we outline three popular TAN methods and attacks against each of them from the related work. **All of these methods offer a 2FA(Two-factor authentication (2FA) ) as well as full transaction authentication and, hence, display the transfer details**—i.e., the beneficiary’s **account number and the amount**—on a second device.



1.SMS Authentication. 

基于短信的身份验证程序（smsTAN）依靠短信服务（SMS）从银行向客户传输一条包含转账详情和TAN的短信。In 2008 and 2014, Engel discovered several vulnerabilities in the Signalling System No. 7 (SS7) protocol that forms the foundation SMS messages are built on [21]. Also Long-Term Evolution (LTE)—the to date latest mobile communication standard—is prone to attacks [22]. Mulliner et al. also addressed the security of the SMS [18] and showed how to abuse flaws to attack



2.Smartcard Authentication. 

Particularly European banks rely on the Chip and PIN (EMV) standard to create a TAN using the customer’s bank card. 这种方法需要一个专门的读卡器设备，同时向客户显示转账细节。2009年，Drimer等人发现了英国各银行在各自实施过程中存在的各种设计和协议缺陷，一年后，Murdoch等人成功发起了针对EMV协议的攻击，可以在不知道PIN码的情况下使用被盗卡[20]。



3.智能手机认证

工作原理与短信TAN方法类似，但使用银行开发的专用应用程序通过互联网传递数据。



## 3 Threat Model

我们假设一个客户在网上订购了一个产品，并通过她的网上银行以银行电汇的方式支付。该客户使用2FA(Two-factor authentication)与TAN方法提供完整的交易验证。为此，TAN方法在第二个独立的设备上显示转账细节以进行验证。

攻击者的目标是将客户的转账指令重定向到另一个账户。攻击者只替换了受益人的账号，而没有改变金额。 This happens due to the following reason: when paying an invoice, the customer is usually aware of the amount but frequently unaware of the beneficiary’s account number. 

为了操纵交易，我们假设adversary can completely compromise the transfer-issuing channel, 这使她能够观察或篡改客户收到、看到、输入或发送的所有细节。然而，攻击者无法控制受害者的TAN method. 

The assumed threat model is rather weak as it does not require infection of both devices that are involved in the transaction authentication process.



## 4 Attacks and Challenges

### 4.1 Clipboard Hijacking

在桌面和移动操作系统上，剪贴板是一种共享资源，每个应用程序都可以读取和写入。这就允许通过监控系统剪贴板的内容来窃取[11]和操纵[26]数据。

The international banking account number (IBAN)—the default within the SEPA—adheres to a well defined ISO standard. According to that standard, an IBAN can consist of up to 34 alphanumeric characters. In the case that a customer receives a **digital invoice**, e.g., a PDF, a customer is likely going to use the copy and paste method to avoid entering the IBAN manually.



Attack: 由于IBAN还包含两个校验数字，it is easy to validate the correctness of a given candidate. Consequently, 因此，监控剪贴板的攻击者也可以检测到系统剪贴板中的IBAN，并将其替换为攻击者控制的账户的IBAN.

由于客户粘贴了IBAN，她可能会认为它一定是正确的, hence, skips the account number verification.



防御措施:为了减轻这种攻击，银行应该禁止将剪贴板数据粘贴到表单元素的可能性。开发人员可以通过为粘贴事件安装自定义监听器来防止在Web和移动应用程序中出现这种情况。



### 4.2 SMS Autofill on iOS and macOS

2016年，Konoth等人已经批评了iOS到macOS的短信同步，as this allows for an attacker to only infect the transfer-issuing channel to control both authentication elements.

随着2018年9月iOS 12和macOS 10.14(Mojave)的发布，这种整合变得更加紧密: if a customer visits a webpage that asks for an OTP (one-time password (OTP)  sent by SMS, macOS 10.14上的Safari会提供在预定义字段中自动插入OTP的功能. (in a predefined field)



Attack: 自动填写OTP. 

**Autofilling the TAN encourages the customer to omit this verification step**. 攻击者如果入侵设备并操纵转账，就会触发自动填写。因此，受害者可能不会验证转账的完整性，因为TAN会自动填写。



Defense: Our tests show that the keyword “code” is necessary within the SMS to trigger this feature. 因此，银行应该避免在短信内使用这个词。



### 4.3 Stealthy Transaction Manipulation

在我们的研究过程中，我们注意到，许多重要的德国大型银行--例如Sparkassen以及Volksbanken und Raiffeisenbanken--also show a transaction’s details on the **confirmation webpage that asks the customer for a TAN**. 



它会让用户习惯性地进行错误的交易验证：instead of comparing the details shown on the **customer’s TAN** device to the **original invoice**, she might compare them to the details shown within the **transfer-issuing channel**, e.g., the web browser.



- transfer-issuing channel is trustworthy.



防御措施: banks should stop displaying transaction details within the transfer-issuing channel, 因为这种行为显然是不合规的。



### 4.4 Digital Invoice Manipulation
After purchase, online shops send out an e-mail to their customers that contains a PDF invoice or a link that displays the invoice and payment details within the browser.



Attack. Instead of tampering篡改 with the transfer order, a malware might as well directly modify the invoice. Due to the IBAN’s well defined format, it is easy to detect and replace occurrences within a PDF or HTML page. Hence, an attacker could manipulate the invoiced account number directly. 



防御方法。网店可以只通过邮寄的方式发送支付详情。然而，特别是对于预付款，这可能不是一个选择。





4.5 Transfer Templates

为了避免让客户为经常性的收款人输入账号，很多银行提供了显性和隐性转账模板。对于显性转账模板，客户需要在网上银行内主动新建一个包含受益人姓名和账号的条目。当客户要对其保存为转账模板的联系人进行信用转账时。

When a customer wants to perform a credit transfer to one of her contacts saved as transfer template, she can just select this contact from a list.  隐性转账模板的工作原理类似，但不需要客户主动创建条目：当客户在转账单中输入受益人姓名时，网上银行会自动搜索过去的交易并建议填写相应的账号。





防御：转账模板很难与所见即所得的原则相协调。因此，很难创造一个解决方案，一方面提供转账模板的舒适性，另一方面也鼓励客户验证交易的账号。

