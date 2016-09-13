

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
    // 这里是 parent 不为空，而且parent没有实现 ReactClippingViewGroup接口的情况
    view.getDrawingRect(outputRect);
  }
}


可能这里有点乱，我们还是回到示例代码的案例中，一个 ScrollView 包裹一个多个 View，这样的话，调用流程如下图：