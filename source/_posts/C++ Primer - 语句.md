---
title: C++ Primer -语句
date: 2018-11-20 19:25:11
categories: programming-language
tags:
	- cpp
	- cpp-primer

---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第五章“语句”时所做的笔记。

<!--more-->

# C++ Primer - 语句

## 简单语句

空语句是最简单的语句，只包含一个单独的分号。

```cpp
while(cin >> s && s != sought)
  	;
```

## 语句作用域

if、switch、while和for中可以定义变量，这些变量仅在语句内部可见，还是那句话，大括号定律。

## 条件语句

### if

单if形式：

```cpp
if(condition)
  	statement
```

if-else形式：

```cpp
if(condition)
  	statement
else
	statement2
```

其中*condition*是判断条件，可以是一个表达式或者初始化了的变量声明。*condition*必须用圆括号括起来。

- 如果*condition*为真，则执行*statement*。执行完成后，程序继续执行if语句后面的其他语句。
- 如果*condition*为假，则跳过*statement*。对于简单if语句来说，程序直接执行if语句后面的其他语句；对于if-else语句来说，程序先执行*statement2*，再执行if语句后面的其他语句。

if可以嵌套，else与离它最近的尚未匹配的if相匹配。

悬垂还是嵌套根据自己编码风格选择。

### switch

形式：

```cpp
switch(number)
{
case 1: statement 1;
    	break;
case 2: statement 2;
    	break;
default: statement 3;
    	break;
}
```

switch语句先对括号里的表达式求值，值转换成整数类型后再与每个`case`标签的值进行比较。如果表达式的值和某个case标签匹配，程序从该标签之后的第一条语句开始执行，直到到达switch的结尾或者遇到break语句为止。case标签必须是整型常量表达式。

如果没有break，case会向下穿透。

一般不要省略case后面的break，如果没有写break也最好加上注释说明程序的逻辑，同样最后一个标签的break虽然可以省略，但是还是建议不要省略，起码如果还要新增一个case 的话，就不用额外写break了。也最好不要省略switch中的default，这样是为了告诉阅读代码的人，已经考虑到了default的情况。

## 迭代

迭代语句通常称为循环，它重复执行操作直到满足某个条件才停止。while和for语句在执行循环体之前检查条件，do-while语句先执行循环体再检查条件。

### while

形式：

```cpp
while(condition)
	statement
```

只要*condition*的求值结果为真，就一直执行*statement*（通常是一个块）。*condition*不能为空，如果*condition*第一次求值就是false，*statement*一次都不会执行。

定义在while条件部分或者循环体内的变量每次迭代都经历从创建到销毁的过程。

### for

C++的for有两种形式，第一种继承了C的风格

```cpp
for(initializer; condition; expression)
  	statement
```

相比较while来说，for更适合条件、迭代次数确定性等较为严格的情景。

其他的与while如出一辙。

另一种形式则是范围for，比较简明，由C++11引入：

```cpp
for(declaration : expression)
  	statement
```

其中*expression*表示一个序列，拥有能返回迭代器的begin和end成员。*declaration*定义一个变量，序列中的每个元素都应该能转换成该变量的类型（可以使用auto）。如果需要对序列中的元素执行写操作，循环变量必须声明成引用类型（因为auto会忽略引用类型）。

### do-while

while的一个变种，也是继承于C：

```cpp
do
  	statement
while(condition);
```

计算condition值之前会先执行一次statement，因此和while的差别就在于无论如何都会至少执行一次statement，在很多情况下特别有用，可以简化逻辑。

## 跳转语句

跳转语句中断当前的执行过程。

### break

break可以出现在迭代语句或switch语句的内部，它会终止离它最近的迭代、switch结构，从这些语句之后的第一条语句开始执行。简单来说，就是跳出最内层的嵌套结构。

### continue

和break很像，但它只用于迭代结构，它会终止本次的迭代立即开始下一次迭代。

### goto

无条件跳转到同一函数内的另一条语句，往往会有一个tag来辅助。

```cpp
// ...
if(condition)
	goto end;
...
end:
	...
```

程序设计的课程往往对goto讳莫如深，甚至禁止初学者使用，实际上很多情境下，goto可以让逻辑变得简洁，但也容易挖坑。

## try块和异常处理

异常处理机制由异常检测和异常处理两部分协作来支持。C++的异常处理包括：

- throw表达式，异常检测部分使用throw来表示它遇到了无法处理的问题。我们说throw引发了异常。
- try语句块，异常处理部分使用try块处理异常。try以关键字try开始，并以一个或多个catch子句处理。因为catch子句“处理”异常，所以它们也被称作异常处理代码。
- 一套异常类，用于在throw表达式和相关的catch子句之间传递异常的具体信息。

### throw

throw表达式包含关键字throw和紧随其后的一个表达式，其中表达式的类型就是抛出的异常类型。

### try

形式：

```cpp
try{
    program-statements
}catch(exception-declaration){
    handler-statements
}catch(exception-declaration){
    handler-statements
} // ...
```

try语句块中的*program-statements*组成程序的正常逻辑，其内部声明的变量在块外无法访问，即使在catch子句中也不行。catch子句包含关键字catch、括号内一个对象的声明（异常声明，exception declaration）和一个块。当选中了某个catch子句处理异常后，执行与之对应的块。catch一旦完成，程序会跳过剩余的所有catch子句，继续执行后面的语句。

如果最终没能找到与异常相匹配的catch子句，程序会执行名为`terminate`的标准库函数。该函数的行为与系统有关，一般情况下，执行该函数将导致程序非正常退出。类似的，如果一段程序没有try语句块且发生了异常，系统也会调用terminate函数并终止当前程序的执行。

> 这个实际上和系统平台如何实现异常机制有关。

### 标准异常

C++标准库定义了一组类，也就是默认的异常类，可以直接拿来用，它们分布在4个头文件：

- exception头文件定义了最通用的异常类exception。它只报告异常的发生，不提供任何额外信息。

- stdexcept头文件定义了几种常用的异常类。

  | -                | -                         |
  | ---------------- | ------------------------- |
  | exception        | 最常见的问题（祖先类）               |
  | runtime_error    | 只有在运行时才能检测出的问题（下面3个继承了该类） |
  | range_error      | 运行时错误：生成的结果超出了有意义的值域范围    |
  | overflow_error   | 运行时错误：计算上溢                |
  | underflow_error  | 运行时错误：计算下溢                |
  | logic_error      | 程序逻辑错误（下面4个继承了该类）         |
  | domain_error     | 逻辑错误：参数对应的结果值不存在          |
  | invalid_argument | 逻辑错误：无效参数                 |
  | length_error     | 逻辑错误：试图创建一个超出该类型最大长度的对象   |
  | out_of_range     | 逻辑错误：使用一个超出有效范围的值         |

- new头文件定义了bad_alloc异常类型。

- type_info头文件定义了bad_char异常类型。

只能以默认初始化的方式初始化exception、bad_alloc和bad_cast对象，不允许为这些对象提供初始值。其他异常类的对象在初始化时必须提供一个string或一个C风格字符串，通常表示异常信息。`what`成员函数可以返回该字符串的string副本。对于其他无初始值的异常类型来说，what返回的内容由编译器决定。