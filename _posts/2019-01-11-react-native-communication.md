---
title: ReactNative源码分析-通信机制(基于iOS)
date: 2019-01-10 13:40:45
tags:
  - ReactNative
  - 源码
  - uml
  - iOS
categories: ReactNative
author: shanks
---

## 准备-JavaScriptCore
在开篇，我们先简单准备下JavaScriptCore的知识。这是整个Native和JS沟通的最底层的桥梁。

<!--more-->

***Classes***
- JSContext <br>
JS的执行上下文<br>
![](/assets/jscontext.jpg)

- JSManagedValue<br>
主要用于防止Native导出对象时持有JSValue导致循环引用

- JSValue<br>
JavaScript值的引用，转换JavaScript和Native之间的基本数据<br>
![](/assets/jsvalue.jpg)

- JSVirtualMachine<br>
提供JS执行环境<br>
![](/assets/jsvm.jpg)

- JSExport<br>
导出Native到JavaScript

***相对应的底层C实现***
- JSBase
- JSContextRef
- JSObjectRef
- JSStringRef
- JSStringRefCF
- JSValueRef

## ReactNative通信原理

从ReactNative的demo开始，入口我们只看到两个东西RCTRootView和RCTBridge。RCTRootView负责展示，RCTBridge则和ReactNative的通信有关。

### 入口

```
- (BOOL)application:(__unused UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  _bridge = [[RCTBridge alloc] initWithDelegate:self
                                  launchOptions:launchOptions];

  //...

  RCTRootView *rootView = [[RCTRootView alloc] initWithBridge:_bridge
                                                   moduleName:@"RNTesterApp"
                                            initialProperties:initProps];

  self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
  UIViewController *rootViewController = [UIViewController new];
  rootViewController.view = rootView;
  self.window.rootViewController = rootViewController;
  [self.window makeKeyAndVisible];
  return YES;
}
```
### RCTCxxBridge初始化

跟着调用链会看到，bridge的初始化会走到RCTCxxBridge中的start
```
//RCTCxxBrige.mm
//只贴了关键代码
- (void)start {
  dispatch_group_t prepareBridge = dispatch_group_create();

  // 初始化native modules
  (void)[self _initializeModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO];

  // 初始化底层Instance
  dispatch_group_enter(prepareBridge);
  [self ensureOnJavaScriptThread:^{
    [weakSelf _initializeBridge:executorFactory];
    dispatch_group_leave(prepareBridge);
  }];

  // 加载js代码
  dispatch_group_enter(prepareBridge);
  __block NSData *sourceCode;
  [self loadSource:^(NSError *error, RCTSource *source) {
    if (error) {
      [weakSelf handleError:error];
    }

    sourceCode = source.data;
    dispatch_group_leave(prepareBridge);
  } onProgress:^(RCTLoadingProgress *progressData) {
  }];

  // 执行js代码
  dispatch_group_notify(prepareBridge, dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0), ^{
    RCTCxxBridge *strongSelf = weakSelf;
    if (sourceCode && strongSelf.loading) {
      [strongSelf executeSourceCode:sourceCode sync:NO];
    }
  });
}
```
start方法里做了这么几件事：
- 初始化native modules
- 初始化Instance->底层负责通信
- 加载本地js

要想了解native modules的初始化，我们要先看下之前的准备工作。

### native modules导出模块

通过RCT_EXPORT_MODULE将本地的模块导出供JS使用

```
//RCTBridgeModule.h
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }

void RCTRegisterModule(Class moduleClass) {
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
    RCTModuleClassesSyncQueue = dispatch_queue_create("com.facebook.react.ModuleClassesSyncQueue", DISPATCH_QUEUE_CONCURRENT);
  });
  dispatch_barrier_async(RCTModuleClassesSyncQueue, ^{
    [RCTModuleClasses addObject:moduleClass];
  });
}
```

