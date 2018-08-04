---
title: PWN选手的自我修养
date: 2017-09-11 20:49:11
categories: ctf
tags:
	- pwn
---
i春秋上观摩Atum大佬对CTF的归纳，适合有一定基础、想要入坑PWN的玩家。先立flag：每一个用例，日后都会写writeup。

<!--more-->

# PWN选手的自我修养
## 基础知识
这些如果不会，还玩个毛的PWN。

### 寄存器
rsp/esp,pc,rbp/ebp,rax/eax,rdi,rsi,rdx,rcx

![](/images/ctf/pwn/20170911_1.jpg)

### 栈
特性：后进先出
活在操作系统中的栈：
- 内存的一片区间，栈型结构管理，高地址向低地址增长。
- esp指向栈顶
- push sth -> [esp]=sth, esp=esp-4
- pop sth -> sth=[esp], esp=esp+4
- 以栈帧切割，保存函数调用信息和局部变量

### 调用约定
函数调用: call, ret
调用约定： _stdcall, __cdecl, __fastcall, __thiscall, __nakedcall, __pascal
参数传递：取决于调用约定，x86默认从右向左，x64优先寄存器，然后用栈

- call func -> push pc, jmp func
- leave -> mov esp, ebp, pop ebp
- ret -> pop pc

## 栈溢出的保护机制
1. NX/DEP
2. ASLR
3. Stack Canary/Cookie
4. PIE
5. RELNO

## 栈溢出利用方法
1. ROP
	- 用法
		- 第一次触发漏洞，通过ROP泄露libc的address(如puts_got)，计算system地址，然后返回到一个可以重现触发漏洞的位置(如main)，再次触发漏洞，通过ROP调用system("/bin/sh")
		- 直接execve("/bin/sh",["/bin/sh"],NULL)，通常在静态链接时比较常用
	- 资源
		- defcon 2015 qualifier: R0pbaby
		- AliCTF 2016: vss
		- PlaidCTF 2013: ropasaurusrex
2. SROP
	- 用法
		- 系统Signal Dispatch之前会将所有寄存器压栈，然后调用signal handler, signal handler返回时会将栈内容还原
		- 如果事先填充栈，然后直接调用signal handler，那在返回的时候就可以控制寄存器的值
	- 资源
		- http://angelboy.logdown.com/posts/283221-srop
		- http://www.2cto.com/article/201512/452080.html
		- defcon 2015 qualifier: fuckup
	- 写个demo
3. BROP
	- 用法
		- 目标binary未给出
		- 必须先存在一个已知的stack overflow漏洞，而且攻击者知道如何触发这个漏洞
		- 服务器进程在crash之后会重新复活，复活的进程不会被re-rand
	- 资源
		- http://ytliu.info/blog/2014/05/31/blind-return-oriented-programming-brop-attack-yi/
		- http://ytliu.info/blog/2014/06/01/blind-return-oriented-programming-brop-attack-er/
		- HCTF 2016 出题人跑路了(pwn50)
4. stack pivot
	- 用法
		- 存在地址已知且内容可控的buffer
			- bss段，由于bss段尾端常有大量空余空间，所以bss段尾端也往往是stack pivot的目标
			- 堆块，如果堆地址已泄且堆上的数据可控，那堆也可以作为stack pivot目标
		- 控制流可劫持
		- 存在劫持栈指针的gadgets
			- 如pop esp,ret，具体binary具体分析
	- 资源
		- EKOPARTY CTF 2016 fuckzing-exploit-200(基于栈溢出)
		- HACKIM CTF 2015 - Exploitation 5(基于堆溢出)
5. ret2dl_resovle, fake linkmap绕过ASLR
	- 用法
		- 动态链接就是从函数名到地址的转换过程，所以可以通过动态链接器解析任何函数，无需leak
		- 理论上任何可以stack pivot且FULLRELRO未开的题目都可以利用这种技术
	- 资源
		- http://blog.chinaunix.net/uid-24774106-id-3053007.html
		- 《程序员的自我修养》
		- http://rk700.github.io/2015/08/09/return-to-dl-resolve/
		- http://angelboy.logdown.com/posts/283218-return-to-dl-resolve
		- http://www.inforec.org/wp/?p=389
		- Codegate CTF Finals 2015 yocto(fake relplt) http://o0xmuhe.me/2016/10/25/yocto-writeup
		- HITCON QUALS CTF 2015 readable(fake linkmap)
		- Hack.lu's 2015 OREO(http://wapiflapi.github.io/2014/11/17/hacklu-oreo-with-ret2dl-resolve/) 
6. 低地址12bit不变，Partial Overwrite绕过ASLR
	- 用法
		- PIE开启时，一个32地址的高20位被随机化，低12bit不变。
		- 改写低12bit绕过PIE，不仅在栈溢出使用，各种利用都经常使用。
	- 资源
		- http://ly0n.me/2015/07/30/bypass-aslr-with-partial-eip-overwrite/
		- HCTF 2016 fheap(基于堆溢出)
7. leak canary，overwrite canary，改写指针与局部变量绕过stack canary
	- 用法
		- 以上所有套路，遇到stack canary均无效
		- 不覆盖stack canary，只覆盖stack canary前的局部变量、指针
			- 几乎不行，编译器会根据占用内存大小从小到大排列变量
			- 某些极限情况可以，一般都是精心构造的
		- leak canary
			- printf泄露，canary一般从00开始
		- overwrite canary
			- canary在TLS, TLS地址被随机化
8. 溢出位数不够：覆盖ebp，Partial Overwrite
	- 用法
		-可以覆盖Func2的ebp，会影响到Func1的esp，进而影响func1的ip
```cpp
Func1:
	call func2
	leave(mov esp ebp, pop ebp)
	ret(pop ip)

Func2:
	stack overflow
	leave(move esp, ebp, pop ebp)
	ret(pop ip)
```
	- 资源
		- XMAN 2016 广外女生-pwn
		- Codegate CTF Finals 2015，chess
![](/images/ctf/pwn/20170911_2.jpg)

## 待续。。。