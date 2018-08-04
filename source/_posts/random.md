---
title: Writeup.pwnable.kr系列之random
date: 2018-03-05 20:52:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之random

##审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/random_01.jpg)

看起来是和随机数相关的，上去看看。

直接cat random.c:

```cpp

```

好吧，这个题就太简单了，学过C的都知道rand()是伪随机，它的实现使用线性同余法做的，尽管生成的随机数周期很长，但每次实现随机数的生成序列都是一致的。

一般来说，在使用rand()前，都会使用srand()来随机一个种子（如果不使用，那么rand()相当于每一次都是srand(1)），为了保证随机性，一般用时间来作为srand()的参数。

## 分析

所以这道题就一目了然了，我们只需要执行一次，在gdb中看看这个random的值是多少，然后倒推出应该输入的值即可。

## Pwn

![](/images/ctf/pwn/pwnable.kr/random_02.jpg)

在cmp指令处下断点。此后输入20后，直接看看rpb-4就是生成的随机数了：

![](/images/ctf/pwn/pwnable.kr/random_03.jpg)

也就是0x6b8b4567，每次运行都是这个值。

于是异或算出应该输入的值为`0x6b8b4567^0xdeadbeef=3039230856`

再来一次：

![](/images/ctf/pwn/pwnable.kr/random_04.jpg)

