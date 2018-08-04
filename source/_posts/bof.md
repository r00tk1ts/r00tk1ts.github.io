---
title: Writeup.pwnable.kr系列之bof
date: 2018-03-04 13:36:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之bof

## 审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/bof_01.jpg)

看起来是一道栈溢出相关的题目，看看bof.c。

```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```

非常简单的程序，可以看出以下几个要点：

1. overflowme是一个局部数组变量，尺寸为32字节，且通过gets获取，即用户可控。
2. main中调用func时参数key指定为0xdeadbeef，在进入func后会落入else分支。
3. 想要进入func后落入if分支，需要通过某种方式修改key。

## 分析

把bof可执行文件拉入IDA看一下：

![](/images/ctf/pwn/pwnable.kr/bof_02.jpg)

这是一个32位程序，参数key（对应ida中a1）通过栈传递。根据IDA的反汇编，可以看到key的位置在`ebp+8h`处，而局部变量overflowme（对应s）首地址在`ebp-2ch`处。因此，二者的距离为0x34。

看到这里，思路也就显而易见了，因为`gets(overflowme)`并没有对输入的长度进行限制，因此我们可以溢出overflowme数组来改写参数key，改写的值就是0xCAFEBABE。

## Pwn

题目比较简单，所以写不写exp都是可以的，直接用题目给的nc吧：

![](/images/ctf/pwn/pwnable.kr/bof_03.jpg)

之所以加cat -，是为了防止shell一闪而过，保持与shell的连接。