---
layout: post
title: RecyclerView引发的内存泄露
---

### 背景说明

为了使问题更加清晰，我将出现问题的场景进行简化抽象。现在有一个`Activity`，其主体是一个`ListView`。`ListView`包含了多个模块，每个模块都对应着自己的视图。每个模块都实现了一个接口`Section`:

```java
public interface Section {
    public View getView(int position, View convertView, ViewGroup parent);
}
```

`ListView`的adapter的`getView`会调用各个`Section`的`getView`来获取不同模块的视图。

现在有一个模块`TestSection`对应的视图是一个横向的`RecyclerView`，核心代码如下：

```java
public class TestSection implements Section {
    RecyclerView mRecyclerView;
    public View getView(int position, View convertView, ViewGroup parent) {
        if (mRecyclerView == null) {
            mRecyclerView = new RecyclerView(parent.getContext());
            mRecyclerView.setLayoutManager(new LinearLayoutManager(parent.getContext(), LinearLayoutManager.HORIZONTAL, false));
        }
        return mRecyclerView;
    }
}
```
`ListView`支持下拉刷新。刷新之后，`ListView`会清除原有的所有`Section`，然后根据新的数据创建新的`Section`集合。而内存泄漏就在下拉刷新之后出现了！

<!-- more -->

LeakCanary给出的信息如下：

> * org.chromium.base.SystemMessageHandler.mLooper
* references android.os.Looper.mThread
* references thread java.lang.Thread.localValues (named 'main')
* references java.lang.ThreadLocal$Values.table
* references array java.lang.Object[].[31]
* references android.support.v7.widget.GapWorker.mRecyclerViews
* references java.util.ArrayList.array
* references array java.lang.Object[].[23]
* references android.support.v7.widget.RecyclerView.mContext
* references com.test.TestActivity

### 问题探究

LeakCanary给出的信息中有一个比较好的入手点，就是`android.support.v7.widget.GapWorker.mRecyclerViews`. 那就来看看这个`GapWorker`是何方神圣。

```java
final class GapWorker implements Runnable {

    static final ThreadLocal<GapWorker> sGapWorker = new ThreadLocal<>();

    ArrayList<RecyclerView> mRecyclerViews = new ArrayList<>();
    ...
}
```

`GapWorker`里有两个关键的成员`sGapWorker`和`mRecyclerViews`。根据LeakCanary的信息正是这个`mRecyclerViews`引用了`TestSection`中的`mRecyclerView`导致了内存泄露。注意到`sGapWorker`是`static`的，初步可以推断是这个静态的`sGapWorker`引用了一个`GapWorker`实例，而那个`GapWorker`实例中的`mRecyclerViews`又引用了`TestSection`中的`mRecyclerView`导致了内存泄露。接下来就要寻找`GapWorker`和`RecyclerView`的联系。关键代码如下：

```java
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild {
    ...
    GapWorker mGapWorker;
    ...
    private static final boolean ALLOW_THREAD_GAP_WORK = Build.VERSION.SDK_INT >= 21;
    ...
    
    @Override
    protected void onAttachedToWindow() {
        ...
 
        if (ALLOW_THREAD_GAP_WORK) {
            // Register with gap worker
            mGapWorker = GapWorker.sGapWorker.get();
            if (mGapWorker == null) {
                mGapWorker = new GapWorker();

                ...
                GapWorker.sGapWorker.set(mGapWorker);
            }
            mGapWorker.add(this);
        }
    }
    
    @Override
    protected void onDetachedFromWindow() {
        ...

        if (ALLOW_THREAD_GAP_WORK) {
            // Unregister with gap worker
            mGapWorker.remove(this);
            mGapWorker = null;
        }
    }
}
```

可以看到`RecyclerView`中有一个`GapWorker`类型的成员变量`mGapWorker`，这个`mGapWorker`实际上引用的是一个全局的`GapWorker`实例。在`onAttachedToWindow`中`RecyclerView`将自己加入到那个全局的`GapWorker`实例的`mRecyclerViews`列表里，而在`onDetachedFromWindow`中把自己从那个全局列表中移除。按理有`onAttachedToWindow`就会有`onDetachedFromWindow`，现在看来问题出现在`onDetachedFromWindow`没有被调用。

为了找到问题的真相，让我们回到现在的应用场景。`ListView`在下拉刷新之后会清除原有的所有`Section`，然后创建新的`Section`集合。这也就意味着一个新的`TestSection`实例被创建。再来看一下`TestSection`的`getView`的实现：

```java
public class TestSection implements Section {
    RecyclerView mRecyclerView;
    public View getView(int position, View convertView, ViewGroup parent) {
        if (mRecyclerView == null) {
            mRecyclerView = new RecyclerView(parent.getContext());
            mRecyclerView.setLayoutManager(new LinearLayoutManager(parent.getContext(), LinearLayoutManager.HORIZONTAL, false));
        }
        return mRecyclerView;
    }
}
```
当这个新的`TestSection`实例的`getView`第一次被调用时，`mRecyclerView`为`null`。由于`ListView`的复用机制，此时参数`convertView`并不为`null`，而实际上它引用了之前那个`TestSection`实例的`mRecyclerView`！于是现在出现了两个`RecyclerView`，我们将新的`mRecyclerView`称为*NewRV*，原先的`mRecyclerView`称为*OldRV*。在`getView`返回后，*NewRV*
成为了`ListView`的子view，它的`onDetachedFromWindow`会被正常调用。然而*OldRV*就成为了一个无人管的“野孩子”，没有谁会调用它的`onDetachedFromWindow`。于是它就静静地待在那个全局的`GapWorker`实例的`mRecyclerViews`列表里，很无辜地泄露了整个`Activity`！

### 解决方法

既然已经找到问题的真相，那解决方法也就明了了——正确地复用`convertView`即可。

```java
public class TestSection implements Section {
    RecyclerView mRecyclerView;
    public View getView(int position, View convertView, ViewGroup parent) {
        if (mRecyclerView == null) {
            if (convertView instanceof RecyclerView) {
                mRecyclerView = (RecyclerView) convertView;
            } else {
                mRecyclerView = new RecyclerView(parent.getContext());
                mRecyclerView.setLayoutManager(new LinearLayoutManager(parent.getContext(), LinearLayoutManager.HORIZONTAL, false));
            }
        }
        return mRecyclerView;
    }
}
```

以后在`ListView`中嵌套`RecyclerView`时真的要小心内存泄漏了!
