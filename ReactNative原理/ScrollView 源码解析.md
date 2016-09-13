# ScrollView 源码解析 #

**前序：ScrollView 有一个 removeClippedSubviews 属性，如果设置为 true，并且设置contentContainerStyle属性的 overflow数值为'hidden' ScrollView Component 不在屏幕显示的 子View 会被移除，从而减少内存的使用和减少不必要的绘制。**

## 性能分析 ##
我们简单的进行一下性能分析，工具是android的静态布局检查工具 hierarchyviewer。

默认不开启的状态：

// png

开启之后：
// png


可以看到，不在视图内的View是不会被绘制出来的，React ScrollView 在 OnScroll时候，会不停的去更新显示的子View，渲染时间的对比如下图：
//png


这张体大概说明 ScrollView 设置了removeClippedSubviews 属性为 true，也确实在 View tree 中去掉了不可见的 View，但是组件的加载时间消耗资源还是随着子组件的数量成正比，也就是所谓的响应时间。


## 源码分析 ##

ScrollView的源码位置位于 node_modules/react-native\Libraries\Components\ScrollView\ScrollView.js 我在 ScrollView 目录下惊喜的发现了：

//scrollview.png


我似乎看到了 RecycleView 身影，这点在后面的展望里面，我在详细说一下。


由于 ScrollView 的源码比较长，而且比较复杂，所以这里就不把完整的代码贴出来了，这里先把关注点放在 render() 函数里面，render里面 return 语句如下：

    if (refreshControl) {
      if (Platform.OS === 'ios') {
        // On iOS the RefreshControl is a child of the ScrollView.
        return (
          <ScrollViewClass {...props} ref={this._setScrollViewRef}>
            {refreshControl}
            {contentContainer}
          </ScrollViewClass>
        );
      } else if (Platform.OS === 'android') {
        // On Android wrap the ScrollView with a AndroidSwipeRefreshLayout.
        // Since the ScrollView is wrapped add the style props to the
        // AndroidSwipeRefreshLayout and use flex: 1 for the ScrollView.
        return React.cloneElement(
          refreshControl,
          {style: props.style},
          <ScrollViewClass {...props} style={baseStyle} ref={this._setScrollViewRef}>
            {contentContainer}
          </ScrollViewClass>
        );
      }
    }
    return (
      <ScrollViewClass {...props} ref={this._setScrollViewRef}>
        {contentContainer}
      </ScrollViewClass>
    );


第一行 if 语句里面根据是否为 ScrollView 指定了 RefreshControl 刷新控件，如果有则会生成，如果没有设置，则生成的控件只包含 contentContainer。这里出现了我们不熟悉的 Component ScrollViewClass，其实这个是 ScrollView 的具体实现类。我们往上翻一翻就可以看到这个 ScrollViewClass 的身影：

    let ScrollViewClass;
    if (Platform.OS === 'ios') {
      ScrollViewClass = RCTScrollView;
    } else if (Platform.OS === 'android') {
      if (this.props.horizontal) {
        ScrollViewClass = AndroidHorizontalScrollView;
      } else {
        ScrollViewClass = AndroidScrollView;
      }
    }

上面的代码根据不同的操作系统平台，分别将不同的 Component 赋予 ScrollViewClass 变量，这里可以看到，ios 用的是 RCTScrollView这个类，android 上根据 主轴(horizontal属性 true或者false)方向是水平或者垂直，来使用不同的两个 Component 控件，具体的实现类是由下面的代码决定的（这里只展示一种）

      AndroidScrollView = requireNativeComponent('RCTScrollView', ScrollView, nativeOnlyProps);


