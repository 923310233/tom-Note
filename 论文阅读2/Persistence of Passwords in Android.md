# Total Recall: Persistence of Passwords in Android

## I. INTRODUCTION

在处理敏感数据（如密码）时，良好的安全实践是，一旦数据不再使用，就用零覆盖数据缓冲区。This protects against attackers who gain a snapshot of a device’s physical memory, whether by in- person physical attacks, or by remote attacks like Meltdown and Spectre。

本文关注了Android手机中流行的应用、安全密码管理应用，甚至锁屏系统过程中不必要的密码保留。我们对Android框架和各种应用进行了全面分析，发现密码可以在各种位置存活，including UI widgets where users enter their passwords, apps that retain passwords rather than exchange them for tokens, old copies not yet reused by garbage collectors, and buffers in keyboard apps. 

我们已经开发了一些解决方案，通过适度的代码修改，成功解决了这些问题。





在内存泄露攻击中，无权限的攻击者可以从设备内存中窃取敏感数据。

内存泄露攻击构成了严重的威胁，因为敏感数据(如加密私钥和密码)如果被盗，很容易被重复使用。

因此，我们应该在不再使用敏感数据时，立即将其从内存中删除。加密库早已认识到这种安全实践的重要性。一些软件，如OpenSSL[44]和GnuTLS[39]，在会话结束后明确地将密钥材料归零。

Aware of this issue, the Java Cryptography Architecture (JCA) [45], in 2003, was engineered to use mutable character arrays rather than String objects, which are immutable, for the explicit purpose of making its keys easier to overwrite.

在本研究中，我们特别关注一种类型的敏感数据--用户密码--以及它们在实际中如何被真实的Android应用使用。

密码库已经集成了许多广为人知的安全实践，开发人员往往坚持使用相对成熟的库（例如，OpenSSL）。When it comes to password-based authentication, developers may be tempted to follow idiosyncratic security practices, unaware of the dangers of keeping passwords live in memory。

考虑到应用程序开发人员的经验水平不同，以及市场上有大量的应用程序，可以预计不同应用程序的认证功能的安全性会有很大差异。

 A recent study has also revealed that some developers simply store passwords in plaintext on disk [43]. In this paper, we ask a question: How well does Android manage passwords?





Android平台有许多复杂的交互层（如Dalvik/ART运行时系统、操作系统内核和应用程序），因此这些层中任何一层的不良做法都可能导致安全问题。此外，安卓应用有一个复杂的生命周期；一个应用可能会被放入 "后台 "甚至 "停止"，而不一定有机会在生命周期改变之前清理其内存中的敏感值。

Additionally, user passwords go through a long chain of custody before authentication takes place, sometimes even passing from one app to another via IPC / Binder calls. Each of the steps in this chain may inadvertently retain passwords for longer than necessary. 

最后但并非最不重要的是，之前的研究发现，Android应用程序在执行安全deallocation方面存在不足[57]，它们可能会在内存中保留TLS密钥材料[34]。



Using system memory dumping and code analysis, 发现许多流行的应用程序，包括银行应用程序、密码管理器，甚至是Android锁屏程序，都会在不需要用户密码的情况下，长期在内存中保留用户密码。这些密码可以通过简单的脚本轻松地从内存转储memory dumps 中提取出来。这是一个严重的问题，因为用户经常在不同的应用程序中使用相似或相同的密码



我们的评估表明，我们的解决方案在我们测试的所有应用程序中消除了密码保留，使系统对内存泄露攻击进行了加固。





## II. BACKGROUND AND MOTIVATION

最近的Android版本已经开始使用指纹、人脸识别和语音识别作为authentication的手段。然而，到目前为止，*密码*仍然是Android authentication的主流，

Broadly, Android authentication apps fall into two categories: *remote authentication*, where an app needs to send some secret to a remote server (e.g., social networking apps), and *local authentication*, where authentication is handled entirely on the local device（如密码管理器或锁屏应用）.





Remote authentication:

图1显示了远程认证的典型工作流程，主要有三个阶段

