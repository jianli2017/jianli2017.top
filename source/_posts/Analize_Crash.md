---
title: 收集、符号化IOS崩溃日志
date: 2016-10-17 12:18:26
categories: IOS Crash
tags: Crash 
---
由于代码的缺陷，我们千辛万苦发布发布出去的APP，在用户手中，偶尔会出现Crash现象。为了及时查找到Crash的原因，我们需要收集这些Crash信息。本文就是从分析系统Crash日志开始，一直到定制个性化的Crash日志收集系统，一步步的说明如何完成Crash日志收集系统。
<!--more-->
千里之行，始于足下，我们先从系统Crash日志开始。
<!--合理的应用收集、解析崩溃日志的方法，有N种方式可以完美的消灭大部分的Crash。为了便于理解，我从CrashReporter捕获的标准Crash日志结构、系统Crash收集解析、 定制的Crash报告收集分析。
若了解CrashReporter收集的崩溃日志的各个部分的含义，对消除Crash应该大有裨益。-->
# 系统Crash日志结构分析


系统的Crash日志相信大家都见过，但不一定都认真分析过，所以我想，有必要重新对它做个认识。先贴出一张Crash日志截图，方便大家认识我啊~~~。

![收集、解析IOS崩溃日志3](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BC%8F3.png)

Crash日志内容非常，截图只能展示一部分内容，后续会依次贴出每个模块的内容，分别讲解。一个Crash日志由六部分组成：基本信息模块、系统信息模块、异常信息模块、线程回溯、CUP寄存器信息、镜像信息。

## 基本信息模块

	///崩溃报告的唯一标识符，标识不同的Crash
	Incident Identifier: 2964A813-5C6B-4E43-B7FD-40965E97F720   
	CrashReporter Key:   d280e6d8a2446a3c34436cf9d1c6e98448d0c5ca
	///代表发生Crash的设备类型
	Hardware Model:      iPhone7,1    
	///Crash的APP名称、APP进程id       
	Process:             Simple-Example [2279]  
	///Crash的APP在设备上的存储路径 
	Path:                /private/var/containers/Bundle/Application/A86C469E-5A97-4948-8BBF-98C0B814088E/Simple-Example.app/Simple-Example    
	///APP 的描述符       
	Identifier:          com.gome.gomeEShop 
	///APP的版本   
	Version:             1.0 (1.0)   
	///代码的构架，可以通过file命名查看代码的编译构架，
	///了解更过的构架信息可以到IOS SDK路径下的/usr/include/mach/machine.h文件中查找  
	Code Type:           ARM-64 (Native)     
	Role:                Foreground
	///父进程
	Parent Process:      launchd [1]     
	Coalition:           com.gome.gomeEShop [1008]
	
## 系统信息模块

系统信息模块包含Crash时间、 APP启动时间、OS 版本信息、Crash日志的格式 

	///Crash发生的时间
	Date/Time:           2016-10-17 16:11:43.7670 +0800
	///Crash的APP的启动时间   
	Launch Time:         2016-10-17 16:11:40.0612 +0800  
	///系统版本，（）内的数字代表的是Bulid号 
	OS Version:          iPhone OS 10.0.2 (14A456)  
	Crash日志的格式，一般为104  
	Report Version:      104 
	 
## 异常信息模块

异常信息模块包含异常类型（Mach异常、Unix信号异常）、异常子类型、异常原因、异常线程ID。

	///异常类型
	Exception Type:  EXC_BAD_ACCESS (SIGSEGV)  
	///异常子类型、异常的地址
	Exception Subtype: KERN_INVALID_ADDRESS at 0xffffffffffffffff   
	Termination Signal: Segmentation fault: 11
	///异常原因（非常重要）
	Termination Reason: Namespace SIGNAL, Code 0xb
	Terminating Process: exc handler [0]
	///发生异常的线程ID
	Triggered by Thread:  0    
	
异常类型(Exception Type)由两部分构成：Mach异常、Unix信号异常。 

