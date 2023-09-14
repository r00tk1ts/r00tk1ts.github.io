---
title: 线性代数笔记(八)——求解Ax=b：可解性和解的结构
date: 2022-05-04 19:10:12
category: 线性代数
tags: MIT-18.06-linear_algebra
math: true
---

这一讲系统的讲解了线性方程组$Ax=b$的求解，对可解性和解的结构进行了展开说明，得到了具体的通用解法：$X_p+X_n$，并按照秩与$m,n$的关系对解做了归类。

# 求解$Ax=b$：可解性和解的结构
我们知道$Ax=b$未必有解，当$A$的列空间无法线性组合出$b$时，方程组是无解的。而在有解时可能存在唯一解，也可能存在无穷多个解，那么这其中又有什么规律呢？我们尝试按照上一讲对$Ax=0$的研究方法来进行消元操作。

## 可解性
依然采用上一讲的矩阵：$A=\begin{bmatrix}
1 & 2 & 2 & 2
\\ 2 & 4 & 6 & 8
\\ 3 & 6 & 8 & 10
\end{bmatrix}$，求$Ax=b$的特解。

这一次$b$不再是$0$向量，我们消元时写出增广矩阵：
$$
\left[
\begin{array}{c c c c|c}
1 & 2 & 2 & 2 & b_1 
\\ 2 & 4 & 6 & 8 & b_2 
\\ 3 & 6 & 8 & 10 & b_3 
\end{array}
\right]
\underrightarrow{消元}
\left[
\begin{array}{c c c c|c}
1 & 2 & 2 & 2 & b_1 
\\ 0 & 0 & 2 & 4 & b_2-2b_1 
\\ 0 & 0 & 0 & 0 & b_3-b_2-b_1 
\end{array}
\right]
$$

从最后一行可见，要使$Ax=b$有解，则必须满足$b_3-b_2-b_1=0$，而这就是可解性。

我们此前按照列空间视角，对此可以给出这样的解释：$b$必须要属于$A$的列空间才有解，也就是$A$中各列的线性组合。

现在让我们换个角度来理解可解性：**如果$A$中各行线性组合产生了零行，那么向量$b$的分量在同样的线性组合后也必须为零。**

## 解的结构
设$b=\begin{bmatrix}
1\\ 5\\ 6
\end{bmatrix}$满足可解性，那么$Ax=b$的解是什么呢？
直观上来看，消元以后我们得到了两个方程，但是未知数有4个，因此，理论上我们可以找到无穷多个解（因为存在自由变量）。
$$
\begin{cases}
x_1+2x_2+2x_3+2x_4 = 1 
\\ 2x_3+4x_4 = 3
\end{cases}
$$

想要求得通解，首先我们要找出特解：先让所有的自由变量取$0$，以解出此时主变量的值。这里自由变量是$x_2$和$x_4$，回代求得$x_1=-2,x_3=\frac{3}{2}$，故特解$x_p=\begin{bmatrix}-2\\ 0\\ \frac{3}{2}\\ 0\end{bmatrix}$。

> 自由变量取$0$只是为了方便计算，实际上你想取啥都行，因为它们会被主变量抵消、毫无贡献。

显然，将特解$x_p$加上上一讲所学的零空间中的任意向量(写为$x_n$)就可以得到通解：$x=x_p+x_n$。这是因为零空间的向量带入方程后结果永远是$0$，它不会影响等式（$A(x_p+x_n)=Ax_p+Ax_n=b+0=b$）。

而上一讲中我们得到了零空间解集为：$x_n=c\begin{bmatrix}
-2\\ 1\\ 0\\ 0
\end{bmatrix}+d\begin{bmatrix}
2\\ 0\\ -2\\ 1
\end{bmatrix}$

因此，$Ax=b$的通解为$\begin{bmatrix}-2\\ 0\\ \frac{3}{2}\\ 0\end{bmatrix}+c\begin{bmatrix}
-2\\ 1\\ 0\\ 0\end{bmatrix}+d\begin{bmatrix}2\\ 0\\ -2\\ 1\end{bmatrix}$

## 解与秩的关系
考虑秩为$r$的$m*n$矩阵$A$，显然$r\leq m, r\leq n$。

- 当列满秩时($r=n$)，消元后$R$为$\begin{bmatrix}I \\ 0\end{bmatrix}$，意味着每一列都有主元，那么也就没有自由变量，此时零空间里只有零向量，因此若$b$满足可解性，则解必唯一。
- 当行满秩时($r=m$)，消元后$R$为$\begin{bmatrix}I & F\end{bmatrix}$，意味着没有零行，此时对$b$就没有任何约束，那么$Ax=b$是必然有解，自由变量有$n-m$个，此时解有无穷多个。
- 当$m=n=r$时，行列皆满秩，消元后$R$为单位阵，不存在自由列，也没有零行对$b$进行约束，因此必有解且解唯一。
- 当不满秩，即$r<n$且$r<m$时，消元后$R$为$\begin{bmatrix}I & F \\ 0 & 0\end{bmatrix}$的形式，此时有两种情况：
  - 无解。零行约束了$b$，但$b$不满足约束条件。
  - 无穷多个解。$b$满足零行约束，解集为：特解+零空间任意向量

总结如下：
$$
\begin{array}{c|c|c|c}
r=m=n&r=n\lt m&r=m\lt n&r\lt m,r\lt n
\\ R=I&R=\begin{bmatrix}I\\ 0\end{bmatrix}&R=\begin{bmatrix}I&F\end{bmatrix}&R=\begin{bmatrix}I&F\\0&0\end{bmatrix}\\ 1\ solution&0\ or\ 1\ solution&\infty\ solution&0\ or\ \infty\ solution\end{array}
$$

# 附录
## 视频
<iframe src="//player.bilibili.com/player.html?aid=382989698&bvid=BV16Z4y1U7oU&cid=569898565&p=8&autoplay=0" scrolling="no" border="0" width="100%" height="500" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>
## 参考链接

- [Ax=b 的可解性和解的结构](https://github.com/MLNLP-World/MIT-Linear-Algebra-Notes/blob/master/%5B08%5D%20Ax%3Db%20%E7%9A%84%E5%8F%AF%E8%A7%A3%E6%80%A7%E5%92%8C%E8%A7%A3%E7%9A%84%E7%BB%93%E6%9E%84/%E7%BA%BF%E6%80%A7%E4%BB%A3%E6%95%B08.pdf)
- [线性代数的艺术](https://github.com/kf-liu/The-Art-of-Linear-Algebra-zh-CN)