RCTScrollView 是原生 android 代码封装了 原生 ScrollView 控件的一个组件类，能在react natives android 相关源码上看到相关的实现（由于这里将某个组件类注册为RCTScrollView，在 android 上具体的类名不一定是这个，实际上其对应的类名是 ReactScrollView）我们这里暂且不管，总之需要知道，RCTScrollView 对应的是一个封装的好的android 原生 ScrollView 就好了，我们回到之前的代码。看下这行代码：

      <ScrollViewClass {...props} ref={this._setScrollViewRef}> //这里标签未闭合只是为了说明语法

这里将 ScrollView 的属性，通过 jsx 语法 "...props"全部传递到 ScrollViewClass 里面去了，然后通过 ref 指向自己。接着我们看到这样两行代码：

            {refreshControl}
            {contentContainer}

在 React 的语法中，碰到{}就解析为 javascript 表达式，所以这里两个 javascript 表达式，前者是生成了 refreshControl 下拉刷新控件，后者是 contentContainer 表示的是一个 ScrollView Item 的内容，具体可看下面的代码：

    // 定义了一个 const 常量，不可变常量(ES6语法),把一个 View 赋值给了这个常量
    const contentContainer =
      <View
        {...contentSizeChangeProps}
        ref={this._setInnerViewRef}
        style={contentContainerStyle}
        removeClippedSubviews={this.props.removeClippedSubviews}
        collapsable={false}>
        {this.props.children}
      </View>;

上面的代码基本就是设置了一堆属性，我们先跳过，我们看到这里似乎还不知道，一个 ScrollView 怎么添加一个 Item 的把，这似乎不太对呀！！直观的，应该有类似的 add()方法之类的或者需要滑动事件的处理之类的。由于JS源码可读性比较差，反正我是找不到类似的信息，所以这会我们便跑去看看呗（你可以直接使用 AS 查看源码，在Project 视图下 去到 External Libaries目录下找到react-native源码包或者你可以解压Jar直接看）


    源码路径 com.facebook.react.views.scroll.ReactScrollView 



public class ReactScrollView extends ScrollView implements ReactClippingViewGroup {
    .........
    }

这段代码清楚的向我们展示了 ReactScrollView 的继承关系，它继承了 ScrollView，实现了 ReactClippingViewGroup 接口，其中继承 ScrollView 的话，我们只需呀关注，React ScrollView 如何添加 Item 到 ScrollView 中（因为在React 中它更像Native的ListView的特性，有添加Item的能力，而不像Native的ScrollView只有容器能力）。

我们先来看看后面这个接口是干嘛的

     // 这里省去了所有的注释
    public interface ReactClippingViewGroup {
      void updateClippingRect();
      void getClippingRect(Rect outClippingRect);
      void setRemoveClippedSubviews(boolean removeClippedSubviews);
      boolean getRemoveClippedSubviews();
    }

根据注释，我们可以知道实现了这个接口的 View 组件类，就能够在 JS 中支持 removeClippedSubviews 的属性。如果一个 ViewGroup 组件类具有这个属性，就可以移除被这个 ViewGroup 边界切割的子 View。具体的区分一个子 View 是否被切割，要根据 ReactClippingViewGroupHelper 类的 calculateClippingRect()方法来计算。

简单来说，就是实现了这个接口的 ViewGroup 组件类，就可以移除不可见的子 View，回收内存(没考虑复用，因为复用的代价比较大，后面会说到)。

我们回到源码简单看一下构造函数：

    public ReactScrollView(Context context, @Nullable FpsListener fpsListener) {
    super(context);
    mFpsListener = fpsListener;
    // sTriedToGetScrollerField 默认是 false，所以一定会执行一次第一个if里面的代码。
    if (!sTriedToGetScrollerField) {
      sTriedToGetScrollerField = true; // 设置为 true
      try {
    // 通过反射获取 mScroller字段，mScroller其实是一个 OverScroller 对象
    sScrollerField = ScrollView.class.getDeclaredField("mScroller");
    sScrollerField.setAccessible(true);// 为了避免 IllegalAccessExceptions 异常
      } catch (NoSuchFieldException e) {
    Log.w(
      ReactConstants.TAG,
      "Failed to get mScroller field for ScrollView! " +
    "This app will exhibit the bounce-back scrolling bug :(");
      }
    }
    // 如果 OverScroller 对象不为空，则根据字段拿到 OverScroller 对象。
    if (sScrollerField != null) {
      try {
    mScroller = (OverScroller) sScrollerField.get(this);
      } catch (IllegalAccessException e) {
    throw new RuntimeException("Failed to get mScroller from ScrollView!", e);
      }
    } else {
      mScroller = null;
    }
    }


