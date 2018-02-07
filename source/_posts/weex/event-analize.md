---
title: weex 事件原理分析
date: 2016-12-16 17:46:54
categories: weex
tags: weex
---
weex内置多种事件，包括视图的滑动事件：appear、disappear事件；手势事件：click、swipe、longpress、panstart、panmove、panend、touchstart、touchmove、touchend、touchcancel。本文主要分析这些事件的传递原理。
<!--more-->

# 事件传递过程分析 

本节通过一段示例代码，分析事件的传递过程。如果要实现一个点击事件，只需要在模板的某个标签下添加类似的代码onclick="clickTest"，具体如下：

	<template>
	    <div>
	      <text class="btn" onclick="clickTest">测试点击事件</text>
	    </div>
	</template>
	
	<script>
	  module.exports = {
	    data: 
	    {
	    },
	    methods: 
	    {
	      clickTest: function (e) 
	      {
	          console.log('$event.type + $event.detail');
	      },
	    }
	  }
	</script>

上面这段代码就是给text标签添加点击事件，事件的响应函数为clickTest。执行上面的JS代码，weex渲染text标签为Native的WXText组件，会调用WXText的初始化函数，定义如下：

	- (instancetype)initWithRef:(NSString *)ref
	                       type:(NSString*)type
	                     styles:(nullable NSDictionary *)styles
	                 attributes:(nullable NSDictionary *)attributes
	                     events:(nullable NSArray *)events
	               weexInstance:(WXSDKInstance *)weexInstance;
               
从定义可以看到，参数events会传入组件包含的事件列表，也就是说，weex将text标签中的on开头的属性都当做事件，渲染时，传递到Native中，本例中，events的值为click，接下来继续看initWithRef:type:styles:attributes:events:weexInstance中怎么处理events的。

	@property (nonatomic, readonly, strong) NSArray *events;
	
	@property(nonatomic, readonly, strong) UIView *view;
	
	
	- (instancetype)initWithRef:(NSString *)ref
	                       type:(NSString *)type
	                     styles:(NSDictionary *)styles
	                 attributes:(NSDictionary *)attributes
	                     events:(NSArray *)events
	               weexInstance:(WXSDKInstance *)weexInstance
	{
			....
	    _events = events ? [NSMutableArray arrayWithArray:events] : [NSMutableArray array];
	    ....     
	}
	
	- (UIView *)view
	{
			...
	    [self _initEvents:self.events];
	    ...
	}

上面代码说明，在初始化文本组件WXText（WXComponent的子类）时，将事件列表保存到events属性中。当加载组件的view到父视图的时候，会调用属性view的get方法，在get方法中初始化所有的事件。下面是初始化事件的代码。

	- (void)_initEvents:(NSArray *)events
	{
	    NSArray *eventsCopy = [events copy];
	    for (NSString *addEventName in eventsCopy) {
	        [self _addEventOnMainThread:addEventName];
	    }
	}
	
	- (void)_addEventOnMainThread:(NSString *)addEventName
	{
	    WX_ADD_EVENT(appear, addAppearEvent)
	    WX_ADD_EVENT(disappear, addDisappearEvent)
	    
	    WX_ADD_EVENT(click, addClickEvent)
	    WX_ADD_EVENT(swipe, addSwipeEvent)
	    WX_ADD_EVENT(longpress, addLongPressEvent)
	    
	    WX_ADD_EVENT(panstart, addPanStartEvent)
	    WX_ADD_EVENT(panmove, addPanMoveEvent)
	    WX_ADD_EVENT(panend, addPanEndEvent)
	    
	    WX_ADD_EVENT(touchstart, addTouchStartEvent)
	    WX_ADD_EVENT(touchmove, addTouchMoveEvent)
	    WX_ADD_EVENT(touchend, addTouchEndEvent)
	    WX_ADD_EVENT(touchcancel, addTouchCancelEvent)
	    
	    [self addEvent:addEventName];
	}
	
	
	#define WX_ADD_EVENT(eventName, addSelector) \
	if ([addEventName isEqualToString:@#eventName]) {\
	    [self addSelector];\
	}
	
	- (void)addClickEvent
	{
	    if (!_tapGesture) {
	        _tapGesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(onClick:)];
	        _tapGesture.delegate = self;
	        [self.view addGestureRecognizer:_tapGesture];
	    }
	}

