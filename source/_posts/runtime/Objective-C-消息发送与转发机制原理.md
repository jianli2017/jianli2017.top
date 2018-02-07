---
title: Objective-C 消息发送与转发机制原理
date: 2016-12-02 14:42:23
tags: runtime
categories: runtime
---
消息发送和转发流程可以概括为：<font color=red size=4 face="黑体">  消息发送（Messaging）是 Runtime 通过 selector 快速查找 IMP 的过程，有了函数指针就可以执行对应的方法实现；消息转发（Message Forwarding）是在查找 IMP 失败后执行的一系列转发流程，如果不作转发处理，则会打日志和抛出异常。 </font>
<!--more-->
本文不讲述开发者在消息发送和转发流程中需要做的事，而是讲述原理。能够很好地阅读本文的前提是你对 Objective-C Runtime 已经有一定的了解，关于什么是消息，Class 的结构，selector、IMP、元类等概念将不再赘述。本文用到的源码为 objc4-680 和 CF-1153.18，逆向 CoreFoundation.framework 的系统版本为 macOS 10.11.5，汇编语言架构为 x86_64。
# 八面玲珑的 objc_msgSend
此函数是消息发送必经之路，但只要一提 objc_msgSend，都会说它的伪代码如下或类似的逻辑，反正就是获取 IMP 并调用：

	id objc_msgSend(id self, SEL _cmd, ...) {
	  Class class = object_getClass(self);
	  IMP imp = class_getMethodImplementation(class, _cmd);
	  return imp ? imp(self, _cmd, ...) : 0;
	}
## 源码解析
为啥老用伪代码？因为 objc_msgSend 是用汇编语言写的，针对不同架构有不同的实现。如下为 x86_64 架构下的源码，可以在 objc-msg-x86_64.s 文件中找到，关键代码如下：

	ENTRY	_objc_msgSend
	MESSENGER_START
	
	NilTest	NORMAL
	
	GetIsaFast NORMAL		// r11 = self->isa
	CacheLookup NORMAL		// calls IMP on success
	
	NilTestSupport	NORMAL
	
	GetIsaSupport	   NORMAL
	
	// cache miss: go search the method lists
	LCacheMiss:
	// isa still in r11
	MethodTableLookup %a1, %a2	// r11 = IMP
	cmp	%r11, %r11		// set eq (nonstret) for forwarding
	jmp	*%r11			// goto *imp
	
	END_ENTRY	_objc_msgSend
这里面包含一些有意义的宏：
1. NilTest 宏，判断被发送消息的对象是否为 nil 的。如果为 nil，那就直接返回 nil。这就是为啥也可以对 nil 发消息。
2. GetIsaFast 宏可以『快速地』获取到对象的 isa 指针地址（放到 r11 寄存器，r10 会被重写；在 arm 架构上是直接赋值到 r9）
3. CacheLookup 这个宏是在类的缓存中查找 selector 对应的 IMP（放到 r10）并执行。如果缓存没中，那就得到 Class 的方法表中查找了。
4. MethodTableLookup 宏是重点，负责在缓存没命中时在方法表中负责查找 IMP：


	.macro MethodTableLookup
	
		MESSENGER_END_SLOW
		
		SaveRegisters
		
		// _class_lookupMethodAndLoadCache3(receiver, selector, class)
		
		movq	$0, %a1
		movq	$1, %a2
		movq	%r11, %a3
		call	__class_lookupMethodAndLoadCache3
		
		// IMP is now in %rax
		movq	%rax, %r11
		
		RestoreRegisters
	
	.endmacro
	
从上面的代码可以看出方法查找 IMP 的工作交给了 OC 中的 _class_lookupMethodAndLoadCache3 函数，并将 IMP 返回（从 r11 挪到 rax）。最后在 objc_msgSend 中调用 IMP。
## 为什么使用汇编语言
其实在 objc-msg-x86_64.s 中包含了多个版本的 objc_msgSend 方法，它们是根据返回值的类型和调用者的类型分别处理的：
1. objc_msgSendSuper:向父类发消息，返回值类型为 id
2. objc_msgSend_fpret:返回值类型为 floating-point，其中包含 objc_msgSend_fp2ret 入口处理返回值类型为 long double 的情况
3. objc_msgSend_stret:返回值为结构体
4. objc_msgSendSuper_stret:向父类发消息，返回值类型为结构体

