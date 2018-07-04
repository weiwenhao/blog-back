---
layout: post
title: Amazing tree —— 二叉查找树
date: 2018-04-20 16:12:02
categories: algorithm
tags: algorithm
---


> 二分查找很好的解决了查找问题，将时间复杂度从 O(n)降到了O(logn)。
> 但是二分查找的前提条件是数据必须是有序的，并且具有线性的下标。
> 对于线性表，可以很好的应用二分查找，但是在插入和删除操作时则可能会造成整个线性表的动荡，时间复杂度达到了O(n)
> 链表更是没法应用二分查找。

于是有了下面将要介绍的算法，其在查找、插入、删除都能够达到O(logn)的时间复杂度 —— **二叉查找树**

<!-- more -->

见名知意，其数据结构基础为二叉树，初次接触到二叉树时并没有感觉到其有什么突出之处。但看到通过二叉树构建出的二叉查找树方案时，确被深深的震撼了。

## 定义
二叉查找树（英语：Binary Search Tree），也称二叉搜索树、有序二叉树（英语：ordered binary tree），排序二叉树（英语：sorted binary tree），是指一棵空树或者具有下列性质的二叉树：

若任意结点的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
若任意结点的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
任意结点的左、右子树也分别为二叉查找树；
没有键值相等的结点。

根据上面的规则我们先来定义一颗二叉树
![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0a99f8188?w=478&h=468&f=jpeg&s=29956)


这里可以很容易看出其规律，不需要过多的解释。

### 插入
现在再插入一个元素13。 13>12所以往右边走来到14，13 < 14则左走，发现14没有左孩子，所以将13插入之，得到下面这张图
![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0a9a9885d?w=534&h=483&f=jpeg&s=35531)
### 查找
按照上面插入的思路，可以很容易实现搜索操作。并且发现其查找的时间复杂度就为这颗树的深度。

> 根据完全二叉树的性质，具有n个结点的完全二叉树的深度为 `[logn] + 1`

忽略掉`+1`得到二叉查找树的查找时间复杂度为 `O(logn)`，但是实际上并非如此，后面我们分析。

### 遍历
二叉树的遍历有前序、中序、后序遍历三种方式，这里着重介绍后序遍历。
对二差查找树进行中序遍历时，可以得到一个`asc`的排序结果。如上面的树中序遍历的结果是 **3, 8, 9, 12, 13, 14**。 
`中序遍历从一颗子树最左的节点开始输出，既该树的最小值`。实现中序遍历只需要将**数据收集点**置于左递归点与右递归点之间，这样说还是有些含糊了，看代码吧


```php
/**
 * 中序遍历
 * @param $root
 * @return array
 */
public function inorder($root)
{
    $data = [];

    if ($root->left) {
        $data = array_merge($data, $this->inorder($root->left)); //左孩子递归点
    }

    $data[] = $root->data; // 这里是中序遍历的数据收集点

    if ($root->right) {
        $data = array_merge($data, $this->inorder($root->right)); // 右孩子递归点
    }

    return $data;
}
```

> 前驱与后继， 以9节点为例， 12属于9的后继，8属于9的前驱。

### 删除
我们给这颗树多加几个结点
![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0add13dd2?w=968&h=703&f=jpeg&s=76963)


删除树中的结点分为很多种情况，如被删除的结点不存在子结点，只存在左子树/右子树，左右子树都存在，这里已覆盖率最广的左右子树都存在为例。

> 分析一个需求时要并不是需求存在多少中情况我们就写多少种情况。而应该分析情况之间的关系，是否存在重复，或者属于关系等，程序员应该做的就是提取需求的本质，力求于最简洁的实现

现在我们打算删除25这个结点，你会怎么做？
如果只是简单把18来顶替原来25的位置，则需要对18这颗子树的孩子们进行重新调整。18只有三个孩子还好，但是当孩子成千上万时，显然会造成大面积的调整。
所以我希望能够找到一个更好的节点来代替25，按照算法导论中的描述，我们应该寻找该结点的前驱或者后继来代替，比如图中的24和27分别是25的前驱和后继。

