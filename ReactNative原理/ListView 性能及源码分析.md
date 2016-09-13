# ListView 性能及源码分析 #

RN 中的 ListView 就是基于 ScrollView 的，但是有一些优化。这里简要介绍一些 ListView 的原理。

首先去到 ListView.js 源码看一下，去到 render()函数的 return 语句

    return cloneReferencedElement(renderScrollComponent(props), {
      ref: this._setScrollComponentRef,
      onContentSizeChange: this._onContentSizeChange,
      onLayout: this._onLayout,
    }, header, bodyComponents, footer);

其中使用的 cloneReferencedElement()函数作用是传递 ref 属性到实际的 ScrollView 控件中，从而使当前 Component 能够调用相应的方法。去到 getDefaultProps()函数，这个函数是 ES5写法，初始化 props函数，这里有行代 renderScrollComponent 属性是一个S从容哦了View，使用 jsx 语法把当前属性传递给 ScrollView，然后在上面的 render 的 renturn 语句中，返回的也是这个 renderScrollComponent 属性，所以到这里，基本就能看出，ListView 的实现就是 ScrollView 来的。

      renderScrollComponent: props => <ScrollView {...props} />,


但是 ListView 在显示列表内容的时候，会根据滑动距离，逐步向 ScrollView 中添加子组件（通过调用 renderRow() 方法）。注意到 ListView 有 initialListSize 属性，表示第一次加载的时候添加多少个子项，默认是 10，还有 pageSize 属性，表示每次需要添加的时候，增加多少个子项，默认是 1。

通过上面的分析我们可以看到，ListView 在第一次加载的时候，不论你的列表有多大，默认最多加载 initialListSize 个子项，所以能保证启动速度，如果还没有充满，或者在向下滑动过程中，再组件添加子项。这样的操作似乎比较合理，但是注意到，整个操作中，会逐渐向 ListView 中添加子项，新出现的子项，都是通过创建新的 View，而完全没有复用的过程。所以，如果在应用中，ListView 中的子项数量特别多，ListView 往下滑动的过程中，内存会逐渐上涨的。

值得一提的是，ListView 提供了 renderScrollComponent，可以使用其他 Scroll 组件来替换 ScrollView，并且 RecyclerViewBackedScrollView 组件来作为备选。看到这个名字我很欣喜，说明它支持子项的回收复用（Recycler）。首先，看到 iOS 的实现 RecyclerViewBackedScrollView.ios.js，其实它就是 ScrollView，并没有实现所谓的复用，失望了一半。继续看 Android 的实现，它实际上是对应 Native 的 com.facebook.react.views.recyclerview.AndroidRecyclerViewBackedScrollView，它继承与 Android 的 RecyclerView。看到这里，如果使用这种方法，我直观感觉 RN 的 ListView 性能在 Android 上表现应该会比 iOS 好。

我们继续来看它是怎么实现回收复用的，AndroidRecyclerViewBackedScrollView 内部实现了一个 RecyclerView.Adapter，如下：

    static class ReactListAdapter extends Adapter<ConcreteViewHolder> {
    
      private final List<View> mViews = new ArrayList<>();
    
      public void addView(View child, int index) {
    mViews.add(index, child);
    ...
      }
    
      public void removeViewAt(int index) {
    View child = mViews.get(index);
    if (child != null) {
      mViews.remove(index);
      ...
    }
      }
    
      @Override
      public ConcreteViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    return new ConcreteViewHolder(new RecyclableWrapperViewGroup(parent.getContext()));
      }
    
      @Override
      public void onBindViewHolder(ConcreteViewHolder holder, int position) {
    RecyclableWrapperViewGroup vg = (RecyclableWrapperViewGroup) holder.itemView;
    View row = mViews.get(position);
    if (row.getParent() != vg) {
      vg.addView(row, 0);
    }
      }
    
      @Override
      public void onViewRecycled(ConcreteViewHolder holder) {
    super.onViewRecycled(holder);
    ((RecyclableWrapperViewGroup) holder.itemView).removeAllViews();
      }  
    }


注意到这里有一个 mViews，用来保存所有的子 View，绑定 View 的时候只是简单用一个空的 View（RecyclableWrapperViewGroup）包了一下。这样一来，ListView 完全没有什么起到复用的作用呀！测试一下，确实也是这样，性能问题还是很严重。