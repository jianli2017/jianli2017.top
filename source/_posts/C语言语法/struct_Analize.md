---
title: struct定义语法
date: 2016-10-26 18:13:26
tags: C 语言语法
categories: C 语言语法
---

在C语言中，若使用struct node {}这样来定义结构体的话。在定义变量时，需要这样写：struct node n;
若用typedef定义结构体，typedef struct node{}NODE; 。在定义变量时可以这样写，NODE n;
区别就在于使用时，是否可以省去struct这个关键字。

<!--more-->

# 定义结构体

## C 定义结构体 
在C中定义一个结构体类型要用typedef:

	typedef struct Student
	{
		int a;
	}Stu;

Stu就变为了结构体类型，等同于struct Student，相当于一个别名。于是声明变量可以使用：
1. Stu stu1;
2. struct Student stu1;


另外这里也可以不写Student（于是也不能struct Student stu1;了）

	typedef struct
	{
		int a;
	}Stu;
## C++ 定义结构体 
但在c++里很简单，直接

	struct Student
	{
	int a;
	};
	
于是就定义了结构体类型Student，声明变量时直接Student stu2；

在c++中如果用typedef的话，又会造成区别：
	
	struct Student
	{
	int a;
	}stu1;//stu1是一个变量
	
	typedef struct Student2
	{
	int a;
	}stu2;//stu2是一个结构体类型
	
使用时可以直接访问stu1.a
但是stu2则必须先 stu2 s2;
然后 s2.a=10;

# struct 与class 的区别
在C++里struct 关键字与class 关键字一般可以通用，只有一个很小的区别。struct 的成员默认情况下属性是public 的，而class 成员却是private 的。很多人觉得不好记，其实很容易。你平时用结构体时用public 修饰它的成员了吗？既然struct 关键字与class 关键字可以通用，你也不要认为结构体内不能放函数了。

#位域
有些信息在存储时，并不需要占用一个完整的字节， 而只需占几个或一个二进制位。为了节省存储空间，并使处理简便，C语言提供了一种数据结构，称为“位域”或“位段”。所谓“位域”是把一个字节中的二进位划分为几 个不同的区域，并说明每个区域的位数。每个域有一个域名，允许在程序中按域名进行操作。 这样就可以把几个不同的对象用一个字节的二进制位域来表示。
一、位域的定义和位域变量的说明位域定义与结构定义相仿，其形式为： 

	struct 位域结构名 
	{ 位域列表 };
其中位域列表的形式为： 类型说明符 位域名：位域长度 
例如：

	struct bs
	{
	int a:8;
	int b:2;
	int c:6;
	};
下面是IOS runtime中使用位域的一个例子

	 struct {
	        uintptr_t indexed           : 1; // 0表示普通的isa指针 1表示优化过的，存储引用计数
	        uintptr_t has_assoc         : 1; // 对象是否包含 associated object，如果没有，析构时会更快
	        uintptr_t has_cxx_dtor      : 1; // 是否有C++或ARC的析构函数，如果没有，析构时会更快
	        uintptr_t shiftcls          : 33; // 最重要的原来的Class cls部分，占33个bit，与 ISA_MASK 进行 & 操作可以得到  // MACH_VM_MAX_ADDRESS 0x1000000000
	        uintptr_t magic             : 6; // 用于调试时分辨对象是否完成初始化
	        uintptr_t weakly_referenced : 1; // 对象是否有过weak引用，如果没有，析构时会更快
	        uintptr_t deallocating      : 1; // 对象是否正在析构
	        uintptr_t has_sidetable_rc  : 1; // 表示对象的引用计数过大，无法存储在isa指针，只能存在side table中
	        uintptr_t extra_rc          : 19; // 存储引用计数，不过好像是减 1 后的值，可以在 rootRetainCount 方法中看到
	        // 在 64 位环境下，优化的 isa 指针并不是就一定会存储引用计数，毕竟用 19bit （iOS 系统）保存引用计数不一定够。需要注意的是这 19 位保存的是引用计数的值减一。has_sidetable_rc 的值如果为 1，那么引用计数会存储在一个叫 SideTable 的类的属性中。
	#       define RC_ONE   (1ULL<<45) // 左移 45 bit，正好是extra_rc 所在的位置
	#       define RC_HALF  (1ULL<<18) // extra_rc 总共是19位，RC_HALF是18位，也就是全部引用计数的一半
	    };


# 总结
在C语言和C++语言中，struct的定义感觉非常难记忆，我记忆的方法是：首先记住C语言中的struct的定义，“typedef struct Student{}Stu;”，struct Student整体才当做是类型，至于C++的struct的定义不用记，就把struct当成是class，class怎么使用，struct就怎么使用。

