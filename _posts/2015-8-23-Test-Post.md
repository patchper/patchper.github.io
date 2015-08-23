---
layout: post
title: GNU datamash介绍
---
## Introduction
GNU datamash(http://www.gnu.org/software/datamash/)是一个在命令行下做基本统计汇总（例如均值，标准差，quantils，最大最小值等）的简单工具，对于经常需要在命令行下做一些data valadation和基本的统计汇总的人来说是一个非常有用的工具。

> seq 12|paste - - -|datamash sum 1 sum 2 sum 3
> 22      26      30

同时还可以分组进行以上的统计：

> seq 12|paste <(printf "%s\n" a b b c c c) - - | datamash -g 1 sum 2 sum 3
> a       1       2
> b       8       10
> c       27      30

以前这种工作一般是写一些one-liner或者小脚本然后做alias实现的，实话说写多了容易忘。有了datamash之后就能轻松搞定了，速度也快了不少。