当需要发送消息时，编译器会生成中间代码，根据情况分别调用 objc_msgSend, objc_msgSend_stret, objc_msgSendSuper, 或 objc_msgSendSuper_stret 其中之一。
这也是为什么 objc_msgSend 要用汇编语言而不是 OC、C 或 C++ 语言来实现，因为单独一个方法定义满足不了多种类型返回值，有的方法返回 id，有的返回 int。考虑到不同类型参数返回值排列组合映射不同方法签名（method signature）的问题，那 switch 语句得老长了。。。这些原因可以总结为 Calling Convention，也就是说函数调用者与被调用者必须约定好参数与返回值在不同架构处理器上的存取规则，比如参数是以何种顺序存储在栈上，或是存储在哪些寄存器上。除此之外还有其他原因，比如其可变参数用汇编处理起来最方便，因为找到 IMP 地址后参数都在栈上。要是用 C++ 传递可变参数那就悲剧了，prologue 机制会弄乱地址（比如 i386 上为了存储 ebp 向后移位 4byte），最后还要用 epilogue 打扫战场。而且汇编程序执行效率高，在 Objective-C Runtime 中调用频率较高的函数好多都用汇编写的。

## 使用 lookUpImpOrForward 快速查找 IMP
上一节中说到的 _class_lookupMethodAndLoadCache3 函数其实只是简单的调用了 lookUpImpOrForward 函数：

	IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
	{
	    return lookUpImpOrForward(cls, sel, obj, 
	                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
	}
注意 lookUpImpOrForward 调用时使用缓存参数传入为 NO，因为之前已经尝试过查找缓存了。IMP lookUpImpOrForward(Class cls, SEL sel, id inst, bool initialize, bool cache, bool resolver) 实现了一套查找 IMP 的标准路径，也就是在消息转发（Forward）之前的逻辑。
### 优化缓存查找&类的初始化

先对 debug 模式下的 assert 进行 unlock：

	runtimeLock.assertUnlocked();
runtimeLock 本质上是对 Darwin 提供的线程读写锁 pthread_rwlock_t 的一层封装，提供了一些便捷的方法。
lookUpImpOrForward 接着做了如下两件事：
1. 如果使用缓存（cache 参数为 YES），那就调用 cache_getImp 方法从缓存查找 IMP。cache_getImp 是用汇编语言写的，也可以在 objc-msg-x86_64.s 找到，其依然用了之前说过的 CacheLookup 宏。因为 _class_lookupMethodAndLoadCache3 调用 lookUpImpOrForward 时 cache 参数为 NO，这步直接略过。
2. 如果是第一次用到这个类且 initialize 参数为 YES（initialize && !cls->isInitialized()），需要进行初始化工作，也就是开辟一个用于读写数据的空间。先对 runtimeLock 写操作加锁，然后调用 cls 的 initialize 方法。如果 sel == initialize 也没关系，虽然 initialize 还会被调用一次，但不会起作用啦，因为 cls->isInitialized() 已经是 YES 啦。

### 继续在类的继承体系中查找
考虑到运行时类中的方法可能会增加，需要先做读操作加锁，使得方法查找和缓存填充成为原子操作。添加 category 会刷新缓存，之后如果旧数据又被重填到缓存中，category 添加操作就会被忽略掉。

	runtimeLock.read();
之后的逻辑整理如下：
1. 如果 selector 是需要被忽略的垃圾回收用到的方法，则将 IMP 结果设为 _objc_ignored_method，这是个汇编程序入口，可以理解为一个标记。对此种情况进行缓存填充操作后，跳到第 7 步；否则执行下一步。
2. 查找当前类中的缓存，跟之前一样，使用 cache_getImp 汇编程序入口。如果命中缓存获取到了 IMP，则直接跳到第 7 步；否则执行下一步。
3. 在当前类中的方法列表（method list）中进行查找，也就是根据 selector 查找到 Method 后，获取 Method 中的 IMP（也就是 method_imp 属性），并填充到缓存中。查找过程比较复杂，会针对已经排序的列表使用二分法查找，未排序的列表则是线性遍历。如果成功查找到 Method 对象，就直接跳到第 7 步；否则执行下一步。
4. 在继承层级中递归向父类中查找，情况跟上一步类似，也是先查找缓存，缓存没中就查找方法列表。这里跟上一步不同的地方在于缓存策略，有个 _objc_msgForward_impcache 汇编程序入口作为缓存中消息转发的标记。也就是说如果在缓存中找到了 IMP，但如果发现其内容是 _objc_msgForward_impcache，那就终止在类的继承层级中递归查找，进入下一步；否则跳到第 7 步。
5. 当传入 lookUpImpOrForward 的参数 resolver 为 YES 并且是第一次进入第 5 步时，时进入动态方法解析；否则进入下一步。这步消息转发前的最后一次机会。此时释放读入锁（runtimeLock.unlockRead()），接着间接地发送 +resolveInstanceMethod 或 +resolveClassMethod 消息。这相当于告诉程序员『赶紧用 Runtime 给类里这个 selector 弄个对应的 IMP 吧』，因为此时锁已经 unlock 了所以不会缓存结果，甚至还需要软性地处理缓存过期问题可能带来的错误。这里的业务逻辑稍微复杂些，后面会总结。因为这些工作都是在非线程安全下进行的，完成后需要回到第 1 步再次查找 IMP。
6. 此时不仅没查找到 IMP，动态方法解析也不奏效，只能将 _objc_msgForward_impcache 当做 IMP 并写入缓存。这也就是之前第 4 步中为何查找到 _objc_msgForward_impcache 就表明了要进入消息转发了。
7. 读操作解锁，并将之前找到的 IMP 返回。（无论是正经 IMP 还是不正经的 _objc_msgForward_impcache）这步还偏执地做了一些脑洞略大的 assert，很有趣。

对于第 5 步，其实是直接调用 _class_resolveMethod 函数，在这个函数中实现了复杂的方法解析逻辑。如果 cls 是元类则会发送 +resolveClassMethod，然后根据 lookUpImpOrNil(cls, sel, inst, NO/*initialize*/, YES/*cache*/, NO/*resolver*/) 函数的结果来判断是否发送 +resolveInstanceMethod；如果不是元类，则只需要发送 +resolveInstanceMethod 消息。这里调用 +resolveInstanceMethod 或 +resolveClassMethod 时再次用到了 objc_msgSend，而且第三个参数正是传入 lookUpImpOrForward 的那个 sel。在发送方法解析消息之后还会调用 lookUpImpOrNil(cls, sel, inst, NO/*initialize*/, YES/*cache*/, NO/*resolver*/) 来判断是否已经添加上 sel 对应的 IMP 了，打印出结果。
最后 lookUpImpOrForward 方法也会把真正的 IMP 或者需要消息转发的 _objc_msgForward_impcache 返回，并最终专递到 objc_msgSend 中。而 _objc_msgForward_impcache 会在转化成 _objc_msgForward 或 _objc_msgForward_stret。这个后面会讲解原理。
## 回顾 objc_msgSend 伪代码
回过头来会发现 objc_msgSend 的伪代码描述得很传神啊，因为class_getMethodImplementation 的实现如下：

	IMP class_getMethodImplementation(Class cls, SEL sel)
	{
	    IMP imp;
	    if (!cls  ||  !sel) return nil;
	    imp = lookUpImpOrNil(cls, sel, nil, YES/*initialize*/, YES/*cache*/, YES/*resolver*/);
	    // Translate forwarding function to C-callable external version
	    if (!imp) {
	        return _objc_msgForward;
	    }
	    return imp;
	}

lookUpImpOrNil 函数获取不到 IMP 时就返回 _objc_msgForward，后面会讲到它。lookUpImpOrNil 跟 lookUpImpOrForward 的功能很相似，只是将 lookUpImpOrForward 实现中的 _objc_msgForward_impcache 替换成了 nil:

	IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
	                   bool initialize, bool cache, bool resolver)
	{
	    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
	    if (imp == _objc_msgForward_impcache) return nil;
	    else return imp;
	}