最终导出的模块会被保存到RCTModuleClasses，使用时通过RCTGetModuleClasses()获取。RCTGetModuleClasses()就是在上面初始化native modules时用到的。
```
//RCTBridge.m
NSArray<Class> *RCTGetModuleClasses(void) {
  __block NSArray<Class> *result;
  dispatch_sync(RCTModuleClassesSyncQueue, ^{
    result = [RCTModuleClasses copy];
  });
  return result;
}
```

### native modules导出方法

```
//RCTBridgeModule.h
#define RCT_EXPORT_METHOD(method) \
  RCT_REMAP_METHOD(, method)

#define RCT_REMAP_METHOD(js_name, method) \
  _RCT_EXTERN_REMAP_METHOD(js_name, method, NO) \
  - (void)method RCT_DYNAMIC;

#define _RCT_EXTERN_REMAP_METHOD(js_name, method, is_blocking_synchronous_method) \
  + (const RCTMethodInfo *)RCT_CONCAT(__rct_export__, RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) { \
    static RCTMethodInfo config = {#js_name, #method, is_blocking_synchronous_method}; \
    return &config; \
  }

```

将方法导出，最终生成以下方法提供给外部调用。通过遍历这个类中所有以
"\_\_rct_export\_\_"开头的方法就可以获取属于这个类的所有导出方法。

```
//RCTBridgeModule.h
typedef struct RCTMethodInfo {
  const char *const jsName;
  const char *const objcName;
  const BOOL isSync;
} RCTMethodInfo;

+(const RCTMethodInfo *)__rct_export__+js_name+__LINE__+__COUNTER__ {
    static RCTMethodInfo config = {
        js_name,
        method,
        is_blocking_synchronous_method
    }
    return &config;
}
```

### 生成RCTModuleData

初始化native modules的工作，其实就是根据之前导出的类和方法，生成对应的RCTModuleData对象。

```
//RCTCxxBridge.mm
- (NSArray<RCTModuleData *> *)_registerModulesForClasses:(NSArray<Class> *)moduleClasses
                                        lazilyDiscovered:(BOOL)lazilyDiscovered {
  NSArray *moduleClassesCopy = [moduleClasses copy];
  NSMutableArray<RCTModuleData *> *moduleDataByID = [NSMutableArray arrayWithCapacity:moduleClassesCopy.count];
  for (Class moduleClass in moduleClassesCopy) {
    NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);
    RCTModuleData *moduleData = _moduleDataByName[moduleName];
    if (moduleData) {
       continue;
    }
    moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass bridge:self];
    _moduleDataByName[moduleName] = moduleData;
    [_moduleClassesByID addObject:moduleClass];
    [moduleDataByID addObject:moduleData];
  }
  [_moduleDataByID addObjectsFromArray:moduleDataByID];
  return moduleDataByID;
}

```
至此，我们完成了本地模块和方法的导出，并且生成了一组RCTModuleData对象来表示他们。

### 初始化Instance

我们继续看Instance的初始化。

```
// 初始化底层Instance
  dispatch_group_enter(prepareBridge);
  [self ensureOnJavaScriptThread:^{
    [weakSelf _initializeBridge:executorFactory];
    dispatch_group_leave(prepareBridge);
  }];

//RCTCxxBridge.mm
  - (void)_initializeBridge:(std::shared_ptr<JSExecutorFactory>)executorFactory {
  if (_reactInstance) {
    [self _initializeBridgeLocked:executorFactory];
  }
}
//RCTCxxBridge.mm
- (void)_initializeBridgeLocked:(std::shared_ptr<JSExecutorFactory>)executorFactory {
  _reactInstance->initializeBridge(
                                   std::make_unique<RCTInstanceCallback>(self),
                                   executorFactory,
                                   _jsMessageThread,
                                   [self _buildModuleRegistryUnlocked]);
}

//Instance.cpp
void Instance::initializeBridge(
    std::unique_ptr<InstanceCallback> callback,
    std::shared_ptr<JSExecutorFactory> jsef,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<ModuleRegistry> moduleRegistry) {
  callback_ = std::move(callback);
  moduleRegistry_ = std::move(moduleRegistry);

  jsQueue->runOnQueueSync([this, &jsef, jsQueue]() mutable {
    nativeToJsBridge_ = folly::make_unique<NativeToJsBridge>(
        jsef.get(), moduleRegistry_, jsQueue, callback_);

    std::lock_guard<std::mutex> lock(m_syncMutex);
    m_syncReady = true;
    m_syncCV.notify_all();
  });
}
```

