---
title: Writeup.pwnable.kr系列之flag
date: 2018-03-04 16:47:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之flag

## 审题

![](/images/ctf/pwn/pwnable.kr/flag_01.jpg)

题目描述来看，这是一个reverse题，不是pwn。于是下载flag文件看看，发现是一个ELF64文件。

##分析

拖入IDA：

![](/images/ctf/pwn/pwnable.kr/flag_02.jpg)

函数少的可怜，这种布局一看就是加壳了，继而找找有没有相关的壳信息字符串或特征。

Hex View中看到如下字样：

![](/images/ctf/pwn/pwnable.kr/flag_03.jpg)

根据字符info来看是UPX壳，UPX是一个非常经典常用的压缩壳，用`upx -d`解压即可。

解压的程序就一目了然了：

![](/images/ctf/pwn/pwnable.kr/flag_04.jpg)

有着很明显的说明，malloc一个buffer，然后strcpy把cs:flag处的字符串拷贝进buffer。

这个`cs:flag`应该就是flag。

另外，跟踪sub_400320调用发现最终是定位到一个.got.plt表项：

![](/images/ctf/pwn/pwnable.kr/flag_05.jpg)

![](/images/ctf/pwn/pwnable.kr/flag_06.jpg)

通过gdb跟踪发现，这个函数实际上是：

![](/images/ctf/pwn/pwnable.kr/flag_07.jpg)

当然，实际上分析这一步已是画蛇添足了。

跟踪到`cs:flag`处：

![](/images/ctf/pwn/pwnable.kr/flag_08.jpg)

完整显示字符串：![](/images/ctf/pwn/pwnable.kr/flag_09.jpg)

所以flag就是`UPX...? sounds like a delivery service :)`