上面的代码说明初始化事件就是给文本组件的view属性添加一个点击手势tapGesture， 手势的响应函数为onClick。

	- (void)onClick:(__unused UITapGestureRecognizer *)recognizer
	{
	    NSMutableDictionary *position = [[NSMutableDictionary alloc] initWithCapacity:4];
	    
	    if (!CGRectEqualToRect(self.calculatedFrame, CGRectZero)) {
	        CGRect frame = [self.view.superview convertRect:self.calculatedFrame toView:self.view.window];
	        position[@"x"] = @(frame.origin.x);
	        position[@"y"] = @(frame.origin.y);
	        position[@"width"] = @(frame.size.width);
	        position[@"height"] = @(frame.size.height);
	    }
	
	    [self fireEvent:@"click" params:@{@"position":position}];
	}
	
	- (void)fireEvent:(NSString *)eventName params:(NSDictionary *)params
	{
	    [self fireEvent:eventName params:params domChanges:nil];
	}
	
	- (void)fireEvent:(NSString *)eventName params:(NSDictionary *)params domChanges:(NSDictionary *)domChanges
	{
	    NSMutableDictionary *dict = [NSMutableDictionary dictionary];
	    NSTimeInterval timeSp = [[NSDate date] timeIntervalSince1970] * 1000;
	    [dict setObject:@(timeSp) forKey:@"timestamp"];
	    if (params) {
	        [dict addEntriesFromDictionary:params];
	    }
	    
	    [[WXSDKManager bridgeMgr] fireEvent:self.weexInstance.instanceId ref:self.ref type:eventName params:dict domChanges:domChanges];
	}

上面的代码说明：在手势响应函数中，最后调用weex的WXBridgeManager中的fireEvent函数。代码如下：

	- (void)fireEvent:(NSString *)instanceId ref:(NSString *)ref type:(NSString *)type params:(NSDictionary *)params domChanges:(NSDictionary *)domChanges
	{
	    if (!type || !ref) {
	        WXLogError(@"Event type and component ref should not be nil");
	        return;
	    }
	    
	    NSArray *args = @[ref, type, params?:@{}, domChanges?:@{}];
	    NSMutableDictionary *methodDict = [NSMutableDictionary dictionaryWithObjectsAndKeys:
	                                       @"fireEvent", @"method",
	                                       args, @"args", nil];
	    WXBridgeMethod *method = [[WXBridgeMethod alloc] initWithInstance:instanceId data:methodDict];
	    [self executeJsMethod:method];
	}
	
	- (void)executeJsMethod:(WXBridgeMethod *)method
	{
	    if (!method) return;
	    
	    __weak typeof(self) weakSelf = self;
	    WXPerformBlockOnBridgeThread(^(){
	        [weakSelf.bridgeCtx executeJsMethod:method];
	    });
	}
上面的代码说明： fireEvent实际上就是执行JS的fireEvent函数，JS的fireEvent函数会根据传递的参数找到组件、组件的事件、事件响应函数，最后执行事件。本例中就是第一段JS代码的clickTest函数。其中传递的参数本例如下：

	{
	    args =(
	    		///ref（组件的标记）
	        5, 
	        ///type（事件类型）
	        click, 
	        ///params   
	        {   ///事件触发的位置
	            position ={
	                height = "39.5";
	                width = 768;
	                x = 0;
	                y = 64;
	            };
	            timestamp = "1481882724298.018";
	        },
	        ///domChanges（是否更新界面）
	        {
	        }
	    );
	    ///JS的方法
	    method = fireEvent;
	}
	
到目前为止，weex的事件传递机制讲完了，下面做个简单的总结。

# 总结
weex的事件处理机制是： 在JS中定义标签的事件类型、事件的处理函数，在Native实现事件。原理是：将标签初始化为组件的时候，初始化标签的事件。其中click、swipe、longpress、panstart、panmove、panend、touchstart、touchmove、touchend、touchcancel初始化为手势事件，appear、disappear初始化为滑动视图的出现事件和消息事件。当事件触发时，通过WXBridgeManager调用JS的fireEvent函数，在JS中调用标签的事件处理函数。下面是做了一张大概的原理图。
![weex-event-detail1](http://of685p9vy.bkt.clouddn.com/weex-event-detail1.png) 
图中红色部分是JS逻辑，黑色部分是组件的逻辑，蓝色部分是WXBridgeManager逻辑。

 