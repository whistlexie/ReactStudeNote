# React-Native 优化之路 #


### 布局篇 ###


从 ReactNative的设计结构知道，使用 JS（JSX） 代码编写类似 html 的元素组装的代码，实际实现的是一套 Virtual Dom的虚拟的js数据结构，然后通过 JSBridge 和 native 通信，根据设置的属性值，生成android 原生的 View。android上的JS运行环境是，webkit.org 的jsc.so。从UI角度考虑，一段js代码到生成原生UI界面的过程如下：



首先从 JS 端考虑，这个渲染过程的优化，JS端的渲染流程如下图：


由于 ReactNative 组件开发的思想非常好用



### 使用初始化 getLaunchOptions()方法 ###

使用初始化 getLaunchOptions()方法，返回初始化数值，减少一次 render() 方法的调用。

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
    ......// some resouce code
    }

在 ReactRootView 的 startReactApplication()方法里面可以传递一个初始化启动的参数，作为入口 Component 的初始化参数，之前的逻辑是，入口 Component 在初始化完成之后，接收到新的参数重新 Update。修改后的逻辑是，在进入 ReactRootView 对应的 activity之后，发送网络请求，然后在数据响应回来之后，才 attach ReactRootView，这样就减少了一次无效的 render，同样也是也对一些静态场景最好的逻辑解释。如下如：
// render.png

具体的代码怎么实现呢？使用过 Bundle 的同学都知道，这里面直接存储的是键值对，所以很符合 JS 对象的属性名/属性值的特点，我们暂时这样试一下。

android 代码：

如果你的 Activity 是继承 ReactActivity 那么只需要重载 getLaunchOptions()方法：

    @Override
    protected Bundle getLaunchOptions() {
    // 测试初始化参数
    Bundle bundle = new Bundle();
    bundle.putCharSequence("name","kotlon");
    bundle.putInt("age",18);
    return bundle;
    //return super.getLaunchOptions();
    }

如果不是的话，需要在 ReactRootView 对象的 startReactApplication（）方法里面，传递一个 bundle 对象。

JS端的代码，直接通过 this.props.你在bundle传递的key值，示例代码如下：

    render() {
        return( <Text style={Styles.text} >name:{this.props.name}{'\n'}age:{this.props.age}</Text>);
    }

运行结果显示：
//运行结果.png



tip：然而在实际开发中，数据比较复杂，多为复杂的json对象，使用 bundle 传递这种复杂的数据，效率并不高，而且对代码编写来说，尤为繁琐，我争寻找更好的解决办法。这里一个巧妙的办法是，将显示 loading交给 React 去完成，然后网络请求完成之后，则通知 react 更新视图，或者说网络请求也在 react 中完成，因为 react 也是封装了 okhttp 的。