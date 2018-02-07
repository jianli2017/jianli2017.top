---
title: flexBox 伸缩盒子模型
date: 2016-12-2 16:18:26
categories: weex
tags: flexBox
---
本文主要说明下flexBox的各个效果
<!--more-->

#  一、盒子模型

## 1.1 标准盒子模型

标准的盒子模型如下图所示：

![17](http://of685p9vy.bkt.clouddn.com/17.png)

## 1.2 伸缩盒子模型

任何一个元素都可以指定为flexbox 布局，设置为display:flex或display:inline-flex的元素称为伸缩容器，伸缩容器的子元素称为伸缩项目，下面是伸缩的模型：
![1.1](http://of685p9vy.bkt.clouddn.com/1.1.png)

# 二、React Native中使用flexBox

1. flexDirection（伸缩容器）

2. alignItems（伸缩容器）

3. flexWrap（伸缩容器）
4. justifyContent（伸缩容器）
5. alignSelf（伸缩项目）
6. flex （伸缩项目）


## 2.1 flexDirection 指定主轴方向

~~~
flexDirection:row|column
~~~

row、column的效果图如下：

![1](http://of685p9vy.bkt.clouddn.com/1.png)

![2](http://of685p9vy.bkt.clouddn.com/1.png)


## 2.2 alignItems 该属性用来定义伸缩项目在伸缩容器的交叉轴上的对齐方式

~~~
alignSelf:auto|flex-start|flex-end|center|stretch
~~~

flex-start、flex-end、center的效果图如下：

![3](http://of685p9vy.bkt.clouddn.com/3.png)

![4](http://of685p9vy.bkt.clouddn.com/4.png)

![5](http://of685p9vy.bkt.clouddn.com/5.png)



## 2.3 flexWrap 指定伸缩容器的主轴方向空间不足的情况下，是否换行以及如何换行

~~~
flexWrap:wrap|nowrap
~~~
wrap、nowrap的效果图如下：

![6](http://of685p9vy.bkt.clouddn.com/6.png)

![7](http://of685p9vy.bkt.clouddn.com/7.png)

## 2.4 justifyContent指定伸缩项目沿主轴线的对齐方式

~~~
justifyContent:flex-start|flex-end|center|space-between|space-around
~~~

flex-start、flex-end、center、space-between、space-around的效果图如下：

![8](http://of685p9vy.bkt.clouddn.com/8.png)

![9](http://of685p9vy.bkt.clouddn.com/9.png)

![10](http://of685p9vy.bkt.clouddn.com/10.png)

![11](http://of685p9vy.bkt.clouddn.com/11.png)

![12](http://of685p9vy.bkt.clouddn.com/12.png)

## 2.5  alignSelf 设置单独的伸缩项目在交叉轴上的对齐方式，会覆写默认的对齐方式

~~~
alignSelf:auto|flex-start|flex-end|center|stretch
~~~

lex-start、flex-end、center 的效果图如下：

![13](http://of685p9vy.bkt.clouddn.com/13.png)

![14](http://of685p9vy.bkt.clouddn.com/14.png)

![15](http://of685p9vy.bkt.clouddn.com/15.png)

## 2.6 flex

~~~
flex:number
~~~

分别设置四个伸缩项目的 flex为3、2、1、4，效果图如下：

![16](http://of685p9vy.bkt.clouddn.com/16.png)