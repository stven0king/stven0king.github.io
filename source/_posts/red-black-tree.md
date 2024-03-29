---
title: 寻找红黑树的操作手册
date: 2018-03-018 21:48:00
tags: [红黑树,算法]
categories: 数据结构
description: "红黑树查找的最坏时间复杂度也是O(logN)。为了它这么高的性能，我感觉自己费了这么多脑细胞和时间来学习也是值得的(自己前前后后看了好多次)。这篇文章和插入图也是我自己用心根据自己的理解来做，希望大家能在学习红黑树的时候提高效率，不走弯路。"
---

# 前言

[二叉树知识点回忆以及整理](http://dandanlove.com/2017/10/20/about-binary-tree/)这篇文章中我们说过“二叉树是一个简单的二分查找，但其性能取决于二叉树的层数”。
- 最好的情况是O(logn),存在于完全二叉树情况下，其访问性能近似于折半查找；
- 最差的情况是O(n),比如插入的元素所有节点都没有左子树（右子树），这种情况需要将二叉树的全部节点遍历一次。
![二叉排序树](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktYzgyN2YwYjg2ZjUxNzk3ZS5wbmc?x-oss-process=image/format,png#pic_center)

红黑树本质上是一种二叉查找树，在节点类中添加类一个用来标识颜色的字段，同时具有一定的规则。同时具备这亮点使得红黑树的性能达到理想中的均衡状态（插入、删除、查找的最坏时间负责度为O(logn)）。

在Java1.8中HashMap使用的就是链表和红黑树，[迟到一年HashMap解读](http://dandanlove.com/2017/10/27/late-one-year-hashmap/)。Java集合中的TreeSet和TreeMap，C++ STL中的set、map，以及Linux虚拟内存的管理，都是通过红黑树去实现的。


# 简介

红黑树（英语：Red–black tree）是一种自平衡二叉查找树，是在计算机科学中用到的一种数据结构，典型的用途是实现关联数组。它是在1972年由鲁道夫·贝尔发明的，他称之为"对称二叉B树"，它现代的名字是在Leo J. Guibas和Robert Sedgewick于1978年写的一篇论文中获得的。它是复杂的，但它的操作有着良好的最坏情况运行时间，并且在实践中是高效的：它可以在 O(logn)时间内做查找，插入和删除，这里的n是树中元素的数目，[摘自：维基百科-红黑树](https://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91)。

```java
public class RBTree<T extends Comparable<T>> {
    public RBNode<T> mRoot = null;    // 根结点
    public static boolean RED = true;
    public static boolean BLACK = false;
    class RBNode <T extends Comparable<T>> {
        //颜色
        boolean color;
        //关键字（键值）
        T key;
        //左子节点
        RBNode<T> left;
        //右子节点
        RBNode<T> right;
        //父节点
        RBNode<T> parent;

        public RBNode(T key, boolean color, RBNode<T> parent, RBNode<T> left, RBNode<T> right) {
            this.key = key;
            this.color = color;
            this.parent = parent;
            this.left = left;
            this.right = right;
        }

        public T getKey() {
            return key;
        }

        @Override
        public String toString() {
            return "" + key + (this.color == RED ? "RED" : "BLACK");
        }
    }
}
```

# 红黑树特点

>- 1、每个节点不是红色就是黑色的；
>- 2、根节点总是黑色的；
>- 3、所有的叶节点都是是黑色的（红黑树的叶子节点都是空节点（NIL或者NULL））；
>- 4、如果节点是红色的，则它的子节点必须是黑色的（反之不一定）；
>- 5、从根节点到叶节点或空子节点的每条路径，必须包含相同数目的黑色节点（即相同的黑色高度）。

需要注意：
> 特性3中的叶子节点，是只为空(NIL或null)的节点。
> 特性5，确保没有一条路径会比其他路径长出俩倍。因而，红黑树是相对是接近平衡的二叉树。

# 红黑树的修正

> 变色、左旋、右旋是红黑树在二叉树上的扩展操作，同时也是基于这三个操作才能遵守红黑树的五个特性。所以熟悉二叉树操作的同学只要掌握这红黑树这三个操作那么就能更加容易的理解进行红黑树的添加和删除之后怎么保证其平衡性，不熟悉二叉树的也可以先看看[《二叉树知识点回忆以及整理》](http://dandanlove.com/2017/10/20/about-binary-tree/)这片文章。

## 变色

> 变色仅仅指的是红黑树节点的变色。因为红黑树节点必须是【红】或者【黑】这两中颜色，所以变色只是将当前的节点颜色进行变化，以满足特性（2，3，4，5）。

## 左旋

通常左旋操作用于将一个向右倾斜的红色链接旋转为向左链接。示意图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224152901205.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

左旋操作动画（更加容易理解和记忆）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224152932810.gif#pic_center)

```java
/*************对红黑树节点x进行左旋操作 ******************/
/*
 * 左旋示意图：对节点x进行左旋
 *     p                       p
 *    /                       /
 *   x                       y
 *  / \                     / \
 * lx  y      ----->       x  ry
 *    / \                 / \
 *   ly ry               lx ly
 * 左旋做了三件事：
 * 1. 将y的左子节点赋给x的右子节点,并将x赋给y左子节点的父节点(y左子节点非空时)
 * 2. 将x的父节点p(非空时)赋给y的父节点，同时更新p的子节点为y(左或右)
 * 3. 将y的左子节点设为x，将x的父节点设为y
 */
public void leftRotate(RBNode<T> x) {
    if (x == null) return;
    //1. 将y的左子节点赋给x的右子节点,并将x赋给y左子节点的父节点(y左子节点非空时)
    RBNode<T> y = x.right;
    x.right = y.left;
    if (y.left != null) {
        y.left.parent = x;
    }
    //2. 将x的父节点p(非空时)赋给y的父节点，同时更新p的子节点为y(左或右)
    y.parent = x.parent;
    if (x.parent == null) {
        //mRoot是RBTree的根节点
        this.mRoot = y;
    } else {
        if (x == x.parent.left) {
            x.parent.left = y;
        } else {
            x.parent.right = y;
        }
    }
    //3. 将y的左子节点设为x，将x的父节点设为y
    y.left = x;
    x.parent = y;
}
```

## 右旋

右旋可左旋刚好相反，这里不再赘述，直接看示意图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153005170.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

右旋操作动画（更加容易理解和记忆）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153029893.gif#pic_center)

```java
/*************对红黑树节点y进行右旋操作 ******************/
/*
 * 右旋示意图：对节点y进行右旋
 *        p                   p
 *       /                   /
 *      y                   x
 *     / \                 / \
 *    x  ry   ----->      lx  y
 *   / \                     / \
 * lx  rx                   rx ry
 * 右旋做了三件事：
 * 1. 将x的右子节点赋给y的左子节点,并将y赋给x右子节点的父节点(x右子节点非空时)
 * 2. 将y的父节点p(非空时)赋给x的父节点，同时更新p的子节点为x(左或右)
 * 3. 将x的右子节点设为y，将y的父节点设为x
 */
public void rightRotate(RBNode<T> y) {
    if (y == null) return;
    //1. 将x的右子节点赋给y的左子节点,并将y赋给x右子节点的父节点(x右子节点非空时)
    RBNode<T> x = y.left;
    y.left = x.right;
    if (x.right != null) {
        x.right.parent = y;
    }
    //2. 将y的父节点p(非空时)赋给x的父节点，同时更新p的子节点为x(左或右)
    x.parent = y.parent;
    if (y.parent == null) {
        this.mRoot = x;
    } else {
        if (y == y.parent.left) {
            y.parent.left = x;
        } else {
            y.parent.right = x;
        }
    }
    //3. 将x的右子节点设为y，将y的父节点设为x
    x.right = y;
    y.parent = x;
}
```

# 红黑树节点的添加

红黑树是在二叉树的基础上进行扩展的，其添加节点也是像二叉树一样进行添加，然后再做调整。[二叉树知识点回忆以及整理#创建二叉树](http://dandanlove.com/2017/10/20/about-binary-tree/) 这部分讲述了二叉树节点的添加。所以这里我们重点讲述二叉树的添加完后的调整。

> - 红黑树的第 5 条特征规定，任一节点到它子树的每个叶子节点的路径中都包含同样数量的黑节点。也就是说当我们往红黑树中插入一个黑色节点时，会违背这条特征。
> - 同时第 4 条特征规定红色节点的左右孩子一定都是黑色节点，当我们给一个红色节点下插入一个红色节点时，会违背这条特征。

> 因此我们需要在插入黑色节点后进行结构调整，保证红黑树始终满足这 5 条特征。

# 红黑树插入后节点的调整思想

> 数学里最常用的一个解题技巧就是把多个未知数化解成一个未知数。

我们插入黑色节点的时候担心违背第5条，插入红色节点时担心违背第4条，所以我们将将插入的节点改为红色，然后判断插入的节点的父亲是不是红色，是的话进行修改调整（变色、左旋、右旋）。同时在调整的过程中我们需要遵守`5条特性`。

因为左右子树的操作是对称的，我们下边讲述需要添加的节点的父节点是祖父节点的左孩子的情况，右子树添加与其相反。

>- 1、如果我们添加的【红色节点】的【父节点】是黑色，那么树不需要做调整。
>- 2、如果我们添加的【红色节点】的【父节点】是红色，那么树需要做调整。
    - 1）、父节点是红色，叔叔节点（父节点的兄弟节点）是红色的。
    - 2）、父节点是红色，叔叔节点是黑色，添加的节点是父节点的左孩子。
    - 3）、父节点是红色，叔叔节点是黑色，添加的节点是父节点的右孩子。

`父节点是黑色，祖父节点一定是黑色的，因为红色节点的父节点不可能是红色（特性4:每个红色节点的两个子节点一定都是黑色）。`

## 调整-情况（1）：父节点是红色，叔叔节点（父节点的兄弟节点）是红色的。

下图是这样情况的红黑树的修改过程（上边是目标节点为左孩子，下边是目标节点是右孩子）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153143480.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

为了新添加的节点也满足特性4:

- 将父节点和叔叔节点全部染成黑色（节点T满足了特性四），但是这样父亲和叔叔节点的分支都多了一个黑色；
- 将祖父节点染成红色，这样祖父节点的两个分支满足了所有特性，但是我们需要检验祖父节点是否符合红黑树的特性；
- 将祖父节点当前插入节点，继续向树根方向进行修改；

这样我们一直向上循环，直到父节点变为黑色，或者达到树根为止。

## 调整-情况（2）：父节点是红色，叔叔节点是黑色，添加的节点是父节点的左孩子。

下图是这样情况的红黑树的修改过程:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153215672.png#pic_center)


我们通过将祖父节点的左孩子分支上的连续两个红色节点，转移一个插入到祖父节点和他的右孩子之间（保证左边没有两个连续红点、右边插入的红点满足所有特性）。

- 先将父节点染成黑色；
- 将祖父节点染成红色；
- 将父节点进行右旋；

我们仅仅通过以上3个步骤就调整完使整个红黑树的节点满足5个特性。

## 调整-情况（3）：父节点是红色，叔叔节点是黑色，添加的节点是父节点的右孩子。

下图是这样情况的红黑树的修改过程:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153238966.png#pic_center)

我们父节点进行左旋操作，这样就变成了`调整-情况（2）`的状态，然后再按照其调整操作继续进行调整。

> 通过以上三个情况对红黑树的调整，我们可以解决红黑树插入红色节点中的所有问题。

# 红黑树插入代码实现：

```java
/*********************** 向红黑树中插入节点 **********************/
public void insert(T key) {
    RBNode<T> node = new RBNode<>(key, BLACK, null, null, null);
    insert(node);
}

/**
 * 1、将节点插入到红黑树中，这个过程与二叉搜索树是一样的
 * 2、将插入的节点着色为"红色"；将插入的节点着色为红色，不会违背"特性5"！
 *    少违背了一条特性，意味着我们需要处理的情况越少。
 * 3、通过一系列的旋转或者着色等操作，使之重新成为一颗红黑树。
 * @param node 插入的节点
 */
public void insert(RBNode<T> node) {
    //node的父节点
    RBNode<T> current = null;
    RBNode<T> x = mRoot;

    while (x != null) {
        current = x;
        int cmp = node.key.compareTo(x.key);
        if (cmp < 0) {
            x = x.left;
        } else {
            x = x.right;
        }
    }
    //找到位置，将当前current作为node的父节点
    node.parent = current;
    //2. 接下来判断node是插在左子节点还是右子节点
    if (current != null) {
        int cmp = node.key.compareTo(current.key);
        if (cmp < 0) {
            current.left = node;
        } else {
            current.right = node;
        }
        node.color = RED;
        insertFixUp(node);
    } else {
        this.mRoot = node;
    }
}

/**
 * 修改整插入node节点之后的红黑树
 * @param node
 */
public void insertFixUp(RBNode<T> node) {
    //定义父节点和祖父节点
    RBNode<T> parent, gparent;
    //需要修整的条件：父节点存在，且父节点的颜色是红色
    while (((parent = node.parent) != null) && isRed(parent)) {
        //祖父节点
        gparent = parent.parent;
        //若父节点是祖父节点的左子节点
        if (parent == gparent.left) {
            //获取叔叔点点
            RBNode<T> uncle = gparent.right;
            //case1:叔叔节点是红色
            if (uncle != null && isRed(uncle)) {
                //把父亲和叔叔节点涂黑色
                parent.color = BLACK;
                uncle.color = BLACK;
                //把祖父节点图红色
                gparent.color = RED;
                //将位置放到祖父节点
                node = gparent;
                //继续往上循环判断
                continue;
            }

            //case2：叔叔节点是黑色，且当前节点是右子节点
            if (node == parent.right) {
                //从父亲即诶单处左旋
                leftRotate(parent);
                //将父节点和自己调换一下，为右旋左准备
                RBNode<T> tmp = parent;
                parent = node;
                node = tmp;
            }
            //case3：叔叔节点是黑色，且当前节点是左子节点
            parent.color = BLACK;
            gparent.color = RED;
            rightRotate(gparent);
        } else {
            //若父亲节点是祖父节点的右子节点，与上面的完全相反，本质一样的
            RBNode<T> uncle = gparent.left;
            //case1:叔叔节点也是红色
            if (uncle != null & isRed(uncle)) {
                parent.color = BLACK;
                uncle.color = BLACK;
                gparent.color = RED;
                node = gparent;
                continue;
            }

            //case2: 叔叔节点是黑色的，且当前节点是左子节点
            if (node == parent.left) {
                rightRotate(parent);
                RBNode<T> tmp = parent;
                parent = node;
                node = tmp;
            }
            //case3: 叔叔节点是黑色的，且当前节点是右子节点
            parent.color = BLACK;
            gparent.color = RED;
            leftRotate(gparent);
        }
    }
    //将根节点设置为黑色
    this.mRoot.color = BLACK;
}
```

# 红黑树节点的删除

上面部分讨论了红黑树添加新节点，接下来的部分讲述红黑树的删除。红黑树的删除是红黑树操作中最重要的部分（为什么说最重要呢？因为它最难理解）。

同样红黑树的删除是在二叉树进行删除操作的基础上进行调整的，使之满足红黑树的所有特性。
[二叉树知识点回忆以及整理#二叉树节点删除](http://dandanlove.com/2017/10/20/about-binary-tree/) 这部分讲述了二叉树节点的添加，所以这里我们重点讲述二叉树的添加完后的调整。

## 二叉树节点删除的思路

>- 如果要删除的节点正好是叶子节点，直接删除就 Ok 了；
>- 如果要删除的节点还有子节点，就需要建立父节点和子节点的关系：
    - 如果只有左孩子或者右孩子，直接把这个孩子上移放到要删除的位置就好了；
    - 如果有两个孩子，就需要选一个合适的孩子节点作为新的根节点，该节点称为 继承节点。（新节点要求比所有左子树要大、比右子树要小，我们可以选择左子树中的最大节点，或者选择右子树中的最小的节点。）

## 红黑树删除总纲

> 我们需要在二叉树删除的思路上，再考虑对删除完后的树进行调整。还记得文章说过`数学里最常用的一个解题技巧就是把多个未知数化解成一个未知数。`这句话么？二叉树的删除分为两个大case或者三个小case。我们首先把这些case合并为一个case，再进行调整是不是就更加简单来？

## 红黑树删除之三派合并

上述将删除的步骤总结在一下就是：

- 1、如果删除节点的左孩子和右孩子不同时为null，那么只需要让其子树继承删除该节点的位置；
- 2、如果删除的节点是叶子节点，我们直接进行调整；
- 假如删除节点的左右孩子都不是null，需要`后继节点`替换被删除的节点和值和颜色，这样才不会引起红黑树平衡的破坏，只需要对 `后继节点`删除后进行调整，这样我们就回归处理情况 1 和 2 的状态；
    - `后继节点`为左子树最右边的子叶节点
    - `后继节点`为右子树最左边的叶子节点；

> 如果需要删除的节点颜色为`红色`，那么红黑树的结构不被破坏，也就不需要进行调整。如果我们判断删除节点的颜色为`黑色`，那么就进行调整；

代码以及解析：
```java
/*********************** 删除红黑树中的节点 **********************/
public void remove(T key) {
    RBNode<T> node;
    if ((node = search(mRoot, key)) != null) {
        remove(node);
    }
}

/**
 * 1、被删除的节点没有儿子，即删除的是叶子节点。那么，直接删除该节点。
 * 2、被删除的节点只有一个儿子。那么直接删除该节点，并用该节点的唯一子节点顶替它的初始位置。
 * 3、被删除的节点有两个儿子。那么先找出它的后继节点（右孩子中的最小的，该孩子没有子节点或者只有一右孩子）。
 *    然后把"它的后继节点的内容"复制给"该节点的内容"；之后，删除"它的后继节点"。
 *    在这里后继节点相当与替身，在将后继节点的内容复制给"被删除节点"之后，再将后继节点删除。
 *    ------这样问题就转化为怎么删除后继即节点的问题？
 *    在"被删除节点"有两个非空子节点的情况下，它的后继节点不可能是双子都非空。
 *    注：后继节点为补充被删除的节点；
 *    即：意味着"要么没有儿子，要么只有一个儿子"。
 *    若没有儿子，则回归到（1）。
 *    若只有一个儿子，则回归到（2）。
 *
 * @param node  需要删除的节点
 */
public void remove(RBNode<T> node) {
    RBNode<T> child, parent;
    boolean color;
    //1、删除的节点的左右孩子都不为空
    if ((node.left != null) && (node.right != null)) {
        //先找到被删除节点的后继节点，用它来取代被删除节点的位置
        RBNode<T> replace = node;
        //1).获取后继节点[右孩子中的最小]
        replace = replace.right;
        while (replace.left != null) {
            replace = replace.left;
        }
        //2).处理【后继节点的子节点】和【被删除节点的子节点】之间的关系
        if (node.parent != null) {
            //要删除的节点不是根节点
            if (node == node.parent.left) {
                node.parent.left = replace;
            } else {
                node.parent.right = replace;
            }
        } else {
            mRoot = replace;
        }

        //3).处理【后继节点的子节点】和【被删除节点的子节点】之间的关系
        //后继节点肯定不存在左子节点
        child = replace.right;
        parent = replace.parent;
        //保存后继节点的颜色
        color = replace.color;
        //后继节点是被删除的节点
        if (parent == node) {
            parent =replace;
        } else {
            if (child != null) {
                child.parent = parent;
            }
            parent.left = child;
            replace.right = node.right;
            node.right.parent = replace;
        }
        replace.parent = node.parent;
        replace.color = node.color;
        replace.left = node.left;
        node.left.parent = replace;
        //4。如果移走的后继节点颜色是黑色，重新修正红黑树
        if (color == BLACK) {
            removeFixUp(child, parent);
        }
        node = null;
    } else {
        //被删除的节点是叶子节点，或者只有一个孩子。
        if (node.left != null) {
            child = node.left;
        } else {
            child = node.right;
        }
        parent = node.parent;
        //保存"取代节点"的颜色
        color = node.color;
        if (child != null) {
            child.parent = parent;
        }
        //"node节点"不是根节点
        if (parent != null) {
            if (parent.left == node) {
                parent.left = child;
            } else {
                parent.right = child;
            }
        } else {
            mRoot = child;
        }
        if (color == BLACK) {
            removeFixUp(child, parent);
        }
        node = null;
    }
}
```

完成删除操作后接下来进行我们的调整操作，看完上面代码后我们知道调整时需要传递的参数是`后继节点`和`删除的父节点`。

## 红黑树删除之节点调整

删除后那么`后继节点`就成才删除节点的孩子，那么接下的过程中我们将`后继节点`定义`目标节点`。

下边我们讨论一下节点的颜色情况：因为当前节点的颜色一定是黑色的，我们只根据兄弟节点的颜色做讨论。

- 1、当前节点是黑色的，且兄弟节点是红色的（那么父节点和兄弟节点的子节点肯定是黑色的）；
- 2、当前节点是黑色的，且兄弟节点是黑色的，
    - 1）、兄弟节点的两个子节点均为黑色的；
    - 2）、兄弟节点的左子节点是红色，右子节点时黑色的；
    - 3）、兄弟节点的右子节点是红色，左子节点任意颜色；


#### 调整情况（1）:当前节点是黑色的，且兄弟节点是红色的

下图是这样情况的红黑树的修改过程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153433246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

- 将父节点涂红，将兄弟节点涂黑，然后将当前节点的父节点进行支点左旋。这样就会转化为情况2中的某种状态。

### 调整情况（2）:当前节点是黑色的，且兄弟节点是黑色的

#### 2.1、兄弟节点的左子节点是红色，右子节点时黑色的；

下图是这样情况的红黑树的修改过程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153500801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)


- 将兄弟节点涂红，将当前节点指向其父节点，将其父节点指向当前节点的祖父节点，继续往树根递归判断以及调整；

#### 2.2、兄弟节点的两个子节点均为黑色的；

下图是这样情况的红黑树的修改过程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153536670.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

- 把当前节点的兄弟节点涂红，把兄弟节点的左子节点涂黑，然后以兄弟节点作为支点做右旋操作。

#### 2.3、兄弟节点的右子节点是红色，左子节点任意颜色；

下图是这样情况的红黑树的修改过程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224153615785.png#pic_center)

- 把兄弟节点涂成父节点的颜色，再把父节点涂黑，把兄弟节点的右子节点涂黑，然后以当前节点的父节点为支点做左旋操作。

## 红黑树删除调整case总结&代码实现

如果是从`case:1`开始发生的，可能`case:2，3，4`中的一种：如果是`case:2`，就不可能再出现`case:3和4`；如果是`case:3`，必然会导致`case:4`的出现；如果`case:2和3`都不是，那必然是`case:4`。

```java
/**
 * 红黑树删除修正函数
 *
 * 在从红黑树中删除插入节点之后(红黑树失去平衡)，再调用该函数；
 * 目的是将它重新塑造成一颗红黑树。
 * 如果当前待删除节点是红色的，它被删除之后对当前树的特性不会造成任何破坏影响。
 * 而如果被删除的节点是黑色的，这就需要进行进一步的调整来保证后续的树结构满足要求。
 * 这里我们只修正删除的节点是黑色的情况：
 *
 * 调整思想：
 * 为了保证删除节点的父节点左右两边黑色节点数一致，需要重点关注父节点没删除的那一边节点是不是黑色。
 * 如果删除后父亲节点另一边比删除的一边黑色多，就要想办法搞到平衡。
 * 1、把父亲节点另一边（即删除节点的兄弟树）其中一个节点弄成红色，也少了一个黑色。
 * 2、或者把另一边多的节点（染成黑色）转过来一个
 *
 * 1）、当前节点是黑色的，且兄弟节点是红色的（那么父节点和兄弟节点的子节点肯定是黑色的）；
 * 2）、当前节点是黑色的，且兄弟节点是黑色的，且兄弟节点的两个子节点均为黑色的；
 * 3）、当前节点是黑色的，且兄弟节点是黑色的，且兄弟节点的左子节点是红色，右子节点时黑色的；
 * 4）、当前节点是黑色的，且兄弟节点是黑色的，且兄弟节点的右子节点是红色，左子节点任意颜色。
 *
 * 以上四种情况中，2，3，4都是（当前节点是黑色的，且兄弟节点是黑色的）的子集。
 *
 * 参数说明：
 * @param node 删除之后代替的节点（后序节点）
 * @param parent 后序节点的父节点
 */
private void removeFixUp(RBNode<T> node, RBNode<T> parent) {
    RBNode<T> other;
    RBNode<T> root = mRoot;
    while ((node == null || node.color == BLACK) && node != root) {
        if (parent.left == node) {
            other = parent.right;
            if (other.color == RED) {
                //case 1：x的兄弟w是红色的【对应状态1，将其转化为2，3，4】
                other.color = BLACK;
                parent.color = RED;
                leftRotate(parent);
                other = parent.right;
            }

            if ((other.left == null || other.left.color == BLACK)
                    && (other.right == null || other.right.color == BLACK)) {
                //case 2：x的兄弟w是黑色，且w的两个孩子都是黑色的【对应状态2，利用调整思想1网树的根部做递归】
                other.color = RED;
                node = parent;
                parent = node.parent;
            } else {
                if (other.right == null || other.right.color == BLACK) {
                    //case 3:x的兄弟w是黑色的，并且w的左孩子是红色的，右孩子是黑色的【对应状态3，调整到状态4】
                    other.left.color = BLACK;
                    other.color = RED;
                    rightRotate(other);
                    other = parent.right;
                }
                //case 4:x的兄弟w是黑色的；并且w的右孩子是红色的，左孩子任意颜色【对应状态4，利用调整思想2做调整】
                other.color = parent.color;
                parent.color = BLACK;
                other.right.color = BLACK;
                leftRotate(parent);
                node = root;
                break;
            }
        } else {
            other = parent.left;
            if (other.color == RED) {
                //case 1:x的兄弟w是红色的
                other.color = BLACK;
                parent.color = RED;
                rightRotate(parent);
                other = parent.left;
            }

            if ((other.left == null || other.left.color == BLACK)
                    && (other.right == null || other.right.color == BLACK)) {
                //case 2:x兄弟w是黑色，且w的两个孩子也都是黑色的
                other.color = RED;
                node = parent;
                parent = node.parent;
            } else {
                if (other.left == null || other.left.color == BLACK) {
                    //case 3:x的兄弟w是黑色的，并且w的左孩子是红色，右孩子为黑色。
                    other.right.color = BLACK;
                    other.color = RED;
                    leftRotate(other);
                    other = parent.left;
                }
                //case 4:x的兄弟w是黑色的；并且w的右孩子是红色的，左孩子任意颜色。
                other.color = parent.color;
                parent.color = BLACK;
                other.left.color = BLACK;
                rightRotate(parent);
                node = root;
                break;
            }
        }
    }
    if (node != null) {
        node.color = BLACK;
    }
}
```

# 总结

红黑树查找的最坏时间复杂度也是O(logN)。为了它这么高的性能，我感觉自己费了这么多脑细胞和时间来学习也是值得的(自己前前后后看了好多次)。这篇文章和插入图也是我自己用心根据自己的理解来做，希望大家能在学习红黑树的时候提高效率，不走弯路。

# 参考文章

红黑树的文章有好多，仔细阅读几篇之后感觉这三篇讲的不错。第一篇的红黑树插入，第二篇的红黑树左旋和右旋（动画做的非常好）和红黑树的删除，最后一篇的代码写的很棒并且提供的测试的case。文章的主要内容都是以下三篇文章的总结。

[重温数据结构：深入理解红黑树](https://www.cnblogs.com/skywang12345/p/3624343.html)
[【数据结构和算法05】 红-黑树（看完包懂~）](http://blog.csdn.net/eson_15/article/details/51144079)
[红黑树(五)之 Java的实现](https://www.cnblogs.com/skywang12345/p/3624343.html)

源码：[红黑树操作&代码注释](https://gitee.com/dandanlove/codes/xezpa7g3q1dybw5mutvk643)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！