总的来说，构造函数里面就是为了获取一个 一个 OverScroller 对象。 OverScroller 是一个 Scoller 类，用于滚动的辅助计算，最早的实现是 Scroller 类，大概在 api 1的时候，后来 google 为了完善 Scroller 类，就给出了 OverScroller 这个类，用于处理超出边界的滚动及其他扩展情况。然后，我们还忘记了  mFpsListener = fpsListener;这一行代码，这个类和我们 React 的 onScroll 函数回调有关，后面有用到再解释。

ReactScrollView 对应这一个 ReactScrollViewManager 具体的位置位于 com.facebook.react.views.scroll.ReactScrollViewManager,这个类管理着 ReactScrollView 的一个示例，并且暴露了创建实例和设置属性的方法给 JS。

      @Override
      public String getName() {
    return REACT_CLASS;
      }
    
Manager 通过 放回一个 String 的变量供 JS 调用，然后通过 @ReactProp（或@ReactPropGroup）注解来导出属性的设置方法，提供给 JS 设置，也就大概反映了着他们之间的通信模式。如下，便是一个属性设置方法的示例代码：

      @ReactProp(name = "scrollEnabled", defaultBoolean = true)
      public void setScrollEnabled(ReactScrollView view, boolean value) {
    view.setScrollEnabled(value);
      }

上面的方法，暴露给 JS，JS可以通过这个方法设置 scrollEnabled 属性，来决定 ScrollView 的内容是否可以被滚动。

我们大概了解了一下 ReactScrollView 之后，就可以着手进行分析了，我们首先从 Native ScrollView 上出发，考虑的几个问题和前提：

- ScrollView 继承子 FrameLayout，它是一个 ViewGroup.
- ScrollView 只允许一个直接子View，然后这个子 View 在包裹这需要滚动的内容，这个子 View 一般是 LinearLayout 之类的布局类。 
- ScrollView 相当于把全被子View的内容一次显示出来，或者说，它不像 ListView 一样是动态添加的，但是 React ScrollView 却可以实现动态添加，是怎样一种实现机制？


就第三个问题来分析一下，我们知道 View的移动（内容滑动，并非View本身移动）就是调用了 View 的 scrollBy()和scrollTo()方法，而scrollBy()和scrollTo()方法最终会导致 View 的 onScrollChanged()方法的调用，那我每年进去 ReactScrollView 看下这个方法：

      @Override
      protected void onScrollChanged(int x, int y, int oldX, int oldY) {
    super.onScrollChanged(x, y, oldX, oldY);// 继承了 onScrollChanged()，这里只需要知道这里完成了
    // 判断移动是否有效
    if (mOnScrollDispatchHelper.onScrollChanged(x, y)) {
      if (mRemoveClippedSubviews) { //默认是false，这里的 mRemoveClippedSubviews 设置值是在哪里设置的？
    updateClippingRect();
      }
    
      if (mFlinging) {
    mDoneFlinging = false;
      }
    
      ReactScrollViewHelper.emitScrollEvent(this);
    }
      }

