---
layout: post
title: GNU datamash介绍
---
## Introduction
[GNU datamash](http://www.gnu.org/software/datamash/)是一个在命令行下做基本统计汇总（例如均值，标准差，quantils，最大最小值等）的简单工具，对于经常需要在命令行下做一些data valadation和基本的统计汇总的人来说是一个非常有用的工具。

```
$ seq 12|paste - - -|datamash sum 1 sum 2 sum 3
22      26      30
```

同时还可以分组进行以上的统计：

```
 $ seq 12|paste <(printf "%s\n" a b b c c c) - - | datamash -g 1 sum 2 mean 3
 a       1       2
 b       8       5
 c       27      10
```

以前这种工作一般是写一些one-liner或者小脚本然后做alias实现的，实话说写多了容易忘。有了datamash之后就能轻松搞定了，速度也快了不少。

## Install
安装没什么可说的，debian的软件仓库里有，编译也就是普通的./configure make make install三部曲。

## Examples
datamash 的使用方法和一般的命令行程序相似，格式为datamash [options] operation column [operation2 column2] ...
以下用例子说明使用方法：
```
$ seq 12|paste - - -|datamash sum 1 sum 2 sum 3
22      26      30
```
首先sum就是operation，1 2 3都是列号。上面的命令就是对1，2，3列分别求和。
当然operation不止sum，datamash --help就可以看到，其中与数值统计相关的有：
```
Numeric Grouping operations:
  sum, min, max, absmin, absmax  ##求和，最小值，最大值，绝对值最小，绝对值最大
Statistical Grouping operations:
  mean, median, q1, q3, iqr, mode, antimode  ##平均，中位数，第一个四分位，第三个四分位，四分位间距，众数
  pstdev, sstdev, pvar, svar, mad, madraw    ##总体标准差，样本标准差，总体方差，样本方差，Median Absolute Deviation，Median Absolute Deviation raw
  pskew, sskew, pkurt, skurt, dpo, jarque    ##这里不太懂，可以去看文档。
```
