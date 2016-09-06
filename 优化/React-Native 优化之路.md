# React-Native 优化之路 #

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