lookUpImpOrNil 方法可以查找到 selector 对应的 IMP 或是 nil，所以如果不考虑返回值类型为结构体的情况，用那几行伪代码来表示复杂的汇编实现还是挺恰当的。

# forwarding 转发消息

## objc_msgForward_impcache 的转换

_objc_msgForward_impcache 只是个内部的函数指针，只存储于上节提到的类的方法缓存中，需要被转化为 _objc_msgForward 和 _objc_msgForward_stret 才能被外部调用。但在 Mac OS X macOS 10.6 及更早版本的 libobjc.A.dylib 中是不能直接调用的，况且我们根本不会直接用到它。带 stret 后缀的函数依旧是返回值为结构体的版本。
上一节最后讲到如果没找到 IMP，就会将 _objc_msgForward_impcache 返回到 objc_msgSend 函数，而正是因为它是用汇编语言写的，所以将内部使用的 _objc_msgForward_impcache 转化成外部可调用的 _objc_msgForward 或 _objc_msgForward_stret 也是由汇编代码来完成。实现原理很简单，就是增加个静态入口 __objc_msgForward_impcache，然后根据此时 CPU 的状态寄存器的内容来决定转换成哪个。如果是 NE(Not Equal) 则转换成 _objc_msgForward_stret，反之是 EQ(Equal) 则转换成 _objc_msgForward:
	
	jne	__objc_msgForward_stret
	jmp	__objc_msgForward
	
为何根据状态寄存器的值来判断转换成哪个函数指针呢？回过头来看看 objc_msgSend 中调用完 MethodTableLookup 之后干了什么：

	MethodTableLookup %a1, %a2	// r11 = IMP
	cmp	%r11, %r11		// set eq (nonstret) for forwarding
	jmp	*%r11			// goto *imp
	
