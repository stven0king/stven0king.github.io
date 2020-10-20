---
title: DiffUtil和它的差量算法
date: 2020-10-20 16:57:35
tags: [Diff]
categories: Arithmetic
description: "学习Myers'Diff 算法是从 DiffUtils 源代码开始的，但DiffUtil和它的差量算法这篇却是文章是在写完 Myers‘Diff之贪婪算法 和 Myers‘Diff之线性空间细化 这两篇算法文章之后着手的。比较先需要学会算法才能理解代码实现并更好的进行使用。 "
---

# DiffUtil和它的差量算法

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201016193724905.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

## 前言
学习`Myers'Diff` 算法是从 `DiffUtils` 源代码开始的，但`DiffUtil和它的差量算法`这篇却是文章是在写完 [Myers‘Diff之贪婪算法](https://dandanlove.blog.csdn.net/article/details/108979400) 和 [Myers‘Diff之线性空间细化](https://dandanlove.blog.csdn.net/article/details/108996040) 这两篇算法文章之后着手的。比较先需要学会算法才能理解代码实现并更好的进行使用。
## DiffUtil介绍

在正式分析`DiffUtil`之前，我们先来对`DiffUtil`有一个大概的了解--`DiffUtil`到底是什么东西。

> 类的路径：`androidx.recyclerview.widget.DiffUtil.java`

大家在开发关于列表页面的时候可能会遇到下面的情况：
> 在一次操作里面可能会同时出现`remove`、`add`、`change`三种操作。像这种情况，我们不能调用`notifyItemRemoved`、`notifyItemInserted`或者`notifyItemChanged`方法。为了视图立即刷新，我们只能通过调用`notifyDataSetChanged`方法来实现。但`notifyDataSetChanged` 刷新是全部刷新没有动画效果。

那么有一种能通过对比知道两个列表的数据的差异，然后进行`remove`、`add`或`change`么？

**Google**提供的`DiffUtil`是一个实用程序类，它计算两个列表之间的差异，并输出将第一个列表转换为第二个列表的更新操作列表。

## DiffUtil.DiffResult
> DiffUtil在计算两个列表之间的差异时使用的Callback类。
```java
public abstract static class Callback {
	//旧数据集的长度；
    public abstract int getOldListSize();
    //新数据集的长度
    public abstract int getNewListSize();
    //判断是否是同一个item；
    public abstract boolean areItemsTheSame(int oldItemPosition, int newItemPosition);
    //如果item相同，此方法用于判断是否同一个 Item 的内容也相同
    public abstract boolean areContentsTheSame(int oldItemPosition, int newItemPosition);
    @Nullable
    //如果item相同，内容不同，用 payLoad 记录这个 ViewHolder 中，具体需要更新那个View
    public Object getChangePayload(int oldItemPosition, int newItemPosition){
        return null;
    }
}
```
## DiffUtil.DiffResult
此类包含有关`DiffUtil＃calculateDiff`调用的结果的信息。可以通过`dispatchUpdatesTo`使用`DiffResult`中的更新，也可以通过`dispatchUpdatesTo`直接将结果流式传输到`RecyclerView.Adapter`。

## DiffUtil使用

```java
public class RecyclerItemCallback extends DiffUtil.Callback {
    private List<Bean> mOldDataList;
    private List<Bean> mNewDataList;
    public RecyclerItemCallback(List<Bean> oldDataList, List<Bean> newDataList) {
        this.mOldDataList = oldDataList;
        this.mNewDataList = newDataList;
    }
    @Override
    public int getOldListSize() {
        return mOldDataList.size();
    }
    @Override
    public int getNewListSize() {
        return mNewDataList.size();
    }
    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        return Objects.equals(mNewDataList.get(newItemPosition).getId(), mOldDataList.get(oldItemPosition).getId());
    }
    @Override
    public boolean areContentsTheSame(int i, int i1) {
        return Objects.equals(mOldDataList.get(i).getContent(), mNewDataList.get(i1).getContent());
    }
}
private void refreshData(List<Bean> oldDataList,List<Bean> newDataList) {
    RecyclerItemCallback recyclerItemCallback = new RecyclerItemCallback(oldDataList, newDataList);
    DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(recyclerItemCallback, false);
    diffResult.dispatchUpdatesTo(mRecyclerAdapter);
}
```

## DiffUtil中Myers算法代码

> 再次提醒一下代码阅读需要先了解 [Myers‘Diff之贪婪算法](https://dandanlove.blog.csdn.net/article/details/108979400) 和 [Myers‘Diff之线性空间细化](https://dandanlove.blog.csdn.net/article/details/108996040)  这两篇文章中的算法知识。

```java
public class DiffUtil {
    //部分代码省略
    @NonNull
    public static DiffResult calculateDiff(@NonNull Callback cb, boolean detectMoves) {
        final int oldSize = cb.getOldListSize();
        final int newSize = cb.getNewListSize();

        final List<Snake> snakes = new ArrayList<>();

        // instead of a recursive implementation, we keep our own stack to avoid potential stack
        // overflow exceptions
        final List<Range> stack = new ArrayList<>();

        stack.add(new Range(0, oldSize, 0, newSize));

        final int max = oldSize + newSize + Math.abs(oldSize - newSize);
        // allocate forward and backward k-lines. K lines are diagonal lines in the matrix. (see the
        // paper for details)
        // These arrays lines keep the max reachable position for each k-line.
        final int[] forward = new int[max * 2];
        final int[] backward = new int[max * 2];

        // We pool the ranges to avoid allocations for each recursive call.
        final List<Range> rangePool = new ArrayList<>();
        while (!stack.isEmpty()) {
            final Range range = stack.remove(stack.size() - 1);
            final Snake snake = diffPartial(cb, range.oldListStart, range.oldListEnd,
                    range.newListStart, range.newListEnd, forward, backward, max);
            if (snake != null) {
                if (snake.size > 0) {
                    snakes.add(snake);
                }
                // offset the snake to convert its coordinates from the Range's area to global
                //使路径点的偏移以将其坐标从范围区域转换为全局
                snake.x += range.oldListStart;
                snake.y += range.newListStart;
                //拆分左上角和右下角进行递归
                // add new ranges for left and right
                final Range left = rangePool.isEmpty() ? new Range() : rangePool.remove(
                        rangePool.size() - 1);
                //起点为上一次的起点
                left.oldListStart = range.oldListStart;
                left.newListStart = range.newListStart;
                //如果是逆向得到的中间路径，那么左上角的终点为该中间路径的起点
                if (snake.reverse) {
                    left.oldListEnd = snake.x;
                    left.newListEnd = snake.y;
                } else {
                    if (snake.removal) {//中间路径是向右操作，那么终点的x需要退一
                        left.oldListEnd = snake.x - 1;
                        left.newListEnd = snake.y;
                    } else {//中间路径是向下操作，那么终点的y需要退一
                        left.oldListEnd = snake.x;
                        left.newListEnd = snake.y - 1;
                    }
                }
                stack.add(left);
                // re-use range for right
                //noinspection UnnecessaryLocalVariable
                final Range right = range;//右下角终点和之前的终点相同
                if (snake.reverse) {
                    if (snake.removal) {//中间路径是向右操作，那么起点的x需要进一
                        right.oldListStart = snake.x + snake.size + 1;
                        right.newListStart = snake.y + snake.size;
                    } else {//中间路径是向下操作，那么起点的y需要进一
                        right.oldListStart = snake.x + snake.size;
                        right.newListStart = snake.y + snake.size + 1;
                    }
                } else {//如果是逆向得到的中间路径，那么右下角的起点为该中间路径的终点
                    right.oldListStart = snake.x + snake.size;
                    right.newListStart = snake.y + snake.size;
                }
                stack.add(right);
            } else {
                rangePool.add(range);
            }

        }
        // sort snakes
        Collections.sort(snakes, SNAKE_COMPARATOR);

        return new DiffResult(cb, snakes, forward, backward, detectMoves);

    }
    //diffPartial方法主要是来寻找一条snake，它的核心也就是Myers算法。
    private static Snake diffPartial(Callback cb, int startOld, int endOld,
            int startNew, int endNew, int[] forward, int[] backward, int kOffset) {
        final int oldSize = endOld - startOld;
        final int newSize = endNew - startNew;
        if (endOld - startOld < 1 || endNew - startNew < 1) {
            return null;
        }
        //差异增量
        final int delta = oldSize - newSize;
        //最双向最长路径
        final int dLimit = (oldSize + newSize + 1) / 2;
        //进行初始化设置
        Arrays.fill(forward, kOffset - dLimit - 1, kOffset + dLimit + 1, 0);
        Arrays.fill(backward, kOffset - dLimit - 1 + delta, kOffset + dLimit + 1 + delta, oldSize);
        /**
         * 差异量为奇数
         * 每个差异-水平删除或垂直插入-都是从一千行移到其相邻行。
         * 由于增量是正向和反向算法中心之间的差异，因此我们知道需要检查中间snack的d值。
         * 对于奇数增量，我们必须寻找差异为d的前向路径与差异为d-1的反向路径重叠。
         * 类似地，对于偶数增量，重叠将是当正向和反向路径具有相同数量的差异时
         */
        final boolean checkInFwd = delta % 2 != 0;
        for (int d = 0; d <= dLimit; d++) {
            /**
             * 这一循环是从(0,0)出发找到移动d步能达到的最远点
             * 引理：d和k同奇同偶，所以每次k都递增2
             */
            for (int k = -d; k <= d; k += 2) {
                // find forward path
                // we can reach k from k - 1 or k + 1. Check which one is further in the graph
                //找到前进路径
                //我们可以从k-1或k + 1到达k。检查图中的哪个更远 
                int x;
                final boolean removal;//向下
                //bool down = ( k == -d || ( k != d && V[ k - 1 ] < V[ k + 1 ] ) );
                if (k == -d || (k != d && forward[kOffset + k - 1] < forward[kOffset + k + 1])) {
                    x = forward[kOffset + k + 1];
                    removal = false;
                } else {
                    x = forward[kOffset + k - 1] + 1;
                    removal = true;
                }
                // set y based on x
                //k = x - y
                int y = x - k;
                // move diagonal as long as items match
                //只要item匹配就移动对角线
                while (x < oldSize && y < newSize
                        && cb.areItemsTheSame(startOld + x, startNew + y)) {
                    x++;
                    y++;
                }
                forward[kOffset + k] = x;
                //如果delta为奇数，那么相连通的节点一定是向前移动的节点，也就是执行forward操作所触发的节点
                //if delta is odd and ( k >= delta - ( d - 1 ) and k <= delta + ( d - 1 ) )
                if (checkInFwd && k >= delta - d + 1 && k <= delta + d - 1) {
                    //if overlap with reverse[ d - 1 ] on line k
                    //forward'x >= backward'x，如果在k线上正向查找能到到的位置的x坐标比反向查找达到的y坐标小
                
                    if (forward[kOffset + k] >= backward[kOffset + k]) {
                        Snake outSnake = new Snake();
                        outSnake.x = backward[kOffset + k];
                        outSnake.y = outSnake.x - k;
                        outSnake.size = forward[kOffset + k] - backward[kOffset + k];
                        outSnake.removal = removal;
                        outSnake.reverse = false;
                        return outSnake;
                    }
                }
            }
            /**
             * 这一循环是从(m,n)出发找到移动d步能达到的最远点
             */
            for (int k = -d; k <= d; k += 2) {
                // find reverse path at k + delta, in reverse
                //以k + delta,找到反向路径。backwardK相当于反向转化之后的正向的k
                final int backwardK = k + delta;
                int x;
                final boolean removal;
                //与k线类似
                //bool down = ( k == -d || ( k != d && V[ k - 1 ] < V[ k + 1 ] ) );
                if (backwardK == d + delta || (backwardK != -d + delta 
                        && backward[kOffset + backwardK - 1] < backward[kOffset + backwardK + 1])) {
                    x = backward[kOffset + backwardK - 1];
                    removal = false;
                } else {
                    x = backward[kOffset + backwardK + 1] - 1;
                    removal = true;
                }
                // set y based on x
                int y = x - backwardK;
                // move diagonal as long as items match
                //只要item匹配就移动对角线
                while (x > 0 && y > 0
                        && cb.areItemsTheSame(startOld + x - 1, startNew + y - 1)) {
                    x--;
                    y--;
                }
                backward[kOffset + backwardK] = x;
                //如果delta为偶数，那么相连通的节点一定是反向移动的节点，也就是执行backward操作所触发的节点
                //if delta is even and ( k >= -d - delta and k <= d - delta )
                if (!checkInFwd && k + delta >= -d && k + delta <= d) {
                    //if overlap with forward[ d ] on line k
                    //forward'x >= backward'x，判断正向反向是否连通了
                    if (forward[kOffset + backwardK] >= backward[kOffset + backwardK]) {
                        Snake outSnake = new Snake();
                        outSnake.x = backward[kOffset + backwardK];
                        outSnake.y = outSnake.x - backwardK;
                        outSnake.size =
                                forward[kOffset + backwardK] - backward[kOffset + backwardK];
                        outSnake.removal = removal;
                        outSnake.reverse = true;
                        return outSnake;
                    }
                }
            }
        }
        throw new IllegalStateException("DiffUtil hit an unexpected case while trying to calculate"
                + " the optimal path. Please make sure your data is not changing during the"
                + " diff calculation.");
    }
    //部分代码省略
}
```

参考链接：
[代码:diff-match-patch](https://github.com/google/diff-match-patch)
[diff2论文](http://xmailserver.org/diff2.pdf)
[Myers diff alogrithm:part 1](https://www.codeproject.com/Articles/42279/Investigating-Myers-diff-algorithm-Part-1-of-2)
[Myers diff alogrithm:part 2](https://www.codeproject.com/Articles/42280/Investigating-Myers-Diff-Algorithm-Part-2-of-2)


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我的公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>