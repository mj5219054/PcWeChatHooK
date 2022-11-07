# 微信 
提到微信，网友是又爱又恨，日常沟通已经彻底离不了，但体积却臃肿不堪，动不动就占用三四十GB的空间，成为手机中占内存最大的APP。其实，最初的微信确实是“小而美”的，在2011年1月发布的微信1.0版本，安卓APK安装包的体积仅457KB，还没有一张照片体积大。已经能够实现微信发消息这个核心功能。
而2022年6月发布的微信8.0.24版本，安卓APK安装包的体积已经膨胀到了257MB，比很多PC软件的体积还要大，11年来膨胀了575倍。

微信APK安装包，看了看它膨胀575倍到底更新了啥。解压发现，微信8.0.24版本APK共包含12639个文件，而微信1.0版本只有199个文件，该UP主调侃：“新版微信有98%的文件都是垃圾。”

微信8.0.24中，文件夹主要有：assets文件夹：体积78.4MB，里面装着微信的资源文件，比如自带emoji表情、字体、收款音频、微信电话铃声等等

lib文件夹：体积337MB，里面都是第三方动态库，一共157个库，比如解码、解压缩等，塞入的功能越多，需要调用的库也就越多，安装包体积也就越臃肿。而在微信1.0版本中，只有一个库，体积仅127KB。

META-INF文件夹：体积2.1MB，里面存储了开发者的数字签名

r文件夹：体积12.8MB，里面存放着APP资源库，还有杂七杂八的素材图片。

resdec文件夹：体积0MB，是个空文件夹。

此外在根目录下还有17个文件：
AndroidManifest.xml，

aseInfo.dat，记录着classes.dex文件的MD5值

resources.arsc，记录着文件之间的对应关系

此外，还有14个classes.dex文件，也就是微信编译后的程序本体。新版共占用161MB，而初代只有1个classes.dex文件，体积仅256KB。11年暴涨644倍。

Cheat Engine 是一款内存修改编辑工具 ，它允许你修改你的游戏或软件内存数据，以得到一些其他功能。它包括16进制编辑，反汇编程序，内存查找工具。与同类修改工具相比，它具有强大的反汇编功能，且自身附带了外挂制作工具，可以用它直接生成外挂。

在我看来，CE做的最好的就是各种策略的内存搜索能力。

支持准确数据（整数、字符串、十六进制、浮点数、字节数组等等）搜索，针对目标数据明确效果显著，比如金币数。

支持数据范围的搜索，比如大于某个值，小于某个值等等。比如想找到没有显示数值的血量数据。

支持多组数据同时搜索，针对数据结构复杂的情况

支持搜索结果的多次过滤（图中框选的Next Scan），最终找到目标数据。比如血量未知时，通过加血、减血多次搜索最终找到血量地址。

0x2. 分析
进入正题，本文是要拿到微信聊天的语音消息，然后dump保存下来。

要按以前我的思路，会通过网络通信找到接受消息的函数，然后找到语音数据，看起来很简单，但是有点难。

因为函数真的很多，网络消息也会受到很多干扰。

现在用CE了，应该怎么办呢？

通过查看微信APK的文件结构，我们发现存在三个不同的dex文件，分别是classes.dex、classes2.dex、classes3.dex。dex文件是Android平台上（Dalvik虚拟机）的可执行文件，也叫作Dalvik字节码文件。通过Dalvik字节码我们不能直接查看原来的逻辑代码，但可以借助其它工具来帮助查看。


* assets: 存放资源文件
* lib：存放.so动态链接库
* META-INF：存放工程的属性文件，例如签名文件
* AndroidManifest.xml：Android工程的基础配置属性文件
* resources.arsc：主资源文件，相当于资源索引表

