---
title: ViewPager2+Fragment操作笔记
date: 2021-5-14 10:17:28
tags: [ViewPager2,Fragment]
categories: Android
description: "这篇文章本来想叫ViewPager2+Fragment+TabLayout，你躺过这些坑么？"
---

# ViewPager2+Fragment操作笔记

![心情好](https://img-blog.csdnimg.cn/img_convert/883382632cde26ae934f5a628e278ead.png#pic_center)

## ViewPager2简介

[ViewPager2官网介绍](https://developer.android.com/jetpack/androidx/releases/viewpager2)

[ViewPager2官网Samples](https://github.com/android/views-widgets-samples/tree/main/ViewPager2)

距离`ViewPager2`正式版的发布已经一年多了，目前`ViewPager`早已停止更新，官方鼓励使用ViewPager2替代。 `ViewPager2`底层基于`RecyclerView`实现，因此可以获得`RecyclerView`带来的诸多收益：

- 抛弃传统的`PagerAdapter`，统一了`Adapter`的`API`；
- 横向、竖向布局都可以实现自由滑动；
- 支持`DiffUitl`，可以实现局部刷新；
- 支持`RTL`（right-to-left），对于一些有出海需求的APP非常有用；
- 支持`ItemDecorator`，搭配`PageTransformer`实现炫酷的跳转动画；

`ViewPager2`更多的是配合`Fragment`的使用，这需要借助于`FragmentStateAdapter`。

他们偶尔会搭配`TabLayout`一起使用，相关代码直接阅读或者运行 [ViewPager2官网Samples](https://github.com/android/views-widgets-samples/tree/main/ViewPager2) 即可，这里不做重复的讲解。

下面主要讲一下在使用过程中遇到的问题~！

## 实际操作效果


上滑吸顶+标题页面左右滑动+横滑和竖滑列表+标题页面数据和数量更新

> 上滑吸顶
`CoordinatorLayout`+`AppBarLayout`+`CollapsingToolbarLayout`
> 左右滑动
`ViewPager2`+`TabLayout`+`Fragment`
> 横滑和竖滑列表
`RecycleView`+`NestedScrollableHost`
> 标题页面数据和数量
`TabLayoutMediator`+声明周期检测+缓存优化

![viewpage2_tabLayout_nestedscroll.gif](https://img-blog.csdnimg.cn/img_convert/d092f1350385f7da3a65aa5c841452b5.gif#pic_center)



## RecycleView和Viewpage2的滑动冲突

```kotlin
/**
 * Created by Tanzhenxing
 * Date: 2021/4/7 7:04 下午
 * Description:解决 [RecyclerView] 嵌套到 [androidx.viewpager2.widget.ViewPager2] 左右滑动冲突
 * 目前只解决了左右滑动冲突
 */
class RecyclerViewAtViewPager2 : RecyclerView {
    constructor(context: Context) : this(context, null)
    constructor(context: Context, attrs: AttributeSet?) : this(context, attrs, 0)
    constructor(context: Context, attrs: AttributeSet?, defStyleAttr: Int) : super(context, attrs, defStyleAttr)

    var x1 = 0f
    var x2 = 0f
    override fun dispatchTouchEvent(event: MotionEvent?): Boolean {
        if(event!!.action == MotionEvent.ACTION_DOWN) {
            x1 = event.x
        } else if(event.action == MotionEvent.ACTION_MOVE) {
            x2 = event.x
        } else if (event.action == MotionEvent.ACTION_CANCEL
            || event.action == MotionEvent.ACTION_UP) {
            x2 = 0f
            x1 = 0f
        }
        val xOffset= x2-x1
        if (layoutManager is LinearLayoutManager) {
            val linearLayoutManager = layoutManager as LinearLayoutManager
            if (linearLayoutManager.orientation == HORIZONTAL) {
                if ((xOffset <= 0 && canScrollHorizontally(1))
                    || (xOffset >= 0 && canScrollHorizontally(-1))) {
                    this.parent?.requestDisallowInterceptTouchEvent(true)
                } else {
                    this.parent?.requestDisallowInterceptTouchEvent(false)
                }
            } else {
                // TODO: 2021/4/8 目前没有实现上下滑动和 [androidx.viewpager2.widget.ViewPager2]上下滑动的冲突
            }
        } else {
            handleDefaultScroll()
        }

        return super.dispatchTouchEvent(event)
    }

    fun handleDefaultScroll() {
        val canScrollHorizontally = canScrollHorizontally(-1) || canScrollHorizontally(1)
        val canScrollVertically = canScrollVertically(-1) || canScrollVertically(1)
        if (canScrollHorizontally || canScrollVertically) {
            this.parent?.requestDisallowInterceptTouchEvent(true)
        } else {
            this.parent?.requestDisallowInterceptTouchEvent(false)
        }
    }
}
```

## ViewPager2中Fragment的懒加载

### 懒加载

一般我们使用`Fragment`对页面进行数据懒加载的时候都是通过`onHiddenChanged`方法判断显示和隐藏，在**第一次展现**出来的时候再进行接口调用。

```java
 @Override
public final void onHiddenChanged(boolean hidden) {
  super.onHiddenChanged(hidden);
  if (!hidden) {
    onUserVisible();
  } else {
    onUserGone();
  }
}
```

但在`ViewPager2`中，`Fragment`的`setUserVisibleHint`和`onHiddenChanged`方法都是不执行的。

- `ViewPager`展现第一个页面，然后切后台的日志：

```java
04-17 16:45:10.992 D/tanzhenxing:11(22006): onCreate
04-17 16:45:10.992 D/tanzhenxing:11(22006): onCreateView:
04-17 16:45:11.004 D/tanzhenxing:11(22006): onActivityCreated
04-17 16:45:11.004 D/tanzhenxing:11(22006): onViewStateRestored: 184310198
04-17 16:45:11.004 D/tanzhenxing:11(22006): onStart
04-17 16:45:11.004 D/tanzhenxing:11(22006): onResume
04-17 16:45:18.739 D/tanzhenxing:11(22006): onPause
04-17 16:45:18.779 D/tanzhenxing:11(22006): onStop
```

然后在切回前台的日志：

```java
04-17 16:53:40.749 D/tanzhenxing:11(22006): onStart
04-17 16:53:40.752 D/tanzhenxing:11(22006): onResume
```

- `ViewPager`展现第一个页面，然后手动滑动到第二个页面日志：

```java
04-17 16:54:44.168 D/tanzhenxing:12(22006): onCreate
04-17 16:54:44.168 D/tanzhenxing:12(22006): onCreateView:
04-17 16:54:44.178 D/tanzhenxing:12(22006): onActivityCreated
04-17 16:54:44.178 D/tanzhenxing:12(22006): onViewStateRestored: 47009644
04-17 16:54:44.178 D/tanzhenxing:12(22006): onStart
04-17 16:54:44.553 D/tanzhenxing:11(22006): onPause
04-17 16:54:44.554 D/tanzhenxing:12(22006): onResume
```

这样看起来我们是可以利用`Fragment`声明周期中的`onStart`和`onResume`方法做懒加载。

### 预加载

只要讲数据请求写在 `onCreateView` 或者`onStart`中就能进行接口的离屏请求。



## FragmentStateAdapter

`ViewPager2`继承自`RecyclerView`，大概率`FragmentStateAdapter`继承自`RecyclerView.Adapter`：

```kotlin
public abstract class FragmentStateAdapter extends 
  RecyclerView.Adapter<FragmentViewHolder> implements StatefulAdapter {
}
```

### onCreateViewHolder

`onCreateViewHolder`是`RecycleVeiw`用于创建`ViewHolder`的方法：

```java
@NonNull
@Override
public final FragmentViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType){
  return FragmentViewHolder.create(parent);
}
```

`FragmentViewHolder`的主要作用是通过`FrameLayout`为`Fragment`提供用作容器的`container`：

```java
@NonNull
static FragmentViewHolder create(@NonNull
ViewGroup parent) {
    FrameLayout container = new FrameLayout(parent.getContext());
    container.setLayoutParams(new ViewGroup.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.MATCH_PARENT));
    container.setId(ViewCompat.generateViewId());
    container.setSaveEnabled(false);
    return new FragmentViewHolder(container);
}
```

### onBindViewHolder

`onBindViewHolder`是`RecycleVeiw`用于数据绑定的方法：

```java
@Override
public final void onBindViewHolder(final @NonNull FragmentViewHolder holder, int position) {
    /**部分代码省略 */
	  ensureFragment(position);
    /**部分代码省略 */
  	gcFragments();
}
```

`ensureFragment(position)`，其内部会最终回调用`createFragment`创建当前`Fragment`。

```java
private void ensureFragment(int position) {
  long itemId = getItemId(position);
  if (!mFragments.containsKey(itemId)) {
    // TODO(133419201): check if a Fragment provided here is a new Fragment
    Fragment newFragment = createFragment(position);
    newFragment.setInitialSavedState(mSavedStates.get(itemId));
    mFragments.put(itemId, newFragment);
  }
}
```

`mFragments`缓存创建的`Fragment`，供后面`placeFramentInViewholder`使用; `gcFragments`回收已经不再使用的的`Fragment`（**对应的item已经删除**），节省内存开销。

### onViewAttachedToWindow

`onViewAttachedToWindow`是`ViewHolder`出现在页面中回调。

```
@Override
public final void onViewAttachedToWindow(@NonNull final FragmentViewHolder holder) {
    //将FragmentViewHolder的container与当前Fragment绑定
	  placeFragmentInViewHolder(holder);
  	gcFragments();
}
```


### FragmentStateAdapter使用

- `Fragment`对象容器;
- 生产`fragment`识别的`id`；

```kotlin
class MyFragmentStateAdapter(val data: List<Int>, fragment: Fragment) : FragmentStateAdapter(fragment){
    private val fragments = mutableMapOf<Int, Fragment>()
    override fun createFragment(position: Int): Fragment {
        val value = data[position]
        val fragment = fragments[value]
        if (fragment != null) {
            return fragment
        }
        val cardFragment =
            NestedAllRecyclerViewFragment.create(value)
        fragments[value] = cardFragment
        return cardFragment
    }

    /**
     * 根据数据生成唯一id
     *
     * 如果不重写，那么在调用[notifyDataSetChanged]更新的时候
     *
     * 会抛出```new IllegalStateException("Fragment already added")```异常
     */
    override fun getItemId(position: Int): Long {
        return data[position].toLong()
    }

    /**
     * 用来判断当前id对应的fragment是否添加过
     */
    override fun containsItem(itemId: Long): Boolean {
        data.forEach {
            if (it.toLong() == itemId) {
                return true
            }
        }
        return false
    }
    override fun getItemCount(): Int {
        return data.size
    }
}
```

在进行

### 获取Fragment实例

```kotlin
fun getCurrentFragment(position: Int): Fragment? =
        fragment.childFragmentManager.findFragmentByTag("f$position")
```

源码分析：

```java
public abstract class FragmentStateAdapter extends RecyclerView.Adapter<FragmentViewHolder>
    implements StatefulAdapter {
    /**部分代码省略 */
    void placeFragmentInViewHolder(@NonNull
    final FragmentViewHolder holder) {
        /**部分代码省略 */
        if (!shouldDelayFragmentTransactions()) {
            scheduleViewAttach(fragment, container);
            mFragmentManager.beginTransaction()
                            .add(fragment, "f" + holder.getItemId())
                            .setMaxLifecycle(fragment, STARTED).commitNow();
            mFragmentMaxLifecycleEnforcer.updateFragmentMaxLifecycle(false);
        }

        /**部分代码省略 */
    }

    /**部分代码省略 */
    @Override
    public long getItemId(int position) {
        return position;
    }
}
```

给`Fragment`添加的`TAG`是`"f" + holder.getItemId()`。

### 异常处理

- 初始化时遇到的崩溃；

```java
Fragment HomeFragment{b793d14 (e67290fe-7ab1-4b2b-b98c-4e08d146644c)} has not been attached yet.
com.xxx.xxx.xxx.adapter.HomeFragmentStateAdapter.<init>(SourceFile:29)
```

在开发过程中遇到问题，需要在构造`FragmentStateAdapter`的时候对`Fragment`的状态做判断`isAdded()`。

- 更新数据的时候遇到的崩溃：

```
Fragment already added
```

重写`getItemId`方法，该方法返回的值与数据有关而不是与数据在列表中的索引有关。因为它代表着`Fragment`的唯一性，是否可以复用。

## ViewPager2滑动监听

```java
public abstract static class OnPageChangeCallback {
    //当前页面开始滑动时
    public void onPageScrolled(int position, float positionOffset,@Px int positionOffsetPixels) {
    }
    //当页面被选中时
    public void onPageSelected(int position) {
    }
    //当前页面滑动状态变动时
    public void onPageScrollStateChanged(@ScrollState int state) {
    }
}
```

实现：

```kotlin
var pageChangeCallback = object : ViewPager2.OnPageChangeCallback() {
  override fun onPageSelected(position: Int) {
    //需要注意的是postion需要做大于0的判断
  }
}
```

## TabLayout+TabLayoutMediator

方便实现`TAB`和`ViewPager`滑动或跳转的关联。

```java
implementation 'com.google.android.material:material:1.2.0'
```

建议`material`的版本号大约`1.0.0`，否则实现`TAB`的自定义布局宽度展现些问题。

使用：[ViewPager2官网Samples](https://github.com/android/views-widgets-samples/tree/main/ViewPager2)

## DiffUtil 局部更新

[DiffUtil和它的差量算法](https://ishare.58corp.com/articleDetail?id=41897)

## 总结

本文主要介绍了`ViewPager2`配合`Fragment`的使用方法以及在使用过程中需要注意的问题，顺带提到了**TabLayout**、**OnPageChangeCallback**、**DiffUtil**等。