```toml
title = "【React Native填坑之旅】从源码角度看JavaModule注册及重载陷阱"
slug = "02-react-native-overload-trap"
desc = "react native overload trap"
date = "2018-05-22 20:00:00"
update_date = "2018-05-22 20:00:00"
author = "wangganxin"
thumb = ""
tags = ["React Native","overlaod"]
```

>目录<br>
>
>一、前言<br>
>二、需要了解的几个类<br>
>三、JavaModule注册流程<br>
>
>参考资料<br>

### 一、前言

JavaModule是React Native提供给我们封装原生模块的能力，它可以让你复用一些原生的代码，又或者如果React Native还不支持某个你需要的原生特性，可以自己实现该特性的封装。官网文档介绍了如何封装JavaModule的过程，这里不再啰嗦，本文将会更深入一层Java层源码，看下它是如何注册并提供给JS端调用的，以及需要注意的一些问题。

<!--more-->

### 二、需要了解的几个类

- ReactContext

		继承于ContextWrapper，React Native应用的上下文，通过getContext()获得，它和Android中的Context是同一个概念
		
- ReactRootView
		
		继承于FragmeLayout，主要负责native端事件的监听(键盘事件、tounch事件)，并将结果传递给js端，以及负责页面元素的绘制

- ReactInstanceManager
		
		React Native应用的总的管理类，创建ReactContext、CatalystInstance实体，解析ReactPackage生成映射表，并且配合ReactRootView管理View的创建与生命周期等功能

- CatalystInstance

		通信大管家，负责Java层、C++层、JS层三端通信与协调，它的实现类是CatalystInstanceImpl

- NativeModuleRegistry

		JavaModule注册表，负责管理和查找JavaModule
	
- ReactPackage
		
		定义原生模块和原生组件必须继承的一个类，提供NativeModule和ViewManager列表，其实ViewManager也是NativeModule子类，但是它们的行为是不一样的。
		
### 三、JavaModule注册流程
首先我们要明白React Native本质上是一个View，它通过ReactInstanceManager进行一系列的管理，也是我们跟它打交道最多的，如果我们直接创建一个ReactNative项目，在它的Android目录下并不能直接找到ReactInstanceManager这个类，而是通过ReactNativeHost简化了很多初始化的事情，下面先来看下它的代码：

**ReactNativeHost.java**

```Java

public abstract class ReactNativeHost {

  private final Application mApplication;
  private @Nullable ReactInstanceManager mReactInstanceManager;

  ......

  protected ReactInstanceManager createReactInstanceManager() {
    ReactInstanceManagerBuilder builder = ReactInstanceManager.builder()
      .setApplication(mApplication)
      .setJSMainModulePath(getJSMainModuleName())
      .setUseDeveloperSupport(getUseDeveloperSupport())
      .setRedBoxHandler(getRedBoxHandler())
      .setJavaScriptExecutorFactory(getJavaScriptExecutorFactory())
      .setUIImplementationProvider(getUIImplementationProvider())
      .setInitialLifecycleState(LifecycleState.BEFORE_CREATE);

    for (ReactPackage reactPackage : getPackages()) {
      builder.addPackage(reactPackage);
    }

    String jsBundleFile = getJSBundleFile();
    if (jsBundleFile != null) {
      builder.setJSBundleFile(jsBundleFile);
    } else {
      builder.setBundleAssetName(Assertions.assertNotNull(getBundleAssetName()));
    }
    return builder.build();
  }

  /**
   * Returns a list of {@link ReactPackage} used by the app.
   * You'll most likely want to return at least the {@code MainReactPackage}.
   * If your app uses additional views or modules besides the default ones,
   * you'll want to include more packages here.
   */
  protected abstract List<ReactPackage> getPackages();

  ......
}

```

ReactNativeHost主要是负责ReactInstanceManager的实例创建，从它的`createReactInstanceManager()`方法可以看出构建者ReactInstanceManagerBuilder将ReactInstanceManager的构建与表示进行了分离，其中的`getPackages()`方法就是我们注册封装原生JavaModule的地方，并在builder中通过for循环将packagelist逐个add进去，进一步跟进`build()`方法，会发现它调用了ReactInstanceManager的构造方法，参数有点多，但总得来说是把packages放进了ReactInstanceManager的成员变量`mPackages`。

那仅仅是把它放进`mPackages`成员变量就结束了？没有其他操作？别急，我们还要从ReactNative的启动流程讲起，这里以ReactActivityDelegate这个代理类为例，在它的onCreate方法中有一段逻辑loadApp逻辑：

