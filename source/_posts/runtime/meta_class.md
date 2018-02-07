---
title: What is a meta-class in Objective-C(译文)
date: 2016-11-12 19:56:17
tags: runtime
categories: runtime
---

本文是对[What is a meta-class in Objective-C](http://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html)的翻译,这篇文章主要讲解Objective-C中的元类（meta-class），元类对runtime的理解非常有用。不过快翻译完后，发现已经有前辈翻译过了--[Objective-C 中的元类（meta class）是什么？](http://ios.jobbole.com/81657/)。后面就使用它的翻译。

<!--more-->

In this post, I look at one of the stranger concepts in Objective-C — the meta-class. Every class in Objective-C has its own associated meta-class but since you rarely ever use a meta-class directly, they can remain enigmatic. I'll start by looking at how to create a class at runtime. By examining the "class pair" that this creates, I'll explain what the meta-class is and also cover the more general topic of what it means for data to be an object or a class in Objective-C.

在这篇文章中，我讲解Objective-C中的一个陌生概念——meta-class。在Objective-C，每个类都有一个相关meta-class，由于你很少直接使用meta-class，所以meta-class仍感觉很神秘。我从运行时如何创建一个meta-class开始，然后通过查看创建出的两个类（class 和meta-class），解释什么是meta-class，其中，还会涉及它对于Objective-C中对象和类的意义。

# Creating a class at runtime

The following code creates a new subclass of NSError at runtime and adds one method to it:
下面的代码在运行时创建了一个NSError的子类RuntimeErrorSubclass，并且为RuntimeErrorSubclass添加一个类方法：

	Class newClass =
	    objc_allocateClassPair([NSError class], "RuntimeErrorSubclass", 0);
	class_addMethod(newClass, @selector(report), (IMP)ReportFunction, "v@:");
	objc_registerClassPair(newClass);
	
The method added uses the function named ReportFunction as its implementation, which is defined as follows:
新添加的实现 是ReportFunction函数。定义如下：

	void ReportFunction(id self, SEL _cmd)
	{
	    NSLog(@"This object is %p.", self);
	    NSLog(@"Class is %@, and super is %@.", [self class], [self superclass]);
	    
	    Class currentClass = [self class];
	    for (int i = 1; i < 5; i++)
	    {
	        NSLog(@"Following the isa pointer %d times gives %p", i, currentClass);
	        currentClass = object_getClass(currentClass);
	    }
	
	    NSLog(@"NSObject's class is %p", [NSObject class]);
	    NSLog(@"NSObject's meta class is %p", object_getClass([NSObject class]));
	}


On the surface, this is all pretty simple. Creating a class at runtime is just three easy steps:
表面上，这非常的简单，在运行时创建一个类只需要三步：

1. Allocate storage for the "class pair" (using objc_allocateClassPair).
2. Add methods and ivars to the class as needed (I've added one method using class_addMethod).
3. Register the class so that it can be used (using objc_registerClassPair).


1. 使用objc_allocateClassPair函数为两个类（class pair）申请内存。
2. 使用class_addMethod函数为类添加方法。
3. 使用objc_registerClassPair注册类。

However, the immediate question is: what is a "class pair"? The function objc_allocateClassPair only returns one value: the class. Where is the other half of the pair?
然而，问题就马上来了，什么是“class pair”？ objc_allocateClassPair函数只返回了一个值：newClass，一对类的另一半是什么？

I'm sure you've guessed that the other half of the pair is the meta-class (it's the title of this post) but to explain what that is and why you need it, I'm going to give some background on objects and classes in Objective-C.
我确信你已经猜到：一对类的另一半是meta-class。 为了解释它是什么和我们为什么需要它，还需要交代下Objective-C的对象和类的相关背景
# 什么数据结构才能称之为对象
What is needed for a data structure to be an object?

**Every object has a class**. This is a fundamental object-oriented concept but in Objective-C, it is also a fundamental part of the data. Any data structure which has a pointer to a class in the right location can be treated as an object.
每个对象都有一个类。这是面向对象的基本概念。但是在Objective-C中，它对数据结构也一样。含有一个指针且该指针可以正确指向类的数据结构，都可以被视作为对象。

In Objective-C, an object's class is determined by its isa pointer. **The isa pointer points to the object's Class**.
在Objective-C中，对象的类是有isa指针决定的。isa指针指向对象所属的类。

In fact, the basic definition of an object in Objective-C looks like this:
事实上，在Objective-C中，对象的基本定义如下：

	typedef struct objc_object {
	    Class isa;
	} 

What this says is: any structure which starts with a pointer to a Class structure can be treated as an objc_object.
这就是说：以指向类结构指针开始的数据结构，可以被认为是objc_object（这里使用objc_object，表示包括实例和类）；

The most important feature of objects in Objective-C is that you can send messages to them:
在Objective-C中，objc_object的主要特征是可以向它发生消息。

	[@"stringValue"
	    writeToFile:@"/file.txt" atomically:YES encoding:NSUTF8StringEncoding error:NULL]

This works because when you send a message to an Objective-C object (like the NSCFString here), the runtime follows object's isa pointer to get to the object's Class (the NSCFString class in this case). The Class then contains a list of the Methods which apply to all objects of that Class and a pointer to the superclass to look up inherited methods. The runtime looks through the list of Methods on the Class and superclasses to find one that matches the message selector (in the above case, writeToFile:atomically:encoding:error on NSString). The runtime then invokes the function (IMP) for that method.
上面的例子可以运行的原因是：当向对象（NSCFString）发送消息时，runtime根据跟随isa指针找到对象所属类（这里是NSCFString），对象的类包含方法列表（类的所有对象都具有）和指向父类的指针（包含继承来的方法列表）。这个类包含了能应用于这个类的所有实例方法和指向超类的指针以便可以找到父类的实例方法。运行时库检查这个类和其超类的方法列表，找到一个匹配这条消息的方法（在上面的代码里，是NSString类的writeToFile:atomically:encoding:error方法）。运行时库基于那个方法调用函数（IMP）。

The important point is that the Class defines the messages that you can send to an object.
Class的主要特点是：类定义了发送给对象的消息。

# 什么是元类
What is a meta-class?

Now, as you probably already know, a Class in Objective-C is also an object. This means that you can send messages to a Class.
现在，你可能已经知道，在Objective-C中，一个Class也是一个objc_object。也就是说：你可以向Class发送消息。

	NSStringEncoding defaultStringEncoding = [NSString defaultStringEncoding];

In this case, defaultStringEncoding is sent to the NSString class.
上面这个例子里，defaultStringEncoding消息发送给了NSString类。

This works because every Class in Objective-C is an object itself. This means that the Class structure must start with an isa pointer so that it is binary compatible with the objc_object structure I showed above and the next field in the structure must be a pointer to the superclass (or nil for base classes).
可以向Class发送消息的原因是：在Objective-C中，任何一个类都是一个objc_object。这就是说：类的数据结构必须是以isa开头，这样才可以被看做是objc_object，紧跟着isa后面的是指向父类的指针（对于基类为nil）。

As I showed last week, there are a couple different ways that a Class can be defined, depending on the version of the runtime you are running, but yes, they all start with an isa field followed by a superclass field.
如我上周说的，Class在不同的runtime版本中有两种不同的定义。但他们都以isa指针开始，后续跟着指向父类的指针。

	typedef struct objc_class *Class;
	struct objc_class {
	    Class isa;
	    Class super_class;
	    /* followed by runtime specific details... */
	};

However, in order to let us invoke a method on a Class, the isa pointer of the Class must itself point to a Class structure and that Class structure must contain the list of Methods that we can invoke on the Class.
然而，为了可以调用Class的方法，Class的isa指针必须指向包含类方法的类结构体。

This leads to the definition of a meta-class: the meta-class is the class for a Class object.
**引出meta-class的原因：meta-class是类对象的类**

Simply put:
简言之

1. When you send a message to an object, that message is looked up in the method list on the object's class.
2. When you send a message to a class, that message is looked up in the method list on the class' meta-class.


1. 当向对象发送消息的时候，消息在对象的类中查找
2. 当向类发送消息，消息在类的meta-class中查找

The meta-class is essential because it stores the class methods for a Class. There must be a unique meta-class for every Class because every Class has a potentially unique list of class methods.
meta-class重要的原因是：meta-calss存储了类的类方法。每个类都有一个唯一的元类，原因是每个类都有自己唯一的类方法列表。

# 元类的类是什么？
What is the class of the meta-class?

The meta-class, like the Class before it, is also an object. This means that you can invoke methods on it too. Naturally, this means that it must also have a class.
meta-class就像前面提到的Class，也是一个objc_object。就是说可以向meta-class发送消息。自然的，meta-class也有一个类。

All meta-classes use the base class' meta-class (the meta-class of the top Class in their inheritance hierarchy) as their class. This means that for all classes that descend from NSObject (most classes), the meta-class has the NSObject meta-class as its class.
所有元类的类是根基类的元类。也就是说：所有从NSObject继承下来的类，他们的元类的类是NSObject的元类。
这就意味着所有NSObject的子类（大多数类）的元类都会以NSObject的元类作为他们的类

Following the rule that all meta-classes use the base class' meta-class as their class, any base meta-classes will be its own class (their isa pointer points to themselves). This means that the isa pointer on the NSObject meta-class points to itself (it is an instance of itself).
依据下面的规则：所有的元类使用根基类的元类作为他们的类，根元类的元类则就是它自己。也就是说基类（NSObject）的元类的isa指针指向他自己（NSObject）。

#类和元类的继承
Inheritance for classes and meta-classes

In the same way that the Class points to the superclass with its super_class pointer, the meta-class points to the meta-class of the Class' super_class using its own super_class pointer.
类用 super_class指针指向了超类，同样的，元类用super_class指向类的super_class的元类。

As a further quirk, the base class' meta-class sets its super_class to the base class itself.
说的更拗口一点就是，根元类把它自己的基类设置成了super_class。

The result of this inheritance hierarchy is that all instances, classes and meta-classes in the hierarchy inherit from the hierarchy's base class.
在这样的继承体系下，所有实例、类以及元类（meta class）都继承自一个基类。

For all instances, classes and meta-classes in the NSObject hierarchy, this means that all NSObject instance methods are valid. For the classes and meta-classes, all NSObject class methods are also valid.
这意味着对于继承于NSObject的所有实例、类和元类，他们可以使用NSObject的所有实例方法，类和元类可以使用NSObject的所有类方法

All this is pretty confusing in text. Greg Parker has put together an excellent diagram of instances, classes, meta-classes and their super classes and how they all fit together.
这些文字看起来莫名其妙难以理解。Greg Parker给出了一份精彩的图谱来展示这些关系。
![图谱](http://of685p9vy.bkt.clouddn.com/meta-class1.png)
# 实验证明
Experimental confirmation of this

To confirm all of this, let's look at the output of the ReportFunction I gave at the start of this post. The purpose of this function is to follow the isa pointers and log what it finds.

To run the ReportFunction, we need to create an instance of the dynamically created class and invoke the report method on it.

id instanceOfNewClass =
    [[newClass alloc] initWithDomain:@"someDomain" code:0 userInfo:nil];
[instanceOfNewClass performSelector:@selector(report)];
[instanceOfNewClass release];
Since there is no declaration of the report method, I invoke it using performSelector: so the compiler doesn't give a warning.

The ReportFunction will now traverse through the isa pointers and tell us what objects are used as the class, meta-class and class of the meta-class.

Getting the class of an object: the ReportFunction uses object_getClass to follow the isa pointers because the isa pointer is a protected member of the class (you can't directly access other object's isa pointers). The ReportFunction does not use the class method to do this because invoking the class method on a Class object does not return the meta-class, it instead returns the Class again (so [NSString class] will return the NSString class instead of the NSString meta-class).
This is the output (minus NSLog prefixes) when the program runs:

This object is 0x10010c810.
Class is RuntimeErrorSubclass, and super is NSError.
Following the isa pointer 1 times gives 0x10010c600
Following the isa pointer 2 times gives 0x10010c630
Following the isa pointer 3 times gives 0x7fff71038480
Following the isa pointer 4 times gives 0x7fff71038480
NSObject's class is 0x7fff710384a8
NSObject's meta class is 0x7fff71038480
Looking at the addresses reached by following the isa value repeatedly:

the object is address 0x10010c810.
the class is address 0x10010c600.
the meta-class is address 0x10010c630.
the meta-class's class (i.e. the NSObject meta-class) is address 0x7fff71038480.
the NSObject meta-class' class is itself.
The value of the addresses is not really important except that it shows the progress from class to meta-class to NSObject meta-class as discussed.

# 结论（Conclusion）

1. The meta-class is the class for a Class object. Every Class has its own unique meta-class (since every Class can have its own unique list of methods). This means that all Class objects are not themselves all of the same class.
元类是 Class 对象的类。每个类（Class）都有自己独一无二的元类（每个类都有自己第一无二的方法列表）。这意味着所有的类对象都不同。
2. The meta-class will always ensure that the Class object has all the instance and class methods of the base class in the hierarchy, plus all of the class methods in-between. For classes descended from NSObject, this means that all the NSObject instance and protocol methods are defined for all Class (and meta-class) objects.
元类总是会确保类对象和基类的所有实例和类方法。对于从NSObject继承下来的类，这意味着所有的NSObject实例和protocol方法在所有的类（和meta-class）中都可以使用。
3. All meta-classes themselves use the base class' meta-class (NSObject meta-class for NSObject hierarchy classes) as their class, including the base level meta-class which is the only self-defining class in the runtime.
所有的meta-class使用基类的meta-class作为自己的类，对于根基类的meta-class也是一样，只是它指向自己而已。


# 读后总结

翻译完后，总是感觉还是不好理解，所以自己画了一张图，方便理解。
![meta-class总结](http://of685p9vy.bkt.clouddn.com/meta-class2.png)

1. 具有指向Class的isa指针的数据结构可以称之为objc_objcet对象。
2. objc_object对象的主要特点是可以向它发送消息。
3. Object、Class、meta-class都属于objc_class。
4. 对象的isa指针指向对象所属的类，对象所属的类保存对象的实例方法。
5. 类的isa指针指向类的元类（meta-class）。类的元类保存类的类方法。
6. 向实例发送消息，消息查找的路径是：跟随对象的isa指针，找到对象所属的类，在类中查找，如果类中没有，依次从超类中查找直到根基类。
7. 向对象发行消息，消息查找的路径是：跟随类的isa指针，找到类的元类，在元类中查找，如果元类中没有，依次从元类的超类中查找，如果最后根元类也没有，在根基类（NSObject）中查找。
8. 元类就是类的类（就是类的isa指针）。
9. 元类的作用：当向对象和类发送消息时，统一了类对象和对象查找方法的机制---都从isa中查找。

