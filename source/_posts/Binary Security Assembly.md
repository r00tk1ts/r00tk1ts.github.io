---
title: Binary-Security-Assembly
date: 2018-02-26 21:31:11
categories: resources
tags:
	- resources
	- tutorial
---
在二进制研究中对优质的资源进行整合。老年人表示好记性的确不如烂笔头。方便自己lookup&review，也为后来人乘凉。本博文永久更新。

<!--more-->

# Binary-Security-Assembly

## Books

一些基础的必修书籍以及品相平庸的我就不一一枚举了，以下提到的都是我已经读过或者正在阅读的优质书籍。我会简要的写一点评价，鉴于本人才疏学浅，难免有失偏颇，故仅供参考。

- 《0day安全：软件漏洞分析技术》

  ![](/images/misc/20180226_1.jpg)

  > 书评：
  >
  > ​	可以说是Windows漏洞研究入门的圣经。也正是这本书带我入了漏洞分析的大门(keng)。由于本书纂写的时间较早，很多现代的流行保护技术并不能cover，对堆的理解也仅限于2k/xp sp1的时代。但其中对经典栈溢出的几种利用手法、ASLR/DEP/GS的绕过、SEH的机理再到safeSEH的保护，可以说是教科书般的诠释。书中亦曾提到fuzzing相关内容以及web安全的只言片语（彼时web安全尚不成体系），平添趣味。
  >
  > ​	早年我曾有幸购入一本实体书，现已绝版。电子资源pdf比比皆是，不予枚举。
  >
  > 推荐指数：10

- 《加密与解密》

  ![](/images/misc/20180226_2.jpg)

  > 书评：
  >
  > ​	看雪掌门人的教科书，如果说0day安全带我入了漏洞分析的大门，那么这本书带给我的则是基础技能的学习（调试器、静态动态分析、PE结构、加密解密算法）以及软件保护方面知识的扩展。本书面面俱到却也往往点到为止，旨在做一个科普形式的入门级铺叙。看雪论坛此前发起过本书的升级出版事宜，却又无疾而终。
  >
  > ​	时隔多年，依稀记着买到实体书的那个夜晚，通宵达旦，畅快淋漓。
  >
  > 推荐指数：10

- 《漏洞战争》

  ![](/images/misc/20180226_3.jpg)

  > 书评：
  >
  > ​	你可能不认识riusksk这个id，但一定听说过泉哥的传奇故事。这本书记录了泉哥3年来呕心沥血的优质CVE调试笔记，最终于2016年发布。由于时间线颇为靠后，所以涵盖了很多现代漏洞分析与利用的奇技淫巧，书中的CVE主要集中在Windows平台，也有几个Android相关的漏洞解读，横贯了栈溢出、堆溢出、UAF、内核漏洞等方方面面。
  >
  > ​	这本书的阅读门槛较高，需要读者预先掌握漏洞分析各方面的理论基础并具有一定的动手能力。CVE的分析过程限于篇幅无法详尽到每个点（各种巧妙的exp，不是靶场那种一目了然可以比拟的），需要自行复盘以挖掘其中精妙。
  >
  > ​	我是在这本书出版不久的时候，逛书店无意中看到，想来也是有缘，遂匆匆购入。
  >
  > 推荐指数：10

- 《C++反汇编与逆向分析技术揭秘》

  ![](/images/misc/20180226_4.jpg)

  > 书评：
  >
  > ​	手把手教你怎么反汇编C/C++。对C/C++常见的语法糖的反汇编表现形式进行了拆解。本书适合逆向分析初学者入门，想要完全掌握本书的内容，需要对Windows底层有一定了解。
  >
  > ​	书的最后章节还对臭名昭著的蠕虫——熊猫烧香进行了分析，作者也是很有情怀。
  >
  > 推荐指数：9

- 《IDA Pro权威指南》

  ![](/images/misc/20180226_5.jpg)

  > 书评：
  >
  > ​	屠龙宝刀、点击就送！很自信的说，玩二进制的没有人离得开IDA。这本书就是IDA的说明书，全面、准确、不可多得。很惭愧的说，我没完整的读过这本书（枯燥啊），多数时候是作为reference，这可能就是我只会拙劣的使用IDA的原因吧。
  >
  > 推荐指数：7

- 《Windows PE权威指南》

  ![](/images/misc/20180226_6.jpg)

  > 书评：
  >
  > ​	专门讲Windows的PE格式，不仅充当了手册，还涉及了PE相关的编程技术。也是比较枯燥的一本书，但绝对好过官方手册。如果你想对PE有一个全盘的理解，进而研究Windows的可执行文件的保护与壳、PE病毒分析，那么本书是必读的捷径。
  >
  > 推荐指数：7

- 《程序员的自我修养——链接、装载与库》

  ![](/images/misc/20180226_7.jpg)

  > 书评：
  >
  > ​	很早以前就在书店看到过这本书，由于名字过于中二，直接就把武断的归类于什么21天精通XXX这类书籍。后来无意中在看雪论坛看到了该书的节选引用，一时震惊，果断入了实体书。
  >
  > ​	这本书涉及了Windows和Linux两个平台，讲解了PE与ELF两种文件格式，也剖析了两个平台的应用程序在编译、链接、装载、运行四个时刻发生的事宜。美中不足的是每个阶段的阐述还是不够细腻，我在阅读的时候还参考了源码以及大量的其他资源。然在有限的篇幅中能做出如此准确全面的阐释，足以叹为观止。
  >
  > 推荐指数：10

- 《恶意代码分析实战》

  ![](/images/misc/20180226_8.jpg)

  > 书评：
  >
  > ​	诸葛领队译，质量绝对保证，内容比较杂，围绕着恶意代码的常见表征对调试手法进行了讲解。书篇幅较长，不乏干货，美中不足的是技术深度参差不齐，字里行间可以看出作者在攥写本著作时，定位难度系数时的摇摆犹豫。这也是国外很多技术作家的通病。
  >
  > ​	总体来讲还是很适合新手来通读的，一些看不懂的地方就暂且放一放，时机未到不用硬磕。
  >
  > 推荐指数：8

- 《Reverse Engineering for Beginners》

  ![](/images/misc/20180226_9.jpg)

  > 书评：
  >
  > ​	简称RE4B，见名知意。与其说是书籍，不如说是一位逆向工程师精心编排的手稿汇编。本书在各流行编译器、CPU架构上对高级语言（主要是C/C++）的反汇编指令进行了阐释，内容颇丰，动手能力强，非基础章节的阅读还需要自行扩展阅读大量的资料。书是免费的但没有中文译本（其实有但尚未完善），读英文译本即可。
  >
  > 推荐指数：9

- 《软件保护与分析技术——原理与实践》

  ![](/images/misc/20180226_10.jpg)

  > 书评：
  >
  > ​	看雪大牛netsowell的密界心得。本书归纳了流行的软件保护技术以及软件破解手法，但对于VM技术则点到为止（实际上在另一本书中展开）。本书还手把手指导了一些调试软件时需要的小工具的制作，真正做到授人以渔。本书主要是归纳，所以内容不是很详细，适合有了一定的破解经验但不知所以然的安全研究者。
  >
  > 推荐指数：7

- 《代码虚拟与自动化分析》

  ![](/images/misc/20180226_11.jpg)

  > 书评：
  >
  > ​	netsowell的进阶之作，专门讲解密界当下最为主流，难度也最高的VM技术。这本书我还没看完，个人对于VM技术的理解也止步于入门阶段，待我择日研习后，再做注解。
  >
  > 推荐指数：null

- 《软件调试》

  ![](/images/misc/20180226_12.jpg)

  > 书评：
  >
  > ​	微软工程师手把手教你Windows调试。本书讲调试相关的理论，从底层到上层，看罢会对整个调试机制有一个全盘的认识。这本书无疑是硬核的，由于书主要讲调试，所以在谈到Windows内核的中断、异常、陷阱以及ring0/3态时裁剪掉了很多内容。这就需要读者预先熟悉Windows内核。
  >
  > 推荐指数：9

- 《格蠹汇编  软件调试案例集锦》

  ![](/images/misc/20180226_13.jpg)

  > 书评：
  >
  > ​	《软件调试》的姊妹篇，这本书记载了使用Windbg调试解决问题的大量案例。一些理论讲解会充斥在调试过程中，阅读起来趣味横生，悬疑小说即视感。
  >
  > ​	很早以前，我曾在书店看到过这两本，多年以后始觉其价值所在，但恨已无实体书可入，心下戚戚然。
  >
  > 推荐指数：9

- 《黑客攻防技术宝典——系统实战篇》

  ![](/images/misc/20180226_14.jpg)

  > 书评：
  >
  > ​	这本书相当老了，如果没有那本0day，或许这本书会成为入门的圣经。书名虽然中二，但内容确实扎实。书名可以看出这是一个系列，而这本专注二进制安全的则是最老的一本。
  >
  > ​	我只读过前半本，后面的测试审计则没有研读过。这本书的立足点比较高，比起落实下去的奇技淫巧，更多讲授的是归纳出来的漏洞分析方法论。非常希望这本书能够升级换代。
  >
  > 推荐指数：7

- 《Rootkits——Windows内核的安全防护》

  ![](/images/misc/20180226_15.jpg)

  > 书评：
  >
  > ​	Windows内核驱动编程中，恶意程序与防护软件常用的各类手法。倾向于实战性的铺叙，原理层面则谈得不多。时隔多年，不知道书中资源所在的website还健在否？
  >
  > 推荐指数：7

- 《黑客攻防技术宝典——反病毒篇》

  ![](/images/misc/20180226_16.jpg)

  > 书评：
  >
  > ​	比较新的书，不久前刚入了一本实体还没来得及阅读。走马观花了一番，虽涉猎甚广但不够深入。有时间要好好研读。
  >
  > 推荐指数：null

- 《Windows核心编程》

  ![](/images/misc/20180226_17.jpg)

  > 书评：
  >
  > ​	经久不衰的神作，站在内核的角度居高临下看用户态的那些事儿。阅读本书，不仅能够掌握Windows系统各个模块API的正确使用，也能够窥得Windows内核一二。
  >
  > ​	我在阅读这本书的时候，尚还只是堪堪刷过《Windows程序设计》的毛头小子，彼时对于OS的理解也是停留在课本所讲的概念上。这本书启蒙了我对OS的认知，在后期阅读《Windows内核情景分析》、《Windows内核原理与实现》、《Windows Internals》等Windows内核相关书籍时，理解起来也更加平滑。
  >
  > 推荐指数：10

- 《Windows程序设计》

  ![](/images/misc/20180226_18.jpg)

  > 书评：
  >
  > ​	迫于形势，这本书的第六版已经转去讲C#了（确切的说，是.Net），而第五版虽然老旧，确是基于Windows SDK进行API的讲解，段落章节更是渗透了Windows系统繁杂的知识点。本书的第五版被业内人士奉为圣经，教科书般的展示了SDK方方面面正确的开发姿势。
  >
  > ​	这本书非常之厚，对于入门者有着强大的劝退效果，但实际上却并不难啃。另一方面，研究Windows底层二进制是无论如何也绕不开原生API编程的。
  >
  > 推荐指数：9

