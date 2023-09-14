---
title: 线性代数笔记(七)——求解Ax=0：主变量，特解
date: 2022-05-04 12:28:37
category: 线性代数
tags: MIT-18.06-linear_algebra
math: true
---

这一讲深入探讨了求解$Ax=0$过程中，消元法所得到的主列、自由列、自由变量与特解以及它们与矩阵的秩的关系。

<!--more-->

# 求解$Ax=0$: 主变量，特解
给定一个$3\times 4$的矩阵：
$A=\begin{bmatrix}
 1&2&2&2 
 \\ 2&4&6&8
 \\ 3&6&8&10
\end{bmatrix}$

## 求解零空间
零空间由求$Ax=0$的解集合组成，除了零向量以外，欲求其他解，首先要通过高斯消元法找到主变量：
$$
A=
\begin{bmatrix}
1 & 2 & 2 & 2
\\ 2 & 4 & 6 & 8
\\ 3 & 6 & 8 & 10
\end{bmatrix}
\underrightarrow{eliminate}
\begin{bmatrix}
\underline{1} & 2 & 2 & 2
\\ 0 & 0 & \underline{2} & 4
\\ 0 & 0 & 0 & 0
\end{bmatrix}=U
$$

可以看出经过消元后，主变量还剩2个（下划线元素），因此矩阵$A$的秩就是2，即$r=2$。
主变量所在的列我们称为主列，其余列则称为自由列。自由列对应的变量我们称为自由变量，那么自由变量的个数就是：$n-r=4-2=2$。

高斯消元处理后，线性相关的行就会被暴露出来，比如$A$矩阵的行三实际上就是行一和行二的加和，所以在消元后整行都变成了$0$($0$行，即没有主元)。另一方面，线性相关的列也会被暴露，我们在消元过程中发现第二列原本要做主元的元素变成了0，这恰恰是因为第二列和前面的列（这里只有第一列）线性相关（刚好是两倍的列1），因此第二列没有主元，是自由列。而同样的，第四列也没有主元，这就说明第四列实际上是前三列的线性组合（两倍的第三列减第二列）。

消元的本质是行变换，行变换改变的是列向量（从而可能改变列空间），但是无法改变列向量之间的线性相关性。另一方面，消元不会改变零空间，因为$b$为$0$时，随你怎么折腾，他都是$0$，解不会改变。

因此，$Ax=0$求解问题变成了对$Ux=0$的求解，首先我们找出主变量和自由变量，$x_1$和$x_3$是主变量，$x_2$和$x_4$是自由变量。自由变量可以对其分配任意的值（不管你赋什么值，都会被计算出来的主变量抵消掉）。

**我们对自由变量随意赋值，一般采用的一种简单策略是：对其中一个自由变量赋值为$1$，其余赋值为$0$，循环往复。**

通过这样的策略，我们对自由变量进行赋值，并回代求出主变量：

- 令$x_2=1, x_4=0$，得解$x=\begin{bmatrix}-2\\ 1\\ 0\\ 0\end{bmatrix}$
- 令$x_2=0, x_4=1$，得解$x=\begin{bmatrix}2\\ 0\\ -2\\ 1\end{bmatrix}$

这两个解我们称之为特解，**特解的线性组合构成的向量空间就是零空间，零空间的每个向量都是方程组的解**：
$$
x=c\begin{bmatrix}-2\\ 1\\ 0\\ 0\end{bmatrix}+d\begin{bmatrix}2\\ 0\\ -2\\ 1\end{bmatrix}
$$

**特解之间线性无关。**

> 求特解的策略非常优雅直观，采用控制变量的思想，我们每次只关注其中一个自由变量，避免自由变量的相互影响，从而得到所有的线性无关的特解。

**归纳：对于$m*n$矩阵，若其秩为$r$，则主变量有$r$个，自由变量有$n-r$个。**

## RREF(Reduced row echelon form)
上例中消元得到的$U$矩阵是个上三角阵，实际上还可以进一步简化成$R$矩阵，即RREF(Reduced row echelon form)——简化行阶梯形式。

$R$矩阵中主元上下的元素都是0，主元提取公倍数化简为1：
$$
U=
\begin{bmatrix}
\underline{1} & 2 & 2 & 2
\\ 0 & 0 & \underline{2} & 4
\\ 0 & 0 & 0 & 0
\end{bmatrix}
\underrightarrow{自底向上}
\begin{bmatrix}
\underline{1} & 2 & 0 & -2
\\ 0 & 0 & \underline{1} & 2
\\ 0 & 0 & 0 & 0
\end{bmatrix}
=R
$$

将$R$矩阵中的主列放在一起，自由列放在一起（列交换），得到：
$$
R=\begin{bmatrix}
 \underline{1} & 2 & 0 & -2
 \\ 0 & 0 & \underline{1} & 2
 \\ 0 & 0 & 0 & 0
\end{bmatrix}\underrightarrow{列交换}
\left[
\begin{array}{c c | c c}
 1&0&2&-2
 \\0&1&0&2
 \\ \hline 0&0&0&0
\end{array}
\right]=
\begin{bmatrix}
 I & F
 \\ 0 & 0
\end{bmatrix}
\textrm{，其中}I\textrm{为主列组成的$r*r$大小的单位矩阵，}F\textrm{为自由列经过化简后组成的矩阵}
$$

计算零空间矩阵$N$（nullspace matrix），其列为特解，有$RN=0$。
$$
\begin{align}
x_{pivot}=-Fx_{free} 
\\ \begin{bmatrix}
 I&F
\end{bmatrix}
\begin{bmatrix}
 x_{pivot} 
 \\ x_{free}
\end{bmatrix}=0 
\\ N=\begin{bmatrix}
 -F 
 \\ I
\end{bmatrix}
\end{align}
$$

在本例中$N=\begin{bmatrix}
 -2&2
 \\ 0&-2 
 \\ 1&0 
 \\ 0&1 
\end{bmatrix}$，与上面求得的两个$x$特解一致，因此，可以利用$R$直接求解零空间。

> 主元的概念是消元带来的，而消元过程中挖掘的实际上就是行、列的线性相关性。

**总结：$A$ 的主元个数 = $A$ 矩阵线性无关的列的个数 = $A^T$ 矩阵线性无关的行的个数 = $A^T$ 的主元个数。**

# 附录
## 视频
<iframe src="//player.bilibili.com/player.html?aid=382989698&bvid=BV16Z4y1U7oU&cid=569901010&p=7&autoplay=0" scrolling="no" border="0" width="100%" height="500" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

## 参考链接

- [求解 Ax = 0，主变量，特解](https://github.com/MLNLP-World/MIT-Linear-Algebra-Notes/blob/master/%5B07%5D%20%E6%B1%82%E8%A7%A3%20Ax%20%3D%200%EF%BC%8C%E4%B8%BB%E5%8F%98%E9%87%8F%EF%BC%8C%E7%89%B9%E8%A7%A3/%E7%BA%BF%E6%80%A7%E4%BB%A3%E6%95%B0%5B%E4%B8%83%5D.pdf)
- [线性代数的艺术](https://github.com/kf-liu/The-Art-of-Linear-Algebra-zh-CN)