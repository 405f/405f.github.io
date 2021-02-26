---
title: redis-skiplist
date: 2020-01-06 22:37:02
tags: [redis]
---

### AVL 平衡树

#### 定义
左右子树的高度差不超过1 

#### 特性

左右子树的值有如下的特性, 因此 AVL 树在查询和排序方面有优越性,时间复杂度是 logn


```

left < head <right 

```
#### 一些问题,
由于AVL 为了保持, 上面的特性, 因此AVL 树在Insert , Delete 过程中涉及到了,节点的位置调整, 这个也就是左旋,右旋,(我没有弄懂这个,写不出来)



### skiplist

#### redis 引入跳表是为了解决什么问题
为了在大量的数据中位置数据的有序,同时要在数据的查询,删除,过程 取得一个平衡的位置,使得insert, delete 不会有很大的时间复杂度,而跳表具有这个特性, 并且从编码实现方面来说要比红黑书,AVL, B tree , B plus tree 简单的多.


#### redis 在那些地方使用了跳表

####  redis 的有序数据集的底层实现,都使用了跳表 sorted set(有序字典)

#### 大概的实现
##### skiplist 就是一个链表
其中从链表的一个Node 来看,他是一个高层楼房, 其中每一层 都串联到 同样海拔高度的 Node 的楼房,



![Alt text](https://lin19999.oss-cn-beijing.aliyuncs.com/skiplist.png)

那么查找一个数据的过程变变成了从最高的海拔 ,不停的往下找,同时, 选择前进或者后退



https://redisbook.readthedocs.io/en/latest/internal-datastruct/skiplist.html
