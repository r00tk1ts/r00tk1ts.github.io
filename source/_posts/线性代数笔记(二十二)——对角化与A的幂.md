---
title: 线性代数笔记(二十二)——对角化与A的幂
date: 2023-10-10 14:45:37
category: 线性代数
tags: MIT-18.06-linear_algebra
math: true
---

书接上回，对分解出的特征值和特征向量加以利用，利用特征向量构成的可逆矩阵$S$对矩阵$A$做对角化，抽离出特征值和特征向量矩阵。这一手法在研究矩阵的幂时非常有用，我们可以惊奇的发现对角化之后，矩阵的幂仅仅改变的是特征值，特征向量却没有发生变化。利用这一手法，我们可以构造对角矩阵来解决很多问题，比如：差分方程、斐波那契数列。

<!--more-->

# 对角化和A的幂
我们此前学过很多种矩阵$A$的分解，比如$LU$分解，$QR$分解，它们都在特定的场景发挥作用。在我们学习了特征值与特征向量以后，还可以对矩阵$A$进行对角化分解。

## 对角化
过程如下：

1. 首先要求$A$有$n$个线性无关的特征向量，用它们构成矩阵$S=\begin{bmatrix}x_1&x_2&...&x_n\end{bmatrix}$，我们称$S$为特征向量矩阵。
2. 构造$AS=A\begin{bmatrix}x_1&x_2&...&x_n\end{bmatrix}$，根据特征值的定义，得到：$A\begin{bmatrix}x_1&x_2&...&x_n\end{bmatrix}=\begin{bmatrix}\lambda_1x_1&\lambda_2x_2&...&\lambda_n x_n\end{bmatrix}$。这一方程相当于把每一个$Ax_i=\lambda_i x_i$组合起来。
3. $\begin{bmatrix}\lambda_1x_1&\lambda_2x_2&...&\lambda_n x_n\end{bmatrix}$可以展开成：$\begin{bmatrix}x_1&x_2&...&x_n\end{bmatrix}\begin{bmatrix}\lambda_1&0&...&0\\ 0&\lambda_2&...&0\\ ...&...&...&...\\ 0&0&...&\lambda_n \end{bmatrix}$，我们把这个由特征值构成的矩阵记为$\Lambda$，于是有：$AS=S\Lambda$。
4. $S$的各列向量（也就是特征向量）都是线性无关的，因此$S$可逆，故：$S^{-1}AS=\Lambda$；同理，翻转可得：$A=S\Lambda S^{-1}$。

如此，我们就得到了矩阵$A$的一种新型分解方式，根据这一分解手法，我们可以看出矩阵$A$与其特征向量、特征值的关联。由于特征值构成的$\Lambda$矩阵只有对角线有值（且为特征值），故这一分解过程被称之为——对角化。

## A的幂
对角化的分解方式有什么用呢？仔细观察$A=S\Lambda S^{-1}$的形式，我们可以发现在计算$A$的幂时这一分解非常好用：

- 考虑$A^2$，有：$A^2=S\Lambda S^{-1}S\Lambda S^{-1}=S\Lambda^2S^{-1}$，这意味着进行幂运算的前后，特征向量没有发生改变，而特征值进行了幂运算。
- 推广到$A^n$，不难得到$A^n=S\Lambda^nS^{-1}$。

> 这与上一节我们观察得到的：$A^2x=A\lambda x=\lambda^2x$如出一辙。

### 性质延展
根据这一性质，我们可以解答这样一个问题：什么条件下，能够让矩阵$A$的幂$A^k$，在$k$趋于$\infty$时，趋近于$0$？

根据这一分解，答案也就显而易见了：所有的特征值都满足$|\lambda_i|<1$时。

当然，这里的$A$需要有$n$个线性无关的特征向量，这个是前提。

正如上一节的提醒：$n$阶矩阵能否成功对角化分解取决于是否有$n$个线性无关的特征向量，而特征向量与特征值之间有着紧密的关系：

- 若矩阵$A$没有重复的特征值，那么就一定有$n$个线性无关的特征向量（不同的特征值对应的特征向量线性无关）。
- 若有重复的特征值，则矩阵是否有$n$个线性无关的特征向量尚待考究。

> 本节不对拥有重复特征值的$A$做展开，那是另一个更加深入的话题。

### 差分方程
今有向量$u_{k+1}=Au_k$，通项易得：$u_k=A^ku_0$。这是一个一阶差分方程组。

想要求解这一方程，需要将向量$u_0$拆解成矩阵$A$的线性无关的$n$个特征向量的线性组合：
$$
u_0=c_1x_1+c_2x_2+...+c_nx_n=\begin{bmatrix}
x_1&x_2&...&x_n\end{bmatrix}\begin{bmatrix}
c_1\\ c_2\\ ...\\ c_n\end{bmatrix}=Sc
$$

我们把每一个系数构成的向量记为$c$。

根据特征方程：
$$
Au_0=c_1Ax_1+c_2Ax_2+...+c_nAx_n=c_1\lambda_1x_1+c_2\lambda_2x_2+...+c_n\lambda_nx_n
$$

