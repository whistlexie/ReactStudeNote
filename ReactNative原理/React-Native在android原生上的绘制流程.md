# React-Native在android原生上的绘制流程 #


在 android 原生View的绘制流程，可以参考以下郭霖大神的博客：  [Android视图绘制流程完全解析，带你一步步深入了解View(二)](http://blog.csdn.net/guolin_blog/article/details/16330267)

在之前的认知中，在android原生显示的都是原生的 View，即使是显示html也是通 WebView 去解析的渲染 html 从而显示出网页内容。那么在React Native的应用场景下，是如何将 JS代码生成视图的呢？

## 测试代码准备 ##

在React-Native/React 的官网上，没找到相关的说明，那么我们只好自己找源码了，可以通过编写简单的测试代码来验证，下面是我的入口 JS代码，在EasyView.js中，这个js文件的完整路径是:项目主目录/rnjs/view_draw。


    import React,{ Component} from 'react'
    import {View, Text, AppRegistry} from 'react-native'
    
    class EasyView extends Component {
    
    render() {
    return(<View>
    <Text>How to draw a view!</Text>
    </View>)
    }
    }
    
    AppRegistry.registerComponent("TestRN",() =>EasyView); // TextRN是注册的入口Component名称，默认和项目名称一样


然后在 android.index.js 中只有一句代码(这样的话，方便自己写不同场景的测试代码）：


    require('./rnjs/view_draw/EasyView')

## 源码分析 ##

这样，我们去到 android原生，查看一下代码。

首先去到主页（入口和显示）的 Activity 类中，我们直接从 Activity 的生命周期看起，默认生成的 Activity 中继承了 ReactActivity，实现了 DefaultHardwareBackBtnHandler和PermissionAwareActivity接口，这两个接口主要是处理一些//todo
我们直接从 ReactActivity 看起吧，在 ReactActivity 的 onCreate()方法中，这里完成了视图的绘制：


      @Override
      protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // 如果是开发者支持显示悬浮窗（默认是返回true，支持按下菜单栏或者摇一摇就显示开发者菜单）
    //并且是android6.0上,android 6.0需要手动开启一些权限，由于android 6.0 使用了新的权限管理机制，动态权限管理机制，和ios很类型。
    if (getUseDeveloperSupport() && Build.VERSION.SDK_INT >= 23) {
      // Get permission to show redbox in dev builds.
      if (!Settings.canDrawOverlays(this)) {
     //Settings.ACTION_MANAGE_OVERLAY_PERMISSION是申请SYSTEM_ALERT_WINDOW权限对应的intent action，也就是悬浮窗权限
    Intent serviceIntent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
    startActivity(serviceIntent);
    FLog.w(ReactConstants.TAG, REDBOX_PERMISSION_MESSAGE);
    Toast.makeText(this, REDBOX_PERMISSION_MESSAGE, Toast.LENGTH_LONG).show();
      }
    }

     //这个 RootView 就是我们今天关注的重点了    
    mReactRootView = createRootView();
    mReactRootView.startReactApplication(
      getReactNativeHost().getReactInstanceManager(),
      getMainComponentName(),
      getLaunchOptions());
    // 所以我们清楚的知道了，显示的就是这个mReactRootView了
    setContentView(mReactRootView);
    mDoubleTapReloadRecognizer = new DoubleTapReloadRecognizer();
      }

我们简单瞄一下 createRootView()方法里面做了什么。

      /**
       * A subclass may override this method if it needs to use a custom {@link ReactRootView}.
       */
      protected ReactRootView createRootView() {
    return new ReactRootView(this);
      }

这里的话，只是 new 了一个实例出来，并没有涉及交互和参数传递。我们先跳过这个 ReactRootView 是怎么创建的，后面的篇幅里面会着重介绍，这里你只需要知道 ReactRootView 等同于我们的自定义 View 组件。我们着重看一下 ReactRootView 的 startReactApplication()方法。我们去到 startReactApplication()方法里面：

      /**
       * Schedule rendering of the react component rendered by the JS application from the given JS
       * module (@{param moduleName}) using provided {@param reactInstanceManager} to attach to the
       * JS context of that manager. Extra parameter {@param launchOptions} can be used to pass initial
       * properties for the react component.
       */
      public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle launchOptions) {
    UiThreadUtil.assertOnUiThread();
    
    // TODO(6788889): Use POJO instead of bundle here, apparently we can't just use WritableMap
    // here as it may be deallocated in native after passing via JNI bridge, but we want to reuse
    // it in the case of re-creating the catalyst instance
    Assertions.assertCondition(
    mReactInstanceManager == null,
    "This root view has already been attached to a catalyst instance manager");
    
    mReactInstanceManager = reactInstanceManager;
    mJSModuleName = moduleName;
    mLaunchOptions = launchOptions;
    
    if (!mReactInstanceManager.hasStartedCreatingInitialContext()) {
      mReactInstanceManager.createReactContextInBackground();
    }
    
    // We need to wait for the initial onMeasure, if this view has not yet been measured, we set which
    // will make this view startReactApplication itself to instance manager once onMeasure is called.
    if (mWasMeasured) {
      attachToReactInstanceManager();
    }
      }