1. The app prompts the user to enter their password, and then contacts the remote server with the user credential 用户凭证. 
2. The server validates the credential and returns a cookie or authentication token upon success.
3.  2 The app receives the cookie or token, which will be stored in a secure location (e.g., private files of the app) and used for further requests to the server. 
4. 3 Whenever the app needs to contact the server again, it looks up the token from the secure storage, and resumes the session without prompting for the user password again. The user will not need to enter their password again until the shared temporary key expires.



B. 保留密码的风险
不幸的是，有很多机会让密码被安卓系统保留超过必要的时间。



Background applications: Android应用的活动生命周期与传统桌面程序不同。

Android apps can be “paused” and then “stopped” when they are switched to the background, and they can be “resumed” and “restarted” when switched back. 当一个应用进入后台时，它的GUI会被隐藏，停止运行，但底层进程及其内存仍然存在。如果一个应用在 "暂停 "时还保存着用户密码，那么密码可能会在内存中长期保持活态。虽然安卓系统可能会销毁某些后台进程，但通常只有在系统资源耗尽时才会这样做。



TrustZone:

Over the years, Android has been integrated with many security features, such as the ARM TrustZone platform. A program that runs in the “secure world” inside TrustZone will be protected from attackers in the normal world, so secret data will not be visible to external programs. Android uses this feature for many security applications, such as its Keystore service and fingerprint authentication. 因此，这些数据受到保护，不会受到利用软件漏洞的内存泄露攻击，甚至不会受到拥有root权限的攻击者的攻击。不过，普通的Android应用并不使用TrustZone来管理密码。



延迟的垃圾收集。大多数Android应用程序都是用Java编写的，因此它们的内存由垃圾收集器（GC）管理。因此，即使应用程序删除了它对密码的最后一个引用，内存也会保持未清除状态，直到GC重新使用它。这种延迟可能会持续几分钟甚至几个小时，这取决于GC系统的内存压力。Furthermore, in the (seemingly intuitive) case of using Java’s String class to hold passwords, developers cannot manually overwrite them, because String objects are immutable. Thus, the Java Cryptographic Architecture [45] recommends that passwords should be stored in char arrays instead of String objects. 



Java vs. native code: 虽然Android应用通常用Java编写，但它们可能会对应用所包含的或系统上原生安装的底层C库进行本地调用。例如，Android的TLS实现在用C语言编写的BoringSSL加密库上封装了一个Java层（Conscrypt），如果密码从Java层复制到C层，数据也有可能保留在C层[34]。







We assume that an adversary can perform memory disclosure attacks on an Android device. 

For instance, the recent memory dumping vulnerability in the Nexus 5X phone [29] allows an attacker to obtain the full memory dump of the device even if the phone is locked. 另一个例子是，WiFi芯片组的漏洞[5]可以让攻击者远程获取受害者设备的内存快照。然后，攻击者可以分析内存快照，并获得任何未清除的敏感数据，如用户密码。





我们选取了11个Android应用进行密码保留问题的初步研究。其中6个应用非常流行--每个应用的安装量超过1000万，另外4个应用是存储高度敏感用户数据的password managers 。In addition to these apps, we tested the system processes that are in charge of unlocking the phone after receiving the correct password, which are critical to the overall security of the device. 

We installed and launched each app, and manually entered passwords for authentication. After this, we performed a full physical memory dump [56] as well as a per-process dump [34].

If the passwords are anywhere in memory, we will find them. We looked for password encodings in two-byte characters (UTF16, as used by the Java String object), as well as one-byte characters (as used by ASCII). 

We performed such a dump several times for each app: 

1. right after authentication (“login”),  手动的
2. after moving the app to the background (“BG”), 
3. after additionally playing videos from the YouTube application (“YouTube”), a
4. after locking the phone (“lock”).





观察1：所有测试的应用程序都是脆弱的。通过简单的技术，我们成功地检索到了所有应用程序的明文密码。非常流行的应用程序，如Facebook，Chrome和Gmail，已经安装了超过10亿次，在内存中保留了登录密码。 Secure password managers expose master passwords which are typically used to decrypt their internal password databases, so an attacker would be able to capture the master password and gain access to the full databases. 此外，锁屏过程也会将PIN密码留在内存中。由于PIN密码用于全盘加密和解密以及解锁手机，因此Android系统花费了非常多的精力来保护PIN密码--例如，Gatekeeper服务会在TrustZone中验证用户密码的哈希值来保护密码。

