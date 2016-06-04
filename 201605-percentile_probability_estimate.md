### 一种基于概率估计的快速percentile算法
[数据结构与算法][2016-05-29 17:40:00 +0800]

计算percentile在后台服务中挺常见的，一个最典型场景就是用于监控系统中计算请求延时(latency)的percentile值，常用的比如有99.9%分位延时、99%分位延时。这个问题在计算机科学里叫[“选择问题”](https://en.wikipedia.org/wiki/Selection_algorithm)。

这个问题有许多常见的解法，比如局部排序算法、分区算法、分桶算法。局部排序算法(partial selection sort)，简单来说就是建立一个最小堆，堆的大小能覆盖关心分位内的数据，它的时间复杂度O(nlog(k))（n为统计数量、k为堆大小），空间复杂度为O(k)；分区算法(partition-based selection)，如上文中介绍的[QuickSelect](https://en.wikipedia.org/wiki/Quickselect)，时间复杂度O(n)，空间复杂度O(n)；分桶算法是一个工程上常用的优化方法，按延时值分桶，损失精度以换取速度和简单，时间复杂度O(n)（极快），空间复杂度O(m)（m为分桶数）。

上述这些方案在一些场景上都有一些缺点，就是实现偏复杂，要么需要较多地计算，要么需要较多的内存，在一些对精度要求不是特别高的场景下，有点杀鸡用牛刀的感觉。有没有一种更简洁有效的方法呢？我对此问题做了些思考，找到一种基于概率估计的方法。

这个方法的思路很简单，就是计算n个统计量的最大值(Max)，然后以此值作为对percentile对应值的估计，时间和算法复杂度都很底。思路是比较简单，是否真的靠谱，n值应该怎么来定，还得需要严谨的分析。

#### 数学分析

首先我们把延时值的概率分布假设为[指数分布](https://en.wikipedia.org/wiki/Exponential_distribution)来进行建模。近似为指数分布是合情理的，特别是，往往高延时是由少部分的请求组成（并且后面我们也会看到此建模的结果对非指数分布也基本适用）。

我们先回顾下，指数分布的概率密度函数和累积分布函数：

$$
\begin{split}
P(x) &= \lambda\,e^{-\lambda x} \\\\
F(x) &= \int\_0^x P(x)\,dx = 1 - e^{-\lambda x}
\end{split}
$$

假设我们用n个延时值的最大值作为估计，即有$$$ X = Max(X\_1, X\_2, ..., X\_n) $$$，可得X的累积分布函数：

$$
\begin{split}
F\_X(x) &= P(X < x) \\\\
        &= P(Max(X\_1, X\_2, \cdots, X\_n) < x) \\\\
        &= P(X\_1 < x)\,P(X\_2 < x) \cdots P(X\_n < x) \\\\
        &= \left( F(x) \right)^n \\\\ 
        &= \left( 1 - e^{-\lambda x} \right)^n
\end{split}
$$

从X的累积分布函数我们可以直接得出X的期望E(X) （具体推导有点复杂，这里不详述，可以[参考这里](http://www.stat.berkeley.edu/~mlugo/stat134-f11/exponential-maximum.pdf)）：

$$
\begin{split}
   E(X) &= \int\_0^\infty 1 - F\_X(x)\;dx \\\\
        &= \int\_0^\infty 1 - \left( 1 - e^{-\lambda x} \right)^n \;dx \\\\
        &= \frac 1\lambda \,\left( 1 + \frac 1 2 + \frac 1 3 + \cdots + \frac 1 n \right) \\\\ 
        &= \frac 1\lambda \sum\_{i=1}^n \frac 1 i \\\\ 
\end{split}
$$


E(X)是我们的估计值，我们估计的目标是什么来？是perceltile值（如99.9%分位对应的延时值，分位数下面用p表示），它的值为：

$$
\begin{split}
F^{-1}(p) &= \frac {-ln(1-p)} \lambda \\\\
\end{split}
$$

使 $$$ E(X) = F^{-1}(p) $$$，我们可以求解n值，实际上，这个等式可以简化下：

$$
\begin{split}
              E(X) &= F^{-1}(p) \\\\
 \Rightarrow  \sum\_{i=1}^n \frac 1 i &= -ln(1-p) \\\\
\end{split}
$$

从以上公式可看出，给定percentile的百分位p值，我们可以很容易使用程序计算出n值。也即当取这个n值时，我们使用最大值的期望(平均值)可以无偏地估计percentile值。到这里可能有人会问，均值没问题了，方差怎么办，会有大的误差吗？其实无需担心，方差的问题，我们默认已经通过[中心极限定理](https://en.wikipedia.org/wiki/Central_limit_theorem)来解决了：我们不会直接使用max值来做最后的估计，而是不断地算多个max值后求平均，根据中心极限定理最终平均值的方差会随着样本量的大量增加而大大降低。

除了数学推导，我们还可以从下面这张通过程序模拟的概率分布图来更形象地理解估计过程：

![max_pdf_vs_orig_cdf](/res/201605-percentile_probability_estimate/max_pdf_vs_orig_cdf.png)

黑色曲线（左侧y轴）是待统计指标的积累分布，我们的目标是想统计左侧y轴等于一定百分位时，对应的x轴的值。红色是我们使用最大值做为估计量的概率密度分布，可以看出，它的期望值(均值)即对应于x轴上我们的估计目标值。

以上我们是以指数分布对模型进行分析，实际测试对正态分布也用同样的估计方法，误差仍然非常小，所以此方法对真实场景还是比较可靠的。

#### 程序实现

程序实现上是格外简单的，我们以python为例，首先看一下根据百分位计算参数n的值的实现：

```python
import sys
import math
import string

def get_n_for_percentile(percentile):
    n = 1
    expected_value = 0
    percentile_value = -math.log(1 - string.atof(percentile)/100)
    while expected_value < percentile_value:
        expected_value = expected_value + 1.0/n
        n = n + 1
    return n

if __name__ == '__main__':
    print get_n_for_percentile(sys.argv[1])
```

可以像下面这样运行它：

	$ python n_for_percentile.py 99
	57
	$ python n_for_percentile.py 99.9
	562

从运行结果我们可看出，如果要估计99%分位值，我们需要n=57，即统计57个样本的最大值，如果要估计99.9%，则需要n=562。

下面再看一下实际统计的代码，也十分简单，计算上它只需要做比较赋值等运算，空间上则仅需要3个状态变量：

```python
import sys
import random
import n_for_percentile

n = n_for_percentile.get_n_for_percentile('99.9')

# only three status variables needed
counter = 0
max_of_n = 0
sum_of_max = 0

def percentile_estimate_proc(latency):
    global n, counter, max_of_n, sum_of_max
    if latency > max_of_n:
        max_of_n = latency
    counter = counter + 1
    if counter % n == 0:
        sum_of_max = sum_of_max + max_of_n
        max_of_n = 0

def get_cur_percentile_estimate():
    global n, counter, sum_of_max
    return sum_of_max / (counter / n)

if __name__ == '__main__':
    for i in range(0, 100000):
        # use uniform distribution here just for simplicity
        percentile_estimate_proc(random.uniform(0, 1000))
    print get_cur_percentile_estimate()
```

另外，此算法也能很容易适用于集群汇总统计，即合并多台服务器的统计值，实际只需将上述代码中counter与sum\_of\_max进行汇总合并即可。

#### 总结

本文给出一种基于概率估计的计算percentile值的方法，时间和空间复杂度都极低，十分适合在对精度要求不是太高，又需要尽量高效和实现简单的业务场景。




