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
安装没什么可说的，debian的软件仓库里有，编译也是普通的configure make make install三部曲。

## Examples
datamash的使用方法和一般的命令行程序相似，格式为`datamash [options] operation column [operation2 column2] ...`  
以下用例子说明使用方法：
```
$ seq 12|paste - - -|datamash sum 1 sum 2 sum 3
22      26      30
```
首先`sum`就是operation，1 2 3都是列号。上面的命令就是对1，2，3列分别求和。
当然operation不止`sum`，`datamash --help`就可以看到，其中与数值统计相关的有：
```
Numeric Grouping operations:
  sum, min, max, absmin, absmax  ##求和，最小值，最大值，绝对值最小，绝对值最大
Statistical Grouping operations:
  mean, median, q1, q3, iqr, mode, antimode  ##平均，中位数，第一个四分位，第三个四分位，四分位间距，众数
  pstdev, sstdev, pvar, svar, mad, madraw    ##总体标准差，样本标准差，总体方差，样本方差，Median Absolute Deviation，Median Absolute Deviation raw
  pskew, sskew, pkurt, skurt, dpo, jarque    ##这里不太懂，可以去看文档。
```
更多的时候需要的是对某一列进行分类汇总处理，这种情况在awk，perl里就要使用关联数组，但在datamash中只要在选项中使用`-g`参数指定分组的列号（可以有多个）即可。
```
 $ seq 12|paste <(printf "%s\n" a b b c c c) - - | datamash -g 1 sum 2 mean 3
 a       1       2
 b       8       5
 c       27      10
```
以上的例子即是按第一列分组，并计算第二列的和以及第三列的均值。  
值得注意的是，和uniq相似，如果输出没有排序，则分组的结果就会不正确：
```
$ seq 12|paste <(printf "%s\n" a c b c b c) - - | datamash -g 1 sum 2 mean 3
a       1       2
c       3       4
b       5       6
c       7       8
b       9       10
c       11      12
```
这时就要使用`-s`按照group指定的列号先排序再分组，也可以使用sort排序后再pipe到datamash中。同时如果使用 `-i`参数，排序以及分组都会是case insensitive的。
```
$ seq 12|paste <(printf "%s\n" a c b c b c) - - | datamash -s -g 1 sum 2 mean 3
a       1       2
b       14      8
c       21      8
$ seq 12|paste <(printf "%s\n" a c b C b C) - - | datamash -s -g 1 sum 2 mean 3
C       18      10
a       1       2
b       14      8
c       3       4
$ seq 12|paste <(printf "%s\n" a c b C b C) - - | datamash -s -i -g 1 sum 2 mean 3
a       1       2
b       14      8
c       21      8
```
有时我们的输入表格是有表头的，这种表格在awk中就要指定NR做不同的处理，但datamash中有着专门处理表头的参数：  
```
--header-in     ##表示输入表格中有表头
--header-out    ##表示输出表头
-H --headers    ##上面两个参数的和
```
首先输出表头并不需要原表格有表头，如果没有表头就会对统计信息做简单描述作为表头。如果有表头而不指定`--header-in`则会因为数据类型不符出现错误。
```
$ seq 12|paste <(printf "%s\n" a b b c c c) - - | datamash --header-out -g 1 sum 2 mean 3
GroupBy(field-1)        sum(field-2)    mean(field-3)
a       1       2
b       8       5
c       27      10

$ seq 12|paste <(printf "%s\n" a b b c c c) - - | cat <(printf 'group\tlane1\tlane2\n') - | datamash -g 1 sum 2 mean 3
datamash: invalid numeric input in line 1 field 2: 'lane1'

$ seq 12|paste <(printf "%s\n" a b b c c c) - - | cat <(printf 'group\tlane1\tlane2\n') - | datamash --header-in -g 1 sum 2 mean 3
a       1       2
b       8       5
c       27      10

$ seq 12|paste <(printf "%s\n" a b b c c c) - - | cat <(printf 'group\tlane1\tlane2\n') - | datamash --headers -g 1 sum 2 mean 3
GroupBy(group)  sum(lane1)      mean(lane2)
a       1       2
b       8       5
c       27      10
```
由于datamash本身就有用作data valadation的功能，所以它对于数据的要求十分严格，有缺失数据或数据类型不正确就会报错，这种时候就会提醒我们做一些前处理，这比什么都不说就用默认值代替的awk和perl不知道高到哪去了：
```
$ seq 11|paste <(printf "%s\n" a b b c c c) - - | datamash --header-out -g 1 sum 2 mean 3
GroupBy(field-1)        sum(field-2)    mean(field-3)
a       1       2
b       8       5
datamash: invalid numeric input in line 6 field 3: ''

$ seq 11|sed '1ia'|paste <(printf "%s\n" a b b c c c) - - | datamash --filler=0 --header-out -g 1 sum 2 mean 3
datamash: invalid numeric input in line 1 field 2: 'a'
```
有时输入文件不是tab分割的，就要使用`-t`参数指定分割：
```
$ seq 12|paste <(printf "%s\n" a b b c c c) - - |sed 's/\t/,/g' |datamash -t"," --filler=0 --header-out -g 1 sum 2 mean 3
GroupBy(field-1),sum(field-2),mean(field-3)
a,1,2
b,8,5
c,27,10
```
除了对数据的统计以外，datamash还能做一些简单的文本和文件操作  
例如对输入文件的转置：
```
$ seq 12|paste <(printf "%s\n" a b b c c c) - - | cat <(printf 'group\tlane1\tlane2\n') - |datamash transpose
group   a       b       b       c       c       c
lane1   1       3       5       7       9       11
lane2   2       4       6       8       10      12
```
同时还可以对输入文件进行文本统计与操作，这在日常工作中也是非常常用的
```
Textual/Numeric Grouping operations:
  count, first, last, rand        ##计数，第一个值，最后一个值，随机的一个值
  unique, collapse, countunique   ##用逗号分隔的唯一值，用逗号分隔的所有值，值的种类数
```
例子如下：
```
$ cat test 
chr10   100154974       100155064       NR_031619
chr10   101635333       101637093       NM_015221
chr10   101639567       101640118       NM_015221
chr10   101643767       101643966       NM_015221
chr10   101645443       101645623       NM_015221
chr10   101646056       101646389       NM_015221
chr10   101648581       101648710       NM_015221
chr10   101654702       101654807       NM_015221
chr10   101656023       101656154       NM_015221
chr10   101657842       101658042       NM_015221
chr10   101658499       101658517       NM_015221
chr10   101659675       101659823       NM_015221
chr10   101667751       101667851       NM_015221
chr10   101668709       101668903       NM_015221
chr10   101686965       101687056       NR_024130
chr10   101687916       101687997       NR_024130
chr10   101714970       101716962       NM_015221
chr10   101715430       101715589       NR_024130
chr10   101716238       101716348       NR_024130
chr10   101716545       101718755       NR_024130
$ cat test |datamash -g 4 count 4      ##统计第四列的个数
NR_031619       1
NM_015221       13
NR_024130       2
NM_015221       1
NR_024130       3
$ cat test |datamash -g 1,4 unique 2 unique 3
chr10   NR_031619       100154974       100155064
chr10   NM_015221       101635333,101639567,101643767,101645443,101646056,101648581,101654702,101656023,101657842,101658499,101659675,101667751,101668709  101637093,101640118,101643966,101645623,101646389,101648710,101654807,101656154,101658042,101658517,101659823,101667851,101668903
chr10   NR_024130       101686965,101687916     101687056,101687997
chr10   NM_015221       101714970       101716962
chr10   NR_024130       101715430,101716238,101716545   101715589,101716348,101718755
$ cat test |datamash -g 1,4 countunique 2 countunique 3
chr10   NR_031619       1       1
chr10   NM_015221       13      13
chr10   NR_024130       2       2
chr10   NM_015221       1       1
chr10   NR_024130       3       3
```
## Further Readings
以上只是对GNU datamash的一个简单介绍，更多的介绍以及帮助信息可以在GNU datamash的主页<http://www.gnu.org/software/datamash/>找到，同时官方也有一份更详细的Example学习手册<http://www.gnu.org/software/datamash/examples/>，更加完整的帮助可以用man datamash或者info datamash得到。

********