初始化Instance需要一下几个元素：
- InstanceCallback类型的回调，用于底层执行结束后往上层回调。
```
struct InstanceCallback {
  virtual ~InstanceCallback() {}
  virtual void onBatchComplete() {}
  virtual void incrementPendingJSCalls() {}
  virtual void decrementPendingJSCalls() {}
};
```

- JSExecutorFactory类型的对象，用于生成JSExecutor用于真正执行JS。生产返回使用的是JSCExecutorFactory，返回JSIExecutor用于执行JS，调试使用的是RCTObjcExecutorFactory,返回RCTObjcExecutor通过websocket链接chrome执行JS。

```
class JSExecutorFactory {
public:
  virtual std::unique_ptr<JSExecutor> createJSExecutor(
    std::shared_ptr<ExecutorDelegate> delegate,
    std::shared_ptr<MessageQueueThread> jsQueue) = 0;
  virtual ~JSExecutorFactory() {}
};

//生产
class JSCExecutorFactory : public JSExecutorFactory {
public:
  std::unique_ptr<JSExecutor> createJSExecutor(
    std::shared_ptr<ExecutorDelegate> delegate,
    std::shared_ptr<MessageQueueThread> jsQueue) override {
    return folly::make_unique<JSIExecutor>(
      facebook::jsc::makeJSCRuntime(),
      delegate,
      [](const std::string &message, unsigned int logLevel) {
        _RCTLogJavaScriptInternal(
          static_cast<RCTLogLevel>(logLevel),
          [NSString stringWithUTF8String:message.c_str()]);
      },
      JSIExecutor::defaultTimeoutInvoker,
      nullptr);
  }
};
}
//调试
std::unique_ptr<JSExecutor> RCTObjcExecutorFactory::createJSExecutor(
    std::shared_ptr<ExecutorDelegate> delegate,
    std::shared_ptr<MessageQueueThread> jsQueue) {
  return std::unique_ptr<JSExecutor>(
    new RCTObjcExecutor(m_jse, m_errorBlock, jsQueue, delegate));
}
```

- MessageQueueThread类型对象用于提供队列执行。这里是由RCTMessageThread来实现，内部用的是CFRunLoop来实现。

```
//MessageQueueThread.h
class MessageQueueThread {
 public:
  virtual ~MessageQueueThread() {}
  virtual void runOnQueue(std::function<void()>&&) = 0;
  // runOnQueueSync and quitSynchronous are dangerous.  They should only be
  // used for initialization and cleanup.
  virtual void runOnQueueSync(std::function<void()>&&) = 0;
  // Once quitSynchronous() returns, no further work should run on the queue.
  virtual void quitSynchronous() = 0;
};
}}

//RCTCxxBridge.mm
_jsMessageThread = std::make_shared<RCTMessageThread>([NSRunLoop currentRunLoop], ^(NSError *error) {
    if (error) {
      [weakSelf handleError:error];
    }
  });

```

- ModuleRegistry，这个包含native module信息的对象，它的来源就是我们上面看到的RCTModuleData。可以看到最终透传参数生成了RCTNativeModule

