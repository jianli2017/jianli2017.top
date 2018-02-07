---
title: 搭建weex断点调试环境 
date: 2016-12-28 09:48:40
categories: weex
tags: weex断点调试
---
Weex Devtools是为 Weex开发者服务的一款调试工具，可对 .we 代码及 JavaScript 代码断点调试，并能够审查 Weex app 运行时属性，支持 iOS 和 Android 两个平台。本文是基于IOS平台。

<!--more-->
Weex Devtools 基于 Chrome devtools 实现了 Chrome Debugging Protocol，能够使用Chrome devtools调试 Weex 项目，其主要功能分为两大部分—— Debugger 和 Inspector。若使用Devtools调试weex项目，需要搭建调试环境，包括两部分：安装weex-devtool模块、App项目中集成weex-devtool SDK。
# 安装weex-devtool 模块
weex-devtool是node.js的一个模块，用来启动服务器和Chrome页面，安装weex-devtool需要使用npm安装，由于使用npm安装特别慢，有时候一点速度都没有，这里使用淘宝的镜像源，切换方法：

	npm install cnpm -g --registry=https://registry.npm.taobao.org
切换完后，使用cnpm安装weex-devtool：

	cnpm install -g weex-devtool

weex-devtool 的用法：

	weex debug [options] [we_file|bundles_dir]
选项有下面几种：

	-h, --help           显示帮助
	-V, --verbose        显示debug服务器运行时的各种log
	-v, --version        显示版本
	-p, --port [port]    设置debug服务器端口号 默认为8088
	-e, --entry [entry]  debug一个目录时,这个参数指定整个目录的入口bundle文件,这个bundle文件的地址会显示在debug主页上(作为二维码)
	-m, --mode [mode]    设置构建we文件的方式,transformer 最基础的风格适合单文件,loader:wepack风格 适合模块化的多文件.默认为transformer
启动服务和chrome页面：

	weex debug  
	
输出如下：

	bogon:~ lijian$ weex debug
	start debugger server at http://10.144.36.206:8088
	
	The websocket address for native is ws://10.144.36.206:8088/debugProxy/native
	Launching Dev Tools...

输出上面的内容，表示weex-devtool已经并启动。其中ws://10.144.36.206:8088/debugProxy/native是APP连接到Chrome的调试的地址。App集成weex-devtool-iOS后，需要使用这个地址。
# APP集成weex-devtool-iOS SDK

集成weex-devtool-iOS SDK，可以参考[weex-devtool-iOS](https://github.com/weexteam/weex-devtool-iOS/blob/master/README-zh.md)将 weex-devtool-iOS 集成到项目中。这篇文章中介绍了pod集成方法和源码集成方法。这里假设已经集成完成。直接到使用SDK的步骤。
在AppDelegate中添加下面的代码，就可以使APP链接到Chrome的调试环境：

	#import <WXDevtool.h>
	
	[WXDevTool setDebug:YES];
	[WXDevTool launchDevToolDebugWithUrl:@"ws://10.144.36.206:8088/debugProxy/native"];
setDebug:参数为YES时，直接开启debug模式，反之关闭。launchDevToolDebugWithUrl 中的url就是在控制台启动Chrome时输出的地址。

# 调试

到目前为止，环境已配置完成，可以体验下调试过程。启动App，App会连接到chrome，Chrome中会显示出连接上的APP，如下：
![weex-debug-chrome](http://of685p9vy.bkt.clouddn.com/weex-debug-chrome.png)

如图所示，有两个功能debug（调试）、inspector（元素省察）。其中debug可以调试JS的代码；inspector 可以审查界面的元素。点击debuger，进入调试提示页面，界面如下：
![weex-debug-chrome](http://of685p9vy.bkt.clouddn.com/weex-debug-chrome1.png)
界面提示：使用option+commond+j进入调试JS代码界面，点击Sources标签，左边的导航栏显示源码列表，可以切换源码。调试面板包括设置断点、单步执行、查看运行时变量值等功能。
![weex-debug-chrome](http://of685p9vy.bkt.clouddn.com/weex-debug-chrome2.png)

具体详细的调试方法请参考[如何使用 Devtools 调试 Weex 页面](http://weex-project.io/cn/doc/how-to/debug-with-devtools.html)。