```Java

  protected void loadApp(String appKey) {
    if (mReactRootView != null) {
      throw new IllegalStateException("Cannot loadApp while app is already running.");
    }
    mReactRootView = createRootView();
    mReactRootView.startReactApplication(
      getReactNativeHost().getReactInstanceManager(),
      appKey,
      getLaunchOptions());
    getPlainActivity().setContentView(mReactRootView);
  }

```

- 创建一个ReactRootView
- 获取ReactInstanceManager实例
- ReactRootView#startReactApplication()
- 给当前Activity设置setContentView()

ReactRootView.java

```Java

public class ReactRootView extends SizeMonitoringFrameLayout 
		implements RootView, MeasureSpecProvider{
    
    ......
    
      public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle initialProperties) {
    		Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "startReactApplication");
    		try {
      			UiThreadUtil.assertOnUiThread();
      
      			Assertions.assertCondition(
        		mReactInstanceManager == null,
        		"This root view has already been attached to a catalyst instance manager");

      			mReactInstanceManager = reactInstanceManager;
      			mJSModuleName = moduleName;
      			mAppProperties = initialProperties;

      			if (!mReactInstanceManager.hasStartedCreatingInitialContext()) {
        			mReactInstanceManager.createReactContextInBackground();
      			}

      			attachToReactInstanceManager();

    			} finally {
      				Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
   			 }
  		}
     
    ......      
    }

```

在`startReactApplication()`方法中保存相应的参数外，还有重要的一点，如果是第一次启动，`hasStartedCreatingInitialContext()`肯定是false的，那么就会调用ReactInstanceManager#createReactContextInBackground()方法，进行一系列ReactContext的初始化，可以推测我们的mPackage也是在此刻注册进去的，其内部一系列调用链也表明最终会进入ReactInstanceManager#createReactContext()方法：

```Java

 private ReactApplicationContext createReactContext(
      JavaScriptExecutor jsExecutor,
      JSBundleLoader jsBundleLoader) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.createReactContext()");
    ReactMarker.logMarker(CREATE_REACT_CONTEXT_START);

    //创建reactContext，继承于ContextWrapper
    final ReactApplicationContext reactContext = new ReactApplicationContext(mApplicationContext);

    if (mUseDeveloperSupport) { //设置本地模块异常处理
      reactContext.setNativeModuleCallExceptionHandler(mDevSupportManager);
    }

    //解析mPackages，并创建NativeModuleRegistry进行管理
    NativeModuleRegistry nativeModuleRegistry = processPackages(reactContext, mPackages, false);

    NativeModuleCallExceptionHandler exceptionHandler = mNativeModuleCallExceptionHandler != null
      ? mNativeModuleCallExceptionHandler
      : mDevSupportManager;
    CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
      .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
      .setJSExecutor(jsExecutor)
      .setRegistry(nativeModuleRegistry)
      .setJSBundleLoader(jsBundleLoader)
      .setNativeModuleCallExceptionHandler(exceptionHandler);

    ReactMarker.logMarker(CREATE_CATALYST_INSTANCE_START);
    
    //创建CatalystInstance实例
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "createCatalystInstance");
    final CatalystInstance catalystInstance;
    try {
      catalystInstance = catalystInstanceBuilder.build();
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      ReactMarker.logMarker(CREATE_CATALYST_INSTANCE_END);
    }

    if (mBridgeIdleDebugListener != null) {
      catalystInstance.addBridgeIdleDebugListener(mBridgeIdleDebugListener);
    }
    if (Systrace.isTracing(TRACE_TAG_REACT_APPS | TRACE_TAG_REACT_JS_VM_CALLS)) {
      catalystInstance.setGlobalVariable("__RCTProfileIsProfiling", "true");
    }
    ReactMarker.logMarker(ReactMarkerConstants.PRE_RUN_JS_BUNDLE_START);
    catalystInstance.runJSBundle();
    // Transitions functions in the minitFunctions list to catalystInstance, to run after the bundle
    // TODO T20546472
    if (!mInitFunctions.isEmpty()) {
      for (CatalystInstanceImpl.PendingJSCall function : mInitFunctions) {
        ((CatalystInstanceImpl) catalystInstance).callFunction(function);
      }
    }
    reactContext.initializeWithInstance(catalystInstance);

    return reactContext;
  }
  
```

- 通过mApplicationContext来创建ReactApplicationContext
- 创建NativeModuleRegistry，将mPackages中每个ReactPackage返回的List<NativeModule>集合都注册到NativeModuleRegistry中，这里不包含List<ViewManager>集合
- 创建CatalystInstance实例，后续与C++进行交互