这个方法的说明是：通过给定的 JS module（由参数 moduleName确定） 提供的 JS Application 来调度 React Component 的渲染，并且使用提供的 ReactInstanceManager（由参数 reactInstanceManager 确定) 实例将 JS Context 绑定到这个 Manger。launchOptions 参数可以被传递作为 react Component 的初始化属性。

这个方法里面，前面的代码是非空检查，我们直接跳过，然后是赋值代码，也跳过，我们直接去到第一个if里面，在第二个 if 里面，如果 mWasMeasured 为 true 的话，则执行  attachToReactInstanceManager()，那么 mWasMeasured 默认是 false，什么时候才是 true呢？我们在 ReactRootView 的 onMeasure()方法找到了，在这里 mWasMeasured 被赋值为true，也就是只要 ReactRootView 完成了 onMeasure 阶段，下面的 attachToReactInstanceManager() 方法就会被调用。

      private void attachToReactInstanceManager() {
    if (mIsAttachedToInstance) {
      return;
    }
    
    mIsAttachedToInstance = true;
    Assertions.assertNotNull(mReactInstanceManager).attachMeasuredRootView(this);
    getViewTreeObserver().addOnGlobalLayoutListener(getKeyboardListener());
      }

由于 ReactInstanceManager 是抽象类，我们找到它的实现类 XReactInstanceManagerImpl 类（旧版代码还有一个实现类：ReactInstanceManagerImpl 类)，我们简单看一些 XReactInstanceManagerImpl 类中实际调用的方法：

      private void attachMeasuredRootViewToInstance(
      ReactRootView rootView,
      CatalystInstance catalystInstance) {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "attachMeasuredRootViewToInstance");
    UiThreadUtil.assertOnUiThread();
    
    // Reset view content as it's going to be populated by the application content from JS
    rootView.removeAllViews();
    rootView.setId(View.NO_ID);
    
    UIManagerModule uiManagerModule = catalystInstance.getNativeModule(UIManagerModule.class);
    int rootTag = uiManagerModule.addMeasuredRootView(rootView);
    @Nullable Bundle launchOptions = rootView.getLaunchOptions();
    WritableMap initialProps = Arguments.makeNativeMap(launchOptions);
    String jsAppModuleName = rootView.getJSModuleName();
    
    WritableNativeMap appParams = new WritableNativeMap();
    appParams.putDouble("rootTag", rootTag);
    appParams.putMap("initialProps", initialProps);
    catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      }

先看这里的参数，第一个参数还是之前的 ReactRootView，第二个参数是 CatalystInstance 实例，CatalystInstance 是一个高层的 JSC 异步通信的 bridge 接口，这个接口提供了调用 JS 和 Java 互调的一个环境。等会再讲解这个接口的具体作用和相关实现，我们把注意力放在下面的代码。我们跳过前面的代码，来到第十行代码，UIManagerModule uiManagerModule = catalystInstance.getNativeModule(UIManagerModule.class)。这个 UIManagerModule 类就是提供给 JS 创建和更新 native Views 的 module 类了。