苹果系统有一个微内核，叫做[XNU](http://opensource.apple.com/source/xnu/)，它的源码可以在opensource上载到。Mach是XNU的核心，因而，Mach异常就指Mach内核异常。Mach包含三部分内容：thread，task，host。后续的章节中很多地方都会用到Mach。不妨移步到[Mach IPC Interface](http://web.mit.edu/darwin/src/modules/xnu/osfmk/man/)，了解下Mach暴露给用户的API。

Mach暴露给了用户部分API，允许用户和内核交互。用户态的开发者可以通过Mach API设置thread、task、host的异常端口，来捕获Mach异常，抓取Crash事件。

Mach异常包括：

	#define EXC_BAD_ACCESS		1	/* Could not access memory */
		/* Code contains kern_return_t describing error. */
		/* Subcode contains bad memory address. */
	
	#define EXC_BAD_INSTRUCTION	2	/* Instruction failed */
		/* Illegal or undefined instruction or operand */
	
	#define EXC_ARITHMETIC		3	/* Arithmetic exception */
		/* Exact nature of exception is in code field */
	
	#define EXC_EMULATION		4	/* Emulation instruction */
		/* Emulation support instruction encountered */
		/* Details in code and subcode fields	*/
	
	#define EXC_SOFTWARE		5	/* Software generated exception */
		/* Exact exception is in code field. */
		/* Codes 0 - 0xFFFF reserved to hardware */
		/* Codes 0x10000 - 0x1FFFF reserved for OS emulation (Unix) */
	
	#define EXC_BREAKPOINT		6	/* Trace, breakpoint, etc. */
		/* Details in code field. */
	
	#define EXC_SYSCALL		7	/* System calls. */
	
	#define EXC_MACH_SYSCALL	8	/* Mach system calls. */
	
	#define EXC_RPC_ALERT		9	/* RPC alert */
	
	#define EXC_CRASH		10	/* Abnormal process exit */
	
	#define EXC_RESOURCE		11	/* Hit resource consumption limit */
	
Unix信号：信号是通知进程已发生某种情况的软中断技术。例如：某个进程执行了除法操作，其除数为0，则将名为SIGFPE（浮点异常）的信号发送给该进程。

那么，怎么会有两种异常信息呢？

念茜的[漫谈iOS Crash收集框架](http://www.cocoachina.com/cms/wap.php?action=article&id=12301)阐述了两者的关系，我这里再重复下。

苹果系统是基于Unix系统的，苹果的大牛们为了兼容Unix信号，将Mach异常转化为Unix信号，并投射到异常的线程，这样做的目的是：对于不懂Mach异常的人，也可以使用Unix信号捕获异常。所以，Crash日志有两种异常信息。

Mach和Unix关系图：
![收集、解析IOS崩溃日志4](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%974.png)
所有Mach异常都在host层被ux_exception转换为相应的Unix信号，并通过threadsignal将信号投递到出错的线程。<!--iOS系统中的 POSIX API 就是通过 Mach 之上的 BSD 层实现的。-->

捕获Mach异常或者Unix信号都可以抓到crash事件，这两种方式哪个更好呢？ 
优选Mach异常，因为Mach异常的处理会先于Unix信号处理，如果Mach异常的handler让程序exit了，那么Unix信号就永远不会到达这个进程了。<!--转换Unix信号是为了兼容更为流行的POSIX标准，这样不必了解Mach内核也可以通过Unix信号的方式来兼容开发。-->

所以，Crash日志中的EXC_BAD_ACCESS 是Mach异常信息，SIGSEGV是Unix信号异常信息。

小贴士:
因为硬件产生的信号(通过CPU陷阱)被Mach层捕获，然后才转换为对应的Unix信号；苹果为了统一机制，于是操作系统和用户产生的信号(通过调用kill和pthread_kill)也首先沉下来被转换为Mach异常，再转换为Unix信号。

## 线程回溯
### 符号化回溯线程
线程的回溯是APP Crash瞬间，程序中所有线程的逆向调用堆栈。线程回溯对我们修复Crash非常非常的有用，根据线程回溯，可以分析、定位程序崩溃的原因。

下面将崩溃的代码、未符号化崩溃日志、符号化崩溃日志贴出来，做个对比性的理解。


		@implementation ViewController
		
		- (IBAction) onCrash:(__unused id) sender
		{
		    char* ptr = (char*)-1;
		    *ptr = 10;  ///这里程序崩溃了 
		}
		
		@end
未符号化的崩溃日志（图5）
![收集、解析IOS崩溃日志5](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%975_1.png)
符号化的崩溃日志（图6）
![收集、解析IOS崩溃日志6](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%976.png)

图5中红色文字展示了几个名词：镜像文件、加载地址、堆栈地址。还有没有展示出来的一个名词：符号在二进制中的偏移量。他们的含义分别为：

镜像文件：是可执行二进制文件和二进制文件依赖的动态库的总称。
堆栈地址：是代码在内存中执行的内存地址。
镜像的加载地址：程序执行时，内核会将包含程序代码的镜像加载到内存中，镜像在内存中的基地址就是加载地址。程序每次启动时，镜像的加载地址是随机的。所以，同一代码在不同的设备中执行时，堆栈地址是不一样的。
符号在二进制中的偏移量：按照字面意思理解吧。它以通过下面的公式得到：

	符号在二进制中的偏移量 = 堆栈地址 - 镜像的加载地址  
	
符号在二进制中的偏移量非常有用，我们就是根据它，从符号文件中查找出地址对应的代码符号。这里的符号文件指的是：带有符号表的可执行二进制文件、dSYM文件，这两种文件在后续章节中都统称为符号文件。

那么怎么将图5中的Crash日志符号化为图6中的形式呢？

苹果自带的atos命令行工具可以查找地址对应的符号，在终端中输入：

	/usr/bin/atos -o [符号文件] -arch arm64 -l 0x100030000 0x000000010003522c 
输出结果如下：

	-[ViewController onCrash:] (in Simple-Example) (ViewController.m:10)
	
是不是很简单的就将地址转换为符号？是的，只需将符号文件（-o指定）、代码构架（-arch指定）、加载地址（-l指定）、堆栈地址 传入atos命令，就能解析出符号。 atos命令解析出了堆栈地址为0x000000010003522c、加载地址为0x100030000对应的符号。符号为[ViewController onCrash:]，也验证了崩溃发生在onCrash函数中，也验证了崩溃日志中的地址是可以符号化的。

符号化是简单，但是原理是什么？怎么就通过地址找到了Crash代码的符号，要听详细信息，请看《符号化内幕》。

### 符号化内幕

符号化的内幕就是：在符号文件中，通过偏移量查找符号。下面，一步步的来分析，首先计算Crash地址在符号文件中的偏移量，为000000010000522c。

	符号在二进制中的偏移量 = 堆栈地址 - 镜像的加载地址 = 0x000000010003522c -  0x100030000 = 000000010000522c

在符号文件中直接找地址000000010000522c，应该是找不到，在后续你可以理解。我们使用逆向方法，根据符号-[ViewController onCrash:]，找对应的地址，比较是不是000000010000522c，如果是，就充分说明了，通过偏移量是可以查找到内存地址对应的符号的。在终端中输入下面的命令：

	nm [符号文件] | grep "ViewController onCrash:"

输出如下 

	00008320 t -[ViewController onCrash:]
	0000000100005224 t -[ViewController onCrash:]
	
输出的第一行是armv7s构架的符号，第二行是arm64构架的符号，Crash日志显示的代码构架是arm64，使用第二行，符号-[ViewController onCrash:]对应的偏移量是0000000100005224，而不是 000000010000522c，是我给你埋了个坑吗？专门来坑你的？不是的，不是的，这个问题也困扰了我N久、折磨了我N久。这个公式是在stack overflow上找到的，我也怀疑他们骗了我，但是，仔细看两个地址，又那么的相似、那么的相近---就相差8！！！ 虽说相差8，但毕竟不一样。差之毫厘谬以千里啊。感觉就要成功，但就是不对。就永远差了那么一点点，试了好多崩溃日志，都是差那么一点点，相信你也体会过这种感觉，只差一点点，真的只差一点点。就差那么该死的一点点，问题就能解决。百思不得其姐的困扰、抓狂的困扰 。。好久好久我都没想明白，那一点点是怎么差的。这不，今天写日志组织测试用例的时候，忽然明白了为什么差那一点点，踏破铁鞋无觅处，得来全不费功夫！！。 
原来，我们通过nm命令查找出的符号地址对，是函数入口地址和对应的函数调用的符号对，仅仅是函数调用的符号，没有函数内部代码的符号，而程序是崩溃到函数内部，崩溃到*ptr = 10这句话，内部代码的地址怎么可能和入口地址一样呢！相差一点点！ 
下面根据偏移量000000010000522c和代码推算函数的入口地址吧，看看是什么。崩溃代码*ptr = 10前面只有一个语句---定义初始化指针“char* ptr = (char*)-1”，在64位系统上指针的地址占8个字节，000000010000522c - 8= 0000000100005224， 果然是0000000100005224。这个不就是函数的入口地址嘛，对，就是。原来那一点点的原因在这里。 那么偏移量0000000100005224 对应的符号正是-[ViewController onCrash:]。<!--这就是通过偏移量查找符号的原理。--> 

上面通过nm 命令查找符号可能不直观，可以通过可视化工具[MachOView](http://of685p9vy.bkt.clouddn.com/MachOView-2.4.9200.dmg)查看。验证下吧，选择 Debug Symbols（ARM64_ALL）->Symbol Table->Symbols,然后在右上角的搜索框中输入符号：-[ViewController onCrash:]，结果如下，
![收集、解析IOS崩溃日志7](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%977.png)
通过这个工具可以直观的查看到符号和地址的对应关系。

### 小小结
ok，终于可以歇一歇了，我们终于把符号化和符号化原理阐述完了。简单的回顾下：

1. 可以通过系统的atos符号化崩溃日志的单个符号 
2. 符号化内部原理就是：根据符号在二进制中的偏移量，在符号文件中查找对应的符号。其中：符号在二进制中的偏移量 = 堆栈地址 - 镜像的加载地址。

<!--### 1.4.4 弱弱的问题

1. 为什么图5中的回溯线程中，Simple-Example镜像中的地址“0x000000010003522c 0x100030000 + 21036”没有解析出来，显示的是一串十六进制的数字，你是否也遇到过？也被这样的问题困惑过？《1.6小节 Binary Images》， 会阐述该问题。-->
               
## 线程的状态寄存器 

		Thread 0 crashed with ARM Thread State (64-bit):
		    x0: 0x000000010050b460   x1: 0x0000000100102cea   x2: 0x00000001004339d0   x3: 0x00000001740f8f00
		    x4: 0x00000001740f8f00   x5: 0x00000001740f8f00   x6: 0x0000000000000001   x7: 0x0000000000000000
		    x8: 0xffffffffffffffff   x9: 0x000000000000000a  x10: 0x00000001b3ad0018  x11: 0x00c1580100c15880
		   x12: 0x0000000000c15800  x13: 0x0000000000c15900  x14: 0x0000000000c158c0  x15: 0x0000000000c15801
		   x16: 0x0000000000000000  x17: 0x00000001000c1224  x18: 0x0000000000000000  x19: 0x00000001740f8f00
		   x20: 0x00000001004339d0  x21: 0x0000000100102cea  x22: 0x000000010050b460  x23: 0x0000000170240bd0
		   x24: 0x000000017400db90  x25: 0x0000000000000001  x26: 0x0000000000000000  x27: 0x00000001b2822000
		   x28: 0x0000000000000040   fp: 0x000000016fd41ab0   lr: 0x0000000194aea7b0
		    sp: 0x000000016fd41a90   pc: 0x00000001000c122c cpsr: 0x60000000
    
这是APP crash的时候，ARM64 构架CPU的32个寄存器的值， 其中fp 帧指针、sp堆栈指针，lr 是返回地址指针，这三个都比较有用，用来逐级回溯线程调用栈。

## Binary Images 

图8 
![收集、解析IOS崩溃日志8](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%978.png)

镜像文件就是上面讲的可执行程序 和 依赖的所有动态库。
镜像文件中包括镜像的加载地址，和线程回溯中的镜像加载地址指的是一个地址。加载地址后面有个UUID，符号文件中也有个UUID，只有这两个地址一致，才能解析出地址对应的符号。符号文件中的UUID可以通过终端中输入下面的命令得到：
	
	dwarfdump —u [符号文件]
	
输出如下：
	
		UUID: C8E0E6E4-F761-3A19-B231-A31C1BB9037A (armv7) 
		UUID: 39BBB8F4-CCB0-3193-8491-C007931CA05E (arm64) 
第二行的arm64构架的UUID居然和图8中的红色矩形框中UUID惊人的一致。是的。必须得一致，这才表示代码对应的符号能在这个符号文件中找到，如果不一致，就没法解析出地址对应的符号。不论是Xcode，还是symbolicatecrash，都解析不了。
也可以通过MachOView查看符号文件的UUID，结果如下：
![收集、解析IOS崩溃日志9](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%979.png)

## 小结
这节阐述了崩溃日志的组成结构、通过atos命令行工具符号化崩溃日志以及符号化崩溃日志的原理。同时提及了几个工具有用的工具：
1. file,文件类型显示工具（The file-type displaying tool，位于/usr/bin/file）;
2. atos,(将数字地址转换为镜像或可执行程序中的符号工具，convert numeric addresses to symbols of binary images or processes，位于/usr/bin/atos);
3. nm，（符号表展示工具，The symbol table display tool,位于 /usr/bin/nm）;
4. 可视化查看Mach-O工具，MachOView。

# 系统收集、符号化Crash日志

系统如何收集、符号化Crash日志有多种方式，主要有如下几种方式：

1. CrashReporter收集、Xocde或symbolicatecrash符号化。当iOS系统上的某个 APP崩溃时，IOS系统自带的CrashReporter会创建一份crash日志保存在设备上。如果能拿到Crash的手机，就可以通过Xcode或symbolicatecrash符号化Crash日志。
2. 第三方SDK。如友盟，鹅厂的[Bugly](http://bugly.qq.com) 等等  Crash
3. 打造自己的收集、符号化程序。主要方法：使用NSSetUncaughtExceptionHandler注册异常处理函数，当APP 发生Crash时刻，回调到异常处理函数，在异常处理函数中收集Crash信息，然后上传到服务器；当需要分析的时候，从服务器取回Crash日志，如果没有符号化，使用atos命令符号化。
4. 开源框架KSCrash。如果上面的几种收集、符号化的方式依然不能满足你的需求，那么完备的KSCrash框架应该是一个不错的选择。

这章只阐述第一种方式，后续的第三章、第四章分别阐述第三、四种方式。

## CrashReporter收集日志

CrashReporter 是IOS自带的工具，当APP发生崩溃时，CrashReporter会创建一份Crash日志并保存到设备上。上一章阐述的Crash日志，就是出自CrashReporter之手。

可以使用Xcode、iTool导出CrashReporter创建的日志。其中：Xcode导出Crash日志的方法如下：
![导出崩溃日志](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BC%8F2.png)
在Xcode->Window菜单->Devices，弹出的设备面板，选择崩溃的设备 -> 选择右侧的View Device Logs->选中导出的日志，右击，选中export log, 导出.crash后缀的崩溃日志

## Xcode 解析Crash日志
Xcode可以将日志中的地址信息符号化为代码中的符号，但有个前提条件：crash log和dSYM或APP携带的UUID一致。crash log携带的UUID指的是镜像的UUID。这两种UUID在《1.6 Binary Images 》 已经阐明。

- 如果APP是自己电脑编译生成的，Xcode会根据spotlight自动找到对应的符号文件
- 如果不是自己电脑编译生成的，只需要将.app和dSYM放入同一文件夹，然后手动生成索引，这样Xcode也能找到。在命令行中输入如下命令手动创建索引：
	
		mdimport pathName

mdimport ,导入文件到datastore（import file hierarchies into the metadata datastore）。
上面两种方式确保了Xcode能依据UUID找到地址对应的符号文件，这样，Xcode就能解析出崩溃日志。<!--而且Xcode可以批量解析崩溃日志，比symbolicatecrash好用多了。-->

使用Xcode解析崩溃日志的方法：在Xcode->Devices->View Device Logs中，查看设备的所有崩溃日志，如果能解析，Xcode会自动解析崩溃日志。这种方式可以实现批量解析崩溃日志 
![XCode导出崩溃日志](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BC%8F1.png)
Xcode解析崩溃日志的优势：
1. 批量解析，可以一次解析出所有可解析的Crash日志。
2. 稳定性比symbolicatecrash好。网上说symbolicatecrash经常出现解析不出来，解析出错的问题。但是我没有遇到过。

## 那些年我们一起使用的符号化工具

你是不是也在网上搜过“IOS崩溃解析”？网上有好多的文章都讲symbolicatecrash工具。symbolicatecrash，按照字面意思理解，就是符号化异常工具。symbolicatecrash符号化日志的一般步骤为：

1. 查找symbolicatecrash的存储位置，symbolicatecrash在各个Xcode版本中的位置都不一样，我没有办法记住每个版本的位置，所以使用查找命令查找symbolicatecrash 的位置

		find /Applications/Xcode.app -name symbolicatecrash -type f   #查找symbolicatecrash 的路径。
		
	输出如下:
	
		AAA$ find /Applications/Xcode.app -name symbolicatecrash -type f
		/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash
		
2. 将符号文件、Crash日志、symbolicatecrash放在同一目录下。cd 到该目录下，终端执行命令：

		./symbolicatecrash name.crash 符号文件 > out.txt
	如果成功，会将符号化的日志重定向到out.txt中。

3. 备注： 如果执行中遇到下面的错误：

		Error: "DEVELOPER_DIR" is not defined at ./symbolicatecrash line 60.
	执行下面的命令，设置环境变量

		export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer
		
<!--/usr/bin/xcode-select -print-path     # 查找Xcode的安装路径，如果输出”/Developer”或者其他非”/Applications/Xcode.app/Contents/Developer/”的内容，运行xcode-select命令：

sudo /usr/bin/xcode-select -switch /Applications/Xcode.app/Contents/Developer/-->
		
# 打造自己的收集、符号化程序

当APP发布到AppStore后，如果发生了Crash，通常情况下我们拿不到崩溃手机，也就是说拿不到Crash日志。这是一个棘手的问题。有人说可以在开发者中心找到用户上传到苹果的日志，但是，不是所有的用户都会在程序Crash后上传Crash日志，所以有必要打造一个属于我们自己的异常收集系统。下面就讲讲我打造的异常收集系统，主要思路：使用NSSetUncaughtExceptionHandler注册异常处理函数，当APP 发生Crash时，回调到异常处理函数，在异常处理函数中收集Crash信息，然后上传到服务器；当需要分析的时候，从服务器取回Crash日志，如果没有符号化，使用atos命令符号化。由于我没有服务器，就保存到了沙盒路径的Document目录下，可以使用itunes方便的导出日志。这里我提供了一个简单示例代码：[caughtException](https://github.com/jianli2017/caughtException/tree/master/UncaughtException)，有代码才有真相，才有说服力。那就先从代码入手。

## 实现代码 

这里会分别列出关键的代码。下面是 AppDelegate.m中的代码 

	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
	{
	    [LJCaughtException setDefaultHandler];
	    // Override point for customization after application launch.
	    return YES;
	}

在application:didFinishLaunchingWithOptions:中注册异常处理函数，所有的异常注册和异常处理函数的代码都封装到LJCaughtException.m中，如下：

	///先前注册的处理句柄
	NSUncaughtExceptionHandler *preHander;
	
	/// 异常处理函数
	void UncaughtExceptionHandler(NSException * exception)
	{
	    [LJCaughtException  processException:exception];
	}
	
	@implementation LJCaughtException
	
	+ (void)setDefaultHandler
	{
			///首先保存先前注册的异常处理句柄
	    preHander = [LJCaughtException getHandler];
	    ///注册异常处理句柄
	    NSSetUncaughtExceptionHandler(&UncaughtExceptionHandler);
	}
	
	+ (NSUncaughtExceptionHandler *)getHandler
	{
	    return NSGetUncaughtExceptionHandler();
	}
	
	///异常处理句柄
	+ (void)processException:(NSException *)exception
	{
	    /// 异常的堆栈信息
	    NSArray *aryCrashBackTrace = [exception callStackSymbols];
	    if (!aryCrashBackTrace)
	    {
	        return;
	    }
	    /// 出现异常的原因
	    NSString *strCrashReason = [exception reason];
	    
	    /// 异常名称
	    NSString *strCrashName = [exception name];
	    
	    ....
	}
	... 
	
	@end

上面代码可以分解为三个部分理解：

1. 定义异常处理函数：异常处理函数的原型为：

	typedef void NSUncaughtExceptionHandler(NSException *exception);
	
2. 注册异常处理函数：使用NSSetUncaughtExceptionHandler注册异常处理函数,注册的代码为：NSSetUncaughtExceptionHandler(&UncaughtExceptionHandler);

3. 执行异常处理函数：当异常发生时，自动执行异常处理函数。异常处理函数内部完成收集Crash信息的功能。
 
下面是在Debug和Release模式下，Crash时捕获的线程回溯： 
![收集、解析IOS崩溃日式10](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%9710_1.png)

可以看出，使用系统的API可以完美的捕获到崩溃日志，而且符号化了，一行代码（callStackSymbols）就获取了异常线程的回溯并完成了符号化工作，不费吹灰之力。其实，事情没有这么简单，不妨试试发布包，是不是也能像在debug和release模式那样，获取到符号化的异常线程回溯？

## 发布包没带符号表  

将测试程序打为发布包，查看异常线程回溯图，如下：
图11 发布包的Crash日志
![收集、解析IOS崩溃日式11](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BC%8F11.png)

是不是很奇怪，图中红框是异常线程的关键回溯，显示的是镜像的名字，没有被转化为有效的代码符号。这是为什么？

静静的想想。。。。。。前面提到符号化的前提条件，是得有符号表，那么我们推测debug和release的APP包含了符号表，而发布包没有包含符号表，是不是？ 请在终端中使用nm命令验证下。 

![收集、解析IOS崩溃日式13](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BC%8F13.png)

确实是，发布包没有符号表，为什么？

原来，符号表是一个debug产物，如果使用archive模式打包，那么符号表会被剪裁掉。不过你也可以在Xcode的编译选项中配置为符号表不剪裁。方法是设置Strip Style选项为Debugging Symbols，但是会让最后生成的IPA变大约%5。我用我们项目测试，居然大了约%30，可能是代码太多的原因吧。这个对于严格限制APP大小的人来说，是无法接受的。下图是设置发布包带符号表的方法：

![收集、解析IOS崩溃日志18](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%9718.png)

天无绝人之路，在使用archive打包时，生成了一个dSYM符号文件，这个文件不发布，在本地保存着。这个文件太有用了，也是我们符号化的唯一选择了。

显然，对于发布到用户手中的发布包，在程序Crash后，不能在用户设备上完成符号化工作，callStackSymbols只能返回带地址的日志信息，需要我们线下符号化，还好苹果提供了一个命令行工具-----atos，完成符号化工作。 若想通过atos工具在符号文件中查找到地址对应的符号，需要代码构架、镜像加载地址这两个参数，查看图11，这两个参数都没有，怎么办？我只能祭出[OS X ABI Mach-O File Format Reference](http://of685p9vy.bkt.clouddn.com/OS%20X%20ABI%20Mach-O%20File%20Format%20Reference.pdf)和[KSCrash](https://github.com/kstenerud/KSCrash) 开源框架这两个终极神器。[OS X ABI Mach-O File Format Reference](http://of685p9vy.bkt.clouddn.com/OS%20X%20ABI%20Mach-O%20File%20Format%20Reference.pdf)阐述了可执行二进制程序的存储格式，提供原理性的支撑。[KSCrash](https://github.com/kstenerud/KSCrash)包含了获取代码构架和镜像加载地址的代码。依据这两个神器，我们可以顺利的拿到代码构架、镜像加载地址。其中《OS X ABI Mach-O File Format Reference》居然在苹果的官网上找不到了，前段时间都能找到的，幸好我又一个备份，我只能放在七牛上保存起来了。
<!--既然不能符号，也不是一个完整的Crash日志，没法用Xocde解析，也缺少atos解析的条件：代码构架、镜像加载地址。也就是说这个只能开发的时候用，发布到App Store就没有什么用了？是不是觉得一下回到了解放前？ 
不过生成了一个dSYM符号文件。-->

## Mach-O File Format
Mach-O 是Mach object 的意思，就是OS X系统中对象文件的存储格式，对象文件包括： kernel extensions, command-line tools, applications, frameworks, and libraries (shared and static)。 详细的也可以参考[Mach-O Programming Topics](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/MachOTopics/0-Introduction/introduction.html#//apple_ref/doc/uid/TP40001827-SW1)

一个Mach-O 文件包括下面三个部分 
1. Header: Specifies the target architecture of the file, such as PPC, PPC64, IA-32, or x86-64.
2. Load commands: Specify the logical structure of the file and the layout of the file in virtual memory.
3. Raw segment data: Contains raw data for the segments defined in the load commands.

下面是官网上的一张图形化的Mach-O结构示意图： 
![收集、解析IOS崩溃日志14](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%9714.png)  

下面依次讲解这三部分，他们的数据结构定义在mach-o/loader.h中。我会通过三种方式来层显Mach-O文件结构：代码定义、通过命令行工具otool呈现、通过MachOView呈现。其中otool是系统自带的对象文件查看工具。MachOView 是网上下载的可视化查看Mach-O结构工具。由于存在两个代码构架，armv7s、ARM64，他们的定义稍微有点区别，我仅以ARM64构架为例。

<!--在Xcode中输入：

	 #include <mach-o/loader.h>

可以找到这个文件。-->

### header 

header的数据结构的定义如下：
 
	struct mach_header_64 
	{
		uint32_t	magic;		      ///魔数，标记这个是Mach-O文件
		cpu_type_t	cputype;      ///cup 的类型
		cpu_subtype_t	cpusubtype;
		uint32_t	filetype;	
		uint32_t	ncmds;		     /// load commands 个数
		uint32_t	sizeofcmds;  
		uint32_t	flags;		
		uint32_t	reserved;
	};

终端中查看header： 

	otool -hV ~/Desktop/收集、解析IOS崩溃日式/Exception/UncaughtException_archive 

输出如下：
	
	 magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
	   MH_MAGIC     ARM         V7  0x00     EXECUTE    23       2432   NOUNDEFS DYLDLINK TWOLEVEL PIE
	Mach header
	      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
	MH_MAGIC_64   ARM64        ALL  0x00     EXECUTE    23       2872   NOUNDEFS DYLDLINK TWOLEVEL PIE

MachOView显示的结果：
![收集、解析IOS崩溃日志15](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%9715.png) 

magic 是MH_MAGIC_64，固定值：0xfeedfacf，标记这是一个Mach-O文件。
filetype 文件类型是EXECUTE，可执行程序，
ncmds，load commond个数是23个
### load Commond 
 
 load Commond 种类特别多，大概有60多种，每种commond的数据结构是不同的， 我不会去一一的说明，只拿LC_SEGMENT、LC_SYMTAB 做个示例。下面列表了部分load commond。
 
	#define	LC_SEGMENT	0x1	/* segment of this file to be mapped */
	#define	LC_SYMTAB	0x2	/* link-edit stab symbol table info */
	#define	LC_SYMSEG	0x3	/* link-edit gdb symbol table info (obsolete) */
	#define	LC_THREAD	0x4	/* thread */
	#define	LC_UNIXTHREAD	0x5	/* unix thread (includes a stack) */
	#define	LC_LOADFVMLIB	0x6	/* load a specified fixed VM shared library */
	.....   
	
#### LC_SEGMENT 

LC_SEGMENT ,segment load command indicates that a part of this file is to be mapped into a 64-bit task's address space,说白了，就是映射到内存中的所有数据，自然包括代码、数据等等。segment进一步可以分为 __PAGEZERO、__TEXT、__DATA、 __OBJC、__IMPORT、__LINKEDIT。__PAGEZERO 类型的segment是可执行程序的第一个segment，代表指针地址NULL。__TEXT就是可执行代码，当然是只读了。__DATA 是可写的数据segement，应该就是代码中的变量区域。__OBJC 是Objective-C runtime support library。 The __LINKEDIT segment contains raw data used by the dynamic linker, such as symbol, string, and relocation table entries。

每种segement可能包含多种类型的内容，例如__TEXT代码段，可以有代码（__text）、字符串（__cstring） 、常量（__const）、符号（__symbol_stub）、字面量（__literal4、__literal8），所以进一步用二级目录（section）表示。下面是segment、section的数据结构：

	struct segment_command_64 
	{ 
		/* for 64-bit architectures */
			uint32_t	cmd;		/* LC_SEGMENT_64 */
			uint32_t	cmdsize;	/* includes sizeof section_64 structs */
			char		segname[16];	/* segment name */
			uint64_t	vmaddr;		/* memory address of this segment */
			uint64_t	vmsize;		/* memory size of this segment */
			uint64_t	fileoff;	/* file offset of this segment */
			uint64_t	filesize;	/* amount to map from the file */
			vm_prot_t	maxprot;	/* maximum VM protection */
			vm_prot_t	initprot;	/* initial VM protection */
			uint32_t	nsects;		/* number of sections in segment */
			uint32_t	flags;		/* flags */
	};
	
	struct section_64 
	{ 
		/* for 64-bit architectures */
		char		sectname[16];	/* name of this section */
		char		segname[16];	/* segment this section goes in */
		uint64_t	addr;		/* memory address of this section */
		uint64_t	size;		/* size in bytes of this section */
		uint32_t	offset;		/* file offset of this section */
		uint32_t	align;		/* section alignment (power of 2) */
		uint32_t	reloff;		/* file offset of relocation entries */
		uint32_t	nreloc;		/* number of relocation entries */
		uint32_t	flags;		/* flags (section type and attributes)*/
		uint32_t	reserved1;	/* reserved (for offset or index) */
		uint32_t	reserved2;	/* reserved (for count or sizeof) */
		uint32_t	reserved3;	/* reserved */
	};
终端输入：
	otool -lV ~/Desktop/收集、解析IOS崩溃日式/Exception/UncaughtException_archive

输出：

		........
		cmd LC_SEGMENT_64
	  cmdsize 712
	  segname __TEXT
	   vmaddr 0x0000000100000000
	   vmsize 0x0000000000008000
	  fileoff 0
	 filesize 32768
	  maxprot r-x
	  initprot r-x
	   nsects 8
	    flags (none)
		.......


MachOView显示的结果：
![收集、解析IOS崩溃日志16](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%9716.png)
图中直观的显示出了LC_SEGMENT的数据 、LC_SEGMENT的二级目录section的数据。
#### LC_SYMTAB 

LC_SYMTAB的数据结构如下：

	struct symtab_command {
		uint32_t	cmd;		/* LC_SYMTAB */
		uint32_t	cmdsize;	/* sizeof(struct symtab_command) */
		uint32_t	symoff;		/* symbol table offset */
		uint32_t	nsyms;		/* number of symbol table entries */
		uint32_t	stroff;		/* string table offset */
		uint32_t	strsize;	/* string table size in bytes */
	};

终端输出的结果: 

	Load command 6
	     cmd LC_SYMTAB
	 cmdsize 24
	  symoff 132944
	   nsyms 48
	  stroff 133916
	 strsize 1152
MachOView看到的结果：
![收集、解析IOS崩溃日志17](http://of685p9vy.bkt.clouddn.com/%E6%94%B6%E9%9B%86%E3%80%81%E8%A7%A3%E6%9E%90IOS%E5%B4%A9%E6%BA%83%E6%97%A5%E5%BF%9717.png)

LC_SYMTAB 指定了符号的个数和相对Mach-O的偏移量。 

### 数据部分

紧跟着load commond 后面的是数据部分，就是各个load commond 对应的具体数据。

### 小小结

我觉得Mach-O文件的格式非常像 一篇文章的结构，
Header部分是文章的摘要，总体描述了非常重要部分。 
Load commands 相当于目录，Mach-O文件所有内容的索引。
Raw segment data 正文内容。


Mach-O 文件格式就是一个规范，各个部分都有自己的数据格式，内容繁多，只能多看。
不过提到了一个有用的工具---otool，查看Mach-O对象文件的命令行工具。

## 获取构架、镜像加载地址

好了，上面说了那么多Mach-O文件结构，主要是提供原理支撑，目的是通过对Mach-O文件结构的理解，找到获取构架、镜像加载地址的方法。 
构架很好获取，就在Mach-O的文件头中，获取的关键代码如下 ：

	/*
	 获取代码的构架
	 */
	NSString * getCodeArch()
	{
	    NSString *strSystemArch =nil;
	    
	    
	    ///获取应用程序的名称
	    NSDictionary *dicInfo =   [[NSBundle mainBundle] infoDictionary];
	    if (LJM_Dic_Not_Valid(dicInfo))
	    {
	        return strSystemArch;
	    }
	    NSString *strAppName = dicInfo[@"CFBundleName"];
	    if (!strAppName)
	    {
	        return strSystemArch;
	    }
	    
	    ///获取  cpu 的大小版本号
	    uint32_t count = _dyld_image_count();
	    cpu_type_t cpuType = -1;
	    cpu_type_t cpuSubType =-1;
	    
	    for(uint32_t iImg = 0; iImg < count; iImg++)
	    {
	        const char* szName = _dyld_get_image_name(iImg);
	        if (strstr(szName, strAppName.UTF8String) != NULL)
	        {
	            const struct mach_header* machHeader = _dyld_get_image_header(iImg);
	            cpuType = machHeader->cputype;
	            cpuSubType = machHeader->cpusubtype;
	            break;
	        }
	    }
	    
	    if(cpuType < 0 ||  cpuSubType <0)
	    {
	        return  strSystemArch;
	    }
	    ///转化cpu 版本为文字类型
	    switch(cpuType)
	    {
	        case CPU_TYPE_ARM:
	        {
	            strSystemArch = @"arm";
	            switch (cpuSubType)
	            {
	                case CPU_SUBTYPE_ARM_V6:
	                    strSystemArch = @"armv6";
	                    break;
	                case CPU_SUBTYPE_ARM_V7:
	                    strSystemArch = @"armv7";
	                    break;
	                case CPU_SUBTYPE_ARM_V7F:
	                    strSystemArch = @"armv7f";
	                    break;
	                case CPU_SUBTYPE_ARM_V7K:
	                    strSystemArch = @"armv7k";
	                    break;
	#ifdef CPU_SUBTYPE_ARM_V7S
	                case CPU_SUBTYPE_ARM_V7S:
	                    strSystemArch = @"armv7s";
	                    break;
	#endif
	            }
	            break;
	        }
	#ifdef CPU_TYPE_ARM64
	        case CPU_TYPE_ARM64:
	            strSystemArch = @"arm64";
	            break;
	#endif
	        case CPU_TYPE_X86:
	            strSystemArch = @"i386";
	            break;
	        case CPU_TYPE_X86_64:
	            strSystemArch = @"x86_64";
	            break;
	    }
	    return strSystemArch;
	}

主要思路是：通过_dyld_image_count 获取到所有的镜像个数，然后根据镜像索引（0...镜像个数-1），依次枚举出镜像的名字，然后，镜像名字使用_dyld_get_image_header函数获取到镜像的header结构体信息，赋值到：mach_header* machHeader中。最后，通过machHeader->cputype（ CPU的类型）和machHeader->cpusubtype（CPU的子类型）转化为具体的代码构架。

对于镜像的加载地址，其实就是镜像的header结构体的首地址。详细代码如下

	/*
	 获取应用程序的加载地址
	 */
	NSString * getImageLoadAddress()
	{
	    NSString *strLoadAddress =nil;
	    
	    
	    NSString * strAppName = getAppName();
	    if (!strAppName)
	    {
	        return strLoadAddress;
	    }
	    
	    ///获取应用程序的load address
	    uint32_t count = _dyld_image_count();
	    for(uint32_t iImg = 0; iImg < count; iImg++)
	    {
	        const char* szName = _dyld_get_image_name(iImg);
	        if (strstr(szName, strAppName.UTF8String) != NULL)
	        {
	            const struct mach_header* header = _dyld_get_image_header(iImg);
	            strLoadAddress = [NSString stringWithFormat:@"0x%lX",(uintptr_t)header];
	            break;
	        }
	    }
	    return strLoadAddress;
	}

主要思路就是：利用_dyld_get_image_header获取镜像的header结构体，header结构体是整个Mach-O的起始部分，所以，header结构体的首地址就是镜像的加载地址。

好了，到目前为止，使用atos符号化崩溃日志的三个条件条件（符号文件、代码构架、镜像加载地址）都有了，那么我们就可以完成异常地址的符号化工作了。所以，到目前为止，我们定制的异常系统基本完成了，收集功能、符号化动能都有了。下面来看看我们的系统输出的内容。

## 输出Crash日志

本崩溃收集系统的输出格式使用json格式，输出的信息包括arch、CrashName、CrashReason、CrashBackTrace、CrashSystemVerson 。有了这些信息，我们完全可以符号化崩溃地址了。

	{
	  "strCrashArch" : "arm64",         ///代码构架
	  "strCrashName" : "NSRangeException",
	  "strCrashSystemVersion" : "10.0.2",
	  "strCrashReason" : "*** -[__NSArrayI objectAtIndex:]: index 2 beyond bounds [0 .. 1]",
	  "aryCrashBackTrace" : [
	    {
	      "strStackAddress" : "0x000000018ec6c1d8",
	      "strImageName" : "CoreFoundation",
	      "strImageLoadAddress" : "<redacted>"
	    },
	    {
	      "strStackAddress" : "0x000000018d6a455c",
	      "strImageName" : "libobjc.A.dylib",
	      "strImageLoadAddress" : "objc_exception_throw"
	    },
	    {
	      "strStackAddress" : "0x000000018eb48584",
	      "strImageName" : "CoreFoundation",
	      "strImageLoadAddress" : "CFRunLoopRemoveTimer"
	    },
	    {
	      "strStackAddress" : "0x00000001000b48a0",    ///崩溃地址
	      "strImageName" : "UncaughtException",
	      "strImageLoadAddress" : "0x1000B0000"       ///镜像加载地址
	    },
	    {
	      "strStackAddress" : "0x0000000194aea7b0",
	      "strImageName" : "UIKit",
	      "strImageLoadAddress" : "<redacted>"
	    },
	    ........
	    ........
	    {
	      "strStackAddress" : "0x0000000194b1b360",
	      "strImageName" : "UIKit",
	      "strImageLoadAddress" : "UIApplicationMain"
	    },
	    {
	      "strStackAddress" : "0x00000001000b4df0",
	      "strImageName" : "UncaughtException",
	      "strImageLoadAddress" : "0x1000B0000"
	    },
	    {
	      "strStackAddress" : "0x000000018db285b8",
	      "strImageName" : "libdyld.dylib",
	      "strImageLoadAddress" : "<redacted>"
	    }
	  ]
	}

## 小结

这章，我们使用苹果的API完成了Crash日志收集系统，这个系统输出的日志可以使用atos在线下符号化。同时介绍了Mach-O的文件结构。 

## 你被默默的坑了吗

通常一个大型的APP总是会引用第三方的SDK，第三方SDK也会集成一个Crash收集服务，以及时发现他们SDK的问题。当多个收集服务集成到一个APP中时，难免出现时序问题，强行覆盖等等的恶意竞争，总会有人默默被坑。所以NSSetUncaughtExceptionHandler设置自己的异常处理函数前，要保存先前的异常处理函数。当我们的异常处理函数执行完，将先前的异常处理函数注册回去。这样才能保证多个异常收集系统能有序工作。

<!--不过我看我们项目的代码非常的高大上，我们注册异常处理函数延迟执行，当程序启动后10秒，才注册，是不是因为我们被坑过，我不知道。-->


# 深度定制异常收集系统

我们上面定制的系统非常简单，功能单一。只能捕获到Object C异常。不能满足实际项目的需求，所以有必要找一个功能完善的异常收集框架，经过筛选，KSCrash是个不错的选择。

[KSCrash](https://github.com/kstenerud/KSCrash) 是一个异常收集的开源框架。它可以捕获到Mach级内核异常、信号异常、C++异常、Objective-C异常、主线程死锁；当捕获到异常后，KSCrash可以在设备上完成符号化崩溃日志(前提是编译的时候将符号表编译到可执行程序中)；日志的格式你也可以定制，可以是JSON格式的，也可以是Apple crash日志风格。另外，还有僵尸对象查找、内存自省等特性。

由于这篇文章罗列的内容太多，废话太多，所有移到《[KSCrash崩溃原理浅析](https://jianli2017.github.io/2016/07/17/KSCrash_Analize/)》 中单独阐述。