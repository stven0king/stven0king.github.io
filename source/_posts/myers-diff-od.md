---
title: Myers'Diff之线性空间细化
date: 2020-10-20 16:56:3
tags: [Diff]
categories: Arithmetic
description: "在学习完上一篇文章Myers'Diff之贪婪算法 之后，我对Android源码中的DiffUtil类进行了阅读发现其算法的实现和文章中的方式并不尽相同，而是在其基础之上再次进行的优化。所以本篇文章是以上一篇Myers'Diff之贪婪算法 文章内容基础之上对它的变体进行再次研究的过程。 "
---

# Myers'diff

![美图](https://img-blog.csdnimg.cn/20201012201130708.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)


## 前言
在学习完上一篇文章[Myers'Diff之贪婪算法](https://dandanlove.blog.csdn.net/article/details/108979400) 之后，我对`Android`源码中的`DiffUtil`类进行了阅读发现其算法的实现和文章中的方式并不尽相同，而是在其基础之上再次进行的优化。所以本篇文章是以上一篇[Myers'Diff之贪婪算法](https://dandanlove.blog.csdn.net/article/details/108979400) 文章内容基础之上对它的变体进行再次研究的过程。

上一篇文章[Myers'Diff之贪婪算法](https://dandanlove.blog.csdn.net/article/details/108979400) 讲述`diff`怎么从一个抽象的问题转化为数学问题，并对一些名词做了专有的定义（为解决问题的过程提供辅助），`Myers'Diff之贪婪算法`讲述了利用辅助的`k线`进行迭代求解，整改过程并不考虑时间和空间的消耗。所以这篇文章主要是在其基础之上进行时间和空间复杂度的优化。
## 逆向算法
`Myers'Diff之贪婪算反` 是从`(0,0)`到`(N,M)`进行移动的，它的反向工作是从`(N,M)`到`(0,0)`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201010143343944.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

从该图可以看出，该解决方案不同于通过向前工作而生成的解决方案，但是其`LCS`和`SES`的长度相同。 但这是完全正确的，因为通常可以有许多等效的解决方案，并且该算法只是选择找到的第一个解决方案。

#### Delta
> 因为序列`A`和`B`的长度可以不同，所以正向和反向算法的`k线`可以不同。 将此差异作为变量`delta = N-M`隔离是有用的。在示例中，`N = 7`和`M = 6`给出了`delta =1`。这是从`前k行`到`后k行`的偏移量。 您可以说正向路径以`k = 0`为中心，反向路径以`k = delta`为中心。

#### Middle Snake
可以对`D`的连续值同时运行`正向`和`反向`算法。在`D`的某个值处，两条路径将在`k线`上重叠。 本文证明这些路径之一是解决方案的一部分。 由于它将位于中间的某个地方，因此称为中间路径。

该示例的中间路径在此图中以粉红色显示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201010150112608.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

这很有用，因为它将问题分为两部分，然后可以分别递归解决。
这在空间上是线性的，因为只有最后的V向量必须存储，给出`O（D）`。对于时间，此线性算法仍为`O（（N + M）D）`。

这也有助于找到中间路径，其D必须是正向和反向算法D的一半。这意味着随着D的增加，所需时间接近基本算法的一半。

伪代码：
```c
for d = 0 to ( N + M + 1 ) / 2
{
  for k = -d to d step 2
  {
    calculate the furthest reaching forward and reverse paths
    if there is an overlap, we have a middle snake
  }
}
```

#### Odd and Even Deltas
每个差异`水平删除`或`垂直插入`都是从`k行`移到其相邻行。由于增量是正向和反向算法中心之间的差异，因此我们知道需要检查中间路径的`d`值。

对于奇数增量，我们必须寻找差异为`d`的前向路径与差异为`d-1`的反向路径重叠。
下图显示，对于`delta = 3`，当`正向d`为``2而`反向d`为`1`时发生重叠：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201010150752794.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

类似地，对于偶数增量，当正向和反向路径的差异数相同时，就会出现重叠。
下图显示，对于`delta = 2`，当`正向`和`反向` 的 `d`均为`2`时，发生重叠：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201010150901753.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

因此，这是查找中间路径的完整伪代码：

```c
delta = N - M
for d = 0 to ( N + M + 1 ) / 2
{
  for k = -d to d step 2
  {
    calculate the furthest reaching forward path on line k
    if delta is odd and ( k >= delta - ( d - 1 ) and k <= delta + ( d - 1 ) )
      if overlap with reverse[ d - 1 ] on line k
        => found middle snake and SES of length 2D - 1
  }
  
  for k = -d to d step 2
  {
    calculate the furthest reaching reverse path on line k
    if delta is even and ( k >= -d - delta and k <= d - delta )
      if overlap with forward[ d ] on line k
        => found middle snake and SES of length 2D
  }
}
```

- `(N+M+1) / 2 ` 从两端同时出发，意味着外循环次数**大于等于**最长路径的二分之一；
- 如果`delta` 是偶数那么中间路径在向前的方向中出现；
- 如果`delta` 是偶数那么中间路径在向后的方向中出现；

### 递归解决

我们需要以递归方法包装中间路径算法。基本上，我们需要找到一条中间的路径，然后求解保留在**左上角**和**右下角**的矩形。

伪代码：

```c
Compare( A, N, B, M )
{
  if ( M == 0 && N > 0 ) add N deletions to SES
  if ( N == 0 && M > 0 ) add M insertions to SES
  if ( N == 0 || M == 0 ) return  
  calculate middle snake
  suppose it is from ( x, y ) to ( u, v ) with total differences D
  if ( D > 1 )
  {
    Compare( A[ 1 .. x ], x, B[ 1 .. y ], y ) // top left
    Add middle snake to results
    Compare( A[ u + 1 .. N ], N - u, B[ v + 1 .. M ], M - v ) // bottom right
  }
  else if ( D == 1 ) // must be forward snake
  {
    Add d = 0 diagonal to results
    Add middle snake to results
  }
  else if ( D == 0 ) // must be reverse snake
  {
    Add middle snake to results
  }
}
```

我将在稍后解释几个边缘情况。


#### Edge case
上面的伪代码需要考虑两个边界case，`d=0`和`d=1`。

如果中间路径算法找到`D = 0`的解，则两个序列相同。这意味着增量为零，即为偶数。因此，中间路径是一条正好匹配（对角线）的反向路径。因此，我们要做的就是将这条路径添加到结果中。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201010155143841.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

如果中间的路径算法找到`D = 1`的解，那么就存在一个插入或删除。这意味着`delta`是`1`或`-1`，这是奇数，因此中间的路径是前向路径。 对于这种情况，我们可以通过计算`d = 0`对角线并将其与中间路径一起添加到结果中来完成解决方案。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201010155245225.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

## 总结对比
这次的优化还是以递归方法进行的，与[Myers'Diff之贪婪算法](https://dandanlove.blog.csdn.net/article/details/108979400) 的递归不同的是。这次的递归我们需要找到一条中间的路径，然后进行**左上角**和**右下角**的矩形拆分，将拆分之后的矩形再进行递归。`O((M+N)lg(􏰎􏰏M+N)+D2)` 为最坏的情况。时间约为 `O(N + M)D)` 其`D`必须是**正向和反向算法** `D`的一半。这意味着随着`D`的增加，所需时间接近基本算法的一半。

算法实践：[DiffUtil和它的差量算法](https://dandanlove.blog.csdn.net/article/details/109123553)

参考链接：
[代码:diff-match-patch](https://github.com/google/diff-match-patch)
[diff2论文](http://xmailserver.org/diff2.pdf)
[Myers diff alogrithm:part 1](https://www.codeproject.com/Articles/42279/Investigating-Myers-diff-algorithm-Part-1-of-2)
[Myers diff alogrithm:part 2](https://www.codeproject.com/Articles/42280/Investigating-Myers-Diff-Algorithm-Part-2-of-2)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我的公共号：

![振兴书城](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktNjEyYzRjNjZkNDBjZTg1NS5qcGc?x-oss-process=image/format,png#pic_center)