之前提到的 CatalystInstance 实例是在 XReactInstanceManagerImpl 类的 createReactContext() 方法调用了自己实现类 CatalystInstanceImpl 的 build()方法创建的，具体的代码如下：

    CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
        .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
        .setJSExecutor(jsExecutor)
        .setRegistry(nativeModuleRegistry)
        .setJSModuleRegistry(jsModulesBuilder.build())
        .setJSBundleLoader(jsBundleLoader)
        .setNativeModuleCallExceptionHandler(exceptionHandler);
         
         .....      

      final CatalystInstance catalystInstance;
       ......
      catalystInstance = catalystInstanceBuilder.build();

我们来分析一下这里的代码，

然而 createReactContext()方法是在一个 AsyncTask 中执行的，在 XReactInstanceManagerImpl 类里无论是 createReactContext()方法还是 recreateReactContextInBackground() 系列的方法，最终都是调用这个 AsyncTask,所以可以看为一样的机制。


//Java 和 JS通信？我更关注的是，如何将JS注册的参数传递到 Java层？如果知道这点的话，就能更好的，去实现一些代码。
     
## ReactRootView 解析 ##


这个方法只是简单的 New 了一个 ReactRootView 出来，然后方法的说明，一个子类可以覆盖这个方法实现自定义的 ReactRootView。且不说这个，看到这里，我们可以推测，这个 RootView 肯定具有我们常用的 View 一些共同点，因为我们经常也是 new一个button出来什么的。

接着我们去看一下 ReactRootView 类的源码

    /**
     * Default root view for catalyst apps. Provides the ability to listen for size changes so that a UI
     * manager can re-layout its elements.
     * It delegates handling touch events for itself and child views and sending those events to JS by
     * using JSTouchDispatcher.
     * This view is overriding {@link ViewGroup#onInterceptTouchEvent} method in order to be notified
     * about the events for all of it's children and it's also overriding
     * {@link ViewGroup#requestDisallowInterceptTouchEvent} to make sure that
     * {@link ViewGroup#onInterceptTouchEvent} will get events even when some child view start
     * intercepting it. In case when no child view is interested in handling some particular
     * touch event this view's {@link View#onTouchEvent} will still return true in order to be notified
     * about all subsequent touch events related to that gesture (in case when JS code want to handle
     * that gesture).
     */
    public class ReactRootView extends SizeMonitoringFrameLayout implements RootView {
    .......
    .......

      public ReactRootView(Context context) {
    super(context);
      }
    
      public ReactRootView(Context context, AttributeSet attrs) {
    super(context, attrs);
      }
    
      public ReactRootView(Context context, AttributeSet attrs, int defStyle) {
    super(context, attrs, defStyle);
      }
    }

简单的翻译一下就是：构建app的默认 root view，提供了监听大小改变的能力，所以一个UI manager可以重新layout它的元素。它为自己和它的child view 代处理所有的触摸事件，并且会通过 JSTouchDisoatcher 发送事件到JS。这个类重写了 ViewGroup的onInterceptTouchEvent()方法，和ViewGroup的requestDisallowInterceptTouchEvent()方法，保证所有的触摸事件能够被ViewGroup的onInterceptTouchEvent()能够正常分发和接收。如果child view 对当前的触摸事件不感兴趣，当前 child view的 onTouchEvent()方法依旧必须返回 true，这样才能保证可以接收到一些系列的Touch 事件(也可能JS想要接收这些事件)。
简单的来说这个类，它继承自 SizeMonitoringFrameLayout 类，而 SizeMonitoringFrameLayout 继承自 FrameLayout布局类，实现了一个View改变大小的接口回调类，用于监听 View的 onSizeChanged()方法，实现比较简单，所以我们把注意力集中在 ReactRootView 本身。


