---
title: Generalized ESD异常数据检测实践
date: 2018-05-03 16:58:42
categories: Algorithmn
tags: [Grubb's Test, Generalized ESD, Anomaly Detection]
mathjax: true
#cover: banner-write-in-es6.jpg
---

ESD（Extreme Studentized Deviate）检验方法是E. Grubbs博士在1950年提出的离群点（Outlier）检验方法，它具有如下特点：

- 仅适用于服从正态分布的数据集，因此使用前注意检查数据分布特性
- 可判断数据集是否存在离群点，但不能统计出有哪几个离群点

如果要找出所有异常点，就要用到B. Rosner教授在1983改进提出的Generalized ESD检验方法。简单来讲，它就是把ESD检验方法重复K次，以达到检验多个离群点的目的。

# t-分布概念

参照[Scipy工具库](https://docs.scipy.org/doc/scipy-0.14.0/reference/generated/scipy.stats.t.html)

# ESD检验方法

利用样本均值$\mu$和样本标准差$s$来定义检验统计量$G$：

$$ G = \frac{\max_{i = 1,..,n} \left | x_i - \bar{x} \right | }{s} $$


对于给定数据集$X\sim N(\mu ,\sigma^2)$，**没有离群点**的假设以某个显著水平的条件是

$$ G >
\frac{n-1}{\sqrt{n}}
\sqrt{
  \frac
  {t \left (\frac{\alpha}{2n}, n-2 \right )^2}
  {\left ( n-2 \right ) + t\left ( \frac{\alpha}{2n}, n-2 \right )^2} }
$$

其中，$t\left (\alpha/2n, n-2 \right)$表示自由度$\nu = n-2$的t-分布，在显著水平$\alpha/2n$时的临界值。这里提到的显著水平$\alpha/2n$，表示大于$\alpha/2n$的可能性存在离群点，即置信度为$1 - \alpha/2n$。

> 举例来说，如果$n = 9$、$\alpha = 0.1$，则自由度$\nu = 7$、显著水平$\alpha / 2n \approx 0.0056$。利用Python计算得到t-分布的临界值：

```python
import scipy.stats as stats

stats.t.ppf(0.0056, 7)
>>> -3.4157443434384374
```

> 因此，当$G > 2.108$时认为数据集$X$至少存在一个离群点。

# G-ESD检验方法

假设观察数据集存在离群点数量的上限为$r$，则G-ESD检验实质上就是执行$r$次相互独立的假设检验：

> $H_0$： 数据集不存在离群值
> $H_a$： 数据集至多存在$r$个离群值

与ESD检验类似，定义检验统计量$R_j$：

$$ R_j = \frac{\max_{i=1,..,n-j+1} \left | x_i - \bar{x} \right | }{s} $$

其中，$\mu$表示样本均值，$s$表示样本标准差。每次剔除致使$\left | x_i - \bar{x} \right |$最大的$x_i$后，再计算剩余$n-1$个数据对应的$R_{j+1}$，这样$R_1,R_2,...,R_r$便能全部求得。

然后，定义显著水平

# 实战演练

G-ESD检验方法使用时，必须先确定观察数据集是否服从正态分布，以决定有无必要进行离群值检验。例如，[样例数据集](https://www.itl.nist.gov/div898/handbook/datasets/ROSNER.DAT)的正态概率图如下：

![正态概率图](https://focalab.oss-cn-hangzhou.aliyuncs.com/img/normal_probability_plot%20.png)

```python
import scipy.stats as stats
import pylab
import matplotlib.pyplot as plt
plt.style.use('fivethirtyeight')

stats.probplot(dataset, dist="norm", plot=pylab)
pylab.show()
```

可以看到，样例数据集并不符合正态分布的假设，说明它可能存在的一个或多个离群值，有待进一步检验得出结论。

# 参考文章

* [Generalized ESD Test for Outliers](https://www.itl.nist.gov/div898/handbook/eda/section3/eda35h3.htm)
* [Seasonal Hybrid ESD笔记](https://blog.csdn.net/huangbo10/article/details/51942006)
* [Statistics in Python](http://www.scipy-lectures.org/packages/statistics/index.html)


