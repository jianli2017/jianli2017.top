---
title: runtime1
date: 2016-11-03 17:27:29
tags: runtime
categories: runtime
---

现在越来越多的app都使用了JSPatch实现app热修复，而JSPatch 能做到通过 JS 调用和改写 OC 方法最根本的原因是 Objective-C是动态语言，OC上所有**方法的调用/类的生成**都通过Objective-C Runtime在运行时进行，我们可以通过类名/方法名反射得到相应的类和方法，也可以替换某个类的方法为新的实现，理论上你可以在运行时通过类名/方法名调用到任何 OC 方法，替换任何类的实现以及新增任意类。今天就来详细解析一下OC中runtime最为吸引人的地方。
<!--more-->

#objc_msgSend函数简介

最初接触到OC Runtime，一定是从[receiver message]这里开始的。[receiver message]会被编译器转化为：

	id objc_msgSend ( id self, SEL op, ... );
	
这是一个可变参数函数。第二个参数类型是SEL。SEL在OC中是方法选择器。

	typedef struct objc_selector *SEL;
	
objc_selector是一个映射到方法的C字符串。下面就做了个测试：

	-(NSString *) selectorTestWithName:(NSString *)strName sex:(NSString *) strSex
	{
	    NSLog(@"sel = %s", @selector(selectorTestWithName:sex:));
	    return nil;
	}

输出如下：

	sel = selectorTestWithName:sex:

需要注意的是@selector()选择子只与函数名有关。不同类中相同名字的方法所对应的方法选择器是相同的，即使方法名字相同而变量类型不同也会导致它们具有相同的方法选择器。由于这点特性，也导致了OC不支持函数重载。

在receiver拿到对应的selector之后，如果自己无法执行这个方法，那么该条消息要被转发。或者临时动态的添加方法实现。如果转发到最后依旧没法处理，程序就会崩溃。

**所以编译期仅仅是确定了要发送消息，而消息如何处理是要运行期需要解决的事情。**

objc_msgSend函数究竟会干什么事情呢？


1.检测这个 selector是不是要忽略的。 
2.检查target是不是为nil。
如果这里有相应的nil的处理函数，就跳转到相应的函数中。 如果没有处理nil的函数，就自动清理现场并返回。这一点就是为何在OC中给nil发送消息不会崩溃的原因。
3.确定不是给nil发消息之后，在该class的缓存中查找方法对应的IMP实现。
如果找到，就跳转进去执行。 如果没有找到，就在方法分发表里面继续查找，一直找到NSObject为止。
4.如果还没有找到，那就需要开始消息转发阶段了。至此，发送消息Messaging阶段完成。**这一阶段主要完成的是通过select()快速查找IMP的过程**。

# 消息发送Messaging阶段—objc_msgSend源码解析

在这篇文章Obj-C Optimization: The faster objc_msgSend中看到了这样一段C版本的objc_msgSend的源码。

#include <objc/objc-runtime.h>

	id  c_objc_msgSend( struct objc_class /* ahem */ *self, SEL _cmd, ...)  
	{
	   struct objc_class    *cls;
	   struct objc_cache    *cache;
	   unsigned int         hash;
	   struct objc_method   *method;   
	   unsigned int         index;
	
	   if( self)
	   {
	      cls   = self->isa;
	      cache = cls->cache;
	      hash  = cache->mask;
	      index = (unsigned int) _cmd & hash;
	
	      do
	      {
	         method = cache->buckets[ index];
	         if( ! method)
	            goto recache;
	         index = (index + 1) & cache->mask;
	      }
	      while( method->method_name != _cmd);
	      return( (*method->method_imp)( (id) self, _cmd));
	   }
	   return( (id) self);
	
	recache:  
	   /* ... */
	   return( 0);
	}

该源码中有一个do-while循环，这个循环就是上一章里面提到的在方法分发表里面查找method的过程。

不过在obj4-680里面的objc-msg-x86_64.s文件中实现是一段汇编代码。


	/********************************************************************
	 *
	 * id objc_msgSend(id self, SEL _cmd,...);
	 *
	 ********************************************************************/
	
	 .data
	 .align 3
	 .globl _objc_debug_taggedpointer_classes
	_objc_debug_taggedpointer_classes:  
	 .fill 16, 8, 0
	
	 ENTRY _objc_msgSend
	 MESSENGER_START
	
	 NilTest NORMAL
	
	 GetIsaFast NORMAL  // r11 = self->isa
	 CacheLookup NORMAL  // calls IMP on success
	
	 NilTestSupport NORMAL
	
	 GetIsaSupport NORMAL
	
	// cache miss: go search the method lists
	LCacheMiss:  
	 // isa still in r11
	 MethodTableLookup %a1, %a2 // r11 = IMP
	 cmp %r11, %r11  // set eq (nonstret) for forwarding
	 jmp *%r11   // goto *imp
	
	 END_ENTRY _objc_msgSend
	
	
	 ENTRY _objc_msgSend_fixup
	 int3
	 END_ENTRY _objc_msgSend_fixup
	
	
	 STATIC_ENTRY _objc_msgSend_fixedup
	 // Load _cmd from the message_ref
	 movq 8(%a2), %a2
	 jmp _objc_msgSend
	 END_ENTRY _objc_msgSend_fixedup