这个方法里面，继承了 ScrollView 的 onScrollChanged()方法，然后第一个 if 里面判断移动是否有效，然后第二个if里面，根绝 mRemoveClippedSubviews 的布尔值，如果是 true 就执行 updateClippingRect() 方法，这个 mRemoveClippedSubviews 是在哪里设置的呢？ ReactScrollView 里面有个 setRemoveClippedSubviews()方法，这里对 mRemoveClippedSubviews 进行了赋值。但是又是再哪里调用这个方法的呢？

      @Override
      public void setRemoveClippedSubviews(boolean removeClippedSubviews) {
    if (removeClippedSubviews && mClippingRect == null) {
      mClippingRect = new Rect();
    }
    mRemoveClippedSubviews = removeClippedSubviews;
    updateClippingRect();
      }

一路找呀找，我们找到了，ReactScrollViewManager 类发现了下面这个方法：


      @ReactProp(name = ReactClippingViewGroupHelper.PROP_REMOVE_CLIPPED_SUBVIEWS)
      public void setRemoveClippedSubviews(ReactScrollView view, boolean removeClippedSubviews) {
    view.setRemoveClippedSubviews(removeClippedSubviews);
      }


原来，只要你在 React 的ScrollView 里面设置了 removeClippedSubviews 属性值就是，我们之前的 mRemoveClippedSubviews数值了。那么现在就来考虑，updateClippingRect()方法里面的实现把，下面是具体的实现代码：

      @Override
      public void updateClippingRect() {
    if (!mRemoveClippedSubviews) {
      return;// 如果 React 上的 ScrollView 的removeClippedSubviews 属性设置为 false
    }
    
    Assertions.assertNotNull(mClippingRect);
    // 关键代码就在这里了
    ReactClippingViewGroupHelper.calculateClippingRect(this, mClippingRect);
    // 获取直接子 View 
    View contentView = getChildAt(0);
    //如果直接子 View 也是实现了 ReactClippingViewGroup 接口，则子 View 也会调用updateClippingRect()方法。
    if (contentView instanceof ReactClippingViewGroup) {
      ((ReactClippingViewGroup) contentView).updateClippingRect();
    }
      }

这个方法的详细很简单，基本上我都在上面的代码里面注释了，现在我们在重点放在  ReactClippingViewGroupHelper.calculateClippingRect(this, mClippingRect) 这个静态方法里面，先上代码：

     public static void calculateClippingRect(View view, Rect outputRect) {
    ViewParent parent = view.getParent();
    if (parent == null) {
      outputRect.setEmpty();
      return;
    } else if (parent instanceof ReactClippingViewGroup) {
      ReactClippingViewGroup clippingViewGroup = (ReactClippingViewGroup) parent;
      if (clippingViewGroup.getRemoveClippedSubviews()) {
    clippingViewGroup.getClippingRect(sHelperRect);
    // Intersect the view with the parent's rectangle
    // This will result in the overlap with coordinates in the parent space
    if (!sHelperRect.intersect(view.getLeft(), view.getTop(), view.getRight(), view.getBottom())) {
      outputRect.setEmpty();
      return;
    }
    // Now we move the coordinates to the View's coordinate space
    sHelperRect.offset(-view.getLeft(), -view.getTop());
    sHelperRect.offset(view.getScrollX(), view.getScrollY());
    outputRect.set(sHelperRect);
    return;
      }
    }
    view.getDrawingRect(outputRect);
      }

这里的代码也不是很长，但是单单看这里的代码还不能够说明根本问题，我之前提到过 React ScrollView 是一个 ViewGroup 类，实际上在 View 类型的子类也会实现 ReactClippingViewGroup 接口，就拿最简单的 View 来说，它在 Native 对应的类是 ReactViewGroup，先看看 updateClippingRect()方法是怎么实现的：

      @Override
      public void updateClippingRect() {
    if (!mRemoveClippedSubviews) {
      return;
    }
    
    Assertions.assertNotNull(mClippingRect);
    Assertions.assertNotNull(mAllChildren);
    
    ReactClippingViewGroupHelper.calculateClippingRect(this, mClippingRect);
    updateClippingToRect(mClippingRect);
      }

