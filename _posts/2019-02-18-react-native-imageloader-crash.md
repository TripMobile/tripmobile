---
title: 记一次ReactNative的多线程crash
date: 2019-02-18 21:53:03
tags:
  - ReactNative
  - MultiThread
  - Crash
  - iOS
categories: Crash
author: shanks
---

## 背景

在使用ReactNative v0.51版本时，发现线上有个崩溃一直没有解决，为了完成App治理crash的目标，所以花了点时间研究如何解决这个问题。
<!-- more -->

## Crash介绍

![alt text](/assets/rn-crash-reccomend.png)

![alt text](/assets/rn-crash-distribute.png)

可以看到如下信息

1. 崩溃位置

> [RCTHTTPRequestHandler sendRequest:withDelegate:]

2. 崩溃线程

> #27 Thread - 子线程

3. 崩溃原因

> NSGenericException

4. 崩溃描述

> Task created in a session that has been invalidated

5. 崩溃分布

> 没有很明显的特征

## 直接崩溃原因

根据崩溃原因NSGenericException，可以知道这是一个OC异常。我们先尝试找到直接导致崩溃的原因是什么，再找到真正触发的条件。

根据描述"Task created in a session that has been invalidated"，大概能看出一个失效的session对象尝试去创建task导致异常，而sendRequest:withDelegate:方法中创建task的地方只有一个

```objc
NSURLSession *_session;
- (NSURLSessionDataTask *)sendRequest:(NSURLRequest *)request
                         withDelegate:(id<RCTURLRequestDelegate>)delegate {
  //....
  NSURLSessionDataTask *task = [_session dataTaskWithRequest:request];
  //.....
}
```

而session为什么会invalidate的原因也很简单，因为[NSURLSession invalidateAndCancel].

![alt text](/assets/rn-crash-invalidate.png)

## 触发条件

知道直接崩溃原因，我们就要找到触发崩溃的原因。根据上面的证据，我们推测RCTHTTPRequestHandler肯定调用了invalidateAndCancel，事实也正是这样。

```objc
//RCTHTTPRequestHandler.mm
- (void)invalidate {
  [_session invalidateAndCancel];
  _session = nil;
}
```

如果_session被置为nil，则不会发生问题，所以肯定是[_session invalidateAndCancel]和_session = nil执行之间被打断了，结合之前的堆栈信息，卡顿发生在子线程，基本可以肯定这是一个多线程的问题，导致[RCTHTTPRequestHandler invalidate]的方法执行没有保证原子性。

那我们就要找出sendRequest:withDelegate:和invalidate各自的调用链。

### sendRequest

![alt text](/assets/rn-crash-imageloader.png)

根据崩溃堆栈，我们可以看到sendRequest是在RCTImageLoader中发起的。

```objc
_URLRequestQueue = dispatch_queue_create("com.facebook.react.ImageLoaderURLRequestQueue", DISPATCH_QUEUE_SERIAL);

- (void)dequeueTasks
{
    dispatch_async(_URLRequestQueue, ^{
        //...
        [task start];
        //...
    });
}
```

所以sendRequest是在名为"com.facebook.react.ImageLoaderURLRequestQueue"的串行队列中执行.

### invalidate

通过源码，可以找到invalidate的调用路径

```objc
/*
->RCTBridge.dealloc
->RCTBridge.invalidate
->RCTCxxBridge.invalidate
->[moduleData.instance invalidate]
*/

//RCTCxxBridge.mm
- (void)invalidate {
  //...省略
  if ([moduleData.instance respondsToSelector:@selector(invalidate)]) {
    dispatch_group_enter(moduleInvalidation);
    [self dispatchBlock:^{
      [(id<RCTInvalidating>)moduleData.instance invalidate];
      dispatch_group_leave(moduleInvalidation);
    } queue:moduleData.methodQueue];
  }
  [moduleData invalidate];
    }
  //省略...
}
```