```
//RCTCxxBridge.mm
- (std::shared_ptr<ModuleRegistry>)_buildModuleRegistryUnlocked {
  auto registry = std::make_shared<ModuleRegistry>(
         createNativeModules(_moduleDataByID, self, _reactInstance),
         moduleNotFoundCallback);
  return registry;
}

//RCTCxxUtils.mm
std::vector<std::unique_ptr<NativeModule>> createNativeModules(NSArray<RCTModuleData *> *modules, RCTBridge *bridge, const std::shared_ptr<Instance> &instance)
{
  std::vector<std::unique_ptr<NativeModule>> nativeModules;
  for (RCTModuleData *moduleData in modules) {
    if ([moduleData.moduleClass isSubclassOfClass:[RCTCxxModule class]]) {
      //跨平台，Android用
      nativeModules.emplace_back(std::make_unique<CxxNativeModule>(
        instance,
        [moduleData.name UTF8String],
        [moduleData] { return [(RCTCxxModule *)(moduleData.instance) createModule]; },
        std::make_shared<DispatchMessageQueueThread>(moduleData)));
    } else {
      nativeModules.emplace_back(std::make_unique<RCTNativeModule>(bridge, moduleData));
    }
  }
  return nativeModules;
}
```
有必要提一下，这上面的moduleData.instance，其实就是生成这个模块对应实例
```
- (instancetype)initWithModuleClass:(Class)moduleClass
                             bridge:(RCTBridge *)bridge
{
  return [self initWithModuleClass:moduleClass
                    moduleProvider:^id<RCTBridgeModule>{ return [moduleClass new]; }
                            bridge:bridge];
}
```
同时也会准备好它所对应的bridge和method queue
 ```
 //RCTModuleData.mm
 - (void)setBridgeForInstance
{
  if ([_instance respondsToSelector:@selector(bridge)] && _instance.bridge != _bridge) {
    @try {
      [(id)_instance setValue:_bridge forKey:@"bridge"];
    }
    @catch (NSException *exception) {
      //...
    }
  }
}

- (void)setUpMethodQueue
{
  if (_instance && !_methodQueue && _bridge.valid) {
    BOOL implementsMethodQueue = [_instance respondsToSelector:@selector(methodQueue)];
    if (implementsMethodQueue && _bridge.valid) {
      _methodQueue = _instance.methodQueue;
    }
    if (!_methodQueue && _bridge.valid) {
      _queueName = [NSString stringWithFormat:@"com.facebook.react.%@Queue", self.name];
      _methodQueue = dispatch_queue_create(_queueName.UTF8String, DISPATCH_QUEUE_SERIAL);

      // assign it to the module
      if (implementsMethodQueue) {
        @try {
          [(id)_instance setValue:_methodQueue forKey:@"methodQueue"];
        }
        @catch (NSException *exception) {
          //...
        }
      }
    }
    RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"");
  }
}

 ```

同时我们看到Instance::initializeBridge中生成了NativeToJsBridge，到这里Instance的初始化就结束了，下面进入NativeToJsBridge。

### NativeToJsBridge

NativeToJsBridge作用主要是桥接Native和JS，它包含几个关键属性
- &lt;JsToNativeBridge&gt; m_delegate
  JsToNativeBridge类型的引用，主要用于JS call Native

- &lt;JSExecutor&gt; m_executor
  JSExecutor类型引用，主要用于执行Native call JS，这里实际使用是的是JSIExecutor(生产)/RCTObjcExecutor(调试)
```
std::shared_ptr<JSExecutorFactory> executorFactory;
  if (!self.executorClass) {
    if ([self.delegate conformsToProtocol:@protocol(RCTCxxBridgeDelegate)]) {
      id<RCTCxxBridgeDelegate> cxxDelegate = (id<RCTCxxBridgeDelegate>) self.delegate;
      executorFactory = [cxxDelegate jsExecutorFactoryForBridge:self];
    }
    if (!executorFactory) {
      //生产使用JSCExecutorFactory会生成JSIExecuror
      executorFactory = std::make_shared<JSCExecutorFactory>();
    }
  } else {
    //调试用RCTObjcExecutorFactory生成RCTObjcExecutor
    id<RCTJavaScriptExecutor> objcExecutor = [self moduleForClass:self.executorClass];
    executorFactory.reset(new RCTObjcExecutorFactory(objcExecutor, ^(NSError *error) {
      if (error) {
        [weakSelf handleError:error];
      }
    }));
  }
```

