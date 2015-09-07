---
layout: post
title: 笔记：置换检验
tags: permutation\_test
---

罪过罪过，搞了这么久生信居然不知道置换检验，以前碰到非参都是秩和检验对付过去的，还真不知道如此神物，而且这东西居然在R in action里就有，我看的时候愣是跳过去了……(|||ﾟдﾟ)

置换检验的思想是简单的：如果要检验两组数据的某统计量是否相等，就用置换法穷尽这两组数据的可能情况（或者取大量的sample），得到这个统计量的经验分布，然后看我们需要检验的这个p值是否在置信区间内就可以了。

具体的东西在R in action里都有了，我只是做一个简单的benchmark，用到的数据如下：

Sample|lengths
------|------
temp3|29,30,30,32,33,33,33,33,34,34,34,35,35,36,42,43,45,49,53,53,56,58,59,60,62,64,64,68,68,77,78,80,92,97,97
temp4|107,113,114,120,121,28,33,33,37,38,38,38,41,46,48,54,69,73,77,82,84,87,91,80

```R
> temp3 <- c(29,30,30,32,33,33,33,33,34,34,34,35,35,36,42,43,45,49,53,53,56,58,59,60,62,64,64,68,68,77,78,80,92,97,97)
> temp4 <- c(107,113,114,120,121,28,33,33,37,38,38,38,41,46,48,54,69,73,77,82,84,87,91,80)
> length <- c(temp3,temp4)
> sample <- c(rep("temp3",length(temp3)),rep("temp4",length(temp4)))
> data <- data.frame(length,sample)
> library(lattice)
> stripplot(sample~length,data=data)
```
![stripplot]({{ site.baseurl }}/images/permutation-stripplot.png)

以下是一些benchmark：

```
> t.test(length~sample,data=data)

	Welch Two Sample t-test

data:  length by sample
t = -2.3046, df = 36.448, p-value = 0.02699
alternative hypothesis: true difference in means is not equal to 0
95 percent confidence interval:
 -31.318148  -2.005662
sample estimates:
mean in group temp3 mean in group temp4 
           52.17143            68.83333 
> wilcox_test(length~sample,data=data,ditribution="exact")

	Asymptotic Wilcoxon-Mann-Whitney Test

data:  length by sample (temp3, temp4)
Z = -2.131, p-value = 0.03309
alternative hypothesis: true mu is not equal to 0

> oneway_test(length~sample,data=data,ditribution="exact")

	Asymptotic Two-Sample Fisher-Pitman Permutation Test

data:  length by sample (temp3, temp4)
Z = -2.3818, p-value = 0.01723
alternative hypothesis: true mu is not equal to 0
```
从图上看还是有差异的，Sample数量也不多，所以我会更信任oneway\_test一些。这东西对sample数量，重复值的数量的要求都比秩和检验低，保留的信息也多些，虽然秩和检验对out-lier的容忍度要高，但考虑到实际上生物数据里out-lier并不多，从生物信息的一般要求来看基本能够代替秩和检验。

同时oneway\_test还能处理多组数据，这也是其优势的地方所在。

以下代码是一个permutation test的原理演示，但不知为什么其pvalue总比coin算出来的小，仅供记录用：

```R
temp3 <- c(29,30,30,32,33,33,33,33,34,34,34,35,35,36,42,43,45,49,53,53,56,58,59,60,62,64,64,68,68,77,78,80,92,97,97)
temp4 <- c(107,113,114,120,121,28,33,33,37,38,38,38,41,46,48,54,69,73,77,82,84,87,91,80)
length <- c(temp3,temp4)
sample <- c(rep("temp3",length(temp3)),rep("temp4",length(temp4)))
data <- data.frame(length,sample)
find.mean <- function(x){ mean(x[sample=="temp3",2])-mean(x[sample=="temp4",2]) }
results <- replicate(1999,find.mean(data.frame(sample,sample(data[,1]))))
hist(results,breaks=50,prob=TRUE)
lines(density(results))
p.value <- 1-sum(results>mean(data[sample=="temp3",1])-mean(data[sample=="temp4",1]))/2000
```
