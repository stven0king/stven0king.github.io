---
title: Myers‘Diff之贪婪算法
date: 2020-10-20 16:55:35
tags: [Diff]
categories: Arithmetic
description: "最初不知道是什么时候发现 DiffUtil 对比列表 item 数据进行局部刷新，git 文件对比都用到了这个算法。上个月刚好再一次看到了就想深入了解一下。但发现发现国内的博客和帖子，对这个算法的讲述内容比较少，每篇文章都讲述了作者自己认为重要的内容，所以有一个点搞不懂的话没法整体性的进行理解。刚开始我自己就有一个点没想清楚想了好几天，我觉得程序员不能怕算法，书读百遍其义自现，阅读算法代码也是如此，平时多思考偶尔的一点灵光出现会减少你死磕算法浪费的时间。 "
---

# Myers'Diff

![美图](https://img-blog.csdnimg.cn/20201012200724264.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)


## 前言
写这篇文章已经拖了很久了，因为一直在准备后续的 [Myers‘Diff之线性空间细化](https://dandanlove.blog.csdn.net/article/details/108996040) 。最初不知道是什么时候发现 `DiffUtil` 对比列表 `item` 数据进行局部刷新，`git` 文件对比都用到了这个算法。上个月刚好再一次看到了就想深入了解一下。但发现发现国内的博客和帖子，对这个算法的讲述内容比较少，每篇文章都讲述了作者自己认为重要的内容，所以有一个点搞不懂的话没法整体性的进行理解。刚开始我自己就有一个点没想清楚想了好几天，我觉得程序员不能怕算法，书读百遍其义自现，阅读算法代码也是如此，平时多思考偶尔的一点灵光出现会减少你死磕算法浪费的时间。
## Myer差分算法
举一个最常见的例子，我们使用 `git` 进行提交时，通常会查看这次提交做了哪些改动，这里我们先简单定义一下什么是 `diff` ：`diff` 就是目标文本和源文本之间的区别，也就是将源文本变成目标文本所需要的操作。`Myers算法` 由 `Eugene W.Myers` 在 `1986` 年发表在 `《 Algorithmica》` 杂志上。的一篇[论文](http://xmailserver.org/diff2.pdf)中提出，是一个能在大部分情况产生**最短的直观的diff** 的一个算法。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201009164930993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)
**寻找最短的直观的diff** 是一个非常模糊的问题，首先，我们需要把这个问题抽象为一个具体的数学问题，然后再来寻找算法解决。

### 定义
- File A and File B
> The diff algorithm takes two files as input. The first, usually older, one is file A, and the second one is file B. The algorithm generates instructions to turn file A into file B.(diff算法将两个文件作为输入。第一个通常是较旧的文件是文件A，第二个是文件B。算法生成指令以将文件A转换为文件B。)

- Shortest Edit Script ( SES )
> The algorithm finds the Shortest Edit Script that converts file A into file B. The SES contains only two types of commands: deletions from file A and insertions in file B.(该算法找到将文件A转换为文件B的最短编辑脚本。SES仅包含两种类型的命令：从文件A删除和在文件B中插入。)

- Longest Common Subsequence ( LCS )
> Finding the SES is equivalent to finding the Longest Common Subsequence of the two files. The LCS is the longest list of characters that can be generated from both files by removing some characters. The LCS is distinct from the Longest Common Substring, which has to be contiguous.(查找SES等同于找到两个文件的最长公共子序列。 LCS是可以通过删除某些字符从两个文件生成的最长字符列表。 LCS与最长公共子字符串不同，后者必须是连续的。)

### 示例
本文使用与本文相同的示例。文件`A`包含 `ABCABBA`，文件`B`包含`CBABAC`。这些被表示为两个字符数组：`A []`和`B []`。
`A []`的长度为`N`，`B []`的长度为`M`。

我们就可以求解从`A数组`变成`B数组`的问题，转换成为求解从`A字符串`变成`B字符串`的问题（将抽象问题具现）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201009170750344.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)
`数组A`沿`x轴`放在顶部。`数组B`沿`y轴`向下放置。

PS:文章中的图都是由[DiffTutorial](https://www.codeproject.com/KB/recipes/DiffTutorial_2/DiffTutorial_bin.zip)软件制作而成，该应用程序是一种学习辅助工具。它显示算法各个阶段的图形表示。

解决方案：从`左上角（0，0）`到`右下角（7，6）`的最短路径。 您始终可以水平或垂直移动一个字符。水平`（右）`移动表示从`文件A`中删除，垂直`（向下）`移动表示在`文件B`中插入。如果存在匹配的字符，则还可以对角移动，以匹配结束。 解决方案是包含最多对角线的迹线。 `LCS`是轨迹中的对角线，`SES`是轨迹中的水平和垂直移动。例如，`LCS`的长度为4个字符，`SES`的长度为5个差异。

> snake: 一条snake代表走一步。例如从(0,0)->(0,1) / (0,0)->(1,0) / (0,1)->(0,2)->(2,4) 这分别为三条snake，走对角线不计入步数。

> k line: k lines表示长的对角线，其中每个k = x - y。假设一个点m(m,n),那么它所在的k line值为m - n。

> d contour: 每条有效路径(能从(0,0)到(m,n)的路径都称为有效路径)的步数。形象点，一条path有多个snake组成，这里d contours表示snake的个数。

## 贪婪算法

该算法是迭代的。它计算连续 `d` 的每条 `k` 线上最远的到达路径。当路径到达右下角时，将找到解决方案。

> 这里面有很重要的几点：
>1. 路径的终点必然在`k`线上。迭代进行，所以`k`线的上一步操作是`k+1`向下移动或者`k-1`向右移动；
>2. 计算连续的`d`每条`k`线上最远的到达路径（偶数`d`的端点在偶数`k`线，奇数类似）；
>3. 路径到达右下角结束；
>
>其中1和2都是在论文中进行了证明~！

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201009192930777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

- `k line`：棕色线是k的奇数值的k条线。黄线是k的偶数值的k线。 
- `snake`：深蓝色的线条是蛇。红蛇显示溶液痕迹。 
- `d contours`：淡蓝色的线是差异轮廓。例如，标记为“ 2”的直线上的三个端点全部具有2个水平或垂直移动。

### 外循环次数
从（x、y）组成的矩形`左上角`，到`右下角`。最长的路径莫过于所有对角线都不经过。也就是只走`X`和`Y`的长度即`最大长度=N+M`。
```for ( int d = 0 ; d <= N + M ; d++ )```
### 内循环的次数
在此循环内，我们必须为每条`k线`找到最远的到达路径。对于给定的`d`，只能到达的`k线`位于`[-d .. + d]`范围内。当所有移动都向下时，`k = -d` 是可能的；当所有移动都在右侧时，`k = + d` 是可能的。
```for ( int k = -d ; k <= d ; k += 2 )```
看到这里也许就有人产生疑问，为什么是`k+=2`。
> 这块有一个优化，文章前面说过`偶数`d`的端点在偶数`k`线，奇数类似`。
> 解释：移动奇数步长（前进或者后退都行）最终位置一定在奇数的`k线`上，偶数步长的最终位置一定在偶数的`k线`上。
> PS：这里让我纠结了好长时间，最后一下几点思考让我想的更加清楚：
> 1. 从`零`开始一步一步在`k线`上进行移动，一定是从`零`开始。
> 2. 这里的计算不是偶数加偶数得到的还是偶数，奇数加奇数得到的数是奇数或者偶数（这里是计算多个`+1或-1`）。
> 3. 无论偶数还是奇数`+1或-1`之后都会改变自己的奇偶性，所以`d`次操作之后的奇偶性由`d`的奇偶进行决定。由因为起点为偶数`零`，所以说`偶数`d`的端点在偶数`k`线，奇数类似`。

### 举例说明（d=3）
从`d = 3`的示例进行研究，这意味着`k`的取值范围是`[-3，-1，1，3]`。为了帮助您，我将示例中的`snake`的端点转录到下表中：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201010102738607.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

>k = -3：这种情况下，只有当k = -2，d = 2时，向下移动一步(k = -4, d = 2这种情况不存在)。所以，我们可以这么来走，从(2,4)点开始向下走到(2,5),由于(2,5)和(3,6)之间存在一个对角线，可以走到(3,6)。所以着整个snake是：(2,4) -> (2,5) -> (3,6)。

>k = -1：当k = -1时，有两种情况需要来讨论：分别是k = 0，d = 2向下走；k = -2 ，d = 2向右走。
>- 当k = 0，d = 2时，是(2,2)点。所以当从(2,2)点向下走一步，到达(2,3),由于(2,3)没有对角线连接，所以整个snake是：(2,2) -> (2,3)。
>- 当k = -2 ，d = 2时，是(2,4)点。所以当从(2,4)点向右走一步，到达(3,4)，由于(3,4)与(4,5)存在对角线，所以整个snake是：(2,4) -> (3,4) -> (4,5)。
>
>在整个过程中，存在两条snake，我们选择是沿着k line走的最远的snake，所以选择第二条snake。

> k = 1：当k = 1时，存在两种可能性，分别是从k = 0向右走一步，或者k = 2向下走一步，我们分别来讨论一下。
>- 当k = 0，d = 2时，是(2,2)点。所以当从(2,2)向右走一步，到达(3,2),由于(3,2)与(5,4)存在对角线，所以整个snake是：(2,2) ->(3,2) ->(5,4)。
>- 当k = 2，d = 2时，是(3,1)点。所以当从(3,1)向下走一步，到达(3,2)。所以这个snake是：(3,1) ->(3,2) ->(5,4)。
>
> 在整个过程中，存在两条snake，我们选择起点x值较大的snake，所以是：(3,1) ->(3,2) ->(5,4)。

> k = 3：这种情况下，(k = 4， d = 2)是不可能的，所以我们必须在(k = 2，d = 2)时向右移动一步。当k = 2, d = 2时， 是(3,1)点。所以从(3,1)点向右移动一步是(4,1)点。所以整个snake是：(3,1) -> (4,1) -> (5,2).

### 算法实现
我们有两个循环，我们需要一个数据结构。 
请注意，`d（n）`的解仅取决于`d（n-1）`的解。还请记住，对于`d`的偶数值，我们在偶数`k行`上找到端点，而这些端点仅取决于全部在奇数k行上的先前端点。对于`d`的奇数值也是如此。 
我们使用称为`V`的数组，其中`k`为索引，终点的`x`位置为值。我们不需要存储`y`位置，因为我们可以根据`x`和`k`来计算它：`y = x-k`。同样，对于给定的`d`，`k`在`[-d .. d]`范围内。

```bool down = ( k == -d || ( k != d && V[ k - 1 ] < V[ k + 1 ] ) );```

如果`k = -d`，我们一定是向下移动了；如果`k = d`，我们一定是向右移动了。
对于正常的中间情况，我们选择从`x`值较大的任何相邻行开始。这保证了我们到达`k`线上尽可能远的点。

```c
V[ 1 ] = 0;
for ( int d = 0 ; d <= N + M ; d++ )//外循环，进行多少次代表有多少个snake和V数组
{
  for ( int k = -d ; k <= d ; k += 2 )//内循环
  {
    // down or right?
    bool down = ( k == -d || ( k != d && V[ k - 1 ] < V[ k + 1 ] ) );
    int kPrev = down ? k + 1 : k - 1;//如果向下，那么上一步应该是k+1
    // start point，上一个点
    int xStart = V[ kPrev ];//v[k]=x
    int yStart = xStart - kPrev;//y=x-k
    // mid point，下一个点
    int xMid = down ? xStart : xStart + 1;
    int yMid = xMid - k;
    // end point
    int xEnd = xMid;
    int yEnd = yMid;
    // follow diagonal
    int snake = 0;//是否有对角线继续往重点走
    while ( xEnd < N && yEnd < M && A[ xEnd ] == B[ yEnd ] ) 
    { 
    	xEnd++; yEnd++; snake++; 
   	}
    // save end point
    V[ k ] = xEnd;
    // check for solution
    if ( xEnd >= N && yEnd >= M ) /* solution has been found */
  }
}
```

上面的代码寻找一条到达终点的`snake`。因为`V数组`里面存储的是在`k line`最新端点的坐标，所以为了寻找到所有的`snake`，我们在`d`的每次循环完毕之后，从`d(Solution)遍历到0`。如下：

```c
IList<V> Vs; // saved V's indexed on d
IList<Snake> snakes; // list to hold solution
//从后往前推，最后一条snake是到达终点必须经过的路线。
POINT p = new POINT( N, M ); // start at the end
for ( int d = vs.Count - 1 ; p.X > 0 || p.Y > 0 ; d-- )
{
  var V = Vs[ d ];//内循环产生的数据
  int k = p.X - p.Y;//本次循环的起点的k线
  // end point is in V
  int xEnd = V[ k ];
  int yEnd = x - k;
  // down or right?
  bool down = ( k == -d || ( k != d && V[ k - 1 ] < V[ k + 1 ] ) );
  int kPrev = down ? k + 1 : k - 1;//上一个snake的k线
  // start point
  int xStart = V[ kPrev ];
  int yStart = xStart - kPrev;
  // mid point
  int xMid = down ? xStart : xStart + 1;//中间点
  int yMid = xMid - k;
  snakes.Insert( 0, new Snake( /* start, mid and end points */ ) );
  p.X = xStart;
  p.Y = yStart;
}
```
时间复杂度: 期望为`O(M+N+D^2)`，最坏情况为为`O((M+N)D)` 。

有兴趣的可以继续阅读下一篇文章 [Myers‘Diff之线性空间细化](https://dandanlove.blog.csdn.net/article/details/108996040) 。

算法实践：[DiffUtil和它的差量算法](https://dandanlove.blog.csdn.net/article/details/109123553)

参考链接：
[代码:diff-match-patch](https://github.com/google/diff-match-patch)
[diff2论文](http://xmailserver.org/diff2.pdf)
[Myers diff alogrithm:part 1](https://www.codeproject.com/Articles/42279/Investigating-Myers-diff-algorithm-Part-1-of-2)
[Myers diff alogrithm:part 2](https://www.codeproject.com/Articles/42280/Investigating-Myers-Diff-Algorithm-Part-2-of-2)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！