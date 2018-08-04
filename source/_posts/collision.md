---
title: Writeup.pwnable.kr系列之collision
date: 2018-03-03 17:32:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之collision

## 审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/collision_01.jpg)

看描述是和MD5哈希碰撞相关，先上去看看：

![](/images/ctf/pwn/pwnable.kr/collision_02.jpg)

文件权限和fd的一致，所以还是需要看看col.c。

```cpp
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```

从上面的代码可以看出：

1. argv[1]应该是一个20字节长度的字符串。
2. 想要执行`cat flag`分支需要通过`check_password(argv[1])==hashcode`的检查。
3. hashcode是一个硬编码为0x21DD09EC的值；check_password中将20字节长度的argv[1]拆分成5个int值并进行累加。

## 分析

一个典型的简单哈希碰撞。我们只需要找出一组int[5]，满足累加和为0x21DD09EC即可。

最为暴力的解法显然是写一段程序，枚举出来就行了，但杀个鸡还犯不上用宰牛刀。

最直观的想法，我们将前16个字节都设为0x01，后4个字节就是`(0x21DD09EC-0x01010101*4)=0x1DD905E8`。于是，我们可以输入串`0x01010101 * 4 + 0x1DD905E8`。

另一方面，标准输入中，非字母数字以及符号的ASCII字符并不好输入，实际上有这样一种技巧来通过生成的方式处理：

`python -c "print '\x01\x01\x01\x01'*4 + '\xe8\x05\xd9\x1d'"`

这一inline的python表达式可以将生成的结果作为argv[1]参数，这一技巧最早是在看雪上泉哥翻译的Exploit编写系列教程上看到的。除了python，Perl等脚本语言也是可以的。

## Pwn

![](/images/ctf/pwn/pwnable.kr/collision_03.jpg)

这里实际上也可以选择除了0x00以外的任意数，只需要累加和满足条件即可，比如:

![](/images/ctf/pwn/pwnable.kr/collision_04.jpg)

> 之所以不能有0x00是因为0x00是一个截断字符，程序读入argv[1]时如果遇到了0x00会自动截断，从而导致程序结果有误。