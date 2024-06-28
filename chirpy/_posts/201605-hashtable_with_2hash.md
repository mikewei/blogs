---
title: 双哈希短链分配算法浅析
description: >-
  Get started with Chirpy basics in this comprehensive overview.
  You will learn how to install, configure, and use your first Chirpy-based website, as well as deploy it to a web server.
author: mikewei
date: 2016-05-13 23:15:00 +0800
categories: [数据结构与算法]
tags: [数据结构与算法]
pin: true
---
### 双哈希短链分配算法浅析
[数据结构与算法][2016-05-13 23:15:00 +0800]


#### 问题背景

[Balls-into-bins][1] 问题是计算机科学应用中一个经典的概率问题，最典型的是分析随机分配算法中的不均衡性。在该wiki的[讨论][3]中，同时给出一种`部分随机分配`(partially random allocation)的优化，可以使不均衡性降低。它应用到链式哈希表的设计中，就是在写入key时使用2个不同的哈希函数计算出2个bucket值，然后比较两个bucket中的链表长度，选择较短的那个bucket的链表进行写入。我们想分析的问题就是，该优化方法能否有效优化常规链式哈希表的性能。

#### 性能分析

比较直观地分析，基于最短链分配的哈希算法基本不会影响空间效率，对写入的时间效率影响也不大，只是对查找的时间效率可能有一定影响。可以从两个指标来评估查找时间效率：平均查找次数，最大查找次数。对于服务器开发来说，前者最为重要因为它决定了整体的查找性能(qps)，后者对延时有一定影响但一般来说并不十分重要。首先我通过算法模拟的方式，对这两个指标做了一个量化测试。

下图是两种算法在不同数据规模N下(设哈希表的桶数也为N)的平均查找次数对比。可以看出基于双哈希算法的平均查找次数略高于普通算法，且与数据规模基本无关。

![avg_depth](/res/201605-hashtable_with_2hash/linked_ht_1vs2_avg_depth.png)

类似的，下图对比两种算法下的最大查找次数(查找一个Key最坏情况下需要的查找次数)。两种算法都随着N的增长缓慢增长(对数级)，并且双哈希算法比普通算法增长略微慢一些，实际在$$$2^{20}$$$规模以下无明显优势。

![max_depth](/res/201605-hashtable_with_2hash/linked_ht_1vs2_max_depth.png)

这点可以从数学分析上做个形式的分析，普通哈希算法下(大概率下)最大查找次数为[[*][2]]:
$$
M1 = \frac {log(N)} {log\;log(N)}
$$

双哈希算法下的最大查找次数为[[*][3]]：
$$
M2 = \frac {2\;log\;log(N)} {log(2)}
$$

实际有：
$$
\begin{split}
log(M1) &= log\,log(N) - log\,log\,log(N) \\\\
        &\approx O(log\,log(N)) \\\\
        &= O(M2)
\end{split}
$$

所以M2比M1相对N增长更慢，近似于$$$log(N) \rightarrow log\,log(N) $$$的关系。

#### 结论

基于双哈希最短链分配的哈希表算法，在实际工程中并不是很实用，平均查找性能略低代表并不能带来吞吐性能上的提升，它仅可能在对最坏查找次数比较敏感，并且总体数据规模非常大的场景下才有应用意义。


[1]: https://en.wikipedia.org/wiki/Balls_into_bins
[2]: https://en.wikipedia.org/wiki/Balls_into_bins#Random_allocation
[3]: https://en.wikipedia.org/wiki/Balls_into_bins#Partially_random_allocation