所以看到这里，看到构造方法，熟悉自定义组件的同学，应该很容易看出来，这个 ReactRootView 肯定是个自定义组件了。那么我们着重的来看一下 View 类型组件的相关方法：

onMeasure()方法：

      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    // 如SpecMode 为UNSPECIFIED,则直接抛出异常，UNSPECIFIED也就是说高和宽都不确定
    if (widthMode == MeasureSpec.UNSPECIFIED || heightMode == MeasureSpec.UNSPECIFIED) {
      throw new IllegalStateException(
      "The root catalyst view must have a width and height given to it by it's parent view. " +
      "You can do this by specifying MATCH_PARENT or explicit width and height in the layout.");
    }
    
    setMeasuredDimension(
    MeasureSpec.getSize(widthMeasureSpec),
    MeasureSpec.getSize(heightMeasureSpec));
    
    mWasMeasured = true;
    // Check if we were waiting for onMeasure to attach the root view
    if (mReactInstanceManager != null && !mIsAttachedToInstance) {
      // Enqueue it to UIThread not to block onMeasure waiting for the catalyst instance creation
      UiThreadUtil.runOnUiThread(new Runnable() {
    @Override
    public void run() {
      attachToReactInstanceManager();
    }
      });
    }
      }

onMeasure()方法确定了组件的大小，我们看到这里应该考虑，我们在JS中设置了那个Text，Text的属性值，例如颜色，宽高，是怎么传递到android原生的。这里先简单说一下Android原生View的绘制需要一个MeasureSpec类参数，MeasureSpec 由 SpecMode和SpecSize组成，SpecMode有三种类型：

1. EXACTLY
表示父视图希望子视图的大小应该是由specSize的值来决定的，系统默认会按照这个规则来设置子视图的大小，开发人员当然也可以按照自己的意愿设置成任意的大小。
2. AT_MOST
表示子视图最多只能是specSize中指定的大小，开发人员应该尽可能小得去设置这个视图，并且保证不会超过specSize。系统默认会按照这个规则来设置子视图的大小，开发人员当然也可以按照自己的意愿设置成任意的大小。
3. UNSPECIFIED
表示开发人员可以将视图按照自己的意愿设置成任意的大小，没有任何限制。


而 SpecSize则记录了组件的大小信息。
在上面的 onMeasure()方法中，前两行获得了widthMode和heightMode，然后判断它们两个的数值是不是等于 UNSPECIFIED,如果其中一个等于 UNSPECIFIED，则抛出异常。调用 View的setMeasuredDimension（）方法，设置View的宽和高。接着讲 mWasMeasured 设置 true,这个是标志这个View已经完成了Measure操作的标志。再接着跑判断是否为空，并且是还没attached到Instance上去的，然后在UI主线程执行一个 runnable，执行attachToReactInstanceManager()方法。

我们进入 attachToReactInstanceManager()方法看一下:

      private void attachToReactInstanceManager() {
    if (mIsAttachedToInstance) {
      return;
    }
    // 这里将 mIsAttachedToInstance 设置为 true
    mIsAttachedToInstance = true;
    Assertions.assertNotNull(mReactInstanceManager).attachMeasuredRootView(this);// Asserions框架，判断 mReactInstanceManager 是否为空。不为空的话 attach它。
    getViewTreeObserver().addOnGlobalLayoutListener(getKeyboardListener());
      }

这个方法里面，调用了 ReactInstanceManager 类的 attachMeasuredRootView()方法，我们看一下这个方法做了什么，


      /**
       * Attach given {@param rootView} to a catalyst instance manager and start JS application using
       * JS module provided by {@link ReactRootView#getJSModuleName}. If the react context is currently
       * being (re)-created, or if react context has not been created yet, the JS application associated
       * with the provided root view will be started asynchronously, i.e this method won't block.
       * This view will then be tracked by this manager and in case of catalyst instance restart it will
       * be re-attached.
       */
      public abstract void attachMeasuredRootView(ReactRootView rootView);

我们可以看到 ReactInstanceManager 是在 startReactApplication()方法里被赋值，所以我们又回到了 ReactActivity 类的 onCreate()方法中了，这里对 ReactInstanceManager 进行了赋值调用的是：

    getReactNativeHost().getReactInstanceManager()