- &lt;MessageQueueThread&gt; m_executorMessageQueueThread
  MessageQueueThread类型引用，由上层传递，用于队列管理

### JSIExecutor

JSIExecutor主要用来Native call JS，包含几个主要属性：
- &lt;jsi::Runtime&gt; runtime_ 
  Runtime类型指针，代表JS的运行时。这是一个抽象类，其实际上是由JSCRuntime来实现的，JSCRuntime中的功能其实就是通过JavaScriptCode来完成（使用的C函数接口）。JSCRuntime上线了&lt;jsi::Runtime&gt;接口，提供了创建JS上下文的功能，同时可以执行JS。

```
void JSCRuntime::evaluateJavaScript(
    std::unique_ptr<const jsi::Buffer> buffer,
    const std::string& sourceURL) {
  std::string tmp(
      reinterpret_cast<const char*>(buffer->data()), buffer->size());
  JSStringRef sourceRef = JSStringCreateWithUTF8CString(tmp.c_str());
  JSStringRef sourceURLRef = nullptr;
  if (!sourceURL.empty()) {
    sourceURLRef = JSStringCreateWithUTF8CString(sourceURL.c_str());
  }
  JSValueRef exc = nullptr;
  JSValueRef res =
      JSEvaluateScript(ctx_, sourceRef, nullptr, sourceURLRef, 0, &exc);
  JSStringRelease(sourceRef);
  if (sourceURLRef) {
    JSStringRelease(sourceURLRef);
  }
  checkException(res, exc);
}
```

- &lt;ExecutorDelegate&gt; delegate_
  ExecutorDelegate类型的指针，这里的ExecutorDelegate是抽象类，实际是由JsToNative来实现的。也即JSIExecutor引用了JsToNative。

- &lt;JSINativeModules&gt; nativeModules_
  JSINativeModules由上层传入的ModuleRegistry构造而成，同时会将ModuleRegistry中包含的本地模块配置信息通过"__fbGenNativeModule"保存到JS端。

```
//JSINativeModules.cpp
folly::Optional<Object> JSINativeModules::createModule(
    Runtime& rt,
    const std::string& name) {
  if (!m_genNativeModuleJS) {
    m_genNativeModuleJS =
        rt.global().getPropertyAsFunction(rt, "__fbGenNativeModule");
  }
  auto result = m_moduleRegistry->getConfig(name);
  
  Value moduleInfo = m_genNativeModuleJS->call(
      rt,
      valueFromDynamic(rt, result->config),
      static_cast<double>(result->index));
  CHECK(!moduleInfo.isNull()) << "Module returned from genNativeModule is null";
  return module;
}

//NativeModules.js
global.__fbGenNativeModule = genModule;

```
genModule会根据ModuleRegistry生成的module和method信息生成JS端的方法，结构类似：
```
{
  name: moduleName,
  module: {
    methodName: func
  }
}
```

JSIExecutor执行js方法的实现值得说下。
```
void JSIExecutor::callFunction(
    const std::string& moduleId,
    const std::string& methodId,
    const folly::dynamic& arguments) {
  if (!callFunctionReturnFlushedQueue_) {
    bindBridge();
  }

  auto errorProducer = [=] {
    std::stringstream ss;
    ss << "moduleID: " << moduleId << " methodID: " << methodId
       << " arguments: " << folly::toJson(arguments);
    return ss.str();
  };

  Value ret = Value::undefined();
  try {
    scopedTimeoutInvoker_(
        [&] {
          ret = callFunctionReturnFlushedQueue_->call(
              *runtime_,
              moduleId,
              methodId,
              valueFromDynamic(*runtime_, arguments));
        },
        std::move(errorProducer));
  } catch (...) {
    std::throw_with_nested(
        std::runtime_error("Error calling " + moduleId + "." + methodId));
  }

  callNativeModules(ret, true);
}
```
其中callFunctionReturnFlushedQueue_来自和JS端的属性绑定
```
//JSIExecutor.cpp
Object batchedBridge = batchedBridgeValue.asObject(*runtime_);
callFunctionReturnFlushedQueue_ = batchedBridge.getPropertyAsFunction(
        *runtime_, "callFunctionReturnFlushedQueue");

//MessageQueue.js
callFunctionReturnFlushedQueue(module: string, method: string, args: any[]) {
    this.__guard(() => {
      this.__callFunction(module, method, args);
    });

    return this.flushedQueue();
  }
```
可以看到callFunctionReturnFlushedQueue_会调用JS端的callFunctionReturnFlushedQueue方法，最终调用在JS端注册好的JS模块和方法。
而getPropertyAsFunction则是通过runtime来实现。（runtime是JS和Native沟通的桥梁）

