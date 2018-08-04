---
title: Writeup.pwnable.kr系列之passcode
date: 2018-03-05 20:35:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之passcode

## 审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/passcode_01.jpg)

描述上看不出什么，连上去看看再说：

![](/images/ctf/pwn/pwnable.kr/passcode_02.jpg)

flag依然是没有权限的，只能cat一下passcode看看：

```cpp
#include <stdio.h>
#include <stdlib.h>

void login(){
	int passcode1;
	int passcode2;

	printf("enter passcode1 : ");
	scanf("%d", passcode1);			//key 少了个&
	fflush(stdin);

	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
        scanf("%d", passcode2);		//key 少了个&

	printf("checking...\n");
	if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
		exit(0);
        }
}

void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}

int main(){
	printf("Toddler's Secure Login System 1.0 beta.\n");

	welcome();
	login();

	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;	
}
```

有这些线索：

1. 程序需要输入两次密码，但在login中的scanf使用是有误的，passcode1和passcode2两处前均烧了一个"&"符。
2. 这一“笔误”也验证了题目的说法，有警告但是可以编译通过，但很明显，执行时会因为把passcode1和passcode2的值作为地址来进行写入操作，passcode1和passcode2是栈上的局部变量，未初始化所以是垃圾“随机”值，一旦这两个不可控的地址是不可写或访问的，那么程序就crash了。

## 分析

题目已经很清楚了，那么直观的想法就是通过某种方式在`scanf("%d", passcode1);`之前就为passcode1和passcode2精心准备好一个垃圾“随机”值。

可是怎么办呢？一旦进入login()就没戏了，所以我盯上了welcome()。首先，welcome和login之间没有多余的栈操作，因此二者的ebp应该是一致的。其次，在welcome有一个`scanf("%100s", name);`，这是个很大的buffer，所以理论上来说，通过对name进行精心的布局，应该可以覆盖到login()栈帧中passcode1和passcode2的值。这也正是我对“随机”二字加了引号的原因。

那么passcode1和passcode2应该是什么值呢？从逻辑上来看，passcode1为338150（0x000582E6），passcode2为13371337（0x00cc07c9）即可。

然而且不论这两个地址是否是可写的，至少00字节的存在就因为截断而打消念想了。另一方面，浏览一下反汇编代码就会发现，实际上name的地址在`ebp-0x70`的位置，而passcode1在`ebp-0x10`的位置（两个栈帧中ebp值相等），经过计算，name也覆盖不到passcode2。

welcome：

![](/images/ctf/pwn/pwnable.kr/passcode_03.jpg)

login：

![](/images/ctf/pwn/pwnable.kr/passcode_04.jpg)

那么接下来怎么办？

实际上，既然scanf是一个具有写功能的函数，我们完全可以利用scanf来修改此后使用到的某个函数的got表项。例如，程序在`scanf("%d", passcode1);`后立即使用了fflush函数，所以我们完全可以先找到fflush的got表项地址（程序没有开PIE，无需leak），把passcode1布局为该地址，并在调用到scanf("%d", passcode1)时输入程序代码中调用`system("/bin/cat flag");`处的地址即可。

objdump -R passcode找fflush地址：

![](/images/ctf/pwn/pwnable.kr/passcode_05.jpg)

system("cat flag")地址:

![](/images/ctf/pwn/pwnable.kr/passcode_06.jpg)

## Pwn

![](/images/ctf/pwn/pwnable.kr/passcode_07.jpg)

程序NO PIE，所以地址就无需leak了，直接通过gdb找到的就是固定的。

关于got和plt表的关系以及linux的动态延迟加载，我就不详细展开了，给一张图吧：

![](/images/ctf/pwn/pwnable.kr/passcode_08.jpg)

