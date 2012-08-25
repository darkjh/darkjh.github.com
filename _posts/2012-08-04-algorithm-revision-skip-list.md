---
layout: post
title: "Algorithm Revision: Skip List"
description: ""
tags: [algorithm, learning notes]
---

平衡查找树能够让我们在对数时间内管理一个动态数据集, 但是其实现相对比较复杂. 给一小时时间, 不看任何资料, 恐怕很难实现其中任何一种. 上课的时候老师曾经讲过skip list, 当时觉得很无厘头, 以为是纯教学目的的东西. 这次看MIT的课, 才发现skip list也是一种能够在对数时间内(几乎)管理动态数据集的数据结构, 而且实现起来比较容易.

## 结构 ##

skip list顾名思义是一种链表结构. 一般使用链表来存储数据的话, 即便是已排序的, 查询操作也需要`\(\Theta(n)\)`的时间. 这是因为链表不具有随机存取的特性(random access), 无法使用二分查找. 而skip list的思路则是利用多个链表, 跳跃式的存储数据, 如图:

![](/images/2012-08/Skip-list-1.png)

可以把链表中的元素看成是公交车的车站, 含有全部元素的底部列表就是慢车的线路, 它每站都停. 而上层的链表, 则是快车的线路, 它只停靠某些站点. 元素的查找操作就可以看作是从终点站开始的一次出行, 通过乘坐快车加慢车的组合, 快速的到达目的地. 比如对于上图的skip list, 我们要去66号车站, 那么可以先坐快车到42, 然后转慢车到66.

其实skip list本质上是对树的一种链表式的模仿:

    1---------------5---------------*
    1-------3-------5-------7-------*
    1---2---3---4---5---6---7---8---*

这样看起来不就是一颗二叉查找树么..

这里给出实现的结构体代码:

{% highlight java %}
/* data stucture for skip list */
/* node structure */
struct node {
  int data;
  int height;
  struct node *next;
  struct node *down;
};
typedef struct node sl_node;

/* header of list */
typedef struct {
  int height;
  sl_node *head;
  sl_node *bottom;
} sl_head;
{% endhighlight %}

## 查找 ##

一般化的skip list查找算法是从最上层的链表开始, 经过一系列比较, 往下达到底层要找的元素. 我们假设链表有两层`\(L_1, L_2\)`, 而且第二层的元素均分了原始的链表, 那么查找过程在底层链表最多走`\(\frac{|L_{1}|}{|L_{2}|}\)`步. 第一层的链表含有集合所有的元素(n个), 那么显然总的查找过程要经过的节点个数是`\(|L_{2}|+\frac{n}{|L_{2}|}\)`. 这个式子在`\(|L_{2}|=\frac{n}{|L_{2}|}\)`时取得最小值, 即`\(L_2 = \sqrt{n}\)`. 最佳的两层skip list如下图, 每次查询要走`\(2\sqrt{n}\)`个节点

![](/images/2012-08/Skip-list-2.png)

这个结论可以推广到k层的skip list, 则每次查询要经过`\(k\sqrt[k]{n}\)`个节点. 那么怎么确定k的值呢? 首先我们希望查找操作是对数的, 和BBST的一样, 我们又发现k元素是单独在外面的因子, 自然的我们猜测`\(k=lg(n)\)`, 那么原式变成

`\[lg(n)*n^{\frac{1}{lg(n)}}\]`

这个式子看起来... 有点复杂, 但是其实它就是`\(2lg(n)\)`! 你看, __对数时间__! 一共有`\(lg(n)\)`层链表, 是不是很像平衡二叉树?

这个形状的skip list是理想的, 因为当我们插入或者删除节点的时候, 这个形状就会被破坏. 当然, 平均来看, 我们仍旧有非常大的概率在对数时间里完成查找.

查找函数:
{% highlight java %}
sl_node *find(const sl_head *list, const int key) {

  sl_node *p = list->head;

  while(p != NULL) {
    while(p->next != NULL && key > p->next->data)
      p = p->next;

    /* if the node is found */
    if(p->next != NULL && key == p->next->data) {
      p = p->next;
      /* get down to base level */
      while(p->down != NULL)
	p = p->down;
      return p;
    }
    p = p->down;
  }
  return NULL;
}
{% endhighlight %}