> 为什么要使用前驱或者后缀来代替？这点我十分不确定，我给自己的理由是
> 1. 该结点是一个特殊值，属于某颗子树的最大值或者最小值，具有确定性，可以被比较好的定义且查找出来。
> 2. 由于该结点属于被删除节点的前驱或者后继，则删除该结点对数据结构造成的影响最小。*我并不确定是对什么的数据结构造成的影响最小*

上面描述的情况的图解如下 ↓
![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0a9bdc5fc?w=756&h=537&f=jpeg&s=50523)
![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0a9b88949?w=696&h=457&f=jpeg&s=47267)


删除还存在一些其他的情况,比如下面这种情况↓
![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0ae694229?w=909&h=529&f=jpeg&s=52024)


对于这种情况直接将30提升到25即可，接下来看一下看php的代码实现：


```php
public function delete($root, $data)
{
    if (!$root) {
        return null;
    }

    if ($root->data === $data) {
        if ($root->left) {
            // 左转
            $node = $root->left;

            $parent = $root;
            $toward = 'left';

            while ($node->right) {

                $parent = $node;
                $toward = 'right';

                $node = $node->right;
            }

            $root->data = $node->data;

            $parent->{$toward} = $this->delete($node, $node->data);
    

        } else {
            return $root->right;
        }
    } elseif ($root->data > $data) {
        // 如果root的左孩子没有被删除,那就原样返回回来, 如果被删除了,那就找个孩子代替
        $root->left = $this->delete($root->left, $data);
    } else {
        $root->right = $this->delete($root->right, $data);
    }

    return $root;
}
```
由于php有内存回收机制，因此我们没有办法像c一样直接去修改内存，所以这里借助递归的特性来解决这个问题 `$root->left = $this->delete($root->left, $data);` 做类似这样一个处理，这可能会有些理解上的困难。但总归还是能够明白的~

除了递归解决外，也可以用下面这种办法。
即定义一个parent和toward来做一个导向，这在上面的代码中也有体现。**该方法更加适用于迭代处理**

```php
$parent = $root;
$toward = 'left';

while ($node->right) {

    $parent = $node;
    $toward = 'right';

    $node = $node->right;
}

```

更详细的实习细节和调用示例请参考单元测试。

[https://github.com/weiwenhao/algorithm/blob/master/test/BinarySearchTreeTest.php](https://github.com/weiwenhao/algorithm/blob/master/test/BinarySearchTreeTest.php)

## 算法实现

[https://github.com/weiwenhao/algorithm/blob/master/src/BinarySearchTree.php](https://github.com/weiwenhao/algorithm/blob/master/src/BinarySearchTree.php)


## 补充

由于php没有像js一样的字面量对象或者c一样的struct。因此直接使用对象来表示树中的结点

```php
class BiTNode
{
    public $data;
    public $left;
    public $right;

    public function __construct($data, $left = null, $right = null)
    {
        $this->data = $data;
        $this->left = $left;
        $this->right = $right;
    }
}
```


在查找的时候指出了，二叉查找树的查询的时间复杂度并不是严格意义上的O(logn) 是因为有这样的情况发生， 假设需要插入 **12, 10, 9, 5, 4, 1**这几个数据，那么我们会得到这样一颗歪脖子树

![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0e77d4df1?w=628&h=729&f=jpeg&s=29661)
此时的时间复杂度俨然已经变成了O(n)，不过对于这样的问题自然已经有解决方案。下一节将会在**AVL树**和**红黑树**这两种解决方案中选一种来BB~

当然二叉查找树依旧是各种树的根基，还请认真理解。
![](https://user-gold-cdn.xitu.io/2018/4/20/162e36b0e7d2e92c?w=788&h=162&f=jpeg&s=22352)