观察3：一些开发者已经注意到了密码保留问题。我们可以看到一些应用（如大通银行、Dashlane和Keepass2Android）似乎在主动清除密码的证据。对于这三个应用，一旦我们将应用放入后台，密码就会消失。这说明，密码保留的问题似乎至少已经引起了一些Android开发者的重视，至少是可以解决的。我们更希望安卓系统能够为所有应用开发者提供帮助，解决这些问题。

观察4：Password strings are easily recognizable.。对于许多应用程序，我们发现密码字符串与其他容易识别的字符串模式在一起。例如，Facebook应用程序包含ASCII字符串模式，如...&pass=1839172...，Tumblr应用程序有p.a.s.s.w.o.r.d.=.1.8.3.9.7.2.即UTF16编码。





为了达到彻底了解密码保留的根本原因，我们对安卓框架和几个应用进行了深度分析。

A. Methodology
In order to identify where password retention occurs, we have used two key techniques: runtime logging, and password mutation.



Runtime logging: We annotate core modules in Android, using the standard logging facility, giving us a timeline of the use of function calls related to password processing.



Password mutation: In order to precisely pinpoint the location of password retention, we also apply password mutations as the passwords pass through different Android components. When a component Ci receives a password pi, it will index a pre-defined permutation dictionary using pi, and obtain a mutated password pi+1 before passing it to the next component Ci+1. Therefore, when we take the memory dump, we know that instances of pi are hoarded by component Ci, whereas instances of pi+1 are hoarded by Ci+1. Our algorithm also ensures that these password mutants have the same length and unique contents, so we can easily locate a password fragment within the component that’s using it.



为什么不用动态分析：

However, suitable tooling that we might adopt for our experiments appears to be experimental, either targeting outdated Android versions (e.g., NDroid [62] for Android 5.0) or requiring very specific hardware (e.g., DroidScope [63] only supports an Acer 4830T)





图2显示了Android中用户输入密码时的数据流。

- The signals from the touchscreen are transmitted to a software keyboard app, otherwise known as an input method editor (IME) app, via the kernel driver. 

- Then, the keyboard/IME app will send the password to the UI widget (e.g., TextView) in the application (e.g., Facebook) via a dedicated input channel. 

- The UI widget also stores the password internally, so that it can pass the data to the application upon request.

-  Additionally, the widget sends the data to a graphics module，so that the input strokes are echoed back and displayed on the screen as stars (*) by the display device driver. 

  在测试了所有这些阶段后，我们能够将罪魁祸首缩小到UI widget和键盘应用上， Subsequently, we further analyzed the source code of the UI widget, and found that Android does not implement a dedicated class for password widgets, but rather simply reuses the TextView class. 由于密码没有得到任何特别的处理，所以如果出现问题，我们不应该感到奇怪。



问题1：缺乏安全的归零。首先，当应用程序 "暂停"、"停止 "甚至 "销毁 "时，TextView类不会将缓冲区归零或以其他方式清除。因此，当这些生命周期活动之一发生时，保存文本的内存对象仍保持完整。这就把安全归零的责任完全推给了应用开发者，他们既要处理应用的生命周期，又要在登录完成后将TextView缓冲区归零。我们认为，这个责任应该由TextView来处理，而不是由应用开发者来处理。



问题2：不安全的SpannableStringBuilder。TextView类中的缓冲区实际上是一个SpannableStringBuilder，它的实现导致了两个问题。

First, whenever a user types a new character of her password, SpannableStringBuilder will allocate a new array, copy the previous password prefix to this array, and discard the previous array without clearing it.

这就是为什么我们在内存中看到碎片化密码的根本原因。我们还注意到，SpannableStringBuilder类提供了一个清除方法，但它只是将内部数据设置为null，而不是将数据归零。如果应用开发者错误地认为这个方法意味着安全删除，那么密码仍然会留在内存中。