* ![图片](https://user-images.githubusercontent.com/79394963/200309294-27dca0cd-008d-42fe-bf4a-e698932e8ee6.png)

因此我们逆向的重点通常放在dex文件上，这里我们需要两个工具：apktool、dex2jar。

apktool可以对整个apk文件进行反编译，命令：java -jar apktool.jar d weixin.apk

反编译后得到如下文件，其中smali、smali_classes2、smali_classes3下分别存放classes.dex、classes2.dex、classes3.dex文件反编译后的smali文件

逆向大致思路
对于Android代码文件的整个变化过程，大致上可以看作，java转换成smali文件，所有smali文件可以打包成一个jar文件，jar文件可以转换成dex文件，dex文件就存放在apk文件里。而多个dex文件是运用了dex分包技术，不同dex的逻辑代码可以相互调用，但是不会出现重复。

我们可以结合jar和smali文件，对微信代码进行逆向分析。综合使用三个jar文件阅读器jd-gui、Luyten、jadx，使用阅读器自带的文本查找功能定位jar代码。另外综合使用XSearch对所有smali文件进行文本查找，方便定位smali代码。

逆向分析需要进行smali注入又称smali插桩，它是在保证被测程序原有逻辑完整性的基础上在程序中插入一些探针，通过探针的执行并抛出程序运行的特征数据，通过对这些数据的分析，可以获得程序的控制流和数据流信息，进而得到逻辑覆盖等动态信息，从而实现测试目的的方法。这个方法需要对原代码进行修改，需要先定位到相应的smali文件，再按照语法修改虚拟机指令。

smali文件修改后需要把smali文件转换回dex文件，使用如下命令

ShakaApktool: java -jar apktool.jar s [目录或smali文件]
Smali: java -jar smali-[版本号].jar a [目录或smali文件]
使用WinRAR将dex文件重新放回到apk文件中，覆盖更新原有文件。修改后的apk原有签名将会被破坏，需要重新签名才能被安装到Android设备上。删除apk文件里的META-INF文件夹，删除原有签名文件。使用keytool生成自签证书，命令：keytool -genkey -alias android.keystore -keyalg RSA -validity 20000 -keystore android.keystore。然后使用jarsigner对apk进行签名，命令：jarsigner -verbose -keystore android.keystore -signedjar weixin_signed.apk weixin.apk android.keystore。

apk文件签名后，需要使用adb install weixin_signed.apk命令安装修改后的apk至Android设备上，安装前需要使用adb uninstall com.tencent.mm命令卸载原有apk。

逆向目标
对于Android设备本地存储的微信聊天记录，网上有不少方法可以提取。这里尝试逆向分析微信APK，调试输出实时聊天记录。在此之前，我们知道聊天记录存在于网络通信数据包当中，经过之前对微信mmtls协议的研究，通过中间人获取网络明文传输数据的方法，在实际操作上难度比较大。因此，退而求其次，在客户端上进行逆向分析，得到明文数据包应该能够实现。

Android LogCat可以输出系统及其程序的log信息，那么微信也许会输出自己的log信息。同样，我们也可以使用smali插桩，输出自己想要的信息，比如明文通信数据。那么整个逆向分析的关键就是获取微信log信息。

逆向过程
探寻微信的看门人
Android APP开发者都知道，通常情况下，app的log输出由app的一个类负责管理，这个类通常会调用系统函数log()输出log信息，那么我们使用XSearch在smali文件中查找一下“log(”，得到许多匹配的结果。

* ![图片](https://user-images.githubusercontent.com/79394963/200309406-897654d9-7b92-4172-b093-fa91a57088af.png)


我们发现有一个名为XLogSetup.smali的文件，看文件名的意思像是负责Log的Setup。使用jar代码查看器打开，查看一下它的代码。


* ![图片](https://user-images.githubusercontent.com/79394963/200309519-89a857c4-e742-45ee-a9b8-f836f206468f.png)

这个文件确实是负责微信Log的Setup的，而微信的Log系统被命名为XLog。

Xlog.setConsoleLogOpen(isLogcatOpen.booleanValue());看上去这句像是负责LogCat输出Log的开关，那么我们先尝试修改这行对应的smali代码，设置函数的传参为True。



Xlog.appenderOpen(toolsLevel.intValue(), 1, cachePath, logPath, nameprefix);后面的这行调用了appenderOpen函数，这个函数位于com/tencent/mars/xlog/Xlog.smali文件。

* ![图片](https://user-images.githubusercontent.com/79394963/200309749-8aff165f-38a4-4c15-abf3-e60b936e1acd.png)

* ![图片](https://user-images.githubusercontent.com/79394963/200309802-b5725f83-e28e-4312-8449-a3be563c2094.png)

每一种类型log都调用了logWrite2函数，logWrite2函数被声明为native，可以推测调用系统函数输出log由tencentxlog负责。

使用XSearch分别尝试搜索每一种类型log的函数名（logD、logE、logF、logI、logV、logW），发现v.smali每次都出现。

* ![图片](https://user-images.githubusercontent.com/79394963/200309930-6b806bb6-459a-4ea1-aa62-7e5de18fcbd6.png)
发现多次调用public static int getLogLevel()函数，每一种类型的log在输出之前需要检查loglevel（log级别），一共1到7级，1为最高级别。那么我们可以使用smali插桩，输出所有类型的Log。

插入log函数：

.method public static log(Ljava/lang/String;Ljava/lang/String;)V
    .locals 2
    .prologue
    const v0, true
    invoke-static {p0, p1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
    return-void
.end method
删除所有调用getLogLevel函数及其判断结构的代码：

if ((nMo != null) && (nMo.getLogLevel() <= X))
在需要输出log的地方插入调用log函数的代码：

invoke-static {p0, p2}, Lcom/tencent/mm/sdk/platformtools/v;->
log(Ljava/lang/String;Ljava/lang/String;)V
安装修改后的APK，过滤tag:MicroMsg，可以在LogCat中看到微信输出的Log信息

* ![图片](https://user-images.githubusercontent.com/79394963/200310040-8cca3821-940f-4d15-b190-832b3f8f0bbb.png)

微信APK动态链接库里libwechatnetwork.so和libMMProtocalJni.so引起我的注意，看名字前者像是负责微信的网络连接，后者像是负责微信的协议。

使用 IDA6.6 反编译这两个so文件
* ![图片](https://user-images.githubusercontent.com/79394963/200310143-350571e6-72ec-4350-84b0-03d2cf20c142.png)

* ![图片](https://user-images.githubusercontent.com/79394963/200310189-e0695c3e-5947-471f-9b6f-8d2bf27dbff3.png)

libwechatnetwork.so：结合mmtls协议和抓包分析，微信网络使用HTTP短连接和私有协议长连接，Signaling是负责推送的函数，网络传输任务作为Task进入链接库进行Queue。而且可以自定义设置远程服务器IP，获取当前网络连接状态。

libMMProtocalJni.so：分析函数名，本库进行aes加解密，密钥的协商管理（生成和销毁），数据包的封装和解封装。

我们把重点放在pack*和unpack，即数据包的封装和解封装。

使用XSearch查找所有“;->pack”和“;->unpack(”，对所有调用libMMProtocalJni.so的pack*和unpack函数进行smali插桩，log输出函数传参。

注：经过测试，应该重点插桩下面四个文件
com/tencent/mm/u/r.class
com/tencent/mm/u/t.class
com/tencent/mm/protocal/ab.class关键代码位于ab$b.smali
com/tencent/mm/model/ak.class关键代码位于ak$3.smali
传参有PByteArray、Int、Byte、String四种类型，这里log输出PByteArray、Byte和String三种类型，其中PByteArray为自定义类型，封装着一个Byte类型。

插桩方法：在smali中插入下面三个函数，其中com/tencent/mm/sdk/platformtools/bf;->bm(可以将Byte类型格式化为16进制String类型，com/tencent/mm/pointers/PByteArray;->value:[B可以读取PByteArray封装的Byte。

.method public static log(Ljava/lang/String;)V
    .locals 2
    .prologue
    const-string v0, "Test"
    invoke-static {v0, p0}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
    return-void
.end method

.method public static log([B)V
    .locals 2
    .prologue
    const-string v0, "Test"
    move-object v1, p0
    invoke-static {v1}, Lcom/tencent/mm/sdk/platformtools/bf;->bm([B)Ljava/lang/String;
    move-result-object v1
    invoke-static {v0, v1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
    return-void
.end method

.method public static log(Lcom/tencent/mm/pointers/PByteArray;)V
    .locals 2
    .prologue
    const-string v0, "Test"
    iget-object v1, p0, Lcom/tencent/mm/pointers/PByteArray;->value:[B
    invoke-static {v1}, Lcom/tencent/mm/sdk/platformtools/bf;->bm([B)Ljava/lang/String;
    move-result-object v1
    invoke-static {v0, v1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
    return-void
.end method
然后在pack*和unpack对应位置插入调用log函数的代码：

    invoke-static {v1}, Lcom/tencent/mm/u/r;->log(Ljava/lang/String;)V
    invoke-static {v2}, Lcom/tencent/mm/u/r;->log([B)V
    invoke-static {v3}, Lcom/tencent/mm/u/r;->log(Lcom/tencent/mm/pointers/PByteArray;)V
代码按需修改，最后修改所在函数的寄存器声明数.locals。

安装修改后的APK，过滤tag:Test，可以在LogCat中看到微信网络数据包的16进制信息，包含发送和接收的密文和明文数据。
* ![图片](https://user-images.githubusercontent.com/79394963/200310290-a009655b-fea1-472e-b6a0-91668ee4723d.png)

每一段的第一句分别是发送数据包的16进制明文数据

0801124d0a150a13777869645f71797131737566683030667132311221e590ace8afb4e4bda0e59ca8e5819ae5beaee4bfa1e98086e59091e58886e69e90180120d2a9b6c505289891d9fefdffffffff01

![图片](https://user-images.githubusercontent.com/79394963/200310565-b4701033-9ece-47e1-a57c-4efd47f2b540.png)

0801122f0a150a13777869645f717971317375666830306671323112085be5beaee7ac915d180120f6a9b6c50528b4d19c8e01

![图片](https://user-images.githubusercontent.com/79394963/200310616-ee4d93bf-b464-4b06-b904-de95f124fa1e.png)

080112330a150a13777869645f717971317375666830306671323112073233333333333318012082aab6c50528f1e2e5f6ffffffffff01

![图片](https://user-images.githubusercontent.com/79394963/200310664-37b0fdcc-1f1e-4add-84c6-f4b2407ccdcd.png)

“wxid_”开头的是微信id，0000001ch是聊天发送数据的长度，0000001dh是聊天发送数据的开始位置。

提取出聊天发送数据的16进制，分别进行分析

e590ace8afb4e4bda0e59ca8e5819ae5beaee4bfa1e98086e59091e58886e69e90

5be5beaee7ac915d

32333333333333

经过尝试，确定是UTF8编码，只需在开头插入UTF-8编码标志EFBBBF即可。

efbbbfe590ace8afb4e4bda0e59ca8e5819ae5beaee4bfa1e98086e59091e58886e69e90



找到关键数据

关键数据肯定是语音消息了，但是怎么搜索呢，肯定搜语音内容不现实，所以转了弯，先看看文字消息，找到接受文字消息处理函数之后，猜测语音处理函数会相同或者在不同分支。

接着，如何搜索文字消息呢？已经收到的显示在聊天窗口的内容当然可以通过CE找到，但是没用啊，它和接受文字消息处理函数已经没关系了，流程已经处理完成了。

那么在测试中肯定知道发送的消息内容，通过CE来搜索可以吗？

额，我觉得不行，还没收到消息呢，内存中也没有这个文字消息，搜索不到（如果可以，请大佬指点一下）。

能想到的是，在接受到消息某一点通过调试器断下来，然后CE搜索，这样可以，但是这个断点找不到阿，放弃。

那怎么办呢？

看到左侧聊天列表中显示的最新一条消息，有了新的思路。

每次收到新消息后，都会在列表中显示最新消息内容（图中绿框指示位置、注意是unicode字符）。

那么，先用CE（First Scan）搜索当前搜到的消息内容，找到可能的内存地址。多次接受不同消息后，Next Scan按钮搜索每次新的消息内容，最终确定聊天列表中显示的最新消息内容的内存地址。

多次刷选之后，留下两个地址，通过CE修改内容，在界面中查看是否改变，最终确认第二个地址就是我们的目标，暂把该地址记录为MsgAddr。

分析消息接收函数

关键数据地址已经找到，下面的工作复杂也不复杂，就看微信是如何实现的了。

猜测微信实现消息显示的流程是这样的：

recv收到消息，组装完整包后，分发给消息处理函数

根据wxid找到要显示消息的列表项，如果不在已聊天消息列表，就新建一个项

在列表中显示消息，如果是表情显示[文字]，语音显示为[语音]，消息插入wxid对应消息队列，或者存入数据库

步骤3中肯定要写前面找到的MsgAddr内存，把最新消息显示到界面中，这个流程肯定在消息处理函数内部。

So，通过OD对MsgAddr下内存写入断点，回溯堆栈就可以找到消息处理函数。

具体操作如下：

OD挂载Wechat.exe进程后，在左下角内存窗口处Ctrl+G，输入找到的MsgAddr（11A11F34）回车，定位到该数据，然后再HEX数据处，右键弹出菜单，选择断点->内存写入。

* ![图片](https://user-images.githubusercontent.com/79394963/200306024-11d6f885-54af-4276-b587-5fc4d39a3e1c.png)

断点设置完成后，测试发送文字消息，OD断住，代码窗口显示的就是修改MsgAddr的代码位置，如上图10CE412C处。

Alt+K查看当前堆栈。
看到这个调用栈是不是感觉好少，分析起来肯定简单。但，其实是OD显示的并不全，此时真的很想用windbg。

在OD的右下角堆栈窗口，可以看到当前调用栈的参数和预览数据。F8单步（或者Alt+F8执行到返回）逐步的回溯每层堆栈。关注MsgAddr的数据是如何生成的，也就是找到数据来源，然后找到消息处理函数。

* ![图片](https://user-images.githubusercontent.com/79394963/200306148-ed6a11a4-fdf6-4dd6-9eb9-6b77eb7e905c.png)
跟踪过程不赘述（需要熟悉汇编知识），直到看到的最顶层的WeChatWi.10206460处，发现是把收到的消息内容显示到聊天列表处的一个界面功能函数。

那这里不是可以拿到消息了吗，是的，普通文字消息已经可以拿到，但是语音内容不行。

通过观察内存窗口的数据，整理WeChatWi.10206460处的关于消息参数的大致结构。


// 聊天列表框信息
 
* struct chat_list_msg {
* DWORD unk;//
wstring wxid;//
//wchar_t* wxid;//4
//int len;//8
//int maxlen;//c
DWORD unk1;//10
DWORD unk2;//14
wstring name;
//wchar_t* name;//18微信名
//int len;//1c
//int maxlen;//20
…
wstring msg; //
//wchar_t* msg;//3c
//int len;//
//int maxlen;
}
wstring msg字段就是文字消息内容，而语音消息则是预览中看到的[语音]两字，并没有实际能够听到的语音数据，所以还得继续往前找。

* ![图片](https://user-images.githubusercontent.com/79394963/200306274-a64055ec-d709-4c83-9bbd-3398097aadb7.png)

在wstring msg处就是普通文字消息内容，而语音消息并不是我想象的就是直接语音的数据，而是…如下：

1
<msg><voicemsg endflag="1" cancelflag="0" forwardflag="0" voiceformat="4" voicelength="1176" length="1334" bufid="147445261304397871" clientmsgid="416261363964373964444633636200230013013119fdd53b1f494102" fromusername="wxid_xxxxxxxxx" /></msg>
真是一波三折，还不是语音的数据，而是关于语音信息的xml，有语音的大小，来自谁，在语音缓冲区中的id（bufid）等等信息。

继续往前找呗，最后回溯到了所有消息处理的分发函数10323FF0中。这个函数处理逻辑很复杂，我并没有很快就找到如何生成语音消息的xml，以及处理语音数据的函数。

一度卡住，重复分析了很多次。

后来又回神想到了逆向神器IDA，xml中数据如voicemsg肯定是模块中会在代码中用到，看看有没有有用的信息。

用IDA打开Wechatwin.dll，shift+F12分析出所有字符串，Ctrl+F找到关键字voicemsg，看来有戏。


 f_parseVoiceXmlInfo_103148E0
text:103149DD 0E4 0F 84 74 02 00 00                             jz      loc_10314C57
.text:103149E3 0E4 68 D0 06 F0 10                                push    offset aVoicemsg ; "voicemsg"
.text:103149E8 0E8 8B CF                                         mov     ecx, edi
.text:103149EA 0E8 E8 31 28 3E 00                                call    f_xml_subnode_106F7220
.text:103149EF 0E4 85 C0                                         test    eax, eax
.text:103149F1 0E4 0F 84 60 02 00 00                             jz      loc_10314C57
.text:103149F7 0E4 8D 70 2C                                      lea     esi, [eax+2Ch]
.text:103149FA 0E4 68 C4 06 F0 10                                push    offset aClientmsgid ; "clientmsgid"
.text:103149FF 0E8 8B CE                                         mov     ecx, esi
.text:10314A01 0E8 E8 CA 3A 3E 00                                call    f_xml_getvalue_106F84D0

函数103148E0回溯再看看，进入了分发函数10323FF0中，在一个循环中处理了多种流程，包括显示界面最新消息的流程和解码语音的流程。所以前面找的方向并没有问题，只是缺少认真分析数据和代码的耐心。

不过，目的都达到了，找到了数据处理函数，最后通过hook这个函数就能拿到语音数据。

另外可以看到语音数据中包含SILK_V3的字符，这种编码音频格式是Skpye曾经使用的一种编码方式，后来开源了。目前播放器并不能直接播放该编码音频文件，所以需要转码为MP3等格式。不过可喜的是已经有大佬完成了这个工作，并开源了工具silk-v3-decoder。所以把代码拿来整合一下，就可以完整的实现实时dump语音聊天数据，转换为mp3进行保存，完美。
0x3. 总结
这是第一次比较成功的应用CE，整个看来，确实省下来很多定位数据和函数的工作。

但CE并不是万能的，要找对方法，找对目标数据才可能成功，对于某些没有明显数据的功能，可能也是无能为力。

最终还是得提高对大型软件的逆向能力，总体实现思路的猜测以及调试验证。

最后，时间仓促，目前只是将保存语音的demo更新到到SuperWeChatPC项目中，后续会持续更新，欢迎关注。

流程：

枚举句柄，找到_WeChat_App_Instance_Identity_Mutex_Name的mutant

duplicate句柄到本进程，然后close

启动微信

下面是主要代码：

//步骤1和2的代码

//获取到微信所有进程句柄

DWORD Num = GetProcIds(L"WeChat.exe", Pids);
...

Status = ZwQuerySystemInformation(SystemHandleInformation, pbuffer, 0x1000, &dwSize);

PSYSTEM_HANDLE_INFORMATION1 pHandleInfo = (PSYSTEM_HANDLE_INFORMATION1)pbuffer;	

for(nIndex = 0; nIndex < pHandleInfo->NumberOfHandles; nIndex++)
{	   
         //句柄在Pids中，就是微信进程的句柄信息

         if(IsTargetPid(pHandleInfo->Handles[nIndex].UniqueProcessId, Pids, Num))
	{
	   HANDLE hHandle = DuplicateHandleEx(pHandleInfo->Handles[nIndex].UniqueProcessId, 
				(HANDLE)pHandleInfo->Handles[nIndex].HandleValue,
				DUPLICATE_SAME_ACCESS
			);						
	//对象名
	Status = NtQueryObject(hHandle, ObjectNameInformation, szName, 512, &dwFlags);			
         //对象类型名
	Status = NtQueryObject(hHandle,  ObjectTypeInformation, szType, 128, &dwFlags);			
	//找到微信的标志
	if (0 == wcscmp(TypName, L"Mutant"))
	{				
            if (wcsstr(Name, L"_WeChat_App_Instance_Identity_Mutex_Name"))	    
{				    
	    //DUPLICATE_CLOSE_SOURCE标志很重要，不明白的查一查
    		hHandle = DuplicateHandleEx(pHandleInfo->Handles[nIndex].UniqueProcessId, 
	                  (HANDLE)pHandleInfo->Handles[nIndex].HandleValue,
		         DUPLICATE_CLOSE_SOURCE
			);					
                if(hHandle)
		{
                	printf("+ Patch wechat success!\n");
			CloseHandle(hHandle);
		}
		}
	}
		}
		
	}
}


步骤3的代码
//通过注册表找到微信安装目录

if(ERROR_SUCCESS != RegOpenKey(HKEY_CURRENT_USER, L"Software\\Tencent\\WeChat", &hKey))
{	return;
}

DWORD Type = REG_SZ;
WCHAR Path[MAX_PATH] = {0};
DWORD cbData = MAX_PATH*sizeof(WCHAR);

if(ERROR_SUCCESS != RegQueryValueEx(hKey, L"InstallPath", 0, &Type, (LPBYTE)Path, &cbData))
{	goto __exit;
}

PathAppend(Path, L"WeChat.exe");

//启动微信客户端ShellExecute(NULL, L"Open", Path, NULL, NULL, SW_SHOW);
代码就这样，有注释，就不再啰嗦。

完整代码，请看后面的地址。
正在右边输出SendTextMessage搜刮咱们能够看到下面四个该当是咱们所需求的，都翻开看下伪代码。（咱们剖析需求找到函数挪用之处就可以晓得传参，而后再去剖析参数是若何而来。那末除函数界说中央代码，其他的均可以找到。

MMMessageSendLogic ：

/* home.php?mod=space&uid=341152 MMMessageSendLogic */
-(unsigned char)sendTextMessageWithStringvoid *)arg2 mentionedUsersvoid *)arg3 {
    r14 = self;
    r15 = [arg2 retain];
    r12 = [arg3 retain];
    r13 = [[CUtility filterStringForTextMessage:r15] retain];
    [r15 release];
    if ([r13 length] != 0x0) {
            stack[-64] = r12;
            rax = [r13 lengthOfBytesUsingEncoding:0x4];
            rbx = rax;
            if (rax >= 0x4001) {
                    rax = [[NSString alloc] initWithFormat"ERROR: Text too long, length: %lu, utf8 length: %lu", [r13 length], rbx];
                    stack[0] = "-[MMMessageSendLogic sendTextMessageWithString:mentionedUsers:]";
                    [MMLogger logWithMMLogLevel:0x2 module:"ComposeInputView" file:0x103e0e162 line:0x112 func:stack[0] message:rax];
                    [rax release];
                    rax = [NSBundle mainBundle];
                    rax = [rax retain];
                    stack[-72] = rax;
                    r15 = [[rax localizedStringForKey"Message.Input.Too.Long.Title" value"" table:0x0] retain];
                    rax = [NSBundle mainBundle];
                    rax = [rax retain];
                    r14 = rax;
                    rax = [rax localizedStringForKey"Message.Input.Too.Long.Content" value"" table:0x0];
                    rax = [rax retain];
                    [NSAlert showAlertSheetWithTitle:r15 message:rax completion:0x0];
                    [rax release];
                    [r14 release];
                    [r15 release];
                    [stack[-72] release];
                    r14 = 0x0;
                    r12 = stack[-64];
            }
            else {
                    rax = [WeChat sharedInstance];
                    rax = [rax retain];
                    r15 = [[rax CurrentUserName] retain];
                    [rax release];
                    rax = [r14 currnetChatContact];
                    rax = [rax retain];
                    r14 = [[rax m_nsUsrName] retain];
                    [rax release];
                    r12 = [[MMServiceCenter defaultCenter] retain];
                    objc_unsafeClaimAutoreleasedReturnValue([[[r12 getService:[MessageService class]] retain] SendTextMessage:r15 toUsrName:r14 msgText:r13 atUserList:stack[-64]]);
                    [rax release];
                    [r12 release];
                    [r14 release];
                    [r15 release];
                    r14 = 0x1;
                    r12 = stack[-64];
                    r13 = r13;
            }
    }
    else {
            rax = [[NSString alloc] initWithFormat"ERROR: Text is empty, can't send"];
            stack[0] = "-[MMMessageSendLogic sendTextMessageWithString:mentionedUsers:]";
            [MMLogger logWithMMLogLevel:0x2 module:"ComposeInputView" file:0x103e0e162 line:0x10c func:stack[0] message:rax];
            [rax release];
            r14 = 0x0;
    }
    [r13 release];
    [r12 release];
    rax = r14 & 0xff;
    return rax;
}
这个伪代码看的就比拟分明了，

objc_unsafeClaimAutoreleasedReturnValue([[[r12 getService:[MessageService class]] retain] SendTextMessage:r15 toUsrName:r14 msgText:r13 atUserList:stack[-64]]);

咱们能够看到第一个参数是r15，网上追溯r15，

r15 = [[rax CurrentUserName] retain]; r15是这里赋值的，那末再看看CurrentUserName办法内容。

-(void *)CurrentUserName {
    if ([self isLoggedIn] != 0x0) {
            rdi = [[CUtility GetCurrentUserName] retain];
    }
    else {
            rdi = 0x0;
    }
    rax = [rdi autorelease];
    return rax;
}
能够看到是先判别是否是曾经登录，而后挪用CUtility类外面的GetCurrentUserName办法取得的。那末第一个参数咱们就晓得了。其他三个参数咱们也很简单的能够手动结构。咱们编写js剧本代码

7.编写frida剧本
console.log("init success");
function SendTextMessage(wxid, msg) {
    var message = ObjC.chooseSync(ObjC.classes.MessageService)[0]
    var username = ObjC.classes.CUtility.GetCurrentUserName();
    console.log(username)
    console.log("Type of arg[0] -> " + message)
    var toUsrName = ObjC.classes.NSString.stringWithString_(wxid);
    var msgText = ObjC.classes.NSString.stringWithString_(msg);
    message["- SendTextMessage:toUsrName:msgText:atUserList:"](username, toUsrName, msgText, null);
}
SendTextMessage("filehelper","自动挪用发送信息！")
将以上文本保管js文件，而后履行如下饬令：

frida  微信 --debug --runtime=v8 --no-pause -l  test.js

咱们就能够看到微信上发送了一条音讯

 it0365学破解逆向

8.音讯监听
下面咱们完成了微信音讯的窜改及自动发送功用。那末咱们再去看看微信是若何接到音讯信息的！每一当有人活或许群给咱们发送音讯的时分电脑或者手机上普通城市提醒告诉，那末告诉的英文是甚么？notify 翻译便是告诉的意义，咱们碰试试看看看能不克不及找到相干字样。仍是正在MessageService外面咱们找到了- (void)notifyAddMsgOnMainThreadid)arg1 msgDataid)arg2; 这个办法，若何去断定它究竟是否是尼？仍是持续用frida去停止考证。

frida-trace -m "-[MessageService notify*]" 微信
履行上述饬令

$ frida-trace -m "-[MessageService notify*]" 微信
Instrumenting...                                                        
-[MessageService notifyModMsgOnMainThread:msgData:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyModMsgOnMainThread_msgData_.js"
-[MessageService notifyAppMsgUploadProgress:msgData:uploadedBytes:totalBytes:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyAppMsgUploadProgress_msgDa_9b03499e.js"
-[MessageService notifyVideoMsgUploadProgress:msgData:uploadedBytes:totalBytes:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyVideoMsgUploadProgress_msg_e1db5f92.js"
-[MessageService notifyNewMsgNotificationOnMainThread:msgData:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyNewMsgNotificationOnMainTh_d56d83b5.js"
-[MessageService notifyChatSyncMsgsOnMainThread:msgList:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyChatSyncMsgsOnMainThread_msgList_.js"
-[MessageService notifyChatSyncMessagesMergedOnMainThread:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyChatSyncMessagesMergedOnMainThread_.js"
-[MessageService notifyRevokePatMsgOnMainThread:n64MsgId:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyRevokePatMsgOnMainThread_n64MsgId_.js"
-[MessageService notifyAddRevokePromptMsgOnMainThread:msgData:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyAddRevokePromptMsgOnMainTh_81637ebf.js"
-[MessageService notifyDelMsgOnMainThread:msgData:isRevoke:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyDelMsgOnMainThread_msgData_5bbc2297.js"
-[MessageService notifyMsgDeletedForSessionOnMainThread:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyMsgDeletedForSessionOnMainThread_.js"
-[MessageService notifyDelAllMsgOnMainThread:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyDelAllMsgOnMainThread_.js"
-[MessageService notifyAddMsgListForSessionOnMainThread:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyAddMsgListForSessionOnMainThread_.js"
-[MessageService notifyUnreadCntChangeOnMainThread:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyUnreadCntChangeOnMainThread_.js"
-[MessageService notifyMsgResendOnMainThread:msgData:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyMsgResendOnMainThread_msgData_.js"
-[MessageService notifyImgMsgUploadProgress:msgData:uploadedBytes:totalBytes:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyImgMsgUploadProgress_msgDa_e4e0cd43.js"
-[MessageService notifyAppMsgDownloadProgress:msgData:downloadedBytes:totalBytes:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyAppMsgDownloadProgress_msg_4e191704.js"
-[MessageService notifyUIAndSessionOnMainThread:withMsg:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyUIAndSessionOnMainThread_withMsg_.js"
-[MessageService notifyAddMsgOnMainThread:msgData:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyAddMsgOnMainThread_msgData_.js"
Started tracing 18 functions. Press Ctrl+C to stop. 
咱们能够看到有很多的办法被hook了，可是没事。咱们用微信发送一个音讯给本人或许其余人均可以看看输入。

           /* TID 0x307 */
157082 ms  -[MessageService notifyAddMsgOnMainThread:0x6503cfa934d442eb msgData:0x7fd903c9fa00]
           /* TID 0x31e17 */
157092 ms  -[MessageService notifyUnreadCntChangeOnMainThread:0x6503cfa934d442eb]
           /* TID 0xb5c27 */
157228 ms  -[MessageService notifyModMsgOnMainThread:0x6503cfa934d442eb msgData:0x7fd903c9fa00]
咱们能够看到三层相干的挪用，那末咱们就先看第一个notifyAddMsgOnMainThread 修正下js文件。

  onEnter(log, args, state) {
    log(`-[MessageService notifyAddMsgOnMainThread{args[2]} msgData{args[3]}]`);
  },
以咱们下面的经历很快的就能够看出这个该当便是音讯承受的办法，msgdata便是咱们所需求的音讯内容。那末咱们仍是患上持续考证。把参数都打印进去看看。修正增加以下js

console.log("Type of arg[2] -> " + new ObjC.Object(args[2]).$className)
console.log("Type of arg[3] -> " + new ObjC.Object(args[3]).$className)
这两句话是为了输入2个参数的范例。而后也修正下frida饬令履行

frida-trace -m "-[MessageService notifyAddMsgOnMainThread*]" 微信
能够看到第一个参数是String，第二个参数是MessageData

$ frida-trace -m "-[MessageService notifyAddMsgOnMainThread*]" 微信
Instrumenting...                                                        
-[MessageService notifyAddMsgOnMainThread:msgData:]: Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyAddMsgOnMainThread_msgData_.js"
Started tracing 1 function. Press Ctrl+C to stop.                       
Type of arg[2] -> NSTaggedPointerString
Type of arg[3] -> MessageData
           /* TID 0x307 */
  2170 ms  -[MessageService notifyAddMsgOnMainThread:0x6503cfa934d442eb msgData:0x7fd90401c960]
MessageData是音讯的构造体，那末咱们就去头文件中搜刮一下这个MessageData

# n @ localhost in ~/vscodewsp/wechat/dump [7:46:01] C:1
$ ll -l|grep MessageData        
-rw-r--r--  1 n  staff   2.5K  2 15 19:19 FTSFileMessageData.h
-rw-r--r--  1 n  staff   2.0K  2 15 19:19 FTSMessageData.h
-rw-r--r--  1 n  staff   794B  2 15 19:19 IMessageDataExt-Protocol.h
-rw-r--r--  1 n  staff   6.2K  2 15 19:19 MMChatMessageDataSource.h
-rw-r--r--  1 n  staff    25K  2 15 19:19 MessageData.h
-rw-r--r--  1 n  staff   550B  2 15 19:19 MessageDataGroup.h
-rw-r--r--  1 n  staff   2.9K  2 15 19:19 MessageDataPackedInfo.h
-rw-r--r--  1 n  staff   262B  2 15 19:19 NSPasteboard-MessageData.h
能够看到是有MessageData这个文件的。那末咱们翻开看看

@interface MessageData : NSObject <NSPasteboardItemDataProvider, IAppMsgPathMgr, IMsgExtendOperation, NSCopying, WCTTableCoding, WCTColumnCoding>
 {
     unsigned int mesLocalID;
    long long mesSvrID;
    NSString *fromUsrName;
    NSString *toUsrName;
    unsigned int messageType;
    NSString *msgContent;
    NSString *msgVoiceText;
    unsigned int m_uiVoiceToTextStatus;
    unsigned int msgStatus;
    unsigned int msgImgStatus;
    NSString *msgRealChatUsr;
    NSString *msgPushContent;
    unsigned int m_uiTranslateStatus;
    NSString *msgSource;
    unsigned int mesDes;
    unsigned int msgSeq;
    BOOL bForward;
    NSData *m_dtThumbnail;
    unsigned int msgCreateTime;
    unsigned int m_uiSendTime;
    unsigned int m_uiDownloadStatus;
    id <IMsgExtendOperation> m_extendInfoWithMsgType;
    id <IMsgExtendOperation> m_extendInfoWithFromUsr;
    BOOL isAutoIncrement;
    BOOL m_bShouldShowAll;
    BOOL m_bIsMultiForwardMessage;
    BOOL m_shouldReloadOriginal;
    BOOL m_bHasOriginalMessage;
    unsigned int IntRes1;
    unsigned int IntRes2;
    unsigned int m_uiFileUploadStatus;
    unsigned int m_uiOriginalImgHeight;
    unsigned int m_uiOriginalImgWidth;
    unsigned int m_uiSrcCreateTime;
    unsigned int _m_nsMsgCrc32;
    unsigned int _m_uiUploadedBytes;
    unsigned int _m_uiDownloadedBytes;
    unsigned int _m_uiTotalBytes;
    int _m_nCdnServerRetCode;
    unsigned int _m_uiResendMessageCount;
    long long lastInsertedRowID;
    NSString *StrRes1;
    NSString *StrRes2;
    MMTranslateResult *m_nsTranslationResult;
    NSString *m_nsFilePath;
    NSString *m_nsVideoPath;
    NSString *m_nsVideoThumbPath;
    NSString *dataMd5;
    MessageData *m_refMessageData;
    MessageDataPackedInfo *m_packedInfo;
    NSString *m_nsSrcUserName;
    NSString *m_nsSrcNickName;
    NSString *m_nsAtUserList;
    NSString *_m_nsImgFileName;
    NSString *_m_nsBigFileErrMsg;
    SecondMsgNode *_secondMsgNode;
    MessageData *_referHostMsg;
}


                var MessageData = new ObjC.Object(args[3]).$ivars;
    console.log("fromUsrName -> " + MessageData.fromUsrName)
    console.log("toUsrName -> " + MessageData.toUsrName)
    console.log("msgContent -> " + MessageData.msgContent)
运转frida-trace -m "-[MessageService notifyAddMsgOnMainThread*]" 微信

- [MessageService notifyAddMsgOnMainThread:msgData:]:
Loaded handler at "/Users/n/vscodewsp/wechat/__handlers__/MessageService/notifyAddMsgOnMainThread_msgData_.js"
Started tracing 1 function. Press Ctrl+C to stop.                       
Type of arg[2] -> NSTaggedPointerString
Type of arg[3] -> MessageData
fromUsrName -> wxid_pk1reltk63i822
toUsrName -> filehelper
msgContent -> 监听部分
           
           /* TID 0x307 */
 14909 ms  -[MessageService notifyAddMsgOnMainThread:0x6503cfa934d442eb msgData:0x7fd904426980]

#include "pch.h"

#include <iostream>

#include <Windows.h>

#include <openssl/rand.h>

#include <openssl/evp.h>

#include <openssl/aes.h>

#include <openssl/hmac.h>



using namespace std;



#pragma comment(lib, "ssleay32.lib")

#pragma comment(lib, "libeay32.lib")



#if _MSC_VER>=1900

#include "stdio.h" 

_ACRTIMP_ALT FILE* __cdecl __acrt_iob_func(unsigned);

#ifdef __cplusplus 

extern "C"

#endif 

FILE* __cdecl __iob_func(unsigned i) {

  return __acrt_iob_func(i);

}

#endif /* _MSC_VER>=1900 */







#undef _UNICODE

#define SQLITE_FILE_HEADER "SQLite format 3" 

#define IV_SIZE 16

#define HMAC_SHA1_SIZE 20

#define KEY_SIZE 32



#define SL3SIGNLEN 20

#ifndef ANDROID_WECHAT

#define DEFAULT_PAGESIZE 4096       //4048数据 + 16IV + 20 HMAC + 12

#define DEFAULT_ITER 64000

#else

#define NO_USE_HMAC_SHA1

#define DEFAULT_PAGESIZE 1024

#define DEFAULT_ITER 4000

#endif



//pc端密码是经过OllyDbg得到的64位pass,是64位，不是网上传的32位，这里是个坑

unsigned char pass[] = { 0xc7,0x99,0x26,0xc0,0x36,0x6b,0x4f,0xee,0xb8,0xc7,0x48,0x83,0xa

* ![图片](https://user-images.githubusercontent.com/79394963/200308150-bf2bbf78-c57c-4154-ba2f-6c77a6dedb3a.png)

这两个函数是收发包的核心函数，如果我们能够hook到这里，基本上消息的接收是可以完全监控到的。
2.如何找到这两个函数？
   找这两个函数大概有很多种办法，但是我个人觉得最简单的就是 htonl 和 ntohl。这两个函数是导出函数，封包需要用好几次，解包也用好几次，
   相信打开IDA，找到这种一个函数用好几次htonl的函数不是很多，挨个下断点就能够hook出来了。
   比如 2.6.2.31这个版本 封包解包的地址  ： 0x9A6579  0x9A6664

这两个函数是收发包的核心函数，如果我们能够hook到这里，基本上消息的接收是可以完全监控到的。
2.如何找到这两个函数？
   找这两个函数大概有很多种办法，但是我个人觉得最简单的就是 htonl 和 ntohl。这两个函数是导出函数，封包需要用好几次，解包也用好几次，
   相信打开IDA，找到这种一个函数用好几次htonl的函数不是很多，挨个下断点就能够hook出来了。
   比如 2.6.2.31这个版本 封包解包的地址  ： 0x9A6579  0x9A6664

* ![图片](https://user-images.githubusercontent.com/79394963/200308384-8fdcab67-408f-4148-87c1-e71ca8d8e907.png)

* ![图片](https://user-images.githubusercontent.com/79394963/200308477-7ee4cf3c-0813-4baa-9708-3c3d9e59844a.png)
解开了包，就可以用protobuf反序列化，这样我们就能监控到聊天消息了。

4.那如何发送消息？
  1.可以通过hook发送函数来发。
     可以在微信的客户端启动一个PRCServer，然后我们的AI负责把回复消息发送给里面的线程，负责发送，这种Call发送虽然比较死。
  2.可以自己构建包来发。
    发送一个包需要什么，proto文件，这个很好弄。 
    如何封包？
    根据hook pack函数我们揭秘一个就好了。
    拿到version，uin，cookies，然后最后几位标志。
    
  3.怎么发送出去?
     我们走短连接就好了，然后用早期的版本，因为早期的低版本可以关闭mmtls，所以我们在登录以后拿到一个短连接的ip
     封包发送就好了。
   
* ![图片](https://user-images.githubusercontent.com/79394963/200308761-2d9c3bd7-5388-4e19-940c-b0ea0e6d418d.png)

概要總結：

到這裡我們就用靜態方式去破解微信，知道了數據庫加密的密碼，然後看到他是使用的主流的數據庫加密框架：sqlitecipher，而且現在很多app都用這個框架，比如一些小說類的app，這裡就不指定說是誰了，我之前反編譯過幾個小說類的app，有兩個都是用的這個框架進行加密的。

五、破解流程總結

1、猜想信息是保存在本地數據庫

想得到聊天記錄和通訊錄信息，我們的想法是這些信息在設備沒有連接網絡的時候也是可以查看的，所以我們猜想這些信息是保存在本地的數據庫中的。

2、使用sqlite工具查看信息報錯

我們把微信的沙盒數據全部導出到本地，然後查找db文件，找到了EnMicroMsg.db文件，使用sqlite expert工具進行查看報錯，提示數據庫被加密了。

3、根據數據庫的常規使用流程找到入口

我們在Android中使用數據庫的時候都會用到SQLiteDatabase類，所以可以全局搜索這個類，找到這個類的定義，然後再找到他的一些open方法，看看這些方法的調用地方。

4、通過數據庫的入口方法進行代碼跟蹤

知道了open系列方法的調用地方，就開始使用Jadx工具進行代碼跟蹤，最後跟蹤到了有利信息，就是密碼是用戶設備的IMEI+uin值計算MD5值，注意是小寫字符，然後在取MD5的前7位字符構成的密碼。

5、獲取密碼流程

這裡知道了密碼的構成，獲取就比較簡單了，使用*#06#撥號直接獲取IMEI值，然後在去查看SharedPreferences中的auth_info_key_prefs.xml文件中的_auth_uin值就是用戶的uin值，然後進行拼接，使用HashTab計算出MD5值，獲取前7位字符串。

6、使用sqlcipher工具查看數據庫

得到密碼之後，使用sqlcipher工具進行數據庫的查看，可以找到通訊錄表格recontact和聊天記錄表格message。

概要：微信的核心數據庫是EnMicroMsg.db，但是是加密操作的，而加密的密碼是設備的IMEI+用戶的UIN值(在SP中保存了)，計算MD5(字符是小寫)，取出前7位字符即可。

六、延展

1、微信的通訊錄信息和聊天記錄信息對於一個用戶來說是非常重要的隱私，所以這也是微信對數據庫進行加密的原因，但是不管最後怎麼加密，都會需要解密的，所以這就是我們破解的關鍵，只要解密操作在本地進行，密碼肯定能夠獲取到。

2、關於微信的這個密碼獲取的規則不會發生改變的，有的同學會想微信會改掉數據庫加密的密碼獲取算法嗎？答案是不會的，原因很簡單，如果密碼算法改了，就會影響到老用戶，比如新版本中密碼改了，老用戶更新之後，在讀取數據庫的時候進行解密，那麼加密的數據庫是老的，新的加密算法是解密失敗的，這個用戶體驗會瘋的，那有的同學又說了，對數據庫進行升級處理，但是這裡的升級一定要保證老的數據不能丟失，那麼這裡就存在一個老數據遷移的工作，這個工作是巨大的，因為現在很多微信使用的過程中如果不去主動的清除數據，聊天信息非常多，那個數據庫也會非常的大，幾十M很正常的，那麼在數據遷移的時候風險是非常巨大的，所以這樣一來，微信短期內是不會改變密碼算法規則的，其實我已經嘗試了很多老版本的反編譯，發現的確這個算法一直都是這樣的。所以一定要記住微信的數據庫加密算法是：MD5(IMEI+UNI)=>前7個字符。

3、這裡為什麼使用靜態方式去分析呢？原因是微信的包太大了，如果動態調試的話總是出現死機情況，沒辦法後續操作了，所以使用了靜態方式去破解。

七、安全性

通過本文之後，大家應該都知道如何破解微信的聊天記錄信息和通訊錄信息了，只要獲取到加密的數據庫，得到密碼即可，但是這兩步卻不是那麼容易獲取的，首先如何獲取加密的數據庫，這些信息都是保存在微信的沙盒數據中的，所以得設備root之後獲取，設備的imei信息就簡單了，那麼問題就來了，如果一個用戶的設備root了，那麼惡意程序就可以開始盜取信息了。而且在之前的一篇文章中：Android中allowBackup屬性引發的安全問題 介紹了微信在5.1之前的版本allowBackup屬性默認值是true，也就是說沒有root的情況下，可以獲取到微信的沙盒數據，那麼這個安全性就太暴露了。現在也有很多微信通訊錄備份的工具，其實就是把這個數據庫信息同步一下。以後只要有了微信的這個數據庫，那麼破解也是很簡單的，因為密碼是規定的。

八、用途

1、如果你想看周邊的人微信信息，那麼這裡就是給你提供了最好的方案，特別是你最愛的人，比如媳婦總是不讓你看她微信，但是自己又很想看，那麼機會來了。

2、對於root之後的設備可以在後台竊取用戶的微信信息把imei一起上傳到服務端，然後在人工分析獲取聊天記錄中重要信息。

3、當我們的設備中誤刪除了聊天記錄，這時候可以通過導出本地數據庫，然後使用sqlcipher工具進行查看既可以找到之前數據

本文的重點和意圖是：如何使用靜態方式破解apk的思路，而對於微信來說，這個不算是漏洞也不算是問題，因為我們上面介紹中涉及到的數據都是微信的沙盒中的，所以一般情況下是無法獲取到的，所以對於攻擊者來說沒有太多的意義。所以本文的意圖很簡單就是講解靜態方式破解apk的一種思路。
LLDB Remote Debug
上面是静态分析，下面讲的就是动态调试了。

Debug Server 配置
debugServer 顾名思义，这个是用来做远程调试服务器的，而这个文件其实大伙儿已经有了。

我们有两个途径拿到他：

用 scp 从 iDevice /Developer/usr/bin/debugserver 目录下获得

从 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/ DeviceSupport/7.0.3\ \(11B508\)/DeveloperDiskImage.dmg 获得，中间的那个 DeviceSupport 可以根据具体路径替换。

拿到它之后并不能直接使用，因为默认是没有权限的。

我们要给他 task_for_pid 的权限，书写 entitlements.plist 如下：

表示我们 lldb 已经挂载成功了

Address 断点
正常的 App Store 项目中，是没有符号的。所以我们常用的符号断点是不行的，但是我们可以给函数的内存地址下断点。

Class-dump
我们先要知道目标函数的地址是什么。这个就需要 Class-Dump 了。

Class-Dump XApp -H -A -S -o headers/
这句话翻译一下，把 XApp dump出头文件，并标记 IMP 的地址，方法排序输出到 headers 文件夹。

我们在 headers 中就可以找到刚才说的 NSString-Extension.h。

#import "NSString.h"
@interface NSString (Extension) //节选

- (id)md5DecodingString;    // IMP=0x000000010024b99c
- (id)md5StringFor16;    // IMP=0x000000010024b9b8
- (id)objectFromJSONString;    // IMP=0x000000010024d078

@end
于是 IMP 的地址就是: 0x000000010024b99c

获取 ASLR 偏移量
ASLR（Address space layout randomization）是一种针对缓冲区溢出的安全保护技术，通过对堆、栈、共享库映射等线性区布局的随机化，通过增加攻击者预测目的地址的难度，防止攻击者直接定位攻击代码位置，达到阻止溢出攻击的目的。据研究表明ASLR可以有效的降低缓冲区溢出攻击的成功率，如今Linux、FreeBSD、Windows等主流操作系统都已采用了该技术。

我们通过 image 方法获取偏移量

image list -o -f
得到

[  0] 0x00000000000f8000 /var/mobile/Applications/27C2DBB3-34A3-4B71-91F3-EBB81F4E48E8/XApp.app/XApp(0x00000001000f8000)
其中，0x00000000000f8000 就是我们想要的偏移量。

断点
终于跑到了最后一步，根据我们拿到的地址下断点：

br s -a "0x00000000000f8000+0x000000010024b99c"
然后提示：

Breakpoint 1: where = XApp`___lldb_unnamed_function11956$$XApp, address = 0x000000010034399c
表示断点下完了，我们执行 c 或 continue 让 app 跑起来，触发一个网络请求，就会发现断点停下来的。

* thread #1: tid = 0x17f0, 0x000000010034399c XApp`___lldb_unnamed_function11956$$XApp, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x000000010034399c XApp`___lldb_unnamed_function11956$$XApp
XApp`___lldb_unnamed_function11956$$XApp:
->  0x10034399c <+0>:  mov    x8, x0
    0x1003439a0 <+4>:  adrp   x9, 1160
    0x1003439a4 <+8>:  ldr    x0, [x9, #3368]
    0x1003439a8 <+12>: adrp   x9, 1152
由于 OC 方法 $arg1 就是 self，这里的 self 是 NSString，所以仅需

po $arg1
就得到了 MD5 之前的字符串:

/api/recommended/getapp_id=1002client_info={
  "channel" : "0",
  "os" : "iOS",
  "bssid" : "90:72:40:f:e5:a9",
  "model" : "iPad4,4",
  "ssid" : "TieTie Tech 5GHz",
  "start_time" : "1457361686666",
  "version" : "7.1.2",
  "resume_time" : "1457361686666"
}curidentity=0expectId=1148692page=2pageSize=15req_time=1457362529031sortType=1t=BAWEEFQUzATtTYlE2Cz4BZw07UGVXZwZjXDYNNA..uniqid=22B8980704196B96AB30264A2FE2AA46v=4.2bf646f6f09c07e911a6239780ea1b7df
断点变 Log
Xcode 中的 Action 断点，在命令行下也是可以玩的：

breakpoint command add 1

> po $arg1
> c
> DONE
这样，我们每次出发网络请求就不用手动打印并继续了。

# 总结部分代码

  有监听功能少用微信。这些APP都是FW的产物，没有了竟争，再奇葩的规则也不奇怪

  
