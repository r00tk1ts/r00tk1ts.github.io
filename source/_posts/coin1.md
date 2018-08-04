---
title: Writeup.pwnable.kr系列之coin1
date: 2018-03-10 10:25:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之coin1

## 审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/coin1_01.jpg)

上去看看：

![](/images/ctf/pwn/pwnable.kr/coin1_02.jpg)

## 分析

一堆硬币，有一个假币，重量为9，而其他为10，要通过给定的C次机会内找到假币。一共要在30s内找到100个假币，也就是完成100次游戏，最终会给出flag。

二分法处理就行了，又是一道coding。因为网络延迟的问题，所以我们不使用远程连接，直接用其他题目的账号登录上去，在/tmp目录下写个py脚本。

连上去在/tmp下发现已经有个coin1目录了，而且这个目录下也已经有了个别人写的py脚本。。。

直接copy别人写的了，没什么难度自己不浪费时间重新写了，跑一下。

```python
import socket
import sys
import re

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("localhost", 9007))

def get_numbers(ss, ee):
        str_list = []
        for n in range(ss, ee):
                str_list.append(str(n))
        return " ".join(str_list)

start = 0
while True:
        data = s.recv(1024)
        print data
        pattern1 = re.compile("^N=([0-9]*) C=([0-9]*)$")
        match1 = pattern1.match(str(data))
        pattern2 = re.compile("^([0-9]*)$")
        match2 = pattern2.match(str(data))

        if match1:
                end = int(match1.group(1))
                fr = (0,end/2)
                sr = (end/2+1, end)
                s.send(get_numbers(fr[0], fr[1])+"\n")
        elif match2 and len(match2.group(1)) > 0:
                w = int(match2.group(1))
                if (fr[1] - fr[0])*10 == w:
                        fr = (sr[0], (sr[0]+sr[1])/2+(sr[0]+sr[1])%2)
                        sr = (fr[1], sr[1])
                else:
                        sr = (fr[1], fr[1])
                        fr = (fr[0], (fr[0]+fr[1])/2+(fr[0]+fr[1])%2)
                        sr = (fr[1], sr[1])

                s.send(get_numbers(fr[0], fr[1])+"\n")
```

get_numbers用于生成索引序列，比如`2 3 4 5`，满足格式需求。后面就是简单的正则匹配，取出服务器的结果，if/else分支根据当前的weight总和判断假币在左侧还是右侧，如此二分查找下去最终可以找到假币索引。

## Pwn

运行时发现程序有点问题，最后我给补充了一下：

![](/images/ctf/pwn/pwnable.kr/coin1_04.jpg)

这个故事太长了，不完整截图了，给个最后的flag：

![](/images/ctf/pwn/pwnable.kr/coin1_03.jpg)