- 《Windows内核安全与驱动开发》

  ![](/images/misc/20180226_19.jpg)

  > 书评：
  >
  > ​	在Windows驱动编程界内，楚狂人算得上赫赫有名，而本书也基本上出自楚狂人之手。实际上在本书问世前，有着Windows驱动编程的三剑客——《天书夜读》、《寒江独钓》、《竹林蹊径》。本书是前两本的重构（前两本也是楚狂人写的），修改了一些示例代码的bug，更新了一些Windows7以上平台的新知识点。
  >
  > ​	对于二进制安全研究者来说，Windows内核驱动编程是颇为重要的基本功，尤其是从事安全防护、反病毒木马的工程师。
  >
  > ​	我在这本书问世前，已经读过了它的两个蓝本，唯有一本竹林蹊径不曾看过。记得竹林蹊径主要是讲PCI、USB相关，而我一时竟无此意欲，毕竟不同组件的驱动程序在Windows平台上迥然有别。
  >
  > 推荐指数：7

- 《Windows内核原理与实现》

  ![](/images/misc/20180226_20.jpg)

  > 书评：
  >
  > ​	这本书很老了，老到甚至不是在wrk1.2版本上做的阐释。潘老师的这本书深入浅出，对Windows内核的各个层面、各个组件进行了较为完整的陈述。这本书没有通篇的代码引用，采用的是高度的语言概括，这对于一些阅读源码较为吃力或急于求成的人来说，无疑是了解Windows内核最为有效的捷径（当然潘老师也在书中的每个地方给出了源码所在位置，并期望读者去自行阅读）。
  >
  > ​	对于不开源的Windows来说，wrk极具参考价值，不管新的机制如何更迭，最原始的设计思想不会变更。
  >
  > ​	Windows内核体积过巨，这本书也无出意外的无法涵盖内核的方方面面，另一方面，想要通过堪堪千页便通晓内核，也未免太奢侈了。
  >
  > 推荐指数：10

- 《Windows Internals》

  ![](/images/misc/20180226_21.jpg)

  > 书评：
  >
  > ​	微软工程师编纂的神作，所以说解铃还须系铃人，自家人写的剖析注解就是透彻。遗憾的是，这本书仅有第六版上册的译本（深入解析Windows操作系统），下册的译本据说是夭折了，原因未知。而本书的下册内容同样丰富（存储管理、内存管理、文件系统、I/O），所以需要读者有一定的英文功底。好在看雪论坛上曾有前辈翻译过下册的几个比较重要的篇章，难能可贵的是，还加入了自身的理解，不得不感概同是天涯沦落人。
  >
  > ​	当然，本书的作者考虑到需要避嫌，所以并没有直接拉出源代码来层层解读，其所能做的只能是高度的抽象与概括，阐述实现的思想与设计方案。所以，在阅读本书时，多参考wrk，多使用windbg调试kernel，会有不小的收获。
  >
  > 推荐指数：10

- 《Windows内核情景分析》

  ![](/images/misc/20180226_22.jpg)

  > 书评：
  >
  > ​	我时常亲切的称这本书为“毛批”。毛德操老师另外一本姊妹篇是《Linux内核源代码情景分析》，两书的风格相近，这一本的布局表达更成熟老练一些。由于Windows不开源，所以本书是基于ReactOS内核分析的，至于ReactOS是一群逆向爱好者对Windows内核的翻写，可以说除了wrk，它的参考价值最大（尽管一些细节上的处理和wrk不同，但瑕不掩瑜）。
  >
  > ​	我买的第一套（分上下册）通往Windows内核的书籍，尽管由于当初水平有限，尘封多年，却也拓展了视野。
  >
  > 推荐指数：8

- 《Linux内核源代码情景分析》

  ![](/images/misc/20180226_23.jpg)

  > 书评：
  >
  > ​	另一本“毛批”，年代久远，讲解的甚至是2.4的内核。这本书对于细节上的讲解非常细腻，对每个知识点都进行了深度递归。然而，由于篇幅有限，OS的各个模块都只是筛选出几个知识点进行了解读，很多重要的内容并未叙述，盖因毛老师考虑到既然无法深入讲解到每个知识点，那么只言片语也毫无意义，干脆绝口不提。
  >
  > 推荐指数：7

- 《深入理解Linux内核》

  ![](/images/misc/20180226_24.jpg)

  > 书评：
  >
  > ​	大名鼎鼎的ULK，我个人是建议读英文原版的。一方面有些相近的概念翻译过来有点冲突（比如各种组件中的缓存），另一方面一些术语转化成中文丢失了原本的韵味。原版亦是行云流水，而书的内容也类似于潘爱民的《Windows内核情景分析》的风格，摒弃了通篇的代码，从逻辑上概括了每个组件的设计与思想。
  >
  > ​	这本书是基于2.6版本的内核，与当前4.x的内核结构还是有着不小的差别，尽管如此，在当下还没有任何一本讲4.x内核的书可以相形媲美（《奔跑吧Linux内核》还差点火候，内容不够详实且多处字眼不准确起误导作用）。如果想要了解Linux内核，那么ULK无疑是首选。
  >
  > 推荐指数：10

- 《linux内核设计与实现》

  ![](/images/misc/20180226_25.jpg)

  > 书评：
  >
  > ​	这本书只有堪堪300多页，却涵盖了Linux内核的诸多关键组件。这本书对进程调度、中断和内核同步有着非常详细的介绍，可以作为ULK的姊妹篇来通读。本书言简意赅，对内核进行了一番抽丝剥茧，非常适合新手入门Linux内核。
  >
  > 推荐指数：8

- 《Linux设备驱动程序》

  ![](/images/misc/20180226_26.jpg)

  > 书评：
  >
  > ​	Linux驱动开发必备，就好比学Windows驱动要看楚狂人和张帆一样。这本书篇幅较短，且没有对示例程序进行逐一的阐释，所以如果脱离了示例程序，那么只能是看得云里雾里。诚然，对于驱动程序来说，动手实践才是王道！
  >
  > ​	如果你对Linux内核有了较为深入的理解，那么阅读本作当是轻车熟路，也更容易理解什么样的驱动程序是正确的。
  >
  > 推荐指数：9

- 《Unix环境高级编程》

  ![](/images/misc/20180226_27.jpg)

  > 书评：
  >
  > ​	POSIX标准的那些事儿，使用标准的API来进行用户层的C编程，可以说是POSIX编程的圣经。本书涵盖面非常之广，从文件、I/O到进程环境与控制再到进程间通信，教科书般展示了POSIX编程之道。
  >
  > ​	这本书是我入门Linux编程的第一本书，如同《Windows程序设计》一般，博大精深，概念清晰，内容权威，理解起来很舒服。
  >
  > 推荐指数：10

- 《Unix网络编程 卷1：套接字联网API》

  ![](/images/misc/20180226_28.jpg)

  > 书评：
  >
  > ​	Windows有winsock，Linux对应也有自己的socket库，在学习了计算机网络、TCP/IP协议后，就可以来尝试使用socket编程完成各种各样的网络应用。本书着重讲socket编程但绝不仅限制于socket编程。
  >
  > ​	由于工作性质，每每和TCP/IP打交道，所以这本《Unix环境高级编程》的姊妹篇也就一并读了。这本书的卷二是讲进程间通信的，我没有读过（由于此前已读过《Unix环境高级编程》），所以就不提出来了。
  >
  > 推荐指数：7

- 《琢石成器 windows环境下32位汇编语言程序设计》

  ![](/images/misc/20180226_29.jpg)

  > 书评：
  >
  > ​	年代久远的Win32汇编。本书不是讲汇编语言，而是讲如何用masm32来编写Windows程序（调用Windows原生API）。老罗的惊艳之作，算是一本比较另类的Windows程序设计书籍。
  >
  > ​	我早年的时候非常喜欢这本书，一直苦于售罄而未能购入一本实体书。后来在一位毕业学长的书摊前淘得此书，九成新（学长说没有翻过），以高价购入（记得学长很忐忑的要价40，我直接给了50，在学长学姐一片关爱智障的眼神中心满意足的离开），慰矣。
  >
  > 推荐指数：6


## Course

