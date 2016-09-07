# React-Native 优化之路 #


## 布局篇 ##


从 ReactNative的设计结构知道，使用 JS（JSX） 代码编写类似 html 的元素组装的代码，实际实现的是一套 Virtual Dom的虚拟的js数据结构，然后通过 JSBridge 和 native 通信，根据设置的属性值，生成android 原生的 View。android上的JS运行环境是，webkit.org 的jsc.so。从UI角度考虑，一段js代码到生成原生UI界面的过程如下：



首先从 JS 端考虑，这个渲染过程的优化，JS端的渲染流程如下图：

#### 改变了 props或者states ####

一个 React Component 的生命周期方法如下图，它总共经历了三个状态 Mounting，Updating，Unmounted。

// 组件生命周期.png

**接收新的 props**

在初始化之后，接受到新的 Props 的时候，可能会进入 update 状态，有可能再次调用 render()方法。我们在 componentWillReceiveProps()方法里面可以却设置 State

    componentWillReceiveProps(nextProps) {
        // 仅在 loading 改变的时候，更改一些 state
        if(this.props.isLoading!=nextProps.isLoading){
            // 更改 state
        }else{
            // do nothing
        }
    }

tip:ES6 中在 constror 中初始化 state 语法为 

        this.state ={
            isLoading:true,
            number:100,
        }

然而更改 state 需要调用 setState(nextState)方法,传入的是一个 Object 参数，表示新的 state数值，例如以下：

            this.setState ({
                number:195
            })

不要混淆，不是在 constror 使用初始化语法是没有效果的。

**设置新的 state**

旧的 props 通过 this.props 引用，新的 props 就是这个方法传入发参数。通常情况下，props 只能通过父控件来改变，那么是否也有一种机制，让 state 也有这样的判断机制，我们再回到上面的那张生命周期图，可以看到 state 改变之后会回调一个方法 shouldComponentUpdate(nextProps,nextState),这里面一个参数是新的 props 一个是新的 State，如果这个方法放回 true 的话，就会触发下一次的 render()，返回 false 的话，则不会触发，默认是返回 true。所以我们可以在这里做一些逻辑操作，根据业务情况判断需不需要进行重新的 render.


    shouldComponentUpdate(nextProps,nextState) {
        // 仅在 number 变大之后更新视图
        if(this.state.number<nextState.number){
            return true;
        }else{
            return false;
        }
    } 

**-**


#### 使用初始化 getLaunchOptions()方法 ####

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


#### 使用 AsyncStroage 缓存数据减少实际渲染时间 ####

在一般的网络请求中，我们可能直接使用 fetch api，或者使用react-native 内置的XMLHttpRequest API(也ajax)获取网络数据，关于 fetch api可以参考 MDN 的文档 使用 [Fetch](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API/Using_Fetch) ，没有使用缓存之前，我们的渲染流程大概是这样的的：
// 无缓存渲染


使用缓存的情况下：
// 有缓存渲染



我们可以使用 LoaclStroage 管理缓存，但是 React Native 推荐使用 AsyncStroage，AsyncStroage 是一个持久化存储的键值对系统，官方不建议直接使用，我们可以使用 React Native 中文网封装的 [react-native-storage](https://github.com/sunnylqm/react-native-storage)。下面是简单的代码示例：


    // 写进缓存
    writeJSONCache() {
        AsyncStorage.setItem('movies', 'Star Wars Cache', ()=> {
            ToastAndroid.show('写缓存', ToastAndroid.LONG)
        });
    }

    // 读缓存
    getJSONFromCache() {
        AsyncStorage.getItem('movies', (error, result)=> {
            ToastAndroid.show('读缓存', ToastAndroid.LONG)
            this.setState({
                movies: result
            })
        })
    }

将缓存读取，然后和 response 比较。我们不关心这个 response是来自 http 缓存还是网络请求结果，因为无论还是 Fetch API 还是 XMLHttpRequest 都会实行 Http 缓存这一块，但是 http 缓存的一般是 http response body，在粒度上达不到我们想要的程度。关于 AsyncStorage 的具体实现是数据库，会把 key 和 value 存成一张数据库表。在android开发的经验看了，我们操作数据库，都需要释放 Cursor。在这里我挺疑惑，React Native 是怎么去释放 Cursor和断开数据库的连接的。



#### 优化 ListView(JS端) ####

React Native 中ListView 采用的是 ScrollView 实现的，不得不说，这真的是一个性能瓶颈的问题。但是诸如 ListView 的控件，在实际开发中使用频率是非常高的，在 weex 中，weex使用的是 RecycleView 实现的，所以在一些情况上，性能表现要好上很多。我们先来简单的看一段使用 ListView 的示例代码：



通过阅读 API 我们可以找出以下关注点：


在尝试有优化工作前，我们尝试按照之前 android 原生上的 ListView 优化经验，去尝试优化 JS代码。



#### Style样式优化 ####


## 掉帧现象和Event事件 ##

等待调试机到来中....