### JsToNativeBridge

JsToNativeBridge的实现就简单很多，直接通过ModuleRegistry注册好的native信息，调用对应模块的对应方法。

```
void callNativeModules(
      JSExecutor& executor, folly::dynamic&& calls, bool isEndOfBatch) override {
        
    m_batchHadNativeModuleCalls = m_batchHadNativeModuleCalls || !calls.empty();
    for (auto& call : parseMethodCalls(std::move(calls))) {
      m_registry->callNativeMethod(call.moduleId, call.methodId, std::move(call.arguments), call.callId);
    }
    if (isEndOfBatch) {
      if (m_batchHadNativeModuleCalls) {
        m_callback->onBatchComplete();
        m_batchHadNativeModuleCalls = false;
      }
      m_callback->decrementPendingJSCalls();
    }
  }
```
其中m_registry就是上层传入的ModuleRegistry对象

那JS Call Native的整套流程是怎样的呢？

 - JS调用MessageQueue.enqueueNativeCall
```
enqueueNativeCall(
    moduleID: number,
    methodID: number,
    params: any[],
    onFail: ?Function,
    onSucc: ?Function,
  ) {
    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);
    this._queue[PARAMS].push(params);
    const now = Date.now();
    //MIN_TIME_BETWEEN_FLUSHES_MS = 5
    if (
      global.nativeFlushQueueImmediate &&
      now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS
    ) {
      const queue = this._queue;
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
      global.nativeFlushQueueImmediate(queue);
    }
  }
```
可以看到5ms刷新一次

- nativeFlushQueueImmediate对应本地的方法
```
//JSIExecutor.cpp
runtime_->global().setProperty(
      *runtime_,
      "nativeFlushQueueImmediate",
      Function::createFromHostFunction(
          *runtime_,
          PropNameID::forAscii(*runtime_, "nativeFlushQueueImmediate"),
          1,
          [this](
              jsi::Runtime&,
              const jsi::Value&,
              const jsi::Value* args,
              size_t count) {
            if (count != 1) {
              throw std::invalid_argument(
                  "nativeFlushQueueImmediate arg count must be 1");
            }
            callNativeModules(args[0], false);
            return Value::undefined();
          }));
```
可以看到runtime_获取到全局上下文(即JS端的global)，将nativeFlushQueueImmediate属性关联到本地callNativeModules方法。
这里值得一提的是，runtime_->global().setProperty用在很多将JS属性和native方法对象等绑定。

- callNativeModules
```
//JSIExecutor.cpp
void JSIExecutor::callNativeModules(const Value& queue, bool isEndOfBatch) {
  delegate_->callNativeModules(
      *this, dynamicFromValue(*runtime_, queue), isEndOfBatch);
}
```
delegate_就是我们上面提到的&lt;ExecutorDelegate&gt;类型指针，实际就是JsToNativeBridge，最终也就走到了我们上面说的callNativeModules。

到这里，我们已经具备了native call js和js call native的能力。

## 总结

<a href="/assets/uml.jpg"><img src="/assets/uml.jpg"></a>
<!-- ![UML图](/assets/uml.jpg) -->