- [Modern Binary Exploitation](http://security.cs.rpi.edu/courses/binexp-spring2015/)

  - [Syllabus and Review](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/1/01_lecture.pdf)
  - [Tools and Basic Reverse Engineering](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/2/02_lecture.pdf)
  - [Extended Reverse Engineering](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/3/03_lecture.pdf)
  - [Introduction to Memory Corruption](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/5/04_lecture.pdf)
  - [Shellcoding](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/7/05_lecture.pdf)
  - [Format Strings](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/9/06_lecture.pdf)
  - [DEP and ROP](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/11/07_lecture.pdf)
  - [Secure Systems and Game Console Exploitation](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/13/08_lecture.pdf)
  - [Address Space Layout Randomization](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/15/09_lecture.pdf)
  - [Heap Exploitation](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/17/10_lecture.pdf)
  - [Misc concepts & Stack Canaries](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/19/11_lecture.pdf)
  - [C++ Concepts and Differences](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/21/12_lecture.pdf)
  - [Kernel Exploitation](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/23/13_lecture.pdf)
  - [Exploitation on 64bit, ARM, Windows](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/26/14_lecture.pdf)
  - [Automation & The Future of Exploitation](http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/27/15_lecture.pdf)
- [Linux (x86) Exploit Development Series](https://sploitfun.wordpress.com/2015/06/26/linux-x86-exploit-development-tutorial-series/)

  - Basic Vulnerabilities
    - [Classic Stack Based Buffer Overflow](https://sploitfun.wordpress.com/2015/05/08/classic-stack-based-buffer-overflow/)
    - [Integer Overflow](https://sploitfun.wordpress.com/2015/06/23/integer-overflow/)
    - [Off-By-One (Stack Based)](https://sploitfun.wordpress.com/2015/06/07/off-by-one-vulnerability-stack-based-2/)
  - Bypassing Exploit Mitigation Techniques
    - [Bypassing NX bit using return-to-libc](https://sploitfun.wordpress.com/2015/05/08/bypassing-nx-bit-using-return-to-libc/)
    - [Bypassing NX bit using chained return-to-libc](https://sploitfun.wordpress.com/2015/05/08/bypassing-nx-bit-using-chained-return-to-libc/)
    - Bypasing ASLR
      - [Part I using return-to-plt](https://sploitfun.wordpress.com/2015/05/08/bypassing-aslr-part-i/)
      - [Part II using brute force](https://sploitfun.wordpress.com/2015/05/08/bypassing-aslr-part-ii/)
      - [Part III using GOT overwrite and GOT dereference](https://sploitfun.wordpress.com/2015/05/08/bypassing-aslr-part-iii)
  - Heap Vulnerabilites
    - [Heap overflow using unlink](https://sploitfun.wordpress.com/2015/02/26/heap-overflow-using-unlink/)
    - [Heap overflow using Malloc Maleficarum](https://sploitfun.wordpress.com/2015/03/04/heap-overflow-using-malloc-maleficarum/)
    - [Off-By-One (Heap Based)](https://sploitfun.wordpress.com/2015/06/09/off-by-one-vulnerability-heap-based)
    - [Use After Free](https://sploitfun.wordpress.com/2015/06/16/use-after-free/)
- [liveoverflow: Binary Hacking Course](http://liveoverflow.com/binary_hacking/index.html)
- [FuzzySecurity tutorials](https://www.fuzzysecurity.com/tutorials.html)

  - Windows Exploit Development Tutorial Series
    - [Part1: Introduction to Exploit Development](https://www.fuzzysecurity.com/tutorials/expDev/1.html)
    - [Part2: Saved Return Pointer Overflows](https://www.fuzzysecurity.com/tutorials/expDev/2.html)
    - [Part3: Structured Exception Handler(SEH)](https://www.fuzzysecurity.com/tutorials/expDev/3.html)
    - [Part4: Egg Hunters](https://www.fuzzysecurity.com/tutorials/expDev/4.html)
    - [Part5: Unicode 0x00410041](https://www.fuzzysecurity.com/tutorials/expDev/5.html)
    - [Part6: Writing W32 shellcode](https://www.fuzzysecurity.com/tutorials/expDev/6.html)
    - [Part7: Return Oriented Programming](https://www.fuzzysecurity.com/tutorials/expDev/7.html)
    - [Part8: Spraying the Heap <Chapter1: Vanilla EIP>](https://www.fuzzysecurity.com/tutorials/expDev/8.html)
    - [Part9: Spraying the Heap <Chapter2: Use-After-Free>](https://www.fuzzysecurity.com/tutorials/expDev/11.html)
    - [Part10: Kernel Exploitation -> Stack Overflow](https://www.fuzzysecurity.com/tutorials/expDev/14.html)
    - [Part11: Kernel Exploitation -> Write-What-Where](https://www.fuzzysecurity.com/tutorials/expDev/15.html)
    - [Part12: Kernel Exploitation -> Null Pointer Dereference](https://www.fuzzysecurity.com/tutorials/expDev/16.html)
    - [Part13: Kernel Exploitation -> Uninitialized Stack Variable](https://www.fuzzysecurity.com/tutorials/expDev/17.html)
    - [Part14: Kernel Exploitation -> Integer Overflow](https://www.fuzzysecurity.com/tutorials/expDev/18.html)
    - [Part15: Kernel Exploitation -> UAF](https://www.fuzzysecurity.com/tutorials/expDev/19.html)
    - [Part16: Kernel Exploitation -> Pool Overflow](https://www.fuzzysecurity.com/tutorials/expDev/20.html)
    - [Part17: Kernel Exploitation -> GDI Bitmap Abuse(Win7-10 32/64bit)](https://www.fuzzysecurity.com/tutorials/expDev/21.html)
    - [Part18: Kernel Exploitation -> RS2 Bitmap Necromancy](https://www.fuzzysecurity.com/tutorials/expDev/22.html)
    - [Part19: Kernel Exploitation -> Logic bugs in Razer rzpnk.sys](https://www.fuzzysecurity.com/tutorials/expDev/23.html)
  - Windows Heap Exploitation
    - [What's Going On Here b33f?](https://www.fuzzysecurity.com/tutorials/mr_me/1.html)
    - [Heap Overflows For Human 101](https://www.fuzzysecurity.com/tutorials/mr_me/2.html)
    - [Heap Overflows For Human 102](https://www.fuzzysecurity.com/tutorials/mr_me/3.html)
    - [Heap Overflows For Human 102.5](https://www.fuzzysecurity.com/tutorials/mr_me/4.html)
    - [Heap Overflows For Human 103](https://www.fuzzysecurity.com/tutorials/mr_me/5.html)
    - [Heap Overflows For Human 103.5](https://www.fuzzysecurity.com/tutorials/mr_me/6.html)
  - Linux Exploit Development Tutorial Series
    - [Part1: Introduction to Linux Exploit Development](https://www.fuzzysecurity.com/tutorials/expDev/9.html)
    - [Part2: Linux Format String Exploitation](https://www.fuzzysecurity.com/tutorials/expDev/10.html)
    - [Part3: Buffer Overflow(Pwnable.kr -> bof)](https://www.fuzzysecurity.com/tutorials/expDev/12.html)
    - [Part4: Use-After-Free(Pwnable.kr -> uaf)](https://www.fuzzysecurity.com/tutorials/expDev/13.html)
  - Occult Windows Hacking
    - [Windows Privilege Escalation Fundamentals](https://www.fuzzysecurity.com/tutorials/16.html)
    - [I'll Get Your Credentials ... Later!](https://www.fuzzysecurity.com/tutorials/18.html)
    - [Windows Userland Persistence Fundamentals](https://www.fuzzysecurity.com/tutorials/19.html)
    - [Powershell PE Injection: This is not the Calc you are looking for!](https://www.fuzzysecurity.com/tutorials/20.html)
    - [Low-Level Windows API Access From PowerShell](https://www.fuzzysecurity.com/tutorials/24.html)
    - [Windows Domains, Pivot & Profit](https://www.fuzzysecurity.com/tutorials/25.html)
    - [Anatomy of UAC Attacks](https://www.fuzzysecurity.com/tutorials/27.html)
    - [Capcom Rootkit Proof-Of-Concept](https://www.fuzzysecurity.com/tutorials/28.html)
    - [Application Introspection & Hooking With Frida](https://www.fuzzysecurity.com/tutorials/29.html)
  - Malware Analysis
    - [Magnitude EK: Traffic Analysis(08/05/2015)](https://www.fuzzysecurity.com/tutorials/21.html)
    - [Angler EK JavaScript Deobfuscation: The Emperor Has No Clothes](https://www.fuzzysecurity.com/tutorials/22.html)
- Hack The Virtual Memory

  - [Hack The Virtual Memory: C strings & /proc](https://blog.holbertonschool.com/hack-the-virtual-memory-c-strings-proc/)
  - [Hack The Virtual Memory: Python bytes](https://blog.holbertonschool.com/hack-the-virtual-memory-python-bytes/)
  - [Hack the Virtual Memory: drawing the VM diagram](https://blog.holbertonschool.com/hack-the-virtual-memory-drawing-the-vm-diagram/)
  - [Hack the Virtual Memory: malloc, the heap & the program break](https://blog.holbertonschool.com/hack-the-virtual-memory-malloc-the-heap-the-program-break/)
- Exploit writing tutorial

  - [Stack Based Overflows](https://www.corelan.be/index.php/2009/07/19/exploit-writing-tutorial-part-1-stack-based-overflows/)
  - [Stack Based Overflows – jumping to shellcode](https://www.corelan.be/index.php/2009/07/23/writing-buffer-overflow-exploits-a-quick-and-basic-tutorial-part-2/)
  - [SEH Based Exploits](https://www.corelan.be/index.php/2009/07/25/writing-buffer-overflow-exploits-a-quick-and-basic-tutorial-part-3-seh/)
  - [SEH Based Exploits – just another example](https://www.corelan.be/index.php/2009/07/28/seh-based-exploit-writing-tutorial-continued-just-another-example-part-3b/)
  - [From Exploit to Metasploit – The basics](https://www.corelan.be/index.php/2009/08/12/exploit-writing-tutorials-part-4-from-exploit-to-metasploit-the-basics/)
  - [How debugger modules & plugins can speed up basic exploit development](https://www.corelan.be/index.php/2009/09/05/exploit-writing-tutorial-part-5-how-debugger-modules-plugins-can-speed-up-basic-exploit-development/)
  - [Bypassing Stack Cookies, SafeSeh, SEHOP, HW DEP and ASLR](https://www.corelan.be/index.php/2009/09/21/exploit-writing-tutorial-part-6-bypassing-stack-cookies-safeseh-hw-dep-and-aslr/)
  - [Unicode – from 0x00410041 to calc](https://www.corelan.be/index.php/2009/11/06/exploit-writing-tutorial-part-7-unicode-from-0x00410041-to-calc/)
  - [Win32 Egg Hunting](https://www.corelan.be/index.php/2010/01/09/exploit-writing-tutorial-part-8-win32-egg-hunting/)
  - [Introduction to Win32 shellcoding](https://www.corelan.be/index.php/2010/02/25/exploit-writing-tutorial-part-9-introduction-to-win32-shellcoding/)
  - [Chaining DEP with ROP](https://www.corelan.be/index.php/2010/06/16/exploit-writing-tutorial-part-10-chaining-dep-with-rop-the-rubikstm-cube/)
  - [Heap Spraying Demystified](https://www.corelan.be/index.php/2011/12/31/exploit-writing-tutorial-part-11-heap-spraying-demystified/)
  - ​
- [Malware Analysis Tutorials: a Reverse Engineering Approach](https://fumalwareanalysis.blogspot.jp/p/malware-analysis-tutorials-reverse.html)
  - [Malware Analysis Tutorial 1- A Reverse Engineering Approach (Lesson 1: VM Based Analysis Platform) ](http://fumalwareanalysis.blogspot.com/2011/08/malware-analysis-tutorial-reverse.html)
  - [Malware Analysis Tutorial 2- Introduction to Ring3 Debugging  ](http://fumalwareanalysis.blogspot.com/2011/08/malware-analysis-tutorial-reverse_31.html)
  - [Malware Analysis Tutorial 3- Int 2D Anti-Debugging ](http://fumalwareanalysis.blogspot.com/2011/09/malware-analysis-3-int2d-anti-debugging.html).
  - [Malware Analysis Tutorial 4- Int 2D Anti-Debugging (Part II)  ](http://fumalwareanalysis.blogspot.com/2011/10/malware-analysis-tutorial-4-int2dh-anti.html)
  - [Malware Analysis Tutorial 5- Int 2D in Max++ (Part III) ](http://fumalwareanalysis.blogspot.com/2011/10/malware-analysis-tutorial-5-int2d-anti.html).
  - [Malware Analysis Tutorial 6- Self-Decoding and Self-Extracting Code Segment ](http://fumalwareanalysis.blogspot.com/2011/12/malware-analysis-tutorial-6-analyzing.html).
  - [Malware Analysis Tutorial 7: Exploring Kernel Data Structure ](http://fumalwareanalysis.blogspot.com/2011/12/malware-analysis-tutorial-7-exploring.html).
  - [Malware Analysis Tutorial 8: PE Header and Export Table ](http://fumalwareanalysis.blogspot.com/2011/12/malware-analysis-tutorial-8-pe-header.html).
  - [Malware Analysis Tutorial 9: Encoded Export Table ](http://fumalwareanalysis.blogspot.com/2011/12/malware-analysis-tutorial-9-encoded.html).  
  - [Malware Analysis Tutorial 10: Tricks for Confusing Static Analysis Tools ](http://fumalwareanalysis.blogspot.com/2012/01/malware-analysis-tutorial-10-tricks-for.html).
  - [Malware Analysis Tutorial 11: Starling Technique and Hijacking Kernel System Calls using Hardware Breakpoints ](http://fumalwareanalysis.blogspot.com/2012/01/malware-analysis-tutorial-11-starling.html).
  - [Malware Analysis Tutorial 12: Debug the Debugger - Fix Module Information and UDD File ](http://fumalwareanalysis.blogspot.com/2012/01/malware-analysis-tutorial-12-debug.html).
  - [Malware Analysis Tutorial 13: Tracing DLL Entry Point ](http://fumalwareanalysis.blogspot.com/2012/01/malware-tutorial-13-tracing-dll-entry.html).
  - [Malware Analysis Tutorial 14: Retrieve Self-Decoding Key ](http://fumalwareanalysis.blogspot.com/2012/01/malware-analysis-tutorial-14-retrieve.html).
  - [Malware Analysis Tutorial 15: Injecting Thread into a Running Process ](http://fumalwareanalysis.blogspot.com/2012/02/malware-analysis-tutorial-15-injecting.html).
  - [Malware Analysis Tutorial 16: Return Oriented Programming (Return to LIBC) Attack ](http://fumalwareanalysis.blogspot.com/2012/02/malware-analysis-tutorial-16-return.html).
  - [Malware Analysis Tutorial 17: Infection of System Modules (Part I: Randomly Pick a Driver).](http://fumalwareanalysis.blogspot.com/2012/02/malware-analysis-tutorial-17-infecting.html)
  - [Malware Analysis Tutorial 18: Infecting Driver Files (Part II: Simple Infection) ](http://fumalwareanalysis.blogspot.com/2012/02/malware-analysis-tutorial-18-infecting.html).  
  - [Malware Analysis Tutorial 19: Anatomy of Infected Driver ](http://fumalwareanalysis.blogspot.com/2012/03/malware-analysis-tutorial-19-anatomy-of.html)
  - [Malware Analysis Tutorial 20: Kernel Debugging - Intercepting Driver Loading ](http://fumalwareanalysis.blogspot.com/2012/03/malware-analysis-tutorial-20-kernel.html).
  - [Malware Analysis Tutorial 21: Hijacking Disk Driver ](http://fumalwareanalysis.blogspot.com/2012/03/malware-analysis-tutorial-21-hijack.html)
  - [Malware Analysis Tutorial 22: IRP Handler and Infected Disk Driver](http://fumalwareanalysis.blogspot.com/2012/03/malware-analysis-tutorial-22-irp.html)
  - [Malware Tutorial Analysis 23: Tracing Kernel Data Using Data Breakpoints ](http://fumalwareanalysis.blogspot.com/2012/03/malware-tutorial-analysis-23-tracing.html)
  - [Malware Analysis Tutorial 24: Tracing Malicious TDI Network Behaviors of Max++  ](http://fumalwareanalysis.blogspot.com/2012/04/malware-analysis-tutorial-24-tracing.html)
  - [Malware Analysis Tutorial 25: Deferred Procedure Call (DPC) and TCP Connection ](http://fumalwareanalysis.blogspot.com/2012/04/malware-analysis-tutorial-25-deferred.html)
  - [Malware Analysis Tutorial 26: Rootkit Configuration ](http://fumalwareanalysis.blogspot.com/2012/04/malware-analysis-tutorial-26-rootkit.html)
  - [Malware Analysis Tutorial 27: Stealthy Loading of Malicious Driver  ](http://fumalwareanalysis.blogspot.com/2012/05/malware-analysis-tutorial-27-stealthy.html)
  - [Malware Analysis Tutorial 28: Break Max++ Rootkit Hidden Drive Protection](http://fumalwareanalysis.blogspot.com/2012/05/malware-analysis-tutorial-28-break-max.html)
  - [Malware Analysis Tutorial 29: Stealthy Library Loading II (Using Self-Modifying APC) ](http://fumalwareanalysis.blogspot.com/2012/06/malware-analysis-tutorial-29-stealthy.html)
  - [Malware Analysis Tutorial 30: Self-Overwriting COM Loading for Remote Loading DLL](http://fumalwareanalysis.blogspot.com/2012/06/malware-analysis-tutorial-30-loading.html)
  - [Malware Analysis Tutorial 31: Exposing Hidden Control Flow ](http://fumalwareanalysis.blogspot.com/2012/08/malware-analysis-tutorial-31-exposing.html)
  - [Malware Analysis Tutorial 32: Exploration of Botnet Client ](http://fumalwareanalysis.blogspot.com/2012/08/malware-analysis-tutorial-32.html)
  - [Malware Analysis Tutorial 33: Evaluation of Automated Malware Analysis System I (Anubis) ](http://fumalwareanalysis.blogspot.com/2012/09/malware-analysis-tutorial-33-evaluation.html) 
  - [Malware Analysis Tutorial 34: Evaluation of Automated Malware Analysis Tools CWSandBox, PeID, and Other Unpacking Tools](http://fumalwareanalysis.blogspot.com/2012/10/malware-analysis-tutorial-34-evaluation.html)
- [RPISEC Malware Course](https://github.com/RPISEC/Malware)
- [Reverse Engineering Malware 101](https://securedorg.github.io/RE101/)
- Securitysift
  - [Windows Exploit Development – Part 1: The Basics](http://www.securitysift.com/windows-exploit-development-part-1-basics/)
  - [Windows Exploit Development – Part 2: Intro to Stack Based Overflows](http://www.securitysift.com/windows-exploit-development-part-2-intro-stack-overflow/)
  - [Windows Exploit Development – Part 3: Changing Offsets and Rebased Modules](http://www.securitysift.com/windows-exploit-development-part-3-changing-offsets-and-rebased-modules/)
  - [Windows Exploit Development – Part 4: Locating Shellcode With Jumps](http://www.securitysift.com/windows-exploit-development-part-4-locating-shellcode-jumps/)
  - [Windows Exploit Development – Part 5: Locating Shellcode With Egghunting](http://www.securitysift.com/windows-exploit-development-part-5-locating-shellcode-egghunting/)
  - [Windows Exploit Development – Part 6: SEH Exploits](http://www.securitysift.com/windows-exploit-development-part-6-seh-exploits/)
  - [Windows Exploit Development – Part 7: Unicode Buffer Overflows](http://www.securitysift.com/windows-exploit-development-part-7-unicode-buffer-overflows/)
- Whitehatters Academy
  - [Intro to Windows kernel exploitation 1/N: Kernel Debugging](https://www.whitehatters.academy/intro-to-kernel-exploitation-part-1/)
  - [Intro to Windows kernel exploitation 2/N: HackSys Extremely Vulnerable Driver](https://www.whitehatters.academy/intro-to-windows-kernel-exploitation-2-windows-drivers/)
  - [Intro to Windows kernel exploitation 3/N: My first Driver exploit](https://www.whitehatters.academy/intro-to-windows-kernel-exploitation-3-my-first-driver-exploit/)
  - [Intro to Windows kernel exploitation 3.5/N: A bit more of the HackSys Driver](https://www.whitehatters.academy/intro-to-windows-kernel-exploitation-more-of-the-hacksys-driver/)
  - [Backdoor 103: Fully Undetected](https://www.whitehatters.academy/backdoor-103-fully-undetected/)
  - [Backdoor 102](https://www.whitehatters.academy/backdoor-102/)
  - [Backdoor 101](https://www.whitehatters.academy/backdoor101-vysec/)
- TheSprawl
  - [corelan - integer overflows - exercise solution](http://thesprawl.org/research/corelan-integer-overflows-exercise-solution/)
  - [heap overflows for humans - 102 - exercise solution](http://thesprawl.org/research/heap-overflows-humans-102-exercise-solution/)
  - [exploit exercises - protostar - final levels](http://thesprawl.org/research/exploit-exercises-protostar-final/)
  - [exploit exercises - protostar - network levels](http://thesprawl.org/research/exploit-exercises-protostar-network/)
  - [exploit exercises - protostar - heap levels](http://thesprawl.org/research/exploit-exercises-protostar-heap/)
  - [exploit exercises - protostar - format string levels](http://thesprawl.org/research/exploit-exercises-protostar-format/)
  - [exploit exercises - protostar - stack levels](http://thesprawl.org/research/exploit-exercises-protostar-stack/)
  - [open security training - introduction to software exploits - uninitialized variable overflow](http://thesprawl.org/research/ost-introduction-software-exploits-uninit-overflow/)
  - [open security training - introduction to software exploits - off-by-one](http://thesprawl.org/research/ost-introduction-exploits-offbyone/)
  - [open security training - introduction to re - bomb lab secret phase](http://thesprawl.org/research/ost-introduction-re-bomb-secret-phase/)
  - [open security training - introductory x86 - buffer overflow mystery box](http://thesprawl.org/research/ost-introductory-x86-buffer-overflow-mystery-box/)
  - [corelan - tutorial 10 - exercise solution](http://thesprawl.org/research/corelan-tutorial-10-exercise-solution/)
  - [corelan - tutorial 9 - exercise solution](http://thesprawl.org/research/corelan-tutorial-9-exercise-solution/)
  - [corelan - tutorial 7 - exercise solution](http://thesprawl.org/research/corelan-tutorial-7-exercise-solution/)
  - [getting from seh to nseh](http://thesprawl.org/research/seh-to-nseh/)
  - [corelan - tutorial 3b - exercise solution](http://thesprawl.org/research/corelan-tutorial-3b-exercise-solution/)
- Expdev-Kiuhnm
  - [WinDbg](http://expdev-kiuhnm.rhcloud.com/2015/05/17/windbg/)
  - [Mona 2](http://expdev-kiuhnm.rhcloud.com/2015/05/19/mona-2/)
  - [Structure Exception Handling (SEH)](http://expdev-kiuhnm.rhcloud.com/2015/05/19/structured-exception-handling-seh/)
  - [Heap](http://expdev-kiuhnm.rhcloud.com/2015/05/20/heap/)
  - [Windows Basics](http://expdev-kiuhnm.rhcloud.com/2015/05/20/windows-basics/)
  - [Shellcode](http://expdev-kiuhnm.rhcloud.com/2015/05/22/shellcode/)
  - [Exploitme1 (ret eip overwrite)](http://expdev-kiuhnm.rhcloud.com/2015/05/26/exploitme1-ret-eip-overwrite/)
  - [Exploitme2 (Stack cookies & SEH)](http://expdev-kiuhnm.rhcloud.com/2015/05/26/exploitme2-stack-cookies-seh-2/)
  - [Exploitme3 (DEP)](http://expdev-kiuhnm.rhcloud.com/2015/05/27/exploitme3-dep/)
  - [Exploitme4 (ASLR)](http://expdev-kiuhnm.rhcloud.com/2015/05/28/exploitme4-aslr/)
  - [Exploitme5 (Heap Spraying & UAF)](http://expdev-kiuhnm.rhcloud.com/2015/05/29/exploitme5-heap-spraying-uaf/)
  - [EMET 5.2](http://expdev-kiuhnm.rhcloud.com/2015/05/29/emet-5-2-2/)
  - [Internet Explorer 10 - Reverse Engineering IE](http://expdev-kiuhnm.rhcloud.com/2015/05/31/ie10-reverse-engineering-ie/)
  - [Internet Explorer 10 - From one-byte-write to full process space read/write](http://expdev-kiuhnm.rhcloud.com/2015/05/31/ie-10-from-one-byte-write-to-full-process-space-readwrite/)
  - [Internet Explorer 10 - God Mode (1)](http://expdev-kiuhnm.rhcloud.com/2015/05/31/ie10-god-mode-1/)
  - [Internet Explorer 10 - God Mode (2)](http://expdev-kiuhnm.rhcloud.com/2015/06/01/ie10-god-mode-2/)
  - [Internet Explorer 10 - Use-After-Free bug](http://expdev-kiuhnm.rhcloud.com/2015/06/01/ie10-use-free-bug/)
  - [Internet Explorer 11 - Part 1](http://expdev-kiuhnm.rhcloud.com/2015/06/02/ie11-part-1/)
  - [Internet Explorer 11 - Part 2](http://expdev-kiuhnm.rhcloud.com/2015/06/02/ie11-part-2/)

##Papers
### ROP

- ROP萌芽期
  - 【待续】海枫的一些文章
- 基本ROP
  - [一步一步学ROP之linux_x86篇](http://cb.drops.wiki/drops/tips-6597.html)
  - [一步一步学ROP之linux_x86篇](http://cb.drops.wiki/drops/papers-7551.html)
  - [一步一步学ROP之gadgets和2free篇](http://cb.drops.wiki/drops/binary-10638.html)
  - [一步一步学ROP之Android ARM 32位篇](http://cb.drops.wiki/drops/papers-11390.html)
  - [Intro to ROP: ROP Emporium — Split](https://medium.com/@iseethieves/intro-to-rop-rop-emporium-split-9b2ec6d4db08)
  - [ROP Emporium](https://ropemporium.com/)
  - [ropasaurusrex: a primer on return-oriented programming2](https://blog.skullsecurity.org/2013/ropasaurusrex-a-primer-on-return-oriented-programming)
  - [ROP技术入门教程](http://bobao.360.cn/learning/detail/3569.html)
  - [二进制漏洞利用中的ROP技术研究与实例分析](https://xianzhi.aliyun.com/forum/read/840.html?fpage=2)
  - [现代栈溢出利用技术基础：ROP](http://bobao.360.cn/learning/detail/3694.html)
  - [通过ELF动态装载构造ROP链](http://wooyun.jozxing.cc/static/drops/binary-14360.html)
  - [Swing: 基础栈溢出复习 二 之 ROP](http://bestwing.me/2017/03/19/stack-overflow-two-ROP/)
  - [The Geometry of Innocent Flesh on the Bone: Return-into-libc without Function Calls](http://cseweb.ucsd.edu/~hovav/dist/geometry.pdf)
  - [Blind return-oriented programming](http://www.scs.stanford.edu/brop/bittau-brop.pdf)
  - [Sigreturn-oriented Programming](https://www.cs.vu.nl/~herbertb/papers/srop_sp14.pdf)
  - [Jump-Oriented Programming: A New Class of Code-Reuse Attack](http://ftp.ncsu.edu/pub/tech/2010/TR-2010-8.pdf)
  - [Out of control: Overcoming control-flow integrity](http://www.cs.stevens.edu/~gportoka/files/outofcontrol_oakland14.pdf)
  - [ROP is Still Dangerous: Breaking Modern Defenses](http://www.cs.berkeley.edu/~daw/papers/rop-usenix14.pdf)
  - [Loop-Oriented Programming(LOP): A New Code Reuse Attack to Bypass Modern Defenses](https://www.sec.in.tum.de/assets/staff/muntean/Loop-Oriented_Programming_A_New_Code_Reuse_Attack_to_Bypass_Modern0ADefenses.pdf) - by Bingchen Lan, Yan Li, Hao Sun, Chao Su, Yao Liu, Qingkai Zeng [2015]
  - [Systematic Analysis of Defenses Against Return-Oriented Programming](https://people.csail.mit.edu/nickolai/papers/skowyra-rop.pdf) -by R. Skowyra, K. Casteel, H. Okhravi, N. Zeldovich, and W. Streilein [2013]
  - [Return-oriented programming without returns](https://www.cs.uic.edu/~s/papers/noret_ccs2010/noret_ccs2010.pdf) -by S.Checkoway, L. Davi, A. Dmitrienko, A. Sadeghi, H. Shacham, and M. Winandy [2010]
  - [Jump-oriented programming: a new class of code-reuse attack](https://www.comp.nus.edu.sg/~liangzk/papers/asiaccs11.pdf) -by T. K. Bletsch, X. Jiang, V. W. Freeh, and Z. Liang [2011]
  - [Stitching the gadgets: on the ineffectiveness of coarse-grained control-flow integrity protection](https://www.usenix.org/system/files/conference/usenixsecurity14/sec14-paper-davi.pdf) - by L. Davi, A. Sadeghi, and D. Lehmann [2014]
  - [Size does matter: Why using gadget-chain length to prevent code-reuse attacks is hard](https://www.usenix.org/system/files/conference/usenixsecurity14/sec14-paper-goktas.pdf) - by E. Göktas, E.Athanasopoulos, M. Polychronakis, H. Bos, and G.Portokalidis [2014]
  - [Buffer overflow attacks bypassing DEP (NX/XD bits) – part 1](http://www.mastropaolo.com/2005/06/04/buffer-overflow-attacks-bypassing-dep-nxxd-bits-part-1/) - by Marco Mastropaolo [2005]
  - [Buffer overflow attacks bypassing DEP (NX/XD bits) – part 2](http://www.mastropaolo.com/2005/06/05/buffer-overflow-attacks-bypassing-dep-nxxd-bits-part-2-code-injection/) - by Marco Mastropaolo [2005]
  - [Practical Rop](http://trailofbits.files.wordpress.com/2010/04/practical-rop.pdf) - by Dino Dai Zovi [2010]
  - [Exploitation with WriteProcessMemory](https://packetstormsecurity.com/papers/general/Windows-DEP-WPM.txt) - by Spencer Pratt [2010]
  - [Exploitation techniques and mitigations on Windows](http://hick.org/~mmiller/presentations/misc/exploitation_techniques_and_mitigations_on_windows.pdf) - by skape
  - [A little return oriented exploitation on Windows x86 – Part 1](http://blog.harmonysecurity.com/2010/04/little-return-oriented-exploitation-on.html) - by Harmony Security and Stephen Fewer [2010]
  - [A little return oriented exploitation on Windows x86 – Part 2](http://blog.harmonysecurity.com/2010/04/little-return-oriented-exploitation-on_16.html) - by Harmony Security and Stephen Fewer [2010]
- BROP
  - [Blind Return Oriented Programming](http://www.scs.stanford.edu/brop/)
  - [muhe: Have fun with Blind ROP](http://o0xmuhe.me/2017/01/22/Have-fun-with-Blind-ROP/)
  - [Swing: 基础栈溢出复习 四 之 BROP](http://bestwing.me/2017/03/24/stack-overflow-four-BROP/)
- SROP
  - [Sigreturn Oriented Programming (SROP) Attack攻击原理](http://www.freebuf.com/articles/network/87447.html)
  - [Swing: 基础栈溢出复习 三 之 SROP](http://bestwing.me/2017/03/20/stack-overflow-three-SROP/)
- Return-to-dl-resolve
  - [如何在32位系统中使用ROP+Return-to-dl来绕过ASLR+DEP](http://www.freebuf.com/articles/system/149214.html)
  - [通过ELF动态装载构造ROP链 （ Return-to-dl-resolve）](http://www.evil0x.com/posts/19226.html)

### 栈相关

- 栈溢出
  - [手把手教你栈溢出从入门到放弃（上）](http://bobao.360.cn/learning/detail/3717.html)
  - [手把手教你栈溢出从入门到放弃（下）](http://bobao.360.cn/learning/detail/3718.html)
  - [Hcamael: PWN学习总结之基础栈溢出](http://0x48.pw/2016/11/03/0x26/)
  - [Hcamael: PWN学习总结之基础栈溢出2](http://0x48.pw/2016/11/21/0x27/)
  - [Swing: 基础栈溢出复习 之基础](http://bestwing.me/2017/03/18/stack-overflow-one/)
  - [ARM栈溢出攻击实践：从虚拟环境搭建到ROP利用](http://www.freebuf.com/articles/terminal/107276.html)
  - [64-bit Linux stack smashing tutorial: Part 1](https://blog.techorganic.com/2015/04/10/64-bit-linux-stack-smashing-tutorial-part-1/)
  - [64-bit Linux stack smashing tutorial: Part 2](https://blog.techorganic.com/2015/04/21/64-bit-linux-stack-smashing-tutorial-part-2/)
  - [64-bit Linux stack smashing tutorial: Part 3](https://blog.techorganic.com/2016/03/18/64-bit-linux-stack-smashing-tutorial-part-3/)
  - [Offset2lib: bypassing full ASLR on 64bit Linu](http://cybersecurity.upv.es/attacks/offset2lib/offset2lib.html)
  - [return2libc学习笔记](https://www.tuicool.com/articles/VVBz6va)
  - [Win32 Buffer Overflows (Location, Exploitation and Prevention)](http://www.phrack.com/issues.html?issue=55&id=15#article) - by Dark spyrit [1999]
  - [Writing Stack Based Overflows on Windows](http://www.packetstormsecurity.org/papers/win/) - by Nish Bhalla’s [2005]
  - [Stack Smashing as of Today](https://www.blackhat.com/presentations/bh-europe-09/Fritsch/Blackhat-Europe-2009-Fritsch-Bypassing-aslr-slides.pdf) - by Hagen Fritsch [2009]
  - [SMASHING C++ VPTRS](http://phrack.org/issues/56/8.html) - by rix [2000]
- Canary/GS绕过
  - [栈溢出之绕过CANARY保护](http://0x48.pw/2017/03/14/0x2d/)
  - [论canary的几种玩法](http://veritas501.space/2017/04/28/%E8%AE%BAcanary%E7%9A%84%E5%87%A0%E7%A7%8D%E7%8E%A9%E6%B3%95/)
  - [Liunx下关于绕过cancry保护总结](http://yunnigu.dropsec.xyz/2017/03/20/Liunx%E4%B8%8B%E5%85%B3%E4%BA%8E%E7%BB%95%E8%BF%87cancry%E4%BF%9D%E6%8A%A4%E6%80%BB%E7%BB%93/)

### 堆相关

- 堆理论基础
  - [PWN之堆内存管理](http://paper.seebug.org/255/)
  - [Linux堆内存管理深入分析（上）](http://www.freebuf.com/articles/system/104144.html)
  - [Linux堆内存管理深入分析（下）](http://www.freebuf.com/articles/security-management/105285.html)
  - [Windows Exploit开发系列教程——堆喷射（一）](http://bobao.360.cn/learning/detail/3548.html)
  - [Windows Exploit开发系列教程——堆喷射（二）](http://bobao.360.cn/learning/detail/3555.html)
  - [Libc堆管理机制及漏洞利用技术 (一）](http://www.freebuf.com/articles/system/91527.html)
  - [Notes About Heap Overflow Under Linux](https://blog.iret.xyz/article.aspx/linux_heapoverflow_enterance)
  - [如何理解堆和堆溢出漏洞的利用?](http://www.freebuf.com/vuls/98404.html)
  - [Have fun with glibc内存管理](http://o0xmuhe.me/2016/11/21/Have-fun-with-glibc%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)
  - [内存映射mmap](http://www.tuicool.com/articles/A7n2ueq)
  - [glibc malloc学习笔记之fastbin](http://0x48.pw/2017/07/25/0x35/)
  - [malloc.c源码阅读之__libc_free](http://0x48.pw/2017/08/07/0x37/)
  - [Malloc碎碎念](http://www.cnblogs.com/wangaohui/p/5190889.html)
  - [glibc内存分配与回收过程图解](http://blog.csdn.net/maokelong95/article/details/52006379)
  - [理解 glibc malloc](http://blog.csdn.net/maokelong95/article/details/51989081#allocated-chunk)
- 堆利用技术
  - [how2heap总结-上](http://bobao.360.cn/learning/detail/4386.html)
  - [how2heap总结-下](http://bobao.360.cn/learning/detail/4383.html)
  - [溢出科普：heap overflow&溢出保护和绕过](http://wooyun.jozxing.cc/static/drops/binary-14596.html)
  - [现代化的堆相关漏洞利用技巧](http://bobao.360.cn/learning/detail/3197.html)
  - [从一字节溢出到任意代码执行-Linux下堆漏洞利用](http://bobao.360.cn/learning/detail/3113.html)
  - [Heap overflow using unlink](https://sploitfun.wordpress.com/2015/02/26/heap-overflow-using-unlink/?spm=a313e.7916648.0.0.x4nzYZ)
  - [堆溢出的unlink利用方法](https://www.tuicool.com/articles/E3Ezu2u)
  - [Linux堆溢出漏洞利用之unlink](https://jaq.alibaba.com/community/art/show?spm=a313e.7916646.24000001.74.ZP8rXN&articleid=360)
  - [浅析Linux堆溢出之fastbin](http://www.freebuf.com/news/88660.html?utm_source=tuicool&utm_medium=referral)
  - [Linux堆溢出之Fastbin Attack实例详解](http://bobao.360.cn/learning/detail/3996.html)
  - [unsorted bin attack分析](http://bobao.360.cn/learning/detail/3296.html)
  - [Double Free浅析](http://www.vuln.cn/6172)
  - [Understanding the heap by breaking it](http://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf)
  - [An Introduction to Use After Free Vulnerabilities](https://www.purehacking.com/blog/lloyd-simon/an-introduction-to-use-after-free-vulnerabilities)
  - [Use After Free漏洞浅析](http://bobao.360.cn/learning/detail/3379.html?utm_source=tuicool&utm_medium=referral)
  - [Linux堆漏洞之Use after free实例](http://d0m021ng.github.io/2017/03/04/PWN/Linux%E5%A0%86%E6%BC%8F%E6%B4%9E%E4%B9%8BUse-after-free%E5%AE%9E%E4%BE%8B/)
  - [堆之House of Spirit](http://bobao.360.cn/learning/detail/3417.html)
  - [Dance In Heap（一）：浅析堆的申请释放及相应保护机制](http://www.freebuf.com/articles/system/151372.html)
  - [Dance In Heap（二）：一些堆利用的方法（上）](http://www.freebuf.com/articles/system/151407.html)
  - [Dance In Heap（三）：一些堆利用的方法（中）](http://www.freebuf.com/articles/system/151428.html)
  - [Dance In Heap（四）：一些堆利用的方法（下）](http://www.freebuf.com/articles/system/151435.html)
  - [Glibc Adventures：The Forgotten Chunks](https://info.contextis.com/acton/attachment/24535/f-02c8/1/-/-/-/-/Glibc%20Adventures%3A%20The%20forgotten%20chunks.pdf)
  - [Third Generation Exploitation smashing heap on 2k](http://www.blackhat.com/presentations/win-usa-02/halvarflake-winsec02.ppt) - by Halvar Flake [2002]
  - [Exploiting the MSRPC Heap Overflow Part 1](http://freeworld.thc.org/root/docs/exploit_writing/msrpcheap.pdf) - by Dave Aitel (MS03-026) [September 2003]
  - [Exploiting the MSRPC Heap Overflow Part 2](http://freeworld.thc.org/root/docs/exploit_writing/msrpcheap2.pdf) - by Dave Aitel (MS03-026) [September 2003]
  - [Windows heap overflow penetration in black hat](https://www.blackhat.com/presentations/win-usa-04/bh-win-04-litchfield/bh-win-04-litchfield.ppt) - by David Litchfield [2004]
  - [Glibc Adventures: The Forgotten Chunk](http://www.contextis.com/documents/120/Glibc_Adventures-The_Forgotten_Chunks.pdf) - by François Goichon [2015]
  - [Pseudomonarchia jemallocum](http://www.phrack.org/issues/68/10.html) - by argp & huku
  - [The House Of Lore: Reloaded](http://phrack.org/issues/67/8.html) - by blackngel [2010]
  - [Malloc Des-Maleficarum](http://phrack.org/issues/66/10.html) - by blackngel [2009]
  - [free() exploitation technique](http://phrack.org/issues/66/6.html) - by huku
  - [Understanding the heap by breaking it](https://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf) - by Justin N. Ferguson [2007]
  - [The use of set_head to defeat the wilderness](http://phrack.org/issues/64/9.html) - by g463
  - [The Malloc Maleficarum](http://seclists.org/bugtraq/2005/Oct/118) - by Phantasmal Phantasmagoria [2005]
  - [Exploiting The Wilderness](http://seclists.org/vuln-dev/2004/Feb/25) - by Phantasmal Phantasmagoria [2004]
  - [Advanced Doug lea's malloc exploits](http://phrack.org/issues/61/6.html) - by jp

### 格式化字符串

- 格式化字符串漏洞
  - [Exploiting Format String Vulnerabilities](https://crypto.stanford.edu/cs155old/cs155-spring08/papers/formatstring-1.2.pdf)
  - [二进制漏洞之——邪恶的printf](http://cb.drops.wiki/drops/binary-6259.html)
  - [漏洞挖掘基础之格式化字符串](http://cb.drops.wiki/drops/papers-9426.html)
  - [格式化字符串漏洞利用小结（一）](http://bobao.360.cn/learning/detail/3654.html)
  - [格式化字符串漏洞利用小结（二）](http://bobao.360.cn/learning/detail/3674.html)
  - [Linux下的格式化字符串漏洞利用姿势](http://www.cnblogs.com/Ox9A82/p/5429099.html)
  - [Linux系统下格式化字符串利用研究](http://0x48.pw/2017/03/13/0x2c/?utm_source=tuicool&utm_medium=referral)
  - [Advances in format string exploitation](http://phrack.org/issues/59/7.html)
  - [Exploiting Sudo format string vunerability](http://www.vnsecurity.net/research/2012/02/16/exploiting-sudo-format-string-vunerability.html)

### Misc

- FSP溢出
  - [Head First FILE Stream Pointer Overflow](http://wooyun.jozxing.cc/static/drops/binary-12740.html)
  - [abusing the FILE structure](https://outflux.net/blog/archives/2011/12/22/abusing-the-file-structure/)
  - [File Stream Pointer Overflows Paper.](http://repo.thehackademy.net/depot_ouah/fsp-overflows.txt)
  - [溢出利用FILE结构体](http://bobao.360.cn/learning/detail/3219.html)
- 整数溢出
  - [整数溢出漏洞](http://blog.csdn.net/wuxiaobingandbob/article/details/44618925)
- ARM汇编
  - [ARM 汇编基础速成1：ARM汇编以及汇编语言基础介绍](http://bobao.360.cn/learning/detail/4070.html)
  - [ARM 汇编基础速成2：ARM汇编中的数据类型](http://bobao.360.cn/learning/detail/4075.html)
  - [ARM 汇编基础速成3：ARM模式与THUMB模式](http://bobao.360.cn/learning/detail/4082.html)
  - [ARM 汇编基础速成4：ARM汇编内存访问相关指令](http://bobao.360.cn/learning/detail/4087.html)
  - [ARM 汇编基础速成5：连续存取](http://bobao.360.cn/learning/detail/4097.html)
  - [ARM 汇编基础速成6：条件执行与分支](http://bobao.360.cn/learning/detail/4104.html)
  - [ARM 汇编基础速成7：栈与函数](http://bobao.360.cn/learning/detail/4108.html)
- Lua
  - [Lua程序逆向之Luac文件格式分析](http://bobao.360.cn/learning/detail/4534.html)
- 进程注入
  - [10种常见的进程注入技术的总结](http://bobao.360.cn/learning/detail/4131.html)
  - [系统安全攻防战：DLL注入技术详解](http://www.freebuf.com/articles/system/143640.html)
- 符号执行
  - [关于符号执行](https://github.com/enzet/symbolic-execution)
  - [Playing with Dynamic symbolic execution](http://www.miasm.re/blog/2017/10/05/playing_with_dynamic_symbolic_execution.html)
- Windows memory protections
  - [Data Execution Prevention](http://support.microsoft.com/kb/875352)
  - [/GS (Buffer Security Check)](http://msdn.microsoft.com/en-us/library/Aa290051)
  - [/SAFESEH](http://msdn.microsoft.com/en-us/library/9a89h429(VS.80).aspx)
  - [ASLR](http://blogs.msdn.com/michael_howard/archive/2006/05/26/address-space-layout-randomization-in-windows-vista.aspx)
  - [SEHOP](http://blogs.technet.com/srd/archive/2009/02/02/preventing-the-exploitation-of-seh-overwrites-with-sehop.aspx)
- Bypassing filter and protections
  - [Third Generation Exploitation smashing heap on 2k](http://www.blackhat.com/presentations/win-usa-02/halvarflake-winsec02.ppt) - by Halvar Flake [2002]
  - [Creating Arbitrary Shellcode In Unicode Expanded Strings](http://www.net-security.org/dl/articles/unicodebo.pdf) - by Chris Anley
  - [Advanced windows exploitation](http://www.immunityinc.com/downloads/immunity_win32_exploitation.final2.ppt) - by Dave Aitel [2003]
  - [Defeating the Stack Based Buffer Overflow Prevention Mechanism of Microsoft Windows 2003 Server](http://www.ngssoftware.com/papers/defeating-w2k3-stack-protection.pdf) - by David Litchfield
  - [Reliable heap exploits and after that Windows Heap Exploitation (Win2KSP0 through WinXPSP2)](http://cybertech.net/~sh0ksh0k/projects/winheap/XPSP2%20Heap%20Exploitation.ppt) - by Matt Conover in cansecwest 2004
  - [Safely Searching Process Virtual Address Space](http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf) - by Matt Miller [2004]
  - [IE exploit and used a technology called Heap Spray](http://www.exploit-db.com/exploits/612)
  - [Bypassing hardware-enforced DEP](http://www.uninformed.org/?v=2&a=4&t=pdf) - by Skape (Matt Miller) and Skywing (Ken Johnson) [October 2005]
  - [Exploiting Freelist[0\] On XP Service Pack 2](http://www.orkspace.net/secdocs/Windows/Protection/Bypass/Exploiting%20Freelist%5B0%5D%20On%20XP%20Service%20Pack%202.pdf) - by Brett Moore [2005]
  - [Kernel-mode Payloads on Windows in uninformed](http://www.uninformed.org/?v=3&a=4&t=pdf)
  - [Exploiting 802.11 Wireless Driver Vulnerabilities on Windows](http://www.uninformed.org/?v=6&a=2&t=pdf)
  - [Exploiting Comon Flaws In Drivers](http://www.reversemode.com/index.php?option=com_content&task=view&id=38&Itemid=1)
  - [Heap Feng Shui in JavaScript](http://www.blackhat.com/presentations/bh-europe-07/Sotirov/Presentation/bh-eu-07-sotirov-apr19.pdf) by Alexander sotirov [2007]
  - [Understanding and bypassing Windows Heap Protection](http://kkamagui.springnote.com/pages/1350732/attachments/579350) - by Nicolas Waisman [2007]
  - [Heaps About Heaps](http://www.insomniasec.com/publications/Heaps_About_Heaps.ppt) - by Brett moore [2008]
  - [Bypassing browser memory protections in Windows Vista](http://taossa.com/archive/bh08sotirovdowd.pdf) - by Mark Dowd and Alex Sotirov [2008]
  - [Attacking the Vista Heap](http://www.ruxcon.org.au/files/2008/hawkes_ruxcon.pdf) - by ben hawkes [2008]
  - [Return oriented programming Exploitation without Code Injection](http://cseweb.ucsd.edu/~hovav/dist/blackhat08.pdf) - by Hovav Shacham (and others ) [2008]
  - [Token Kidnapping and a super reliable exploit for windows 2k3 and 2k8](http://www.argeniss.com/research/TokenKidnapping.pdf) - by Cesar Cerrudo [2008]
  - [Defeating DEP Immunity Way](http://www.immunityinc.com/downloads/DEPLIB.pdf) - by Pablo Sole [2008]
  - [Practical Windows XP2003 Heap Exploitation](http://www.blackhat.com/presentations/bh-usa-09/MCDONALD/BHUSA09-McDonald-WindowsHeap-PAPER.pdf) - by John McDonald and Chris Valasek [2009]
  - [Bypassing SEHOP](http://www.sysdream.com/articles/sehop_en.pdf) - by Stefan Le Berre Damien Cauquil [2009]
  - [Interpreter Exploitation : Pointer Inference and JIT Spraying](http://www.semantiscope.com/research/BHDC2010/BHDC-2010-Slides-v2.pdf) - by Dionysus Blazakis[2010]
  - [Write-up of Pwn2Own 2010](http://vreugdenhilresearch.nl/Pwn2Own-2010-Windows7-InternetExplorer8.pdf) - by Peter Vreugdenhil
  - [All in one 0day presented in rootedCON](http://wintercore.com/downloads/rootedcon_0day_english.pdf) - by Ruben Santamarta [2010]
  - [DEP/ASLR bypass using 3rd party](http://web.archive.org/web/20130820021520/http://abysssec.com/files/The_Arashi.pdf) - by Shahin Ramezany [2013]
  - [Bypassing EMET 5.0](http://blog.sec-consult.com/2014/10/microsoft-emet-armor-against-zero-days.html) - by René Freingruber [2014]
- Typical windows exploits
  - [Real-world HW-DEP bypass Exploit](http://www.exploit-db.com/exploits/3652) - by Devcode
  - [Bypassing DEP by returning into HeapCreate](http://www.metasploit.com/redmine/projects/framework/repository/revisions/7246/entry/modules/exploits/windows/brightstor/mediasrv_sunrpc.rb) - by Toto
  - [First public ASLR bypass exploit by using partial overwrite ](http://www.metasploit.com/redmine/projects/framework/repository/entry/modules/exploits/windows/email/ani_loadimage_chunksize.rb)- by Skape
  - [Heap spray and bypassing DEP](http://skypher.com/SkyLined/download/www.edup.tudelft.nl/~bjwever/exploits/InternetExploiter2.zip) - by Skylined
  - [First public exploit that used ROP for bypassing DEP in adobe lib TIFF vulnerability](http://www.metasploit.com/redmine/projects/framework/repository/revisions/8833/raw/modules/exploits/windows/fileformat/adobe_libtiff.rb)
  - [Exploit codes of bypassing browsers memory protections](http://phreedom.org/research/bypassing-browser-memory-protections/bypassing-browser-memory-protections-code.zip)
  - [PoC’s on Tokken TokenKidnapping . PoC for 2k3 -part 1](http://www.argeniss.com/research/Churrasco.zip) - by Cesar Cerrudo
  - [PoC’s on Tokken TokenKidnapping . PoC for 2k8 -part 2](http://www.argeniss.com/research/Churrasco2.zip) - by Cesar Cerrudo
  - [An exploit works from win 3.1 to win 7](http://lock.cmpxchg8b.com/c0af0967d904cef2ad4db766a00bc6af/KiTrap0D.zip) - by Tavis Ormandy KiTra0d
  - [Old ms08-067 metasploit module multi-target and DEP bypass](http://metasploit.com/svn/framework3/trunk/modules/exploits/windows/smb/ms08_067_netapi.rb)
  - [PHP 6.0 Dev str_transliterate() Buffer overflow – NX + ASLR Bypass](http://www.exploit-db.com/exploits/12189)
  - [SMBv2 Exploit](http://www.metasploit.com/redmine/projects/framework/repository/revisions/8916/raw/modules/exploits/windows/smb/ms09_050_smb2_negotiate_func_index.rb) - by Stephen Fewer
  - [Microsoft IIS 7.5 remote heap buffer overflow](http://www.phrack.org/issues/68/12.html) - by redpantz
  - [Browser Exploitation Case Study for Internet Explorer 11](https://labs.bluefrostsecurity.de/files/Look_Mom_I_Dont_Use_Shellcode-WP.pdf) - by Moritz Jodeit [2016]

### 内核

- Windows
  - [Some-Kernel-Fuzzing-Paper](https://github.com/k0keoyo/Some-Kernel-Fuzzing-Paper)
  - [Introduction to Windows Kernel Driver Exploitation (Pt. 1) - Environment Setup](https://chybeta.github.io/2017/08/19/Software-Security-Learning/Introduction%20to%20Windows%20Kernel%20Driver%20Exploitation%20(Pt.%201) - Environment Setup)
  - [Introduction to Windows Kernel Driver Exploitation (Pt. 2) - Stack Buffer Overflow to System Shell](https://glennmcgui.re/introduction-to-windows-kernel-driver-exploitation-pt-2/)
  - [HackSysExtremeVulnerableDriver](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver)
  - [Starting with Windows Kernel Exploitation – part 1 – setting up the lab](https://hshrzd.wordpress.com/2017/05/28/starting-with-windows-kernel-exploitation-part-1-setting-up-the-lab/)
  - [Starting with Windows Kernel Exploitation – part 2 – getting familiar with HackSys Extreme Vulnerable Driver](https://hshrzd.wordpress.com/2017/06/05/starting-with-windows-kernel-exploitation-part-2/)
  - [利用WinDbg本地内核调试器攻陷 Windows 内核](http://bobao.360.cn/learning/detail/4477.html)
  - [Windows内核利用之旅：熟悉HEVD（附视频演示）](http://bobao.360.cn/learning/detail/4002.html)
  - [Windows 内核攻击：栈溢出](http://bobao.360.cn/learning/detail/3718.html)
  - Kernel based Windows overflows
    - [How to attack kernel based vulns on windows was done](http://www.derkeiler.com/Mailing-Lists/Full-Disclosure/2003-08/0101.html) - by a Polish group called “sec-labs” [2003]
    - [Sec-lab old whitepaper](http://www.artofhacking.com/tucops/hack/windows/live/aoh_win32dcv.htm)
    - [Sec-lab old exploit](http://www.securityfocus.com/bid/8329/info)
    - [Windows Local Kernel Exploitation (based on sec-lab research)](http://www.packetstormsecurity.org/hitb04/hitb04-sk-chong.pdf) - by S.K Chong [2004]
    - [How to exploit Windows kernel memory pool](http://packetstormsecurity.nl/Xcon2005/Xcon2005_SoBeIt.pdf) - by SoBeIt [2005]
    - [Exploiting remote kernel overflows in windows](http://research.eeye.com/html/papers/download/StepIntoTheRing.pdf) - by Eeye Security
    - [Kernel-mode Payloads on Windows in uninformed](http://www.uninformed.org/?v=3&a=4&t=pdf) - by Matt Miller
    - [Exploiting 802.11 Wireless Driver Vulnerabilities on Windows](http://www.uninformed.org/?v=6&a=2&t=pdf)
    - [BH US 2007 Attacking the Windows Kernel](http://www.blackhat.com/presentations/bh-usa-07/Lindsay/Whitepaper/bh-usa-07-lindsay-WP.pdf)
    - [Remote and Local Exploitation of Network Drivers](http://www.blackhat.com/presentations/bh-usa-07/Bulygin/Presentation/bh-usa-07-bulygin.pdf)
    - [Exploiting Comon Flaws In Drivers](http://www.reversemode.com/index.php?option=com_content&task=view&id=38&Itemid=1)
    - [I2OMGMT Driver Impersonation Attack](http://www.immunityinc.com/downloads/DriverImpersonationAttack_i2omgmt.pdf)
    - [Real World Kernel Pool Exploitation](http://sebug.net/paper/Meeting-Documents/syscanhk/KernelPool.pdf)
    - [Exploit for windows 2k3 and 2k8](http://www.argeniss.com/research/TokenKidnapping.pdf)
    - [Alyzing local privilege escalations in win32k](http://www.uninformed.org/?v=10&a=2&t=pdf)
    - [Intro to Windows Kernel Security Development](http://www.dontstuffbeansupyournose.com/trac/browser/projects/ucon09/Intro_NT_kernel_security_stuff.pdf)
    - [There’s a party at ring0 and you’re invited](http://www.cr0.org/paper/to-jt-party-at-ring0.pdf)
    - [Windows kernel vulnerability exploitation](http://vexillium.org/dl.php?call_gate_exploitation.pdf)
    - [A New CVE-2015-0057 Exploit Technology](https://www.blackhat.com/docs/asia-16/materials/asia-16-Wang-A-New-CVE-2015-0057-Exploit-Technology-wp.pdf) - by Yu Wang [2016]
    - [Exploiting CVE-2014-4113 on Windows 8.1](https://labs.bluefrostsecurity.de/publications/2016/01/07/exploiting-cve-2014-4113-on-windows-8.1/) - by Moritz Jodeit [2016]
    - [Easy local Windows Kernel exploitation](http://media.blackhat.com/bh-us-12/Briefings/Cerrudo/BH_US_12_Cerrudo_Windows_Kernel_WP.pdf) - by Cesar Cerrudo [2012]
    - [Windows Kernel Exploitation ](http://www.hacking-training.com/download/WKE.pdf)- by Simone Cardona 2016
    - [Exploiting MS16-098 RGNOBJ Integer Overflow on Windows 8.1 x64 bit by abusing GDI objects](https://sensepost.com/blog/2017/exploiting-ms16-098-rgnobj-integer-overflow-on-windows-8.1-x64-bit-by-abusing-gdi-objects/) - by Saif Sherei 2017
    - [Windows Kernel Exploitation : This Time Font hunt you down in 4 bytes](http://www.slideshare.net/PeterHlavaty/windows-kernel-exploitation-this-time-font-hunt-you-down-in-4-bytes) - by keen team [2015]
    - [Abusing GDI for ring0 exploit primitives](https://www.coresecurity.com/system/files/publications/2016/10/Abusing-GDI-Reloaded-ekoparty-2016_0.pdf) - [2016]
  - Windows Kernel Memory Corruption
    - [Remote Windows Kernel Exploitation](https://cansecwest.com/core05/windowsremotekernel.pdf) - by Barnaby Jack [2005]
    - [windows kernel-mode payload fundamentals](http://uninformed.org/index.cgi?v=3&a=4&t=sumry) - by Skape [2006]
    - [exploiting 802.11 wireless driver vulnerabilities on windows](http://www.uninformed.org/?v=6&a=2&t=sumry) - by Johnny Cache, H D Moore, skape [2007]
    - [Kernel Pool Exploitation on Windows 7](https://media.blackhat.com/bh-dc-11/Mandt/BlackHat_DC_2011_Mandt_kernelpool-wp.pdf) - by Tarjei Mandt [2011]
    - [Windows Kernel-mode GS Cookies and 1 bit of entropy](https://github.com/enddo/awesome-windows-exploitation/blob/master/vexillium.org/dl.php?/Windows_Kernel-mode_GS_Cookies_subverted.pdf) - [2011]
    - [Subtle information disclosure in WIN32K.SYS syscall return values](http://j00ru.vexillium.org/?p=762) - [2011]
    - [nt!NtMapUserPhysicalPages and Kernel Stack-Spraying Techniques](http://j00ru.vexillium.org/?p=769) - [2011]
    - [SMEP: What is it, and how to beat it on Windows](http://j00ru.vexillium.org/?p=783) - [2011]
    - [Kernel Attacks through User-Mode Callbacks](http://www.mista.nu/research/mandt-win32k-paper.pdf) - by Tarjei Mandt [2011]
    - [Windows Security Hardening Through Kernel Address Protection](http://j00ru.vexillium.org/blog/04_12_11/Windows_Kernel_Address_Protection.pdf) - by Mateusz "j00ru" Jurczyk [2011]
    - [Reversing Windows8: Interesting Features of Kernel Security](http://hitcon.org/2012/download/0720A5_360.MJ0011_Reversing%20Windows8-Interesting%20Features%20of%20Kernel%20Security.pdf) - by MJ0011 [2012]
    - [Smashing The Atom: Extraordinary String Based Attacks](https://github.com/enddo/awesome-windows-exploitation/blob/master/mista.nu/research/smashing_the_atom.pdf) - by Tarjei Mandt [2012]
    - [Easy local Windows Kernel exploitation](http://media.blackhat.com/bh-us-12/Briefings/Cerrudo/BH_US_12_Cerrudo_Windows_Kernel_WP.pdf) - by Cesar Cerrudo [2012]
    - [Using a Patched Vulnerability to Bypass Windows 8 x64 Driver Signature Enforcement](https://github.com/enddo/awesome-windows-exploitation/blob/master/www.powerofcommunity.net/poc2012/mj0011.pdf) - by MJ0011 [2012]
    - [MWR Labs Pwn2Own 2013 Write-up - Kernel Exploit](https://labs.mwrinfosecurity.com/blog/mwr-labs-pwn2own-2013-write-up-kernel-exploit/) - [2013]
    - [KASLR Bypass Mitigations in Windows 8.1](https://github.com/enddo/awesome-windows-exploitation/blob/master/www.alex-ionescu.com/?p=82) - [2013]
    - [First Dip Into the Kernel Pool: MS10-058](http://doar-e.github.io/blog/2014/03/11/first-dip-into-the-kernel-pool-ms10-058/) - by Jeremy [2014]
    - [Windows 8 Kernel Memory Protections Bypass](https://labs.mwrinfosecurity.com/blog/windows-8-kernel-memory-protections-bypass/) - [2014]
    - [An Analysis of A Windows Kernel-Mode Vulnerability (CVE-2014-4113)](http://blog.trendmicro.com/trendlabs-security-intelligence/an-analysis-of-a-windows-kernel-mode-vulnerability-cve-2014-4113/) - by Weimin Wu [2014]
    - [Sheep Year Kernel Heap Fengshui: Spraying in the Big Kids’ Pool](http://www.alex-ionescu.com/?p=231) - [2014]
    - [Exploiting the win32k!xxxEnableWndSBArrows use-after-free (CVE 2015-0057) bug on both 32-bit and 64-bit](https://www.nccgroup.trust/globalassets/newsroom/uk/blog/documents/2015/07/exploiting-cve-2015.pdf) - by Aaron Adams [2015]
    - [Exploiting MS15-061 Microsoft Windows Kernel Use-After-Free (win32k!xxxSetClassLong)](https://www.nccgroup.trust/globalassets/our-research/uk/whitepapers/2015/08/2015-08-27_-_ncc_group_-_exploiting_ms15_061_uaf_-_release.pdf) - by Dominic Wang [2015]
    - [Exploiting CVE-2015-2426, and How I Ported it to a Recent Windows 8.1 64-bit](https://www.nccgroup.trust/globalassets/our-research/uk/whitepapers/2015/09/2015-08-28_-_ncc_group_-_exploiting_cve_2015_2426_-_release.pdf) - by Cedric Halbronn [2015]
    - [Abusing GDI for ring0 exploit primitives](https://www.coresecurity.com/blog/abusing-gdi-for-ring0-exploit-primitives) - by Diego Juarez [2015]
    - [Duqu 2.0 Win32k exploit analysis](https://www.virusbulletin.com/uploads/pdf/conference_slides/2015/OhFlorio-VB2015.pdf) - [2015]
- Linux
  - [Linux 内核漏洞利用教程（一）：环境配置](http://bobao.360.cn/learning/detail/3700.html)
  - [Linux 内核漏洞利用教程（二）：两个Demo](http://bobao.360.cn/learning/detail/3702.html)
  - [Linux 内核漏洞利用教程（三）：实践 CSAW CTF 题目](http://bobao.360.cn/learning/detail/3706.html)
  - [Linux内核ROP姿势详解(一)](http://www.freebuf.com/articles/system/94198.html)
  - [Linux内核ROP姿势详解（二）](http://www.freebuf.com/articles/system/135402.html)

### 虚拟机逃逸

- [Phrack: VM escape - QEMU Case Study](https://www.exploit-db.com/papers/42883/)
- [虚拟机逃逸——QEMU的案例分析（一）](http://bbs.pediy.com/thread-217997.htm)
- [虚拟机逃逸——QEMU的案例分析（二）](http://bbs.pediy.com/thread-217999.htm)
- [虚拟机逃逸——QEMU的案例分析（三）](http://bbs.pediy.com/thread-218045.htm)

### 漏洞挖掘

- [看我如何对Apache进行模糊测试并挖到了一个价值1500刀的漏洞](http://bobao.360.cn/learning/detail/4213.html)

### CTF

- pwn
  - 入门
    - [跟我入坑PWN第一章](http://bobao.360.cn/learning/detail/3300.html)
    - [跟我入坑PWN第二章](http://bobao.360.cn/learning/detail/3339.html)
  - 技巧
    - [借助DynELF实现无libc的漏洞利用小结](http://bobao.360.cn/learning/detail/3298.html?utm_source=tuicool&utm_medium=referral)
    - [what DynELF does basically](http://o0xmuhe.me/2016/12/24/what-DynELF-does-basically/)
    - [Finding Function’s Load Address](http://uaf.io/exploitation/misc/2016/04/02/Finding-Functions.html)
  - 总结
    - [pwn2exploit](https://github.com/jmpews/pwn2exploit)
    - 【待续】muhe师傅的钻石矿
    - [CTF总结](https://github.com/stfpeak/CTF)
    - [pwn tips](http://skysider.com/?p=223)
    - [CTF-pwn-tips](https://github.com/Naetw/CTF-pwn-tips)
    - [pwn 学习总结](http://www.angelwhu.com/blog/?p=460)
    - [CTF中做Linux下漏洞利用的一些心得](http://www.cnblogs.com/Ox9A82/p/5559167.html)
    - [linux常见漏洞利用技术实践](http://drops.xmd5.com/static/drops/binary-6521.html)
  - Writeup
    - [堆溢出学习之0CTF 2017 Babyheap](http://0x48.pw/2017/08/01/0x36/)
    - [一道有趣的CTF PWN题](http://bobao.360.cn/learning/detail/3189.html)
    - [Exploit-Exercises Nebula全攻略](https://github.com/1u4nx/Exploit-Exercises-Nebula)
    - [三个白帽之从pwn me调试到Linux攻防学习](http://wooyun.jozxing.cc/static/drops/binary-16700.html)
- reverse

## Practice

### Reverse

- [Crackmes.de](http://www.crackmes.de/)
- [OSX Crackmes](https://reverse.put.as/crackmes/)
- [ESET Challenges](http://www.joineset.com/jobs-analyst.html)
- [Flare-on Challenges](http://flare-on.com/)
- [Github CTF Archives](http://github.com/ctfs/)
- [Reverse Engineering Challenges](http://challenges.re/)
- [xorpd Advanced Assembly Exercises](http://www.xorpd.net/pages/xchg_rax/snip_00.html)
- [Virusshare.com](http://virusshare.com/)
- [Contagio](http://contagiodump.blogspot.com/)
- [Malware-Traffic-Analysis](https://malware-traffic-analysis.com/)
- [Malshare](http://malshare.com/)
- [Malware Blacklist](http://www.malwareblacklist.com/showMDL.php)
- [malwr.com](https://malwr.com/)
- [vxvault](http://vxvault.net/)