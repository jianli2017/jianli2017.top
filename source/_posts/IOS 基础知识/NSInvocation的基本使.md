---
title: NSInvocation的基本使
date: 2016-11-24 15:35:49
tags: NSInvocation
categories: IOS 基础知识
---
iOS中可以间接向对象发送消息方式有两种：
1. 使用performSelector:withObject；
2. 使用NSInvocation。
performSelector:withObject能完成简单的调用。但是对于大于2个的参数或者有返回值的消息，它就显得有点有心无力了，那么在这种情况下，我们就可以使用NSInvocation来进行这些相对复杂的操作。
<!--more-->


# 方法签名
An NSMethodSignature object records type information for the return value and parameters of a method. It is used to forward messages that the receiving object does not respond to—most notably in the case of distributed objects.
NSMethodSignature对象记录了方法的参数和返回值的类型信息。它被用于转发接受者不能处理的消息。

You typically create an NSMethodSignature object using the NSObject methodSignatureForSelector: instance method . It is then used to create an NSInvocation object,。

NSObject的实例方法methodSignatureForSelector:是创建NSMethodSignature对象的典型方法。 使用NSMethodSignature对象用于创建NSInvocation 对象(通过NSInvocation的invocationWithMethodSignature:类方法)。
下面是创建方法签名的代码：

	///1. 创建方法签名
	NSMethodSignature  *signature = [ViewController instanceMethodSignatureForSelector:@selector(fucWithName:)];
	if (signature == nil)
	{
	    return;
	}
	
使用numberOfArguments获取方法的参数个数，参数个数比sel多两个隐藏参数，一个是self,一个是_cmd。页可以使用getArgumentTypeAtIndex:获取参数。还可以使用methodReturnType获取返回类型。
# 使用NSInvocation发送消息 

	///2、创建NSInvocation对象
	NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
	invocation.target = self;
	//注意：这里的方法名一定要与方法签名类中的方法一致
	invocation.selector = @selector(fucWithName:);
	NSString *strName = @"dog";
	//这里的Index要从2开始，以为0跟1已经被占据了，分别是self（target）,selector(_cmd)
	[invocation setArgument:&strName atIndex:2];
	
	
	//3、调用invoke方法
	[invocation invoke];
	
	id res = nil;
	if (signature.methodReturnLength != 0)
	{
	    [invocation getReturnValue:&res];
	}
	NSLog(@"res = %@",res);
首先使用invocationWithMethodSignature:创建一个NSInvocation对象，然后设置对象的target、selector。NSInvocation对象实际上就是将方法封装为对象。然后使用invoke方法调用消息。使用getReturnValue获取返回值。输出的结果如下：

	ViewController object receive message functionWithName:
	res = dog
	










