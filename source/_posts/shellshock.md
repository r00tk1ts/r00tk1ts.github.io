---
title: Writeup.pwnable.kr系列之shellshock
date: 2018-03-10 09:27:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之shellshock

## 审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/shellshock_01.jpg)

连上去看看：

![](/images/ctf/pwn/pwnable.kr/shellshock_02.jpg)

## 分析

这段程序很简单，我们以shellshock身份启动时，程序的权限是other权限r-x，而在setresuid和setresgid中使用的是effective gid，也就是shellshock_pwn的权限r-s，当程序执行到system时，程序已经具有shellshock_pwn组权限了。

这个组权限对于flag文件来说是可读的（r--），但是问题在于这一段程序并没有涉及对flag的读操作，权限虽然有了，但怎么办呢？

实际上题干描述中或多或少有些提示，关于bash的新闻。不仅如此，当找寻shellshock字符串时，会发现这是在bash 4.1版本以下存在的破壳漏洞的关键字。这个漏洞可以作为一个本地提权漏洞来使用。

这个漏洞就不在这里详细展开了，实际上非常简单，和bash中的环境变量设置与后续的initialize_shell_variables解析有关。参考这两个文章吧：

[关于*ShellShock*漏洞的利用过程和原理解析 - CSDN博客](https://www.baidu.com/link?url=3yWq37n7Y_LHRUa2L-iudGGqqgT-6s4gKmaU0_hOQBDFSacVZ_bxZqpA999gplxb_gOwXfn9XSE9mE53bYkDBK&wd=&eqid=e40eff4200053949000000035aa32eda)

[破壳（ShellShock）漏洞样本分析报告](http://www.freebuf.com/articles/system/45390.html)

为了便于后续使用的说明，简单的说一下流程。

先试试是否有此漏洞：

![](/images/ctf/pwn/pwnable.kr/shellshock_03.jpg)

nice!

实际上，`x='() { :;}; echo vulnerable'`是new出来了一个新的环境变量：

```
KEY=x
VALUE=() {:;}; echo vulnerable
```

而当我们后续执行bash的时候，最终会定位到initialize_shell_variables中，这个函数内部会遍历所有的环境变量，而我们设计的VALUE绕过了其中一个export函数的定义检查，使得最终执行的是后面的`echo vulnerable`串。所以，调用bash的时候，自定义的这个语句就会触发。

## Pwn

于是，我们偷梁换柱，将自定义语句换为`bash -c "cat ./flag"`，bash -c换成`./shellshock`(它内部调用了`bash -c 'echo shock_me'`)。

![](/images/ctf/pwn/pwnable.kr/shellshock_04.jpg)

./shellshock执行时拿到了shellshock_pwn的权限，而破壳漏洞在此权限上执行了`cat ./flag`，这一拼接也就导致了提权后的为所欲为。