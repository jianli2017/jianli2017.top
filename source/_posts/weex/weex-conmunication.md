---
title: weex 通信原理分析
date: 2016-12-13 18:01:31
categories: weex
tags: weex
---
本文首先简单阐述了JavaScriptCore中JS和Native的通信机制，在此基础上，分析了weex中的JS和Native的通信机制。
<!--more-->

# 万物之源 -JavaScriptCore

不论是RN、weex、JSPatch，他们的核心交互机制都使用JavaScriptCore，通过它完成JS和Native的通信。关于JavaScriptCore的简单介绍可以参考
[IOS7开发～JavaScriptCore （一）](http://blog.csdn.net/lizhongfu2013/article/details/9232129)，这里只简单提及下。

## 关键类-JSContext、JSValue
JavaScriptCore中的两个关键类：
1. JSContext(JS脚本的执行环境)
2. JSValue（JS 和Native传值的载体）
 
JSContext 的核心方法- (JSValue *)evaluateScript:(NSString *)script;，功能是执行JS脚本代码，执行完后，会将JS中的对象、方法添加到JSContext上下文中的全局对象中。这样就可以通过JSContext引用到JS代码中的对象、方法。

JSValue是用来呈现JS的对象，它可以将JS中的对象、方法转化为Native的对象、函数，反之亦然。下面是JSValue的两个核心方法：

	- (JSValue *)callWithArguments:(NSArray *)arguments; 
	- (JSValue *)invokeMethod:(NSString *)method 
	            withArguments:(NSArray *)arguments;
	            
如果JSValue实例代表JS中一个函数变量（变量中存储的是函数），用callWithArguments调用JS的函数。如果拿到了JS代码的执行环境的全局对象，可以向全局对象发送invokeMethod消息调用JS中的函数。参数包括JS的函数名称、调用参数。

## Native调用JS机制
可以使用JSValue 的callWithArguments 和 invokeMethod方法 单独调用JS中的某个函数 。
也可以使用JSContext 的evaluateScript方法执行整个JS代码。下面给出一个例子，其中，test.js的内容如下 

	///匿名函数 
	var functionVar = function(num)
	{
	    return num + 1;
	}
	
	function jsFucton(num)
	{
	    return num + 1;
	}
	
	jsFucton(2);
	
测试代码如下： 

	-(void) nativeCallJS
	{
	    NSString *path = [[NSBundle mainBundle]pathForResource:@"test"ofType:@"js"];
	    NSString *testScript = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:nil];
	    JSContext *jsContext = [[JSContext alloc] init];
	    ///1.执行整个JS代码
	    JSValue *resultEvaluate =[jsContext evaluateScript:testScript];
	    NSLog(@"resultEvaluate = %@",[resultEvaluate toObject]);
	    
	    ///2. callWithArguments 调用方式
	    JSValue *fuctionVar =  jsContext[@"functionVar"];
	    JSValue *resultVar = [fuctionVar callWithArguments:@[@(4)]];
	    NSLog(@"resultVar = %@",[resultVar toObject]);
	    
	    ///3. invokeMethod 调用方式
	    JSValue *resultFuction = [[jsContext globalObject] invokeMethod:@"jsFucton" withArguments:@[@(6)]];
	    NSLog(@"resultFuction = %@",[resultFuction toObject]);
	}
输出如下：

	2016-12-14 13:36:27.237 Test2222[15246:172495] resultEvaluate = 3
	2016-12-14 13:36:27.238 Test2222[15246:172495] resultVar = 5
	2016-12-14 13:36:30.166 Test2222[15246:172495] resultFuction = 7

例子中展示了使用JSContext的evaluateScript:方法执行整个JS代码，调用完evaluateScript之后，会将JS代码中的变量、方法添加到jsContext的全局对象中，这样可以使用jsContext[@"functionVar"]访问到JS中的functionVar对象。在JS中，functionVar是一个函数对象，可以向fuctionVar发送callWithArguments消息，执行JS中的functionVar变量中存储的函数。同时也可以使用[[jsContext globalObject] invokeMethod:@"jsFucton" withArguments:@[@(6)]]调用到JS中的jsFucton函数。 上面是Native调用JS的三种方式。每种方式使用的场景不同。

## JS调用Native机制
JS调用Native有两种方式：
1. block
2. JSExport

