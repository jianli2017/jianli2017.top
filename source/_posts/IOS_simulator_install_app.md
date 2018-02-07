---
title: 给IOS模拟器按照APP
date: 2016-10-27 09:15:55
tags: IOS 工具
categories: IOS 工具
---

这里介绍了给IOS模拟器安装APP的方法。

<!--more-->
# 安装步骤

1. 安装HomeBrew ,终端输入 ：

	ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
  
2. 安装node.js, 终端输入：

   brew install node

3. 安装ios-sim，在终端中输入 ：

	 npm install ios-sim -g
	 
4. 安装、启动APP

	ios-sim launch APP路径 --devicetypeid iPhone-6s
	 
	 
	 