## 插入和删除 ##

插入元素需要考虑的主要的问题是: 如何确定新插入的元素的高度(处在第几层链表中). 最简单的一个思路是在最底层链表插入完成后, 抛一个硬币, 如果是正面, 那么这个元素需要提升一层, 如果是反面, 那么插入结束. 这也就是说, 我们以50%的概率把元素插入到更上一层的链表中, 那么:

 + `\(\frac{1}{2}\)`的元素上升1层
 + `\(\frac{1}{4}\)`的元素上升2层
 + `\(\frac{1}{8}\)`的元素上升3层

我们希望的层数是对数的, 比如`\(clg(n)\)`层. 那么其概率是
`\[
\frac{1}{2^{clg(n)}}=\frac{1}{n^{c}}
\]`
我们可以控制常数c, 比如设其为10, 那么skip list的层数依然是对数的, 但是一个元素达到这一高度的概率却是非常非常低的(`\(\frac{1}{n^{10}}\)`). 换句话说, 这种插入策略在绝大多数情况下能够保证对数高度的skip list.

相比插入操作, 删除就没有什么奥妙了. 我们从上到下删除所有该元素的节点即可.

在实现的过程中, 还要考虑几个细节的问题:

 + 我们希望每次搜索都从skip list的"左上角"开始, 所以对于每层的链表, 需要构造一个表头节点, 含有一个无穷小的值
 + 由于插入的元素可能上升到比当前skip list还要高的高度, 意味着在每一层都需要插入这个节点. 所以在向下查找底部链表插入位置的时候, 需要用一个指针数组记录查找的路径, 以方便后续的插入


插入和删除的代码:

{% highlight java %}
/* insert into a skip list */
/* bottom up approach */
int insert(sl_head *list, int key) {

  sl_node *path[list->height + 1];
  sl_node *p = list->head;
  sl_node *down = NULL;
  int i = 0;

  while(p != NULL) {
    while(p->next != NULL && key > p->next->data)
      p = p->next;
    if(p->next != NULL && key == p->next->data)
      return 1;
    path[p->height] = p;
    p = p->down;
  }

  /* insert from the base level */
  p = path[0];
  do {
    /* if it's a new height, create a sentinel node */
    if(i > list->height)
      p = list->head = new_node(-1, i, NULL, list->head);

    /* insertion */
    p->next = new_node(key, i++, p->next, NULL);

    if(down != NULL)
      p->next->down = down;
    down = p->next;

    if(i <= list->height)
      p = path[i];
  } while(getRand(0, 2) > 1);

  /* update the list height */
  list->height = list->head->height;

  /* direct link to the base list */
  if(list->bottom == NULL) {
    list->bottom = list->head;
    while(list->bottom->down != NULL)
      list->bottom = list->bottom->down;
  }

  return 0;
}

/* delete the node containing key */
/* up down approach */
int delete(sl_head* list, int key) {

  sl_node *p = list->head;
  sl_node *to_delete = NULL;

  while(p != NULL) {
    /* go 'right' */
    while(p->next != NULL && key > p->next->data)
      p = p->next;

    /* if the next one is the node, delete it */
    if(p->next != NULL && p->next->data == key) {
      to_delete = p->next;
      p->next = p->next->next;
      free(to_delete);
    }

    /* continue to next level */
    p = p->down;
  }

  return 0;
}
{% endhighlight %}

insert函数比较麻烦的原因就是元素是从下往上插入的. 后来想到可以在插入之前就先计算层数, 然后从上往下插入, 会方便很多(和删除差不多), 懒得改了...

参考[一][1], [二][2], [三][3]

  [1]: http://stackoverflow.com/questions/256511/skip-list-vs-binary-tree
  [2]: http://www.csee.umbc.edu/courses/undergraduate/341/fall01/Lectures/SkipLists/skip_lists/skip_lists.html
  [3]: http://eternallyconfuzzled.com/tuts/datastructures/jsw_tut_skip.aspx