在创建CatalystInstance实例的同时也把NativeModuleRegistry引用给了它，这才真正和C++端建立起Bridge桥连接，让C++端能调用到JavaModule，CatalystInstanceImpl的构造函数内部initializeBridge()可以表明这一点。
我们回过头来看看NativeModuleRegistry的`getJavaModules()`方法

```Java

  /* package */ Collection<JavaModuleWrapper> getJavaModules(
      JSInstance jsInstance) {
    ArrayList<JavaModuleWrapper> javaModules = new ArrayList<>();
    for (Map.Entry<Class<? extends NativeModule>, ModuleHolder> entry : mModules.entrySet()) {
      Class<? extends NativeModule> type = entry.getKey();
      if (!CxxModuleWrapperBase.class.isAssignableFrom(type)) {
        javaModules.add(new JavaModuleWrapper(jsInstance, type, entry.getValue()));
      }
    }
    return javaModules;
  }
  
```

每一个JavaModule都用JavaModuleWrapper进行了包裹，它是C++中Java层BaseJavaModule特有的包装类，通过它可以更方便的阅读和被JNI调用。在JavaModuleWrapper中解析了JavaModule的名字、方法，并通过invoke方式进行调用。

```Java

@DoNotStrip
public class JavaModuleWrapper {

  ......

  @DoNotStrip
  public BaseJavaModule getModule() {
    return (BaseJavaModule) mModuleHolder.getModule();
  }

  @DoNotStrip
  public String getName() {
    return mModuleHolder.getName();
  }

  @DoNotStrip
  private void findMethods() {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "findMethods");
    Set<String> methodNames = new HashSet<>();

    Class<? extends NativeModule> classForMethods = mModuleClass;
    Class<? extends NativeModule> superClass =
        (Class<? extends NativeModule>) mModuleClass.getSuperclass();
    if (ReactModuleWithSpec.class.isAssignableFrom(superClass)) {
  
      classForMethods = superClass;
    }
    Method[] targetMethods = classForMethods.getDeclaredMethods();

    for (Method targetMethod : targetMethods) {
      ReactMethod annotation = targetMethod.getAnnotation(ReactMethod.class);
      if (annotation != null) {
        String methodName = targetMethod.getName();
        if (methodNames.contains(methodName)) {
          //js不支持方法重载否则会抛出这个异常
          throw new IllegalArgumentException(
            "Java Module " + getName() + " method name already registered: " + methodName);
        }
        MethodDescriptor md = new MethodDescriptor();
        JavaMethodWrapper method = new JavaMethodWrapper(this, targetMethod, annotation.isBlockingSynchronousMethod());
        md.name = methodName;
        md.type = method.getType();
        if (md.type == BaseJavaModule.METHOD_TYPE_SYNC) {
          md.signature = method.getSignature();
          md.method = targetMethod;
        }
        mMethods.add(method);
        mDescs.add(md);
      }
    }
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
  }

  //C++端调用，返回这个NativeModule的所有被@ReactMethod注解方法的描述
  @DoNotStrip
  public List<MethodDescriptor> getMethodDescriptors() {
    if (mDescs.isEmpty()) {
      findMethods();
    }
    return mDescs;
  }

  ......

}

```

这里有一点需要注意，`findMethod()`方法中通过反射的方式，获取到了所有被@ReactMethod注解标记的方法，
并用集合判断了是否有方法名重复的方法，否则抛出IllegalArgumentException，我们都知道在Java中可以很方便的根据方法名重复参数不一样来进行方法重载，但是js是没有方法重载的，否则就会出现问题。
我自己写了一个ToastModule测试类，里面进行了方法重载，发现ReactNative并没有把IllegalArgumentException抛回给Java层，直接就应用内部崩溃ANR了，这是一个令人很尴尬的地方，其实我们可以通过ReactInstanceManagerBuilder#setNativeModuleCallExceptionHandler()进行捕获处理.

大概就是这样吧，可能对于ReactNative内部执行流程的还不是特别的熟悉，很多跟着源码一步步点进去看的，如有不对的地方，还请各位看官多多包涵了！

### 参考资料
- [ReactNative源码篇](https://github.com/guoxiaoxing/react-native) 
- [ReactNative Android源码分析](http://tech.lede.com/2017/02/13/rd/android/android_rn/)
- [ReactNative 通信机制_java端源码分析](https://www.jianshu.com/p/d5d63840748c)