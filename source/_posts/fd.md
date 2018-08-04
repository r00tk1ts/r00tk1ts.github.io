---
title: Writeup.pwnable.kr系列之fd
date: 2018-03-03 17:18:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->

# Writeup.pwnable.kr系列之fd

##审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/fd_01.jpg)

看起来是和Linux文件描述符有关的题目，先ssh上去：

![](/images/ctf/pwn/pwnable.kr/fd_02.jpg)

按提示密码guest登录上去后，`ls -al`看看有什么好东西：

![](/images/ctf/pwn/pwnable.kr/fd_03.jpg)

可以看到我们的终极目标flag文件，然而我们作为fd账户并没有对flag的读权限。此时观察到fd.c是有读权限的，cat一下：

![](/images/ctf/pwn/pwnable.kr/fd_04.jpg)

非常简单的程序，有着以下的几个特点：

1. 局部变量fd表示文件描述符，为read所使用。
2. fd是由`atoi(argv[1])-0x1234`计算出来的，其中argv[1]可控，且程序没有做任何check。
3. read的dst buffer是一个全局的数组buf，大小定义为32，且read的第三个参数限定了32，不存在溢出。
4. 对比buf与"LETMEWIN\n"串，如果一致则调用system("/bin/cat flag")。

## 分析

非常直观的，我们只需要将buf填充为"LETMEWIN\n"即可。而buf由read控制，而我们所能控制的仅仅是Console下的输入，也就是stdin。众所周知，stdin、stdout、stderr在Linux系统中对应的文件描述符的值分别为0、1、2。因此，我们只需要满足fd为0即可控制buf，进一步由fd的计算表达式，我们只需要传入argv[1]为0x1234也就是4660即可。

Linux下stdin的定义在/usr/include/unistd.h中：

![](/images/ctf/pwn/pwnable.kr/fd_05.jpg)

另一方面，可能会有读者心存疑问，既然fd账户没有办法直接`cat flag`，为什么在程序中执行却可以读出来呢？实际上，仔细观察fd.c编译出来的fd可执行文件的文件权限就清楚了。fd的owner为fd_pwn，与flag文件一致，且fd设置了执行权限的suid，因此在fd执行时，其权限就临时的视为fd_pwn（关于执行时权限的判定实际上是很复杂的，这里的“临时性”说法只是便于理解，有兴趣可以自己查阅资料）。

## Pwn

直接执行./fd 4660即可：

![](/images/ctf/pwn/pwnable.kr/fd_06.jpg)