---
title: GNU Hash ELF Sections
date: 2017-08-24 19:27:11
categories: glibc
tags:
	- linux
	- glibc
---
最近一直在研究ELF，期间看了很多优质的资源，这一篇文章是我从oracle的博客上翻译过来的。这是我近期见过的讲解最细致、五脏皆全的关于.gnu.hash的文章。我将它翻译了过来，鉴于本人英语渣渣，词不达意处，诸君海涵。

[原文链接](https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections)

还要感谢0xmuhe师傅，在学习Linux PWN相关的姿势时，在muhe师傅的github上发现了大量优质资源，果断fork。也希望自己能够像muhe师傅一样，在调研过后，写几篇干货。
[0xmuhe师傅的钻石矿](https://github.com/o0xmuhe/pwn2exploit)

<!--more-->
# GNU Hash ELF Sections
By: Ali Bahrami | Principal Software Engineer

ELF对象格式在多个不同的操作系统中使用，它们虽同享一个通用的基本设计，但各自又做了自身的扩展。在ELF设计中，有一个很好的特点促进了扩展：它定义一个通用核心，同时储备了空间使得每个实现体可以定义自身的扩展。我们在ELF团体中尝试相互流通，作为新的想法和灵感的源泉，以便于避免重复造轮子，以及失去好奇。

近期，GNU linker开发者为他们的对象增加了一种新风格hash section，并且我已通过优良的详细说明中学习了他们。在完成这样的工作时，我仅仅想要把它写下来并分享。

该文章描述了GNU ELF hash sections的布局和解释。GNU hash section为ELF对象提供了新的hash section，较之原始的SYSV hash具有更好的性能。

这里收集的信息来源如下：
- Jakub Jelinek's posting to the binutils@sourceware.org mailing list for the binutils project: http://sourceware.org/ml/binutils/2006-10/msg00377.html.
- Ulrich Drepper's document How To Write Shared Libraries.
- Email correspondence with Jakub and Ulrich. They were very generous with their time, and filled in details I couldn't intuit from the above.

收集这些信息时，我没有看过GNU binutils源码。

如果你发现了一个错误，发送邮件（First.Last@Oracle.COM, First和Last替换成我的姓名）给我，我会修正它。

## Hash 函数
GNU哈希函数使用DJB(Daniel J Bernstein)哈希，该哈希算法由Bernstein教授在多年以前发表在comp.lang.c 世界性新闻组网络新闻组:

```cpp
uint32_t
dl_new_hash (const char *s)
{
        uint32_t h = 5381;

        for (unsigned char c = *s; c != '\0'; c = *++s)
                h = h * 33 + c;

        return h;
}
```

如果你在线查找该算法，你会找到该哈希表达式：

```cpp
h = h * 33 + c
```

经常被编码成：

```cpp
h = ((h << 5) + h) + c
```

它们是等价的语句，将整型乘法运算替换成成本更低的移位及加法运算。实际中是否能够降低成本取决于使用的CPU。对旧机器来说这是个具有象征意义的差异，但是对当代机器来说整型乘法运算相当的快。

另一个算法的变化围绕着一个31位的返回值。

```cpp
return h & 0x7fffffff;
```

然而，GNU哈希使用完全的无符号32位整型数作为结果。

GNU binutils implementation利用`uint_fast32_t`类型来计算哈希。这一类型被定义成最快的可用整型——当前系统上机器至少能表示32位。它可以用更宽的类型实现，结果在返回前被显式的修剪成一个32位无符号值。

```cpp
static uint_fast32_t
dl_new_hash (const char *s)
{
        uint_fast32_t h = 5381;

        for (unsigned char c = *s; c != '\0'; c = *++s)
                h = h * 33 + c;

        return h & 0xffffffff;
}
```

## Dynamic Section 需求
GNU hash sections在动态符号表中放置了一些额外的有序需求。这和标准的SVR4 hash sections形成对照，SVR4允许符号以任意顺序放置，这被ELF标准所允许。

对一个标准的SVR4 hash table来说，动态符号表中包含其所有符号。然而，动态符号表中的某些符号在哈希表中永远找不到：
- LOCAL符号，除非通过重定向引用（在某些架构）
- FILE符号
- 共享对象: 所有UNDEF符号
- 可执行：任何未在PLT引用的UNDEF符号
- 特殊索引0符号（UNDEF的特殊情况）

哈希表节忽略这些符号对于正确性来说没有任何影响，且结果上来看还会减轻哈希表的拥塞，缩短哈希链，达成更好的哈希性能。

有了GNU哈希，动态符号表被分割成两部分。第一部分接受那些哈希表可以忽略的符号。GNU哈希不会对这一部分符号强加某种特定的顺序。

第二部分接受可从哈希表访问的符号。这些符号需要升序排列（hash % nbuckets），用上面描述的GNU hash函数。哈希桶(nbuckets)的个数在GNU hash section中记录，下面会有所描述。结果上来看，在单一哈希链中找到的符号在内存中是毗邻的，这也是为了更好的缓存性能。

## GNU_HASH section
一个GNU_HASH节由四个分离的部分组成，按顺序如下：
- Header
32位字数组(4)提供节参数：
	- nbuckets
	哈希桶个数
	- symndx
	动态符号表有dynsymcount数量的符号。symndx是动态符号表中可被访问的第一个符号的索引。这意味着有(dynsymcount-symndx)个符号可以通过哈希表访问到。
	- maskwords
	在哈希表节的Bloom filter部分中，ELFCLASS单位大小的掩码个数（每个掩码可以为32位或64位）。这个值必须非零，同时必须是一个2的幂次数，见下面的描述。
	
	注意到值0可以被解释成哈希节中不存在Bloom filter。然而，GNU linkers不会做此处理——GNU哈希节永远至少包含1个掩码。
	- shift2
	Bloom filter使用的位移计数
- Bloom Filter
GNU_HASH sections包含一个Bloom filter。该过滤器用于快速拒绝在对象中不存在的符号查找。对ELFCLASS32对象来说，Bloom filter的掩码是32位的，ELFCLASS64则是64位。
- Hash Buckets
nbuckets长度的32位哈希桶数组。
- Hash Values
32位哈希链值的数组，长度为(dynsymcount-symndx)，每个符号都源自动态符号表的第二部分。

Header，哈希桶以及哈希链永远都是32位字，而Bloom filter可能是32或64位，这取决于对象的类别。这意味着ELFCLASS32 GNU_HASH节仅包含32位字，因此他们节头的sh_entsize域被设置为4。ELFCLASS64 GNU_HASH节包含混合元素尺寸，因此sh_entsize被设置成0。

假设哈希节是对齐的、可正确访问ELFCLASS尺寸的字，在Bloom filter前面的(4)32位字直接确保了filter的掩码永远是对齐的，并且可以在内存中被正确的直接访问。

## Bloom Filter
GNU hash sections包含一个Bloom filter。Bloom filters是概率性的，意味着可能会错检，但是不可能误检（简单来说，就是通过布隆过滤器的不一定在hash表中，但不通过的一定不在表中）。filter用来快速拒绝不在对象中的符号名，避免更多昂贵的哈希查找操作。通常地，只有一个对象在进程中拥有给定的符号。跳过对其他所有对象的哈希操作可以显著地增快符号查询。

filter由maskwords个字组成，每一个字都是32位（ELFCLASS32）或64位(ELFCLASS64)，这取决于对象的类别。在下面的讨论中，C将被用来表示一个掩码的尺寸，单位是位(bits)。所有掩码共同组成一个逻辑的(C * maskwords)的bitmask。

GNU hash使用一个k=2的Bloom filter，这意味着会有两个独立的哈希函数为每个符号所使用。Bloom filter参考包含下面的陈述：

> 设计k个不同的独立哈希函数的需求，当k较大时应被禁止。对一个良好的具有宽输出的哈希函数来说，哈希的不同bit-fields之间的关联应该很小，所以这类哈希可以用来生成多重“不同的”哈希函数，其通过对输出进行切片操作做成多重位域。

GNU哈希使用的哈希函数具有这一特性。事实上，正是利用上面描述的单一哈希函数生成了两个Bloom filter所需要的哈希函数。

```cpp
H1 = dl_new_hash(name);
H2 = H1 >> shift2;
```

如上面讨论，链接编辑器决定使用多少个掩码（maskwords），以及第一个哈希结果右移产生的第二个哈希(shift2)的量。掩码用得越多，hash section就越大、漏检率越低。在私人email中我被告知GNU linker首先从哈希表（dynsymcount-symndx）符号数以2为底的对数得到shift2，对ELFCLASS32最小值为5，对ELFCLASS64最小值为6。这些值显式的记录在hash section中，以便于让链接编辑器可以在未来灵活的修改它们，应更好的去实现启发式。

Bloom filter掩码对两个哈希值的每一个都设置一位。基于Bloom filter参考，该字包含每一位，设置位的计算方法应该如下：

```cpp
N1 = ((H1 / C) % maskwords);
N2 = ((H2 / C) % maskwords);

B1 = H1 % C;
B2 = H2 % C;
```

构筑filter时构造位：

```cpp
bloom[N1] |= (1 << B1);
bloom[N2] |= (1 << B2);
```

并且在随后的测试filter：

```cpp
(bloom[N1] & (1 << B1)) && (bloom[N2] & (1 << B2))
```

GNU hash以一种有意义的方式偏离了上述的定义。并非分别计算N1和N2，而是使用单一掩码计算的和前者中N1一致的值。这是个清醒的决定，GNU hash开发组旨在优化缓存表现：

*比起使用两个不同的Ns，这使得Bloom filter的2个哈希函数依赖性更强，但在我们的测算中对于那些该拒绝的查询仍然具有非常好的拒绝率，且缓存更加友好。在查找中我们尽可能的触碰最少的缓存行，这一点尤为重要。*

因此，在GNU hash中，单一掩码实际上计算如此：

```cpp
N = ((H1 / C) % maskwords);
```

Bloom filter掩码N的两个位设置如下：

```cpp
BITMASK = (1 << (H1 % C)) | (1 << (H2 % C));
```

链接编辑器设置这些位的方法：

```cpp
bloom[N] |= BITMASK;
```

运行时链接器使用的测试如下：

```cpp
(bloom[N] & BITMASK) == BITMASK;
```

## Bit Fiddling: 为何掩码数应为2的幂次数
通常来讲，一个Bloom filter可以使用一个任意的字数构成。然而，如同前面提到的，GNU hash的maskwords必须是2的幂次数。这一需求允许下面的模操作：

```cpp
N = ((H1 / C) % maskwords);
```

可以写成简单的掩码操作替代：

```cpp
N = ((H1 / C) & (maskwords-1));
```

注意到(maskwords-1)可以预计算一次

```cpp
MASKWORDS_BITMASK = maskwords - 1;
```

且被每个哈希使用：

```cpp
N = ((H1 / C) & MASKWORDS_BITMASK);
```

## Bloom Filter特殊情况
Bloom filters有一对有趣的特殊情况：
- 当一个Bloom filter所有位均置位时，所有的测试结果都是True(accept)值。GNU linker利用这一优势设定所有位均置位的单字Bloom filter为“disable”态。filter仍然在那里，也仍然在使用，作为最小限度的上限，它允许所有事物通过。
- Bloom filter所有位均未置位时，测试结果全部返回False。这种情况在ELF文件中相当少见，毕竟一个没有导出任何符号的对象具有有限的应用性。然而，有时候对象就是以这种方式构造，依赖于init/fini节来引起对象代码的执行。

## Hash Buckets
跟随Bloom filter的是哈希桶(nbuckets个)，以32位为单位。数组中第N位置处的值是到动态符号表最低的索引：

```cpp
(dl_new_hash(symname) % nbuckets) == N
```

既然动态符号表由相同的键排序（hash % nbuckets），那么dynsym[buckets[N]]就是哈希链中的第一个符号，哈希链将包含所期望的符号如果存在的话。

一个桶元素会包含索引0如果在哈希表中对于给定的值N没有任何符号。dynsym的索引0是一个保留值，该索引在一个合法的符号上不会出现，也因此没有歧义。

## Hash Values
GNU hash section的最后一部分包含(dynsymcount-symndx)个32位字，每个条目都是动态符号表第二部分中的每个符号。每个字的前31位包含和符号表值一致的31位值。最后一位用做阻塞位。当符号是给定哈希链中最后一个或者两个桶的边界时其置1。

```cpp
lsb = (N == dynsymcount - 1) ||
	((dl_new_hash(name[N]) % nbuckets)
	!= (dl_new_hash(name[N + 1]) % nbuckets))

hashval = (dl_new_hash(name) & ~1) | lsb;
```

### 使用GNU Hash查询符号
下列展示了一个符号怎样在使用了GNU hash section的对象中查找的过程。我们假设包含需要信息的内存记录是存在的。

```cpp
typedef struct {
        const char      *os_dynstr;      /* Dynamic string table */
        Sym             *os_dynsym;      /* Dynamic symbol table */
        Word            os_nbuckets;     /* # hash buckets */
        Word            os_symndx;       /* Index of 1st dynsym in hash */
        Word            os_maskwords_bm; /* # Bloom filter words, minus 1 */
        Word            os_shift2;       /* Bloom filter hash shift */
        const BloomWord *os_bloom;       /* Bloom filter words */
        const Word      *os_buckets;     /* Hash buckets */
        const Word      *os_hashval;     /* Hash value array */
} obj_state_t;
```

为了简化因素，我们省略控制不同ELF classes的细节。上面的情景中，Word是32位无符号数，BloomWord是32或64位数，取决于ELFCLASS，Sym相应就是Elf32_Sym和Elf64_Sym。

把包含上述信息的变量给一个对象，下列的伪代码返回一个指针，如果对象中期望的符号存在，则指向它，否则指向NULL。

```cpp
Sym *
symhash(obj_state_t *os, const char *symname)
{
        Word            c;
        Word            h1, h2;
        Word            n;
        Word            bitmask; 
        const Sym       *sym;
        Word            *hashval;

        /*
         * Hash the name, generate the "second" hash
         * from it for the Bloom filter.
         */
        h1 = dl_new_hash(symname);
        h2 = h1 >> os->os_shift2;

        /* Test against the Bloom filter */
        c = sizeof (BloomWord) * 8;
        n = (h1 / c) & os->os_maskwords_bm;
        bitmask = (1 << (h1 % c)) | (1 << (h2 % c));
        if ((os->os_bloom[n] & bitmask) != bitmask)
                return (NULL);

        /* Locate the hash chain, and corresponding hash value element */
        n = os->os_buckets[h1 % os->os_nbuckets];
        if (n == 0)    /* Empty hash chain, symbol not present */
                return (NULL);
        sym = &os->os_dynsym[n];
        hashval = &os->os_hashval[n - os->os_symndx];

        /*
         * Walk the chain until the symbol is found or
         * the chain is exhausted.
         */
        for (h1 &= ~1; 1; sym++) {
                h2 = *hashval++;

                /*
                 * Compare the strings to verify match. Note that
                 * a given hash chain can contain different hash
                 * values. We'd get the right result by comparing every
                 * string, but comparing the hash values first lets us
                 * screen obvious mismatches at very low cost and avoid
                 * the relatively expensive string compare.
                 *
                 * We are intentionally glossing over some things here:
                 *
                 * - We could test sym->st_name for 0, which indicates
                 *   a NULL string, and avoid a strcmp() in that case.
                 *
                 * - The real runtime linker must also take symbol
                 *   versioning into account. This is an orthogonal
                 *   issue to hashing, and is left out of this
                 *    example for simplicity.
                 *
                 * A real implementation might test (h1 == (h2 & ~1), and then
                 * call a (possibly inline) function to validate the rest.
                 */
                 if ((h1 == (h2 & ~1)) &&
                     !strcmp(symname, os->os_dynstr + sym->st_name))
                         return (sym);

                 /* Done if at end of chain */
                 if (h2 & 1)
                         break;
        }

        /* This object does not have the desired symbol */
        return (NULL);
}
```

### 更新
2010年8月26日
Per Lidén 指出上面例子的一些错误。有3处"sizeof(BloomWord)"的使用应为"sizeof(BloomWord)*8"，因为我们处理的是位，而不是字节。并且，在for()循环中有个打字错误。谢谢你的更正。

2011年9月19日
Subin Gangadharan告诉我另外一个打字错误。在文末的代码实例中，我写了：

```cpp
h1 = dl_new_hash(symname);
h2 = h2 >> os->os_shift2;
```

第二行应该是`h2 = h1 >> os->os_shift2;`。

在翻阅这篇旧文章时，我发现了一些'*'字符被转义成了'\*'。我仅能推测这是当Oracle转换我们Sun的博文时发生的。我修复了原本的字符。我也借此机会更新了我的邮箱地址成当前的Oracle.COM格式。

2013年8月15日
Mikael Vidstedt指出另一个小打字错误。在讨论GNU hash如何计算单一N，而不是N1和N2时，我做了如下的总结：

运行时连接器使用的测试如下：

```cpp
(bloom[N1] & BITMASK) == BITMASK;
```

N1应该是简单的N:

```cpp
(bloom[N] & BITMASK) == BNITMASK;
```

谢谢你善意的文字。