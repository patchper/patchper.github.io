---
layout: post
title: limma Revisited, 1st
tags: R limma
---

limma是Array时代做DEG分析的经典包，也是bioconductor的重要组成部分。基本上我没有经历过Array时代：还在实验室鬼混的时候师兄师姐做过几个，但是其分析过程现在看来简直儿戏（部分原因恐怕也是因为服务提供商不够负责吧），所以一般来说我用不上limma，但是最近在看subRead-featureCount这个pipeline的时候发现它用的DEG软件居然是limma，这才知道它居然也是可以用来做RNA-Seq的DEG。然后又遇到microRNA以及时间序列分析的一些问题，不少的解决方案都指向这个软件，这才想起要把这个元老级的分析软件拿出来研究。

基本上本文的内容都来自limma的[一篇综述](http://nar.oxfordjournals.org/content/43/7/e47)，当然也还有一些来自于limma的[说明书](http://www.bioconductor.org/packages/release/bioc/vignettes/limma/inst/doc/usersguide.pdf)以及一些网络资源。

由于我实在是不做Array分析，所以limma包内仅与Array处理相关的内容（如前处理部分的Background Correction, Within-Array Normalization以及所有的双色Array）我都会略去少谈。相反，中文网络上很少提到的limma基本原理，与RNA-Seq分析软件的关系，以及如何将limma中的各种函数应用到其他分析中去的话题，我都会尝试多谈一点。

##Introduction
其实最好的对这个包的总结就是这个综述的Fig.1 
![Summary]({{ site.baseurl }}/images/limma_overall.jpg )

在这个图片中提到了分析中的几个关键要素：

* 我们的实验可以得到一个Expression Value的矩阵
* 我们对矩阵中的每一个基因做linear model
* 我们需要对每个基因的liear model的两个统计量面进行估计：
  1. 系数的估计值\\(\hat{\beta_{g}}\\)
  2. 离散性（这里是标准差）的估计值\\(s^2_{g}\*\\)
* 而对这两个统计量的估计要用到以下的统计方法：
  1. 对原始数据进行前处理
  2. 对不同的基因进行量化权重处理
  3. 进行基因间的信息共享
  4. 对整体的离散度进行建模

当然一开始读到这些东西基本上不可能理解，以下就对这些要素做一些解释。

##Expression Value Matrix
这个对于处于二代测序时代的人实际上比较平淡无奇，在RNA-Seq的DEG分析中，最终一定会得到这么一张表，它的行是基因，列是raw count number，然后把它喂给edgeR或者DESeq，这么一张表就是Expression Value Matrix。由于基因表达的高度并行性，这张表的每行都可以看作是独立的，所以我们可以对这张表的每一行（每个基因）做单独处理。

然而由于Array设计本身的复杂性，这个简单明了的数据结构被隐藏的很深，通常要经过若干的前处理步骤才能看得出来。

##Linear Model
这个实际说是*limma*的核心也不为过，limma本身即是(LInear Model for MicroArray)的简称。使用线性模型有三个原因：

  1. 线性模型从原理上兼容现有统计方法
  2. 可以在样品间分享信息
  3. 可以使分析的范围更加广泛

###什么是线性模型
实际上总结图上已经有了描述：\\(E(y\_{g}) = X\\beta\_{g}\\)，但是这个稍微有些不太明白，翻译一下就是\\(y\_{g}=X\\beta\_{g}+\\epsilon\\)，仍然不太明白？我来解释一下。  

\\(y_{g}\\)代表的是某一个基因的Expression Value向量。\\(X\\)是一个矩阵，或者特殊的说，设计矩阵。\\(\\beta\\)是我们需要使用线性模型估计的参数向量，有时具有一定的意义。\\(\\epsilon\\)是残差。  
这些东西里最重要的就是设计矩阵\\(X\\)，使用过edgeR的同学应该不会陌生。先用实例说明一下设计矩阵，考虑一个简单的试验设计：野生型细胞测三次，经过某药物处理后再测三次。

```R
library(limma)
f <- as.factor(c(rep("control",3),rep("treatment",3)))
design <- model.matrix(~f)
colnames(design) <- levels(f)
design
  control treatment
1       1         0
2       1         0
3       1         0
4       1         1
5       1         1
6       1         1
attr(,"assign")
[1] 0 1
attr(,"contrasts")
attr(,"contrasts")$f
[1] "contr.treatment"
```
这里的

\\[
\\begin{bmatrix}
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 1 \\\\
 1 & 1 \\\\
 1 & 1 \\\\
\\end{bmatrix}
\\]
就是设计矩阵，带入上面的\\(y\_{g}=X\\beta\_{g}+\\epsilon\\)中就是：

\\[
\\begin{bmatrix}
 y\_{1} \\\\
 y\_{2} \\\\
 y\_{3} \\\\
 y\_{4} \\\\
 y\_{5} \\\\
 y\_{6} \\\\
\\end{bmatrix}=
\\begin{bmatrix}
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 1 \\\\
 1 & 1 \\\\
 1 & 1 \\\\
\\end{bmatrix}
\\begin{bmatrix}
 \\beta\_{1} \\\\
 \\beta\_{2} \\\\
\\end{bmatrix}+
\\begin{bmatrix}
 \\epsilon\_{1} \\\\
 \\epsilon\_{2} \\\\
 \\epsilon\_{3} \\\\
 \\epsilon\_{4} \\\\
 \\epsilon\_{5} \\\\
 \\epsilon\_{6} \\\\
\\end{bmatrix}
\\]
确定\\(\\beta\\)的方法可以有多种，一般来说使用[最小二乘法](https://zh.wikipedia.org/wiki/%E6%9C%80%E5%B0%8F%E4%BA%8C%E4%B9%98%E6%B3%95)。

然而得出的\\(\\beta\\)有什么意义呢？在本例中把上面的矩阵展开看：

\\[
y\_{1}=\\beta\_{1}+\\epsilon\_{1} \\\\
y\_{2}=\\beta\_{1}+\\epsilon\_{2} \\\\
y\_{3}=\\beta\_{1}+\\epsilon\_{3} \\\\
y\_{4}=\\beta\_{1}+\\beta\_{2}+\\epsilon\_{4} \\\\
y\_{5}=\\beta\_{1}+\\beta\_{2}+\\epsilon\_{5} \\\\
y\_{6}=\\beta\_{1}+\\beta\_{2}+\\epsilon\_{6} \\\\
\\]
由最小二乘法的性质可得

\\[
\\beta\_{1}=\\mu\_{wt} \\\\
\\beta\_{1}+\\beta\_{2}=\\mu\_{treatment}
\\]
所以\\(\\beta\_{1},\\beta\_{2}\\)是对某基因在野生型中的表达量和野生型与处理组表达量之差的一个估计，在这里恰好是这两个量的算术平均值。当然在这个例子中使用线性模型有些杀鸡用牛刀的感觉了，和分组比较相比没有明显的优势，而在更加复杂的实验设计中，线性模型则可以充分利用样品间的关系，做统一的处理。

可以看下其他的设计矩阵：

```R
> library(limma)
> f <- as.factor(c(rep("control",3),rep("treatment",3)))
> design <- model.matrix(~0+f)
> colnames(design) <- levels(f)
> design
  control treatment
1       1         0
2       1         0
3       1         0
4       0         1
5       0         1
6       0         1
attr(,"assign")
[1] 1 1
attr(,"contrasts")
attr(,"contrasts")$f
[1] "contr.treatment"
```
这里可以直接得到野生型中的表达量和处理组表达量的估计值。

下面这个是一个pairwise的实验设计：

Sample|Block|Treatment
:---:|:---:|:-------:
     1|    1|        T
2|1|C
3|2|T
4|2|C
5|3|T
6|3|C

```R
> block <- factor(rep(c(1,2,3),each=2))
> Treatment <- as.factor(rep(c("C","T"),3))
> model.matrix(~0+block+Treatment)
  block1 block2 block3 TreatmentT
1      1      0      0          0
2      1      0      0          1
3      0      1      0          0
4      0      1      0          1
5      0      0      1          0
6      0      0      1          1
attr(,"assign")
[1] 1 1 1 2
attr(,"contrasts")
attr(,"contrasts")$block
[1] "contr.treatment"

attr(,"contrasts")$Treatment
[1] "contr.treatment"
```
\\[\\begin{bmatrix}
 y\_{1} \\\\
 y\_{2} \\\\
 y\_{3} \\\\
 y\_{4} \\\\
 y\_{5} \\\\
 y\_{6} \\\\
\\end{bmatrix}=
\\begin{bmatrix}
 1 & 0 & 0 & 0 \\\\
 1 & 0 & 0 & 1 \\\\
 0 & 1 & 0 & 0 \\\\
 0 & 1 & 0 & 1 \\\\
 0 & 0 & 1 & 0 \\\\
 0 & 0 & 1 & 1 \\\\
\\end{bmatrix}
\\begin{bmatrix}
 \\beta\_{1} \\\\
 \\beta\_{2} \\\\
 \\beta\_{3} \\\\
 \\beta\_{4} \\\\
\\end{bmatrix}+
\\begin{bmatrix}
 \\epsilon\_{1} \\\\
 \\epsilon\_{2} \\\\
 \\epsilon\_{3} \\\\
 \\epsilon\_{4} \\\\
 \\epsilon\_{5} \\\\
 \\epsilon\_{6} \\\\
\\end{bmatrix}
\\]
稍加计算就可以看出\\(\\beta\_{4}\\)是\\(y\_{2}-y\_{1},y\_{4}-y\_{3},y\_{6}-y\_{5}\\)的算术平均，而这也正是我们用传统方法估计的值。

对于更加复杂的实验设计例如Factorial experiment，Multi level experiment等，linear model的能量就更加强大，但同时也要引入新的工具：contrastMatrix和contrasts.fit函数。contrastMatrix通常用来描述对于估计参数的比较信息，其行是估计参数的名字，列是比较的方式，其意义与设计矩阵类似。而contrasts.fit函数则可以将linear model的结果进行进一步比较。

例如下面的Factorial experiment，我们希望看到同一处理下野生型和突变型的不同表现：

Sample|Cell Type|Treatment
---|---|---
1|wt|control
2|wt|treatment
3|mu|control
4|mu|control
5|mu|treatment
6|mu|treatment
```R
> treatment = factor(c("control","treatment","control","control","treatment","treatment"),levels=c("control","treatment"))
> type = factor(c("wt","wt",rep("mu",4)),levels=c("wt","mu"))
> model.matrix(~0+type+treatment:type)
  typewt typemu typewt:treatmenttreatment typemu:treatmenttreatment
1      1      0                         0                         0
2      1      0                         1                         0
3      0      1                         0                         0
4      0      1                         0                         0
5      0      1                         0                         1
6      0      1                         0                         1
attr(,"assign")
[1] 1 1 2 2
attr(,"contrasts")
attr(,"contrasts")$type
[1] "contr.treatment"

attr(,"contrasts")$treatment
[1] "contr.treatment"
```
可以看到，第三列的系数代表的是在treatment条件下wt的反应，第四列是treatment条件下mu的反应
所以

```R
> fit2 <- contrasts.fit(fit,c(0,0,-1,1))
```
就是我们所要的在treatment条件下mu与wt不同的表现。同理

```R
> fit2 <- contrasts.fit(fit,c(-1,1,1,-1))
```
是control条件下mu与wt的不同表现。

---
写到这里也写了不少了，但也还仅仅只提到了limma中的线性模型的部分，但是我们也可以看到limma非常强大的部分了。总体说来，线性模型是limma的核心，设计矩阵与比较矩阵结合使得limma可以很方便的做非常广泛的分析，同时还允许在不同样品中分享信息，同时在简单的设计中也能和普通的处理方法保持兼容。后续的RNA-Seq分析软件edgeR也继承了这一系统，不过这就不在这里提了。

更多的例子可以在limma的[说明书](http://www.bioconductor.org/packages/release/bioc/vignettes/limma/inst/doc/usersguide.pdf)中找到，本页的例子多数也是抄的那里的。