参考[RN原理](https://shanks.pro/2019/01/10/react-native-communication/)， 可以知道moduleData是在RN初始化的时候注册的模块信息，RCTHTTPRequestHandler也会生成其中一个moduleData。 那我们看下moduleData.methodQueue是什么，因为这就是invalidate执行的队列。

```objc
- (void)setUpMethodQueue {
  //...
  _queueName = [NSString stringWithFormat:@"com.facebook.react.%@Queue", self.name];
    _methodQueue = dispatch_queue_create(_queueName.UTF8String, DISPATCH_QUEUE_SERIAL);
  //...
 [(id)_instance setValue:_methodQueue forKey:@"methodQueue"];
 //...
}
```
可以看到，每个module都会又一个对应的串行methodQueue，并且名称的规则是"com.facebook.react.%@Queue", 所以RCTHTTPRequestHandler对应的队列就是"com.facebook.react.HTTPRequestHandlerQueue"

也即，invalidate是在串行队列""com.facebook.react.HTTPRequestHandlerQueue""中执行。

### 还不够！

就算知道了sendRequest和invalidate方法在不同队列的线程中执行，还不能百分百确定一定会发生多线程问题，除非RCTCxxBridge.invalidate中触发的moduleData实例和RCTImageLoader触发的sendRequest中RCTHTTPRequestHandler实例是同一个对象。

```objc
//RCTNetworkTask中获取RCTHTTPRequestHandler的方法
- (id<RCTURLRequestHandler>)handlerForRequest:(NSURLRequest *)request {
  _handlers = [[self.bridge modulesConformingToProtocol:@protocol(RCTURLRequestHandler)] sortedArrayUsingComparator:^NSComparisonResult(id<RCTURLRequestHandler> a, id<RCTURLRequestHandler> b) {
        float priorityA = [a respondsToSelector:@selector(handlerPriority)] ? [a handlerPriority] : 0;
        float priorityB = [b respondsToSelector:@selector(handlerPriority)] ? [b handlerPriority] : 0;
        if (priorityA > priorityB) {
          return NSOrderedAscending;
        } else if (priorityA < priorityB) {
          return NSOrderedDescending;
        } else {
          return NSOrderedSame;
        }
      }];
}

//RCTBridge.m
- (NSArray *)modulesConformingToProtocol:(Protocol *)protocol
{
  NSMutableArray *modules = [NSMutableArray new];
  for (Class moduleClass in [self.moduleClasses copy]) {
    if ([moduleClass conformsToProtocol:protocol]) {
      id module = [self moduleForClass:moduleClass];
      if (module) {
        [modules addObject:module];
      }
    }
  }
  return [modules copy];
}
```

可以看到，RCTNetworkTask执行时用到的handler，是从RCTBridge之前注册好的module中去找到符合<RCTURLRequestHandler>协议的对象。最终结果找到也是RCTHTTPRequestHandler对象。

```objc
@interface RCTHTTPRequestHandler : NSObject <RCTURLRequestHandler, RCTInvalidating>
@end
```

至此可以发现，RCTHTTPRequestHandler其实生成的实例对象，在一个RCTBridge周期内只有一个。同一个RCTHTTPRequestHandler对象的invalidate和sendRequest的执行在不同队列的不同子线程。虽然两个队列都是串行，但是两个子线程之间互相之间没有约束，一个执行时可能会被另一个打断，从而导致执行了[_session invalidateAndCancel]之后执行[_session dataTaskWithRequest:request]导致crash。

## 怎么复现

我复现的方式是，在RCTHTTPRequestHandler的invalidate方法中插入sleep，加大RCTHTTPRequestHandler中invalidate方法被打断的概率，同时在外面模拟RCTBridge的invalidate。

```objc
//RCTHTTPRequestHandler
- (void)invalidate {
  [_session invalidateAndCancel];
  sleep(3);
  _session = nil;
}
```

这样很容易能复现这个问题。

## 怎么修复

既然两个线程在不同的队列执行，那最简单的修复方式就是把他们的执行放到同一个队列中去，这样两块代码再执行的时候顺序不会被中途打断。
之前我们也看到，每个moduleData都有自己的methodQueue，那比较好的方式还是在RCTHTTPRequestHandler内部用他自己的methodQueue。

```objc
@synthesize methodQueue = _methodQueue;

- (void)invalidate{
  dispatch_async(self->_methodQueue, ^{
    [self->_session invalidateAndCancel];
    self->_session = nil;
  });
}

- (NSURLSessionDataTask *)sendRequest:(NSURLRequest *)request
                         withDelegate:(id<RCTURLRequestDelegate>)delegate {
  //....
  dispatch_sync(self->_methodQueue, ^{
    NSURLSessionDataTask *task = [self->session dataTaskWithRequest:request];
  }
  //.....
}
```

## 给ReactNative提PR

既然这里存在问题，并且改动还算合理，我就尝试把这个修改提交给ReactNative，看人家会不会采纳。最终PR还是被合并了，Bingo!
https://github.com/facebook/react-native/pull/22746


## 总结

通过这个crash我们可以看到，多线程的问题比较隐蔽，所以我们平时在写代码和做code review时，要特别注意线程安全，对共享变量的使用要比较小心。