再看看返回值为结构体的 objc_msgSend_stret 这里的逻辑：

	MethodTableLookup %a2, %a3	// r11 = IMP
	test	%r11, %r11		// set ne (stret) for forward; r11!=0
	jmp	*%r11			// goto *imp
稍微懂汇编的人一眼就看明白了，不懂的看注释也懂了，我就不墨迹了。现在总算是把消息转发前的逻辑绕回来构成闭环了。
上一节中提到 class_getMethodImplementation 函数的实现，在查找不到 IMP 时返回 _objc_msgForward，而 _objc_msgForward_stret 正好对应着 class_getMethodImplementation_stret:

	IMP class_getMethodImplementation_stret(Class cls, SEL sel)
	{
	    IMP imp = class_getMethodImplementation(cls, sel);
	    // Translate forwarding function to struct-returning version
	    if (imp == (IMP)&_objc_msgForward /* not _internal! */) {
	        return (IMP)&_objc_msgForward_stret;
	    }
	    return imp;
	}
	
也就是说 _objc_msgForward* 系列本质都是函数指针，都用汇编语言实现，都可以与 IMP 类型的值作比较。_objc_msgForward 和 _objc_msgForward_stret 声明在 message.h 文件中。_objc_msgForward_impcache 在早期版本的 Runtime 中叫做 _objc_msgForward_internal。
## objc_msgForward 也只是个入口
从汇编源码可以很容易看出 _objc_msgForward 和 _objc_msgForward_stret 会分别调用 _objc_forward_handler 和 _objc_forward_handler_stret：

	ENTRY	__objc_msgForward
	// Non-stret version
	
	movq	__objc_forward_handler(%rip), %r11
	jmp	*%r11
	
	END_ENTRY	__objc_msgForward
	
	
	ENTRY	__objc_msgForward_stret
	// Struct-return version
	
	movq	__objc_forward_stret_handler(%rip), %r11
	jmp	*%r11
	
	END_ENTRY	__objc_msgForward_stret
这两个 handler 函数的区别从字面上就能看出来，不再赘述。
也就是说，消息转发过程是现将 _objc_msgForward_impcache 强转成 _objc_msgForward 或 _objc_msgForward_stret，再分别调用 _objc_forward_handler 或 _objc_forward_handler_stret。
## objc_setForwardHandler 设置了消息转发的回调
在 Objective-C 2.0 之前，默认的 _objc_forward_handler 或 _objc_forward_handler_stret 都是 nil，而新版本的默认实现是这样的：

	// Default forward handler halts the process.
	__attribute__((noreturn)) void 
	objc_defaultForwardHandler(id self, SEL sel)
	{
	    _objc_fatal("%c[%s %s]: unrecognized selector sent to instance %p "
	                "(no message forward handler is installed)", 
	                class_isMetaClass(object_getClass(self)) ? '+' : '-', 
	                object_getClassName(self), sel_getName(sel), self);
	}
	void *_objc_forward_handler = (void*)objc_defaultForwardHandler;
	
	#if SUPPORT_STRET
	struct stret { int i[100]; };
	__attribute__((noreturn)) struct stret 
	objc_defaultForwardStretHandler(id self, SEL sel)
	{
	    objc_defaultForwardHandler(self, sel);
	}
	void *_objc_forward_stret_handler = (void*)objc_defaultForwardStretHandler;
	#endif
objc_defaultForwardHandler 中的 _objc_fatal 作用就是打日志并调用 __builtin_trap() 触发 crash，可以看到我们最熟悉的那句 “unrecognized selector sent to instance” 日志。__builtin_trap() 在杀掉进程的同时还能生成日志，比调用 exit() 更好。objc_defaultForwardStretHandler 就是装模作样搞个形式主义，把 objc_defaultForwardHandler 包了一层。__attribute__((noreturn)) 属性通知编译器函数从不返回值，当遇到类似函数需要返回值而却不可能运行到返回值处就已经退出来的情况，该属性可以避免出现错误信息。这里正适合此属性，因为要求返回结构体哒。
因为默认的 Handler 干的事儿就是打日志触发 crash，我们想要实现消息转发，就需要替换掉 Handler 并赋值给 _objc_forward_handler 或 _objc_forward_handler_stret，赋值的过程就需要用到 objc_setForwardHandler 函数，实现也是简单粗暴，就是赋值啊：

	void objc_setForwardHandler(void *fwd, void *fwd_stret)
	{
	    _objc_forward_handler = fwd;
	#if SUPPORT_STRET
	    _objc_forward_stret_handler = fwd_stret;
	#endif
	}
	
# 总结
我将整个实现流程绘制出来，过滤了一些不会进入的分支路径和跟主题无关的细节：

![runtime22](http://of685p9vy.bkt.clouddn.com/runtime22.jpg)