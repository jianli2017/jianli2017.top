---
title: 二级指针动态申请内存
date: 2016-10-29 15:53:34
tags: C 语言语法
categories: C 语言语法
---
在读《高质量CC++编程指南》的时候，为了透彻理解二级指针传递内存的原理，写下本文，本文用图形化的方式描述二级指针传递内存的原理。

<!--more-->

# 一级指针不能动态申请内存
如果函数的参数是一个指针，不要指望用该指针去申请动态内存，下面的代码试图用GemMemory函数获取内存失败。

	void GetMemory(char *p,int num)
	{
			p = (char*)malloc(sizeof(char)*num);
	}
	
	void Test(void)
	{
			char * sz = NULL;
			GeyMemory(sz,100);  ///执行完sz任然为NULL
			strcpy(sz,"hello");  ///运行错误
	}
	
下面给出一张图，说明GemMemory函数为什么传递不了内存。

![一级指针示意图](http://of685p9vy.bkt.clouddn.com/%E4%B8%80%E7%BA%A7%E6%8C%87%E9%92%88%E7%A4%BA%E6%84%8F%E5%9B%BE.bmp)
编译器总是为函数的每个参数制作临时副本，在被调用函数内部使用副本，指针p的副本是_p,函数执行前，_p = p ,都指向地址0x00000000处，用虚线头表示；GemMemory函数执行时，_p指向了新申请的内存0xA0101010地址；函数调用结束后，p还指向地址0x00000000处，_p指向了0xA0101010，但应该已经不存在，_p改变不能改变p指针，不能让p指针指向地址0xA0101010出处，执行完的指针指向使用实线表示。所以GemMemory不能传递内存。
# 二级指针动态申请内存

下面是二级指针动态申请内存的示例代码：

	void GetMemory2(char **p,int num)
	{
			*p = (char*)malloc(sizeof(char)*num);
	}
	
	void Test(void)
	{
			char * sz = NULL;
			GeyMemory2(&sz,100);  
			strcpy(sz,"hello");  ///没有问题 
	}
	
下面给出一张图，说明GemMemory2函数可以传递内存的原因。图中所有地址都是随机写的，不是真实地址。

![二级指针示意图](http://of685p9vy.bkt.clouddn.com/%E4%BA%8C%E7%BA%A7%E6%8C%87%E9%92%88%E7%A4%BA%E6%84%8F%E5%9B%BE.bmp)
现在假设，指针p的副本是_p,函数执行前，_p = p，他俩都指向地址0x88888888，0x88888888地址的内容还是个地址，指向地址0x00000000 ；GemMemory2执行时， \*_p指向了0xB0101010地址，也就是说，地址0x88888888中的内容变为0xB0101010，那么，\*p也就指向了0xB0101010地址。修改\*_p，\*p也被修改；函数执行结束：二级指针P最终指向了地址0xB0101010。所以GemMemory2可以动态申请内存。