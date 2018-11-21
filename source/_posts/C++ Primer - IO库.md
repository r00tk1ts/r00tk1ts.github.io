---
title: C++ Primer - IO库
date: 2018-11-21 22:51:11
categories: programming-language
tags:
	- cpp
	- cpp-primer



---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第八章“IO库”时所做的笔记。

<!--more-->

# C++ Primer - IO库

C++不直接处理输入输出，而是通过一族定义在标准库中的类型来处理IO。这些类型支持从设备读取数据、向设备写入数据的IO操作，设备可以是文件、控制台窗口等。还有一些类型允许内存IO，从string读取数据，向string写入数据。

IO库定义了读写内置类型值得操作。此外，一些类，如string，通常也会定义类似的IO操作，来读写自己的对象。

初学C++时，为了控制台的交互，往往会接触到IO的内容，现在可以对以下的设施进行说明了：

- istream类型，提供输入操作
- ostream类型，提供输出操作
- cin，istream对象，从标准输入读取数据
- cout，ostream对象，向标准输出写入数据
- cerr，ostream对象，用于输出程序错误信息，写入到标准错误
- `>>`运算符，从一个istream对象读取输入数据
- `<<`运算符，向一个ostream对象写入输出数据
- getline函数，从给定的istream读取一行数据，存入一个给定的string对象中。

## IO类

| 头文件   | 类型                                                         |
| -------- | ------------------------------------------------------------ |
| iostream | Istream, wistream从流读取数据<br />ostream,wostream向流写入数据<br />iostream,wiostream读写流<br /> |
| fstream  | ifstream,wifstream从文件读取数据<br />ofstream, wofstream向文件写入数据<br />fstream,wfstream读写文件 |
| sstream  | Istringstream, wistringstream从string读取数据<br />ostringstream,wostringstream向string写入数据<br />stringstream,wstringstream读写string |

w是宽字符版本，wcin,wcout,wcerr对应cin,cout,cerr的宽字符版本对象。

### IO对象无拷贝或赋值

不能拷贝和对IO对象赋值：

```cpp
ofstream out1, out2;
out1 = out2;	//错误：不能对流对象赋值
ofstream print(ofstream);	//错误：不能初始化ofstream参数
out2 = print(out2);	//错误：不能拷贝流对象
```

所以形参和返回类型都不能设置成流类型，IO操作的函数往往以引用的方式传递和返回流。读写一个IO对象会改变其状态，因此传递和返回的引用不能是const的。

### 条件状态

| -                 | -                                                            |
| ----------------- | ------------------------------------------------------------ |
| strm::iostate     | strm是一种IO类型。iostate是一种机器相关的类型，提供了表达条件状态的完整功能。 |
| strm::badbit      | strm::badbit用来指出流已崩溃                                 |
| strm::failbit     | strm::failbit用来指出一个IO操作失败了                        |
| strm::eofbit      | strm::eofbit用来指出流到达了文件结束                         |
| strm::goodbit     | strm::goodbit用来指出流未处于错误状态。此值保证为零          |
| s.eof()           | 若流s的eofbit置位，则返回true                                |
| s.fail()          | 若流s的failbit或badbit置位，则返回true                       |
| s.bad()           | 若流s的badbit置位，则返回true                                |
| s.good()          | 若流s处于有效状态，则返回true                                |
| s.clear()         | 将流s中所有条件状态位复位，将流的状态设置为有效。返回void    |
| s.clear(flags)    | 根据给定的flags标志位，将流s中对应条件状态位复位。flags的类型为strm::iostate。返回void |
| s.setstate(flags) | 根据给定的flags标志位，将流s中对应条件状态位置位。flags的类型为strm::iostate，返回void |
| s.rdstate()       | 返回流s的当前条件状态，返回值类型为strm::iostate             |

一个流一旦发生错误，后续的IO操作都会失败。只有当流处于无错状态时，才可以进行读写。

badbit无法恢复，failbit可以修正。如果badbit，failbit和eofbit的任意一个被置位，则检测流状态的条件会失败。它们有对应的函数来查询。

### 输出缓冲

每个输出流都管理一个缓冲区，保存读写数据。组合多个输出操作成单一的设备写操作可以提升性能。

缓冲刷新的契机：

- 程序正常结束
- 缓冲区满，会刷新缓冲
- 使用操纵符如endl和flush，显式刷新缓冲
- 每个输出操作之后，可以用操纵符unitbuf设置流的内部状态，清空缓冲区。默认清空下，对cerr是设置unitbuf的，因此写到cerr的内容都是立即刷新的。
- 一个输出流可能被关联到另一个流。此时，读写被关联的流时，关联到的流的缓冲区会被刷新。例如默认清空下，cin和cerr都关联到cout。因此，读cin或写cerr都会导致cout的缓冲区被刷新。

```cpp
cout << "hi!" << endl;	//输出hi和一个换行，刷新缓冲区
cout << "hi!" << flush;	//输出hi，刷新缓冲区

cout << unitbuf;	//所有输出操作后都会立即刷新缓冲区
cout << nounitbuf;	//回到正常的缓冲方式
```

程序崩溃时，输出缓冲区不会被刷新。

## 文件输入输出

实际上ifstream和ofstream继承于istream和ostream，所以操作和iostream差不多，它们还额外支持一些新的操作，用来管理与流关联的文件。

| -                       | -                                                            |
| ----------------------- | ------------------------------------------------------------ |
| fstream fstrm;          | 创建一个未绑定的文件流。fstream是头文件fstream中定义的一个类型 |
| fstream fstrm(s);       | 创建一个fstream，并打开名为s的文件。s可以是string类型，或者是一个指向C风格字符串的指针。这些构造函数都是explicit的。默认的文件模式mode依赖于fstream类型。 |
| fstream fstrm(s, mode); | 与前一个构造函数类似，但按指定mode打开文件                   |
| fstrm.open(s)           | 打开名为s的文件，并将文件与fstrm绑定。s可以是一个string或C风格字符串指针。默认文件mode依赖于fstream类型。返回void |
| fstrm.close()           | 关闭与fstrm绑定的文件。返回void                              |
| fstrm.is_open()         | 返回一个bool值，指出与fstrm关联的文件是否成功打开并且尚未关闭 |

读写一个文件，需要自己定义文件流对象，建立关联。

文件模式：

- in： 读方式打开
- out：写方式打开
- app：每次写操作前均定位到文件末尾
- ate：打开文件后立即定位到文件末尾
- trunc：截断文件
- binary：二进制方式进行IO

各种模式的指定都有一些条件，比如对ifstream还是ofstream的亲和性，以及是否互相依赖、互斥等等。

## string流

istringstream和ostringstream专门负责对string的读写。也是istream和ostream的继承类。所以，除了雷同的常规操作以外，它们也有自己独有的操作：

| -                | -                                                            |
| ---------------- | ------------------------------------------------------------ |
| sstream strm;    | strm是一个未绑定的stringstream对象。sstream是头文件sstream中定义的一个类型 |
| sstream strm(s); | strm是一个sstream对象，保存string s的一个拷贝。次构造函数是explicit的。 |
| strm.str()       | 返回strm所保存的string的拷贝                                 |
| strm.str(s)      | 将string s拷贝到strm中。返回void                             |

istringstream和ostringstream很少见，其实整个标准IO库都历来饱受诟病，完全不好用，折腾了一大堆，不如C标准库本身提供的那一套机制方便。