上式展开成矩阵乘法写作：
$$
Au_0=\begin{bmatrix}
x_1&x_2&...&x_n
\end{bmatrix}\begin{bmatrix}
\lambda_1&0&...&0 \\
0&\lambda_2&...&0 \\
...&...&...&... \\
0&0&...&\lambda_n
\end{bmatrix}\begin{bmatrix}
c_1 \\
c_2 \\
... \\
c_n
\end{bmatrix}=S\Lambda c
$$

> 相当于用对角化分解直接带入：$Au_0=S\Lambda S^{-1}u_0=S\Lambda S^{-1}Sc=S\Lambda c$，只不过上述的展开更加立体。

因此，对于$u_k$，只需要将$\lambda$做幂运算，保持$c$和特征向量矩阵$S$不变，即可得到。即：$u_k=A^{k}u_0=S\Lambda^k c=c_1\lambda_1^kx_1+c_2\lambda_2^kx_2+...+c_n\lambda_n^kx_n$。

### 斐波那契数列
借助差分方程的通项计算方法，我们可以通过构造矩阵$A$来求解斐波那契数列的通项。

对$F_{k+2}=F_{k}+F_{k+1}$，我们需要把他转化成$u_{k+1}=Au_k$的形式。怎么操作呢？可以借助这样一个小技巧：
$$
令u_k=\begin{bmatrix}F_{k+1}\\ F_{k}\end{bmatrix}
$$

追加一个方程组成方程组：
$$
\begin{cases}
F_{k+2}=F_{k}+F_{k+1} \\
F_{k+1}=F_{k+1}
\end{cases}
$$

对于上述方程组，我们可以这样表达：
$$
\begin{bmatrix}
F_{k+2} \\ F_{k+1}
\end{bmatrix}=\begin{bmatrix}1&1 \\ 1&0\end{bmatrix}\begin{bmatrix}
F_{k+1} \\ F_{k}
\end{bmatrix}
$$

即转化为$u_{k+1}=Au_k$的形式，其中$A=\begin{bmatrix}1&1 \\ 1&0\end{bmatrix}$。

> 本质上是把二阶差分方程改造成一阶向量方程组。

矩阵$A$是对称阵，它的特征值都是实数，利用迹与行列式可以求解得到特征值$\lambda_1=\frac{1+\sqrt{5}}{2},\lambda_2=\frac{1-\sqrt{5}}{2}$。代回特征方程得到特征向量$x_1=\begin{bmatrix}\frac{1+\sqrt{5}}{2} \\ 1\end{bmatrix}, x_2=\begin{bmatrix}\frac{1-\sqrt{5}}{2} \\ 1\end{bmatrix}$。

> 特征方程求解$\lambda^2-\lambda-1=0$本质上就是$F_{k+2}-F_{k+1}-F_{k}=0$。

再根据$u_0=\begin{bmatrix}F_1 \\ F_0\end{bmatrix}=\begin{bmatrix}1 \\ 0\end{bmatrix}=Sc=c_1x_1+c_2x_2$，容易求得系数$c_1=\frac{\sqrt{5}}{5},c_2=-\frac{\sqrt{5}}{5}$。

最终代入$u_k=S\Lambda^kc$，就可以求得任意通项。比如，我们希望预估斐波那契数列的第100项的值：
$$
u_{99}=\left[\begin{array}{c}
F_{100} \\
F_{99}
\end{array}\right]=\left[\begin{array}{cc}
\frac{1+\sqrt{5}}{2} & \frac{1-\sqrt{5}}{2} \\
1 & 1
\end{array}\right]\left[\begin{array}{cc}
\left(\frac{1+\sqrt{5}}{2}\right)^{99} & 0 \\
0 & \left(\frac{1-\sqrt{5}}{2}\right)^{99}
\end{array}\right]\left[\begin{array}{c}
\frac{\sqrt{5}}{5} \\
-\frac{\sqrt{5}}{5}
\end{array}\right]=\left[\begin{array}{c}
c_1 \lambda_1^{100}+c_2 \lambda_2^{100} \\
c_1 \lambda_1^{99}+c_2 \lambda_2^{99}
\end{array}\right]
$$

得通项公式：$F_k=c_1\lambda_1^k+c_2\lambda_2^k$。

此外，由于特征值$\lambda_2\approx-0.618$，在幂增长过程中趋近于$0$，预估时可以忽略。因此斐波那契数列的增长速度可以按照$\lambda_1$来评估，增长速度大约为$1.618$。

# 附录
## 视频
<iframe src="//player.bilibili.com/player.html?aid=382989698&bvid=BV16Z4y1U7oU&cid=570099395&p=22&autoplay=0" scrolling="no" width="100%" height="500" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

## 参考链接

- [对角化和A的幂](https://github.com/MLNLP-World/MIT-Linear-Algebra-Notes/blob/master/%5B22%5D%20%E5%AF%B9%E8%A7%92%E5%8C%96%E5%92%8C%20A%20%E7%9A%84%E5%B9%82/%E7%BA%BF%E6%80%A7%E4%BB%A3%E6%95%B022.pdf)
- [线性代数的艺术](https://github.com/kf-liu/The-Art-of-Linear-Algebra-zh-CN)
