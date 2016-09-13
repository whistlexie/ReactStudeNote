
### 问题描述 ###
在之前的 debug 发现，在 UIViewOperationQueue 类里面（位置：react\uimanager\UIViewOperationQueue.java），在画面渲染完成之后，里面的存储了 UIOperation 的 ArrayList 还是会不停的接受到新的 UIOperation（每一帧），然后不断的被 add进List里里面去，然后执行operate.excute（）方法，这个照理来说不太符合实际情况的，所以在此去了解和发现一下。

### 介绍 ###

**UIViewOperationQueue 类**

根据源码上的注释，这个类是用来缓存动画和Native View 层级管理命令的一个缓冲类。

**NativeViewHierarchyManager 类**

UIManagerModule 的代理类，UIManagerModule 类则是保留 Native View 层级，和被JS使用的Native View类和其对应的通信ViewManager映射。UIManagerModule提供了一系列更新属性，更新Layout，创建View，管理子View的方法给NativeViewHierarchyManager类使用。

**UIManagerModule 类**

一个允许 JS 创建和更新 Native View 的 module 类。

**DispatchUIFrameCallback 类**

实现了 Choreographer.FrameCallback 接口（并非直接实现），然后会在每一帧更新View的时候，调用 

**UIImplementation 类**

从JS接受React 命令，并且把 node shadow 视图层级转化成 Native 视图层级，这里持有了 UIViewOperationQueue，负责调用 UIViewOperationQueue 的各种 operation enqueue(入队)方法。这里也暴露了给React 调用的方法，例如创建View，更新View之类的方法。例如创建View是根据一个 ReactShadowNode 对象，来创建的。


简单的描述一下创建一个 View 的在原生上的绘制流程调用方法链：

![](https://github.com/whistlexie/Images/blob/master/operation/%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%20View.png?raw=true)

测试代码：只显示了一个 Text 在界面上

    import React,{Component} from 'react'
    import {Text, AppRegistry, ToastAndroid} from 'react-native'
    
    class OneText extends Component{
    constructor(props){
    super(props)
    }
    render(){
    return(<Text style={{fontSize:16, color:"#f25"}}
    >A text to test operation!</Text>)
    }
    }
    AppRegistry.registerComponent("TestRN",()=>OneText)



其实这里就是为了验证，每次加入到这个 List 里面的是什么，我们从源码预知这里无非是各种继承了 UIOperation 的事件，但具体是什么是事件，需要进行验证。

      private ArrayList<UIOperation> mOperations = new ArrayList<>();


我们是为了弄清楚，每次添加到 ArrayList<UIOperation> 里面的 Operations 是什么操作，我们在 743行看到，这里将 operations 进行赋值，后面就是new 一个新的 ArrayList，然后存储新的 Operation，那么将断点打在这里就可以分析出，operation究竟是一些什么事件。

    final ArrayList<UIOperation> operations = mOperations.isEmpty() ? null : mOperations;

**调试方式，attach process 断点调试**


#### 首次打开： ####

但是却发现一进入React 页面，就运行了这行代码了，但是 operations 是空的。

一直 step over ，第三次 就不为空了，截图：

![](https://github.com/whistlexie/Images/blob/master/operation/%E7%AC%AC%E4%B8%89%E6%AC%A1%E7%A9%BA%E4%B9%8B%E5%90%8E.png?raw=true)

然后显示内容

然后又会进入，一直都是空的，之后一直能调用，所以断言这个方法肯定的实现，肯定是 handler + message+messagequeue+looper 这种机制。


### 进入React页面之后，再点击 attach process  ###

之后发现进入 React 页面之后，什么也不做，是不会进入这个方法的，但是在屏幕锁屏的时候，又进入了这个方法。其实可以从这里判断，这里的这个方法是触发性的，也就是接口回调的实现。


### 断点打在 List 的 add()方法上 ###

1. enqueueManageChildren()方法 ManageChildrenOperation 大概是和 Child View 有关系
1. enqueueUpdateExtraData()方法 UpdateViewExtraData 和虚拟 Dom 的计算有关
1. enqueueUpdateLayout()方法 UpdateLayoutOperation 

然后便是显示内容了。


### 进入 React 页面，点击 attach process，然后滑动 ###

会进入这个方法，这里的 operations 也是空的。


### 总结 ###

**总结：并没有发现有莫名奇妙的 Operation 会被一直发送和执行，只有当产生交互的时候才会有 Operation 发送。**


那么之前说的，不断的发送 Operation 是怎么回事呢？然后发现另一种情况就是，进入这个方法之后，会不断的执行这个方法（Loop）

这里的问题是，第一次 Create 是怎么调用 dispatchViewUpdates()方法的呢？我只在 UIImplementation的 updateRootNodeSize()方法里面看到了调用这个方法，实际上第一次创建的时候也会调用，仔细找了一下，暂时还是没有找到回调机制。