主要介绍Block方式，因为weex中使用的是Block方式，直接以代码为例：

		-(NSString *) nativeFuction:(NSString *) strParam1 param2:(NSString *)strParam2
		{
		    return [NSString stringWithFormat:@"%@_%@",strParam1,strParam2];
		}
		
		-(void) JSCallNative
		{
		    JSContext *jsContext = [[JSContext alloc] init];
		    
		    ///给JS添加一个方法，JS就能调用到Native的方法
		    __weak typeof(self) weakSelf = self;
		    jsContext[@"jsMethod"] = ^(NSString * strParam1, NSString *strParam2)
		    {
		        NSString * strResult =  [weakSelf nativeFuction:strParam1 param2:strParam2];
		        return strResult;
		    };
		    JSValue *strResult = [jsContext evaluateScript:@"jsMethod('这是第一个参数','这是第二个参数')"];
		    NSLog(@"strResult = %@",[strResult toObject]);
		}  
		输出： strResult = 这是第一个参数_这是第二个参数
在Native中block相当于JS中的function，jsContext[@"jsMethod"]=...相当于给JS添加一个名为jsMethod的function，不过，这个jsMethod的实现是Native的代码。在JS中执行jsMethod方法，就会调用到Native的代码。 
## 小小结
了解了JSContext、JSValue、Native和JS的交互机制，是看懂weex的前提。不论weex如何封装，都离不开上面的基本原理。
Native调用JS有三种方式：
1. JSValue 的callWithArguments
2. JSValue的invokeMethod
3. JSContext的evaluateScript

JS调用Native有两种方式
1. block
2. JSExport
 
# 刨根问底-weex的交互机制
这一节我一步步的分析weex的通信原理，目标两个：
1. 了解weex通信原理
2. 了解通信原理相关的两个重要部分Module、Bridge。

具体的思路是找到SDK初始化的入口，顺着入口逐渐深入分析。直到了解weex的通信机制为止。
## 寻踪觅迹- 分析初始化
weex SDK的入口是WXSDKEngine类，WXSDKEngine 是一个全局类。其中，initSDKEnviroment:完成sdk的初始化工作：

	+ (void)initSDKEnviroment:(NSString *)script
	{
	    [self _registerDefaultComponents];
	    [self _registerDefaultModules];
	    [self _registerDefaultHandlers];
	    
	    [[WXSDKManager bridgeMgr] executeJsFramework:script];
	}

weex有中有三个重要的部分：Components、Modules、Handlers。
Components 是布局Native界面的元素，例如UILabel对应的组件是WXTextComponent，他和html元素text对应。
Modules是Native的功能模块，是JS可以访问的Native类。应该封装了必须有Native完成的功能。
我们主要关注Modules模块，他是weex中Native和JS通信的核心部分。_registerDefaultModules是注册默认的Module：

	+ (void)_registerDefaultModules
	{
	    [self registerModule:@"dom" withClass:NSClassFromString(@"WXDomModule")];
	    [self registerModule:@"navigator" withClass:NSClassFromString(@"WXNavigatorModule")];
	    .....
	}
	
这段代码是注册weex提供给开发者的默认模块。每个模块都实现某种功能，<font color=red size=4 face="黑体">它是JS可以调用的Native类</font>。接着分析registerModule函数： 

	+ (void)registerModule:(NSString *)name withClass:(Class)clazz
	{
			///注册native的module
	    NSString *moduleName = [WXModuleFactory registerModule:name withClass:clazz];
	    
	    ///组成JS的module
	    NSDictionary *dict = [WXModuleFactory moduleMethodMapsWithName:moduleName];
	    [[WXSDKManager bridgeMgr] registerModules:dict];
	}
这段代码完成两个功能，初始化Native端Module的配置信息、初始化JS端Module的配置信息。到目前为止，我们已经找到了weex的入口，后续部分需要分两个单独的模块分析讲解。一个是Module，一个是通信机制。