看到这里，我们似乎清楚了：

      private void updateClippingToRect(Rect clippingRect) {
    Assertions.assertNotNull(mAllChildren);
    int clippedSoFar = 0;
    for (int i = 0; i < mAllChildrenCount; i++) {
      updateSubviewClipStatus(clippingRect, i, clippedSoFar);
      if (mAllChildren[i].getParent() == null) {
    clippedSoFar++;
      }
    }
      }


      /**
       * Notify view that clipping area may have changed and it should recalculate the list of children
       * that should be attached/detached. This method should be called only when property
       * {@code removeClippedSubviews} is set to {@code true} on a view.
       *
       * CAUTION: Views are responsible for calling {@link #updateClippingRect} on it's children. This
       * should happen if child implement {@link ReactClippingViewGroup}, return true from
       * {@link #getRemoveClippedSubviews} and clipping rect change of the current view may affect
       * clipping rect of this child.
       */
      void updateClippingRect();

在 ReactClippingViewGroup 的 updateClippingRect() 方法说明里，如果一个View的裁剪区域发生了改变，需要重新绑定和解绑它对应的子View，但是这个View也必须去调用children的updateClippingRect()方法，因为父View重新计算裁剪区域也会影响到子View的裁剪区域，这种处理必须一直传递下去，直到子View没有实现ReactClippingViewGroup接口，removeClippedSubviews属性设置为 false，或者当前view的getChild(0)返回为null的时候。

      public static void calculateClippingRect(View view, Rect outputRect) {
    ViewParent parent = view.getParent(); // 获得当前 View 的 parent。
    if (parent == null) { // 如果 parent 为空,则输出 Rect 设为(0,0,0,0)
      outputRect.setEmpty();
      return;
    } else if (parent instanceof ReactClippingViewGroup) {//praent 不为空，而且实现了ReactClippingViewGroup接口
      ReactClippingViewGroup clippingViewGroup = (ReactClippingViewGroup) parent;//强制类型转换
      if (clippingViewGroup.getRemoveClippedSubviews()) {// 如果在 React Component 的removeClippedSubviews属性为 true
    clippingViewGroup.getClippingRect(sHelperRect);
    // Intersect the view with the parent's rectangle
    // This will result in the overlap with coordinates in the parent space
    // sHelperRect是一个 final staic 类型的 Rect 对象，它是 empty 的，所以值为(0,0,0,0)，instert()方法可以理解为截取相交，成功则返回ture，失败返回false
    if (!sHelperRect.intersect(view.getLeft(), view.getTop(), view.getRight(), view.getBottom())) {
      outputRect.setEmpty();// 相交截取失败，将输出 Rect 置为emopty
      return;
    }
    // Now we move the coordinates to the View's coordinate space
    sHelperRect.offset(-view.getLeft(), -view.getTop());
    sHelperRect.offset(view.getScrollX(), view.getScrollY());
    outputRect.set(sHelperRect);
    return;
      }
    }
    // 这里是 parent 不为空，而且parent没有实现 ReactClippingViewGroup接口的情况
    view.getDrawingRect(outputRect);
      }
    }
    

可能这里有点乱，我们还是回到示例代码的案例中，一个 ScrollView 包裹一个多个 View，这样的话，调用流程如下图：
//clip.png




这里主要取出滑动距离，然后对 ScrollView的显示区域和子 View的绘制区域进行计算，重新确定需要回收的子View和显示的子View。


## 展望 ##

之前提到 React Native for android 有期望使用 RecyclerView 的迹象，后来去源码里面看了一下，发现这个类的实现，确实是 RecyclerView 可是被加上了 @VisibleForTesting 的注解，如下图：

//png

在未来的版本中，fb完成这块代码之后，应该会开放使用，这样的话，我们就能避免去使用 ScrollView 这种在 大数据量下 性能尴尬的控件了。