这个 ReactInstanceManager 类又是用来干嘛的呢？

ReactInstanceManager 类是在 ReactNativeHost 类的 getReactInstanceManager() 方法下被创建的。

      public ReactInstanceManager getReactInstanceManager() {
    if (mReactInstanceManager == null) {
      mReactInstanceManager = createReactInstanceManager();
    }
    return mReactInstanceManager;
      }

而 ReactNativeHost 类是在 Application 类被创建的，ReactNativeHost 类的实现如下：

    private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
        @Override
        protected boolean getUseDeveloperSupport() {
            return BuildConfig.DEBUG;
        }

        @Override
        protected List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    new MainReactPackage(),
                    new ReactNativePackager()
            );
        }
    };

然而，ReactNativeHost 类又是通过 ReactInstanceManger 实现类的 Builder的build()方法生成一个 ReactInstanceManger 实例的，这种设计模式在 java 中称为 建造者(builder)设计模式。我们看下build()方法里面，做了什么：

    public ReactInstanceManager build() {
     .....// 这里的代码是一些断言判断不为空的代
    
       // 这里的代码是返回一个旧的 ReactInstanceManager，我在0.29的源码里看到有，在0.32的源码已经被删除了
      return new XReactInstanceManagerImpl(
      mApplication,
      mCurrentActivity,
      mDefaultHardwareBackBtnHandler,
      (mJSBundleLoader == null && mJSBundleFile != null) ?
    JSBundleLoader.createFileLoader(mApplication, mJSBundleFile) : mJSBundleLoader,
      mJSMainModuleName,
      mPackages,
      mUseDeveloperSupport,
      mBridgeIdleDebugListener,
      Assertions.assertNotNull(mInitialLifecycleState, "Initial lifecycle state was not set"),
      mUIImplementationProvider,
      mNativeModuleCallExceptionHandler,
      mJSCConfig,
      mRedBoxHandler);
    }

我们简单来看一下 return 语句里面执行的操作，

看到这里，我们在回到 ReactActivity 的 onCreate()方法，似乎就有点峰回路转了。我们找到 ReactInstanceManager 的实现类 XReactInstanceManagerImpl（旧版本是使用ReactInstanceManagerImpl，这两者之间略有不同，后续再分析）。我们先看一下 XReactInstanceManagerImpl 类的 attachMeasuredRootView()方法：

      @Override
      public void attachMeasuredRootView(ReactRootView rootView) {
    UiThreadUtil.assertOnUiThread();
    mAttachedRootViews.add(rootView);
    
    // If react context is being created in the background, JS application will be started
    // automatically when creation completes, as root view is part of the attached root view list.
    if (mReactContextInitAsyncTask == null && mCurrentReactContext != null) {
      attachMeasuredRootViewToInstance(rootView, mCurrentReactContext.getCatalystInstance());
    }
      }


JSBundleLoader.createFileLoader(mApplication, mJSBundleFile) 去加载 JSbundle 文件的。

调用了View的getViewTreeObserver()方法，获取了当前View的 ViewTreeObserver，ViewTreeObserver 是一个视图树的监听类，会在View树重新 layout，draw，measure的时候发送和处理通知。那么这里为什么为这个 ViewTreeObserver 添加一个 Listener 呢？ 


我们继续往下看，SizeMonitoringFrameLayout类 是继承 FrameLayout的，实现了 RootView 接口，而RootView 接口只是为了RN中，操作了原生事件的时候能够回调，先看下 RootView 的代码：


import android.view.MotionEvent;

    /**
     * Interface for the root native view of a React native application.
     */
    public interface RootView {
    
      /**
       * Called when a child starts a native gesture (e.g. a scroll in a ScrollView). Should be called
       * from the child's onTouchIntercepted implementation.
       */
      void onChildStartedNativeGesture(MotionEvent androidEvent);
    }

我们尝试在源码中找出它的调用时机 // todo



参考资料：