## 刨根问底-配置信息和Module
Module 相关的类有WXModuleFactory、WXModuleManager、WXModuleProtocol。三个类的类图如下：
![weex-module](http://of685p9vy.bkt.clouddn.com/weex_Module.png)
WXModuleManager只有一个方法，-dispatchMethod:(WXBridgeMethod *)method，根据传入的method在配置文件中找到方法并执行。
WXModuleFactory是Module的工厂类，存储着所有Module的配置信息。
WXModuleProtocol是Module需要遵守的协议。只有遵守了WXModuleProtocol协议的Native类才叫做Module，他才会被添加到Native和JS的配置信息中，这样JS就可以调用到这个类中的方法。这里贴出WXModuleProtocol的代码如下：

	///1. 定义了module JS 回调native的方法  
	typedef void (^WXModuleCallback)(id result);
	typedef void (^WXModuleKeepAliveCallback)(id result, BOOL keepAlive);
	
	#define WX_EXPORT_METHOD(method) \
	+ (NSString *)WX_CONCAT_WRAPPER(wx_export_method_, __LINE__) { \
	    return NSStringFromSelector(method); \
	}
	
	
	///2. 导出类
	#define WX_EXPORT_MODULE(module)
	
	
	///3. 导出方法 (其实是定义了一个类方法 )
	WX_EXPORT_METHOD(@selector(getNetUrl:page:sucCallBack:failCallBack:)) 
	
	
	///4. 执行方法的线程
	- (NSThread *)targetExecuteThread;
	
	@property (nonatomic, weak) WXSDKInstance *weexInstance;
 
主要有三部分：
1. 定义回调
2. 定义导出Native方法的方法
3. 定义Native方法的执行线程。

回调就是JS执行完Native代码后，可以回调的JS的代码。定义导出方法就是：给每个导出方法定义一个类方法，类方法的名字大概就是wx_export_method_100这种样式，返回导出的方法名称。这样做的目的就是不用初始化Module类，就能获取到Module的导出方法信息，并加入到两端的配置信息中。
targetExecuteThread是定义了所有的导出方法执行的线程。如果没有实现，默认在主线程执行方法。如果实现，返回对应的线程，就在该线程中执行方法。
到目前为止，Native端的Module配置信息都讲完了。总结为：配置信息中包括了所有Module的导出方法、Module的名称、对应的类。
接下来简单分析下JS端的配置信息。注册Module的信息到JS，调用的是WXBridgeManager的registerModules 方法：

	- (void)registerModules:(NSDictionary *)modules
	{
	    if (!modules) return;
	    
	    __weak typeof(self) weakSelf = self;
	    WXPerformBlockOnBridgeThread(^(){
	        [weakSelf.bridgeCtx registerModules:modules];
	    });
	}
	
registerModules: 调用了WXBridgeContext中的registerModules:方法，代码如下：

	- (void)registerModules:(NSDictionary *)modules
	{
	    [self callJSMethod:@"registerModules" args:@[modules]];
	}

内部调用了JS中的方法registerModules，下面转入到main.js，registerModules代码如下：

	function registerModules(modules)
	{
	    if((typeof modules==="undefined"?"undefined":_typeof(modules))==="object")
	    {
	        (0,_register.initModules)(modules)
	    }
	}
	
	
	function initModules(modules,ifReplace)
	{
	    var _loop=function _loop(moduleName)
	    {
	        var methods=nativeModules[moduleName];
	        if(!methods)
	        {
	            methods={};
	            nativeModules[moduleName]=methods
	        }
	        modules[moduleName].forEach(
	                                    function(method)
	                                    {
	                                        if(typeof method==="string")
	                                        {
	                                            method={name:method}
	                                        }
	                                        if(!methods[method.name]||ifReplace)
	                                        {
	                                            methods[method.name]=method
	                                        }
	                                    })
	    };
	    for(var moduleName in modules)
	    {
	        _loop(moduleName)
	    }
	}
大概可以理解为：JS中有个nativeModules变量，这个对象保存着module名称和对应的方法列表。注册就是向nativeModules中添加Module的信息。

## 终极目标-Bridge(通信机制)
接下来了解如何利用这些配置信息完成JS和Native之间的交互。交互就需要一个桥梁，在weex中用Bridge表示。Bridge封装了交互的规则，主要有几个类WXBridgeManager、WXBridgeProtocol、WXBridgeMethod、WXJSCoreBridge、WXBridgeContext。
首先将这几个类的类图贴出来：
![bridge类图](http://of685p9vy.bkt.clouddn.com/weex-bridge.png)
WXBridgeManager是Brider的管理类，承接着Native和JS的交互、调试管理、界面更新、事件传递等任务。具体干活的类是WXBridgeContext和WXJSCoreBridge，其中WXJSCoreBridge是遵守了WXBridgeProtocol协议的类。

图中红色的代码完成Native和JS交互功能的代码。交互分为两部分：JS到Native、Native到JS。首先分析Native到JS这个路径。
executeJsMethod: 是Native调用JS的入口方法。内部调用了WXJSCoreBridge的callJSMethod方法，传递的JS方法名是callJS。具体的代码如下：

	- (void)executeJsMethod:(WXBridgeMethod *)method
	{
	    ....
	    [sendQueue addObject:method];
	    [self performSelector:@selector(_sendQueueLoop) withObject:nil];
	}
	
	- (void)_sendQueueLoop
	{
	    ///构造方法的相关参数
	    ....
	    
	    ///调用方法
	    if ([methods count] > 0 && execIns) {
	        [self callJSMethod:@"callJS" args:@[execIns, methods]];
	    }
	    
	    if (hasTask) {
	        [self performSelector:@selector(_sendQueueLoop) withObject:nil];
	    }
	}
	
	- (void)callJSMethod:(NSString *)method args:(NSArray *)args
	{
	    if (self.frameworkLoadFinished) {
	        [self.jsBridge callJSMethod:method args:args];
	    }
	    else {
	        [_methodQueue addObject:@{@"method":method, @"args":args}];
	    }
	}
	
	///WXJSCoreBridge  的callJSMethod方法
	- (JSValue *)callJSMethod:(NSString *)method args:(NSArray *)args
	{
	    WXLogDebug(@"Calling JS... method:%@, args:%@", method, args);
	    return [[_jsContext globalObject] invokeMethod:method withArguments:args];
	}

上面是Native调用JS的原理 ，下面分析JS调用Native的原理。

当首次调用- (id<WXBridgeProtocol>)jsBridge方法创建Bridge时，会调用Bridge的registerCallNativeJS设置JS调用Native的入口。入口名称为callNative，即JS代码中调用callNative函数，就能调用到Native的代码。当进入Native后，会调用WXBridgeContext的invokeNative方法，内部会查找配置信息，然后invoke相应的方法。代码如下：

	- (id<WXBridgeProtocol>)jsBridge
	{
	    ...
	    _jsBridge = _debugJS ? [NSClassFromString(@"WXDebugger") alloc] : [[WXJSCoreBridge alloc] init];
	     __weak typeof(self) weakSelf = self;
	    [_jsBridge registerCallNative:^NSInteger(NSString *instance, NSArray *tasks, NSString *callback) {
	        return [weakSelf invokeNative:instance tasks:tasks callback:callback];
	    }];
	    ....
	    return _jsBridge;
	}
	
	
	- (void)registerCallNative:(WXJSCallNative)callNative
	{
	    NSInteger (^callNativeBlock)(JSValue *, JSValue *, JSValue *) = ^(JSValue *instance, JSValue *tasks, JSValue *callback){
	        NSString *instanceId = [instance toString];
	        NSArray *tasksArray = [tasks toArray];
	        NSString *callbackId = [callback toString];
	        return callNative(instanceId, tasksArray, callbackId);
	    };
	    _jsContext[@"callNative"] = callNativeBlock;
	}
	
	- (NSInteger)invokeNative:(NSString *)instance tasks:(NSArray *)tasks callback:(NSString *)callback
	{
	   ...
	    ///批量调用方法
	    for (NSDictionary *task in tasks) {
	        WXBridgeMethod *method = [[WXBridgeMethod alloc] initWihData:task];
	        method.instance = instance;
	        [[WXSDKManager moduleMgr] dispatchMethod:method];
	    }
	    
	    ///下面是处理回调的
	    NSMutableArray *sendQueue = [self.sendQueue valueForKey:instance];
	    
	    if (![callback isEqualToString:@"undefined"] && ![callback isEqualToString:@"-1"] && callback) {
	        WXBridgeMethod *method = [self _methodWithCallback:callback];
	        method.instance = instance;
	        [sendQueue addObject:method];
	    }
	    [self performSelector:@selector(_sendQueueLoop) withObject:nil];
	    return 1;
	}
图中蓝色代码是Native直接调用JS的函数，分别为： createInstance、destroyInstance、refreshInstance、registerModules、registerComponents。具体的功能在JS那边。这里不做了解。
# 归纳总结
weex的JS和Native通信原理可以使用下面图简单描述：
![JS_Native](http://of685p9vy.bkt.clouddn.com/weex_JS_Native.png)
在weexSDK初始化的时候，完成JS端和Native端的配置信息。JS端的配置信息的作用应该是确保什么样的Native方法可以在JS中调用；Native配置信息的作用是：当JS调用到Native的方法，需要查找调用的Module、方法、方法的SEL等信息，这样才能invoke到相应的方法。weex这样设计的原因大概是为了统一管理JS和Native的通信机制，假如，有许多Native的方法可以被JS调用，那么就需要配置很多的入口，这样随着代码的增加，无法有效维护，weex借助配置信息，巧妙的将入口控制为两个，一个是JS到Native的入口为callNative，一个是Native到JS的入口为callJS。


