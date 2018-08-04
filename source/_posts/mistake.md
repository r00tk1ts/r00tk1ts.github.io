---
title: Writeup.pwnable.kr系列之mistake
date: 2018-03-09 21:18:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之mistake

## 审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/mistake_01.jpg)

看起来是和操作符优先级有关的一道题。上去看看：

```cpp
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){
	
	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password : ");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}
```

1. 题目从/home/mistake/password文件中读取10字节数据。
2. 我们输入10字节并把这10个字节与0x1异或。
3. 对比上面两个字符串是否一致，一致则输出flag。

## 分析

看起来如果我们能够cat出password中的内容，就可以计算出需要输入的内容了，然而，password我们是没有读权限的：

![](/images/ctf/pwn/pwnable.kr/mistake_02.jpg)
为之奈何？一个很自然的想法诞生了——暴破！

然而，仔细读读题，发现有提示不需要bruteforce，而本题干曾给出了操作符优先级的hint。搜索一番后，发现`if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){`这一句的使用大有问题。

这里因为比较运算符优先级高于赋值运算符，因此实际上fd最终并不是理想的文件描述符，而是0才对，而另一方面，0作为文件描述符，它指向的是stdin。

这一句是一个很常见的错误写法，理想的写法应该是`if((fd=open("/home/mistake/password",O_RDONLY,0400)) < 0){`。

分析到了这里，也就有了眉目，`read(fd,pw_buf,PW_LEN)`实际上是从stdin读取，而不是password文件，而stdin我们可以控制。

到此，两个buffer都可以控制，我们也就为所欲为。

## Pwn

第一次输入1111111111（1是0x31），第二次输入0000000000（0x30），第二次经异或0x1也变为了111111111。

![](/images/ctf/pwn/pwnable.kr/mistake_03.jpg)