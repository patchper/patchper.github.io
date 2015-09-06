---
layout: post
title: Mathjax测试
tags: mathjax
---

MathJax是用来在静态页面中渲染数学公式的好工具。

Here is an example MathJax inline rendering \\( 1/x^{2} \\), and here is a block rendering: 
\\[ \sum_{i=0}^n i^2 = \frac{(n^2+n)(2n+1)}{6} \\]

Test Matrix 
\\[ 
\begin{bmatrix}
 1.2 \\\\
 1.8 \\\\
 2.4 \\\\
 3   \\\\
 3.6 \\\\
 4.2 \\\\
\end{bmatrix}=
\begin{bmatrix}
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 1 \\\\
 1 & 1 \\\\
 1 & 1 
\end{bmatrix}
\begin{bmatrix}
 \mu\_{wt} \\\\
 \mu\_{mut} - \mu\_{wt}
\end{bmatrix}+
\begin{bmatrix}
 \epsilon\_{1} \\\\
 \epsilon\_{2} \\\\
 \epsilon\_{3} \\\\
 \epsilon\_{4} \\\\
 \epsilon\_{5} \\\\
 \epsilon\_{6} \\\\
\end{bmatrix}
\\]

在github-pages中使用MathJax十分容易：直接在模板的`<head></head>`段中加上

```html
<script type="text/javascript"
  src="https://cdn.bootcss.com/mathjax/2.5.3/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
```
就可以了。其中MathJax的主js文件是一个国内镜像。虽然更新会滞后一些，鉴于国内的网络稳定性，还是用它算了。

inline使用`\\(\\)`，block使用`\\[\\]`，这两个中如果有需要转义的字符统统转义。以上matrix的相关代码的源码如下：

```
\\[ 
\begin{bmatrix}
 1.2 \\\\
 1.8 \\\\
 2.4 \\\\
 3   \\\\
 3.6 \\\\
 4.2 \\\\
\end{bmatrix}=
\begin{bmatrix}
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 0 \\\\
 1 & 1 \\\\
 1 & 1 \\\\
 1 & 1 
\end{bmatrix}
\begin{bmatrix}
 \mu\_{wt} \\\\
 \mu\_{mut} - \mu\_{wt}
\end{bmatrix}+
\begin{bmatrix}
 \epsilon\_{1} \\\\
 \epsilon\_{2} \\\\
 \epsilon\_{3} \\\\
 \epsilon\_{4} \\\\
 \epsilon\_{5} \\\\
 \epsilon\_{6} \\\\
\end{bmatrix}
\\]
```
最近可能会写点*limma*和edgeR的文章，先部署下依赖环境。不过说实话，以前虽然用过\\(\LaTeX\\)但是很少插公式，试着写了点，还真是麻烦的很……也许对于熟悉的人来说会好点吧。