问题3：缺乏安全的getPassword API。开发者通常通过调用TextView对象的getText()方法来获取其内容，该方法返回的是一个CharSequence界面，而不是一个SpannableStringBuilder对象。由于CharSequence也是String类的接口，开发者通常将其视为String的一种，并调用getText().toString()方法将密码变成一个String对象。然而，众所周知，字符串会引起安全问题。JCA官方库[45]特别提出 "String类型的对象是不可变的，所以在使用后没有办法覆盖String的内容"；它还进一步建议开发者不要将密码存储在String对象中。





## 深入分析

Next, we analyze several third-party Android apps.

We wish to consider “example” apps using four categories of authentication techniques: 

a) basic password-based authentication apps, which simply send user passwords to a remote server for authentication, 

b) challenge/response apps, which derive secrets from passwords for authentication, 

c) OAuth apps, which delegate authentication to an OAuth service (e.g., Facebook), and 

d) local authentication or standalone apps that do not involve a remote server, including some password manager apps.





we obtain similar types of apps from official sites, open source repositories (e.g., GitHub), security guidebooks, and developers’ website such as Stack Overflow.





我们首先考虑基本的认证应用，即应用通过HTTP/HTTPS向服务器发送原始用户密码。我们收集到的大多数应用都属于这一类，因为直接发送密码是最简单（但不安全）的认证方式。

Sample apps 1 – 2 in Table III are some of the basic password-based authentication apps we have tested, and they use different libraries for network communication (Apache vs. Volley). 结果显示，它们都有很多密码副本。





为了了解为什么这么简单的应用会有这么多的密码拷贝，我们对这些应用进行了逐一修改，并分析了我们修改后的影响。

Android框架在TextView中提供了getText().toString()。事实上，我们看到所有的样本应用都通过调用getText().toString()将密码存储为Java字符串。String的使用对密码保留的贡献有多大？为了衡量String使用的效果，我们删除了样本应用1中对String的所有使用，取而代之的是让应用发送一个空密码。服务器 "运行在我们实验室的本地台式机上；它总是向应用程序发送成功的验证消息。同表中的样本应用1a显示了修改后的结果。相对于原来的密码，这个修改消除了一半以上的内存密码。

缺乏手动清理TextView。回想一下，TextView在不再需要密码后，会将密码保存在其缓冲区中（问题#1和#2）。这就把清理TextView的责任留给了各个应用开发者。





 Challenge-Response Authentication Apps
 Challenge/response apps do not directly send passwords to the network, but rather generate HMAC values from user passwords and use them as secrets for the remote servers to perform authentication.



 接下来我们考虑提供OAuth服务的应用。由于Facebook是OAuth的主流身份提供商，我们按照Facebook的官方指南，使用Facebook OAuth库实现了示例应用4。如果我们启动应用并点击登录按钮，我们的应用就会重定向到Facebook应用，然后Facebook应用会提示用户输入密码，并执行要求的认证；Facebook应用在成功后重定向回我们原来的应用，然后显示登录成功的消息。
表三中的样本应用4显示了结果。虽然它比之前的应用安全得多，但它仍然在内存中保存了不少密码副本，而且在手机被锁定后，其中一个副本仍然留在内存中。这些密码都是在Facebook应用本身的内存中发现的，而不是在我们的样本应用中。



如果说有哪个应用程序开发者会谨慎地控制内存中密码的存在，那肯定是密码管理器的开发者了！从表一中可以看出，密码管理器的安全性比其他应用程序要高，但仍然有很多密码留在内存中。如表一所示，密码管理器相对来说比其他应用更安全，但仍然有很多密码留在内存中。



Keepass2Android由8万多行代码和300多个源代码文件组成。我们发现，它的代码库在处理安全密码删除方面设计得特别好。
- 它将密码转换为char数组，并使用后者生成主密钥。
- 认证后，它将TextView的内容设置为空字符串，并手动将所有与密码相关的对象设置为空。
- 它手动调用垃圾回收器，这可能有助于加速内存的再利用。

另一款密码管理器PasswdSafe在安全措施上也是值得称道的。事实上，在分析了源码之后，他们在处理密码方面的努力程度给我们留下了深刻的印象。首先，这款应用实现了 manual reference  count 而不是依赖于Java本身的自动内存管理。