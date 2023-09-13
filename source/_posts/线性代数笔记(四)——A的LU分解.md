---
title: 线性代数笔记(四)——A的LU分解
date: 2022-05-02 21:23:15
category: 线性代数
tags: MIT-18.06-linear_algebra
math: true
---

本讲主要是围绕$A$的$LU$分解，深入展开矩阵乘法、逆与转置的关系。最后自然而然的引出了置换矩阵。

<!--more-->

# $A$的$LU$分解
## $AB$的逆
$AB$如果有逆，那么其逆显然是$B^{-1}A^{-1}$，因为有：
$$
\begin{align}
(AB)(B^{-1} A^{-1})=I
\\ (B^{-1} A^{-1})(AB)=I
\end{align}
$$

根据结合律法则，我们把括号挪一下，俩俩结合成$I$，上式一目了然。

## $AB$的转置
显然是$B^TA^T$（对结果$C$中的每个元素求和分别计算即可证等），由此可见转置如果想要拆除括号，则内层矩阵的转置顺序要倒置。

## $A^T$的逆
显然：
$$
\begin{align}
(AA^{-1})^T = I^T = I
\\ (A^{-1})^TA^T = I
\end{align}
$$

因此$A^T$的逆矩阵就是$(A^{-1})^T$，即$(A^T)^{-1}=(A^{-1})^T$，因此得出：对单个矩阵，转置和取逆操作顺序可以互换。

## $A=LU$和$EA=U$的关系
通过不断的左乘$E_{xy}$做高斯消元，可以让矩阵$A$变换为$U$，而只需要在等号两边同时左乘$E^{-1}$，就得到$A=E^{-1}U$，显然，$L$和$E$互为逆矩阵。高斯消元本质上做的是行变换，最终得到的$U$是一个上三角阵，而$L$则是一个下三角阵。

$n$阶方阵$A$变换为$LU$需要的计算量有多少呢？对100阶方阵，从第2行开始到第100行针对列1元素都需要进行行变换，每行有100个元素，计算次数为$99\times100$，然后对于列2，则需要从第3行开始到第100行，计算次数为$98\times99$，递归下去，总的时间复杂度可用平方和来预估：$O(n^2+(n-1)^2+\dots+2^2+1^2)$，即$O(\frac{n^3}{3})$。


## 置换矩阵
3阶置换矩阵有6个：
$$
\begin{bmatrix}
1 & 0 & 0 
\\ 0 & 1 & 0 
\\ 0 & 0 & 1 
\end{bmatrix}
\begin{bmatrix}
0 & 1 & 0 
\\ 1 & 0 & 0
\\ 0 & 0 & 1 
\end{bmatrix}
\begin{bmatrix}
0 & 0 & 1
\\ 0 & 1 & 0
\\ 1 & 0 & 0
\end{bmatrix}
\begin{bmatrix}
1 & 0 & 0
\\ 0 & 0 & 1
\\ 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
0 & 1 & 0
\\ 0 & 0 & 1
\\ 1 & 0 & 0
\end{bmatrix}
\begin{bmatrix}
0 & 0 & 1
\\ 1 & 0 & 0
\\ 0 & 1 & 0
\end{bmatrix}
$$

这6个置换矩阵组成了一个很有意思的矩阵群（任取两个矩阵相乘，结果仍在该矩阵群中），它们有非常有意思的性质：置换矩阵的逆等于其转置，这一点非常好理解，相当于被交换的行又再次被交换回来。

根据每行中1的位置，可以知道$n$阶方阵的置换矩阵个数就是全排列数：共$\binom{n}{1}=n!$个。

# 附录

## 视频

<iframe src="//player.bilibili.com/player.html?aid=382989698&bvid=BV16Z4y1U7oU&cid=569890100&p=4&autoplay=0"
width="100%" height="500" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

## 参考链接

- [A的LU分解](https://github.com/MLNLP-World/MIT-Linear-Algebra-Notes/blob/master/%5B04%5DA%20%E7%9A%84%20LU%20%E5%88%86%E8%A7%A3/%E7%BA%BF%E6%80%A7%E4%BB%A3%E6%95%B0%5B%E5%9B%9B%5D.pdf)
- [线性代数的艺术](https://github.com/kf-liu/The-Art-of-Linear-Algebra-zh-CN)