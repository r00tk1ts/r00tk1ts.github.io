---
title: C++ Primer - 泛型算法
date: 2018-11-24 14:33:11
categories: programming-language
tags:
	- cpp
	- cpp-primer




---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十章“泛型算法”时所做的笔记。

<!--more-->

# C++ Primer - 泛型算法

容器提供的接口操作少得可怜，C++通过定义一组泛型算法去应用到各种各样的容器上。

大多数算法都定义在头文件algorithm中。头文件numeric中还定义了一组数值泛型算法。

泛型算法一般不直接操纵容器，而是遍历由两个迭代器指定的元素范围进行操作。

尽管迭代器使得算法不依赖于容器，但算法却依赖于元素类型的操作。

> 比如find用元素类型的==运算符完成元素与给定值的比较，如果元素不支持==，算法就不可用。

**算法永远不会执行容器的操作。**

## 初识泛型算法

标准库提供了100多种算法，可以按一定方式进行分类。

### 只读算法

- find

  ```cpp
  int ia[] = {27,210,12,47,109,83};
  int val = 83;
  int *result = find(begin(ia), end(ia), val);

  string val = "a value";
  auto result = find(lst.cbegin(), lst.cend(), val);
  ```

- count

- accumulate

  ```cpp
  int sum = accumulate(vec.cbegin(), vec.cend(), 0);
  ```

- equal

  ```cpp
  // roster2中的元素数目应该至少与roster1一样多
  equal(roster1.cbegin(), roster1.cend(), roster2.cbegin());
  ```

### 写元素算法

- fill

  ```cpp
  fill(vec.begin(), vec.end(), 0);	//将每个元素重置为0
  ```

- fill_n

  ```cpp
  fill_n(vec.begin(), vec.size(), 0);	//将所有元素重置为0
  ```

插入迭代器（insert iterator）是一种向容器内添加元素的迭代器。通过插入迭代器赋值时，一个与赋值号右侧值相等的元素会被添加到容器中。

`back_inserter`函数（定义在头文件*iterator*中）接受一个指向容器的引用，返回与该容器绑定的插入迭代器。通过此迭代器赋值时，赋值运算符会调用push_back将一个具有给定值的元素添加到容器中。

```cpp
vector<int> vec;//空vector
auto it = back_inserter(vec);	//通过它赋值会将元素添加到vec中
*it = 42;	//vec中现在有一个元素，值为42

fill_n(back_inserter(vec), 10, 0);//添加10个元素到vec
```

- copy

  ```cpp
  int a1[] = { 0,1,2,3,4,5,6,7,8,9 };
  int a2[sizeof(a1) / sizeof(*a1)];     // a2与a1大小一样
  // ret指向拷贝到a2尾元素之后的位置
  auto ret = copy(begin(a1), end(a1), a2);    // 把a1的内容拷贝给a2
  ```

- replace

  ```cpp
  //将所有值为0的元素改为42
  replace(ilst.begin(), ilst.end(), 0, 42);
  ```

- replace_copy

  ```cpp
  //使用back_inserter按需要增长目标序列
  replace_copy(ilst.cbegin(), ilst.cend(), back_inserter(ivec), 0, 42);
  ```

  保留原序列不变，拷贝到了ivec中。

### 排序算法

- sort

- unique

  ```cpp
  void elimDups(vector<string> &words)
  {
      //按字典序排序words，以便查找重复单词
    	sort(words.begin(), words.end());
    	//unique重排输入范围，使得每个单词只出现一次
    	//排列在范围的前部，返回指向不重复区域之后一个位置的迭代器
    	auto end_unique = unique(words.begin(), words.end());
    	//使用向量操作erase删除重复单词
    	words.erase(end_unique, words.end());
  }
  ```

  因为泛型算法只能操作迭代器，不能直接增删容器的元素，所以最后还需要容器自身的erase操作来删除重复单词。

- stable_sort

  排序时会保证相等元素的相对顺序。

## 定制操作

默认情况下，很多比较算法使用元素类型的`<`或`==`运算符完成操作。可以为这些算法提供自定义操作来代替默认运算符。

### 向算法传递函数

谓词（predicate）是一个可调用的表达式，其返回结果是一个能用作条件的值。标准库算法使用的谓词分为一元谓词（unary predicate，接受一个参数）和二元谓词（binary predicate，接受两个参数）。接受谓词参数的算法会对输入序列中的元素调用谓词，因此元素类型必须能转换为谓词的参数类型。

```cpp
//比较函数，用来按长度排序单词
bool isShorter(const string &s1, const string &s2)
{
    return s1.size() < s2.size();
}
//按长度由短到长排序words
sort(words.begin(), words.end(), isShorter);
```

### lambda表达式

对于一个对象或表达式，如果可以对其使用调用运算符`()`，则称它为可调用对象（callable object）。可以向算法传递任何类别的可调用对象。

一个lambda表达式表示一个可调用的代码单元，类似未命名的内联函数，但可以定义在函数内部。其形式如下：

```cpp
[capture list] (parameter list) -> return type { function body }
```

其中，*capture list*（捕获列表）是一个由lambda所在函数定义的局部变量的列表（通常为空）。*return type*、*parameter list*和*function body*与普通函数一样，分别表示返回类型、参数列表和函数体。但与普通函数不同，lambda必须使用尾置返回类型，且不能有默认实参。

定义lambda时可以省略参数列表和返回类型，但必须包含捕获列表和函数体。省略参数列表等价于指定空参数列表。省略返回类型时，若函数体只是一个return语句，则返回类型由返回表达式的类型推断而来。否则返回类型为void。

```cpp
auto f = [] { return 42; };
cout << f() << endl;
```

lambda可以使用其所在函数的局部变量，但必须先将其包含在捕获列表中。捕获列表只能用于局部非static变量，lambda可以直接使用局部static变量和其所在函数之外声明的名字。

lambda可以以值方式或引用方式捕获变量，但引用方式必须保证lambda执行时变量存在。

引用方式有时候很有用，比如使用了ostream对象os，os本身是不能拷贝的，那么lambda使用os就需要引用捕获方式。

可以让编译器根据lambda代码隐式捕获函数变量，方法是在捕获列表中写一个`&`或`=`符号。`&`为引用捕获，`=`为值捕获。

可以混合使用显式捕获和隐式捕获。混合使用时，捕获列表中的第一个元素必须是`&`或`=`符号，用于指定默认捕获方式。显式捕获的变量必须使用与隐式捕获不同的方式。

| lambda捕获列表           |                                          |
| -------------------- | ---------------------------------------- |
| []                   | 空捕获列表。lambda不能使用所在函数中的变量。一个lambda只有捕获变量后才能使用它们 |
| [names]              | names是一个逗号分隔的名字列表，这些名字都是lambda所在函数的局部变量。默认情况下，捕获列表中的变量都被拷贝。名字前如果使用了&, 则采用引用捕获方式 |
| [&]                  | 隐式捕获列表，采用引用捕获方式。lambda体中所使用的来自所在函数的实体都采用引用方式使用 |
| [=]                  | 隐式捕获列表，采用值捕获方式。lambda体将拷贝所使用的来自所在函数的实体的值 |
| [&, identifier_list] | identifier_list是一个逗号分隔的列表，包含0到多个来自所在函数的局部变量。这些变量采用值捕获方式，而任何隐式捕获的变量都采用引用方式捕获。identifier_list中的名字前面不能使用& |
| [=, identifier_list] | identifier_list中的变量都采用引用方式捕获，而任何隐式捕获的变量都采用值方式捕获。identifier_list的名字不能包含this，且这些名字之前必须使用& |

一个例子：

```cpp
// os隐式捕获，引用捕获方式；c显示捕获，值捕获方式
for_each(words.begin(),words.end(),[&, c](const string &s){os << s << c;});
// os显示捕获，引用捕获方式；c隐式捕获，值捕获方式
for_each(words.begin(), words.end(), [=, &os](const string &s){os << s << c;});
```

默认情况下，对于值方式捕获的变量，lambda不能修改其值。如果希望修改，就必须在参数列表后添加关键字`mutable`。

```cpp
size_t v1 = 42;
auto f = [v1] () mutable { return ++v1; };
v1 = 0;
auto j = f();	// j为43
```

对于引用方式捕获的变量，lambda是否可以修改依赖于此引用指向的是否是const类型。

如果函数体语句超出一句，必须指定返回类型，且只能通过尾置方式指定：

```cpp
transform(vi.begin(), vi.end(), vi.begin(),
            [](int i) -> int { if (i < 0) return -i; else return i; });
```

`transform`函数接受三个迭代器参数和一个可调用对象。前两个迭代器参数指定输入序列，第三个迭代器参数表示目的位置。它对输入序列中的每个元素调用可调用对象，并将结果写入目的位置。

### 参数绑定

`bind`函数定义在头文件*functional*中，相当于一个函数适配器，它接受一个可调用对象，生成一个新的可调用对象来适配原对象的参数列表。一般形式如下：

```cpp
auto newCallable = bind(callable, arg_list);
```

其中，*newCallable*本身是一个可调用对象，*arg_list*是一个以逗号分隔的参数列表，对应给定的*callable*的参数。之后调用*newCallable*时，*newCallable*会再调用*callable*，并传递给它*arg_list*中的参数。*arg_list*中可能包含形如`_n`的名字，其中*n*是一个整数。这些参数是占位符，表示*newCallable*的参数，它们占据了传递给*newCallable*的参数的位置。数值*n*表示生成的可调用对象中参数的位置：`_1`为*newCallable*的第一个参数，`_2`为*newCallable*的第二个参数，依次类推。这些名字都定义在命名空间*placeholders*中，它又定义在命名空间*std*中，因此使用时应该进行双重限定。

```cpp
using std::placeholders::_1;
using namespace std::placeholders;
bool check_size(const string &s, string::size_type sz);

// check6是一个可调用对象，接受一个string类型的参数
// 并用此string和值6来调用check_size
auto check6 = bind(check_size, _1, 6);
string s = "hello";
bool b1 = check6(s);    // check6(s) calls check_size(s, 6)
```

bind函数可以调整给定可调用对象中的参数顺序。

```cpp
sort(words.begin(), words.end(), isShorter);
sort(words.begin(), words.end(), bind(isShorter, _2, _1));
```

默认情况下，bind函数的非占位符参数被拷贝到bind返回的可调用对象中。但有些类型不支持拷贝操作。

如果希望传递给bind一个对象而又不拷贝它，则必须使用标准库的`ref`函数。ref函数返回一个对象，包含给定的引用，此对象是可以拷贝的。`cref`函数生成保存const引用的类。

```cpp
ostream &print(ostream &os, const string &s, char c);
for_each(words.begin(), words.end(), bind(print, ref(os), _1, ' '));
```

## 再探迭代器

除了为每种容器定义的迭代器之外，标准库还在头文件*iterator*中定义了另外几种迭代器。

- 插入迭代器（insert iterator）：该类型迭代器被绑定到容器对象上，可用来向容器中插入元素。
- 流迭代器（stream iterator）：该类型迭代器被绑定到输入或输出流上，可用来遍历所关联的IO流。
- 反向迭代器（reverse iterator）：该类型迭代器向后而不是向前移动。除了forward_list之外的标准库容器都有反向迭代器。
- 移动迭代器（move iterator）：该类型迭代器用来移动容器元素。

### 插入迭代器

插入器是一种迭代器适配器，它接受一个容器参数，生成一个插入迭代器。通过插入迭代器赋值时，该迭代器调用容器操作向给定容器的指定位置插入一个元素。

| 插入迭代器操作         |                                          |
| --------------- | ---------------------------------------- |
| it = t          | 在it指定的当前位置插入值t。假定c是it绑定的容器，依赖于插入迭代器的不同种类，此赋值会分别调用c.push_back(t)、c.push_front(t)或c.insert(t, p)，其中p为传递给inserter的迭代器位置 |
| *it, ++it, it++ | 这些操作虽然存在，但不会对it做任何事情。每个操作都返回it           |

插入器有三种类型，差异在于元素插入的位置：

- back_inserter: 创建一个使用push_back的迭代器
- front_inserter: 创建一个使用push_front的迭代器
- inserter创建一个使用inserter的迭代器。此函数接受第二个参数，这个参数必须是一个指向给定容器的迭代器。元素被插入到给定迭代器所表示的元素之前。

> 只有容器支持push_front，才可以使用front_inserter。back_inserter亦如是。

### iostream迭代器

尽管iostream不是容器，但标准库定义了用于这些IO类型对象的迭代器。

`istream_iterator`从输入流读取数据，`ostream_iterator`向输出流写入数据。这些迭代器将流当作特定类型的元素序列处理。

```cpp
istream_iterator<int> int_it(cin);	//从cin读取int
istream_iterator<int> int_eof;	//默认初始化，当成尾后迭代器
ifstream in("afile");
istream_iterator<string> str_it(int);	//从"afile"中读取字符串

istream_iterator<int> in_iter(cin);	//从cin读取int
istream_iterator<int> eof;	//istream尾后迭代器
while(in_iter != eof)
  	// 后置递增运算读取流，返回迭代器的旧值
  	// 解引用迭代器，获得从流读取的前一个值
  	vec.push_back(*in_iter++);
```

| istream_iterator的操作           |                                          |
| ----------------------------- | ---------------------------------------- |
| `istream_iterator<T> in(is);` | in从输入流中读取类型为T的值                          |
| `istream_iterator<T> end;`    | 读取类型为T的值的istream_iterator迭代器，表示尾后位置      |
| in1 == in2  in1!= in2         | in1和in2必须读取相同类型。如果他们都是尾后迭代器，或绑定到相同的输入，则两者相等 |
| *in                           | 返回从流中读取的值                                |
| in->mem                       | 与(*in).mem含义相同                           |
| ++in, in++                    | 使用元素类型所定义的>>运算符从输入流中读取下一个值。与以往一样，前置版本返回一个指向递增后迭代器的引用，后置版本返回旧值 |

将istream_iterator绑定到一个流时，标准库并不保证迭代器立即从流读取数据。但可以保证在第一次解引用迭代器之前，从流中读取数据的操作已经完成了。

定义ostream_iterator对象时，必须将其绑定到一个指定的流。不允许定义空的或者表示尾后位置的ostream_iterator。

| ostream_iterator操作                |                                          |
| --------------------------------- | ---------------------------------------- |
| `ostream_iterator<T> out(os);`    | out将类型为T的值写到输出流os中                       |
| `ostream_iterator<T> out(os, d);` | out将类型为T的值写到输出流os中，每个值后面都输出一个d。d指向一个空字符结尾的字符数组 |
| out = val                         | 用<<运算符将val写入到out所绑定的ostream中。val的类型必须与out可写的类型兼容 |
| *out, ++out, out++                | 这些运算符是存在的，但不会out做任何事情。每个运算符都返回out        |

可以为任何定义了`<<`运算符的类型创建istream_iterator对象，为定义了`>>`运算符的类型创建ostream_iterator对象。

### 反向迭代器

递增反向迭代器会移动到前一个元素，递减会移动到后一个元素。

```cpp
//正常序排序vec
sort(vec.begin(),vec.end());
//逆序排序，最小元素放在vec末尾
sort(vec.rbegin(), vec.rend());
```

不能从forward_list或流迭代器创建反向迭代器。

## 泛型算法结构

算法要求的迭代器操作可以分为5个迭代器类别：

- 输入迭代器 - 只读不写；单遍扫描，只能递增
- 输出迭代器 - 只写不读；单遍扫描，只能递增
- 前向迭代器 - 可读写；多遍扫描，只能递增
- 双向迭代器 - 可读写；多遍扫描，可递增递减
- 随机访问迭代器 - 可读写；多遍扫描，支持全部迭代器运算

C++标准指定了泛型和数值算法的每个迭代器参数的最小类别。对于迭代器实参来说，其能力必须大于或等于规定的最小类别。向算法传递更低级的迭代器参数会产生错误（大部分编译器不会提示错误）。

- 输入迭代器（input iterator）：可以读取序列中的元素，只能用于单遍扫描算法。必须支持以下操作：
- - 用于比较两个迭代器相等性的相等`==`和不等运算符`!=`。
  - 用于推进迭代器位置的前置和后置递增运算符`++`。
  - 用于读取元素的解引用运算符`*`；解引用只能出现在赋值运算符右侧。
  - 用于读取元素的箭头运算符`->`。
- 输出迭代器（output iterator）：可以读写序列中的元素，只能用于单遍扫描算法，通常指向目的位置。必须支持以下操作：
- - 用于推进迭代器位置的前置和后置递增运算符`++`。
  - 用于读取元素的解引用运算符`*`；解引用只能出现在赋值运算符左侧（向已经解引用的输出迭代器赋值，等价于将值写入其指向的元素）。
- 前向迭代器（forward iterator）：可以读写序列中的元素。只能在序列中沿一个方向移动。支持所有输入和输出迭代器的操作，而且可以多次读写同一个元素。因此可以使用前向迭代器对序列进行多遍扫描。
- 双向迭代器（bidirectional iterator）：可以正向/反向读写序列中的元素。除了支持所有前向迭代器的操作之外，还支持前置和后置递减运算符`--`。除forward_list之外的其他标准库容器都提供符合双向迭代器要求的迭代器。
- 随机访问迭代器（random-access iterator）：可以在常量时间内访问序列中的任何元素。除了支持所有双向迭代器的操作之外，还必须支持以下操作：
- - 用于比较两个迭代器相对位置的关系运算符`<`、`<=`、`>`、`>=`。
  - 迭代器和一个整数值的加减法运算`+`、`+=`、`-`、`-=`，计算结果是迭代器在序列中前进或后退给定整数个元素后的位置。
  - 用于两个迭代器上的减法运算符`-`，计算得到两个迭代器的距离。
  - 下标运算符`[]`。

### 算法形参模式

大多数算法的形参模式是以下四种形式之一：

```cpp
alg(beg, end, other args);
alg(beg, end, dest, other args);
alg(beg, end, beg2, other args);
alg(beg, end, beg2, end2, other args);
```

> *alg*是算法名称，*beg*和*end*表示算法所操作的输入范围。几乎所有算法都接受一个输入范围，是否有其他参数依赖于算法操作。*dest*表示输出范围，*beg2*和*end2*表示第二个输入范围。

潜在的规则：

- 向输出迭代器写入数据的算法都假定目标空间足够容纳要写入的数据。
- 接受单独一个迭代器参数表示第二个输入范围的算法都假定从迭代器参数开始的序列至少与第一个输入范围一样大。

### 算法命名规范

- 接受谓词参数的算法都有附加的`_if`后缀。
- 将执行结果写入额外目的空间的算法都有`_copy`后缀。
- 一些算法同时提供`_copy`和`_if`版本。

## 特定容器算法

list和forward_list定义了几个成员函数形式的算法，比如独有的sort，merge，remove，reverse和unique。通用版本的sort要求随机访问迭代器，因此不能用于list和forward_list，这两个类型提供的分别是双向迭代器和前向迭代器。

对于list和forward_list类型，应该优先使用成员函数版本的算法，而非通用算法。

| list和forward_list成员函数版本的算法             |                                          |
| -------------------------------------- | ---------------------------------------- |
| lst.merge(lst2)  lst.merge(lst2, comp) | 将来自lst2的元素合并入lst。lst和lst2都必须是有序的。元素将从lst2中删除。合并之后，lst2变为空。第一个版本使用<运算符；第二个版本使用给定的比较操作 |
| lst.remove(val)  lst.remove_if(pred)   | 调用erase删除掉与给定值相等(==)或令一元谓词为真的每个元素        |
| lst.reverse()                          | 反转lst中元素的顺序                              |
| lst.sort()  lst.sort(comp)             | 使用<或给定比较操作排序元素                           |
| lst.unique()  lst.unique(pred)         | 调用erase删除同一个值的连续拷贝。第一个版本使用==，第二个版本使用二元谓词 |

list和forward_list的`splice`函数可以进行容器合并，其参数如下：

| lst.splice(args)或flst.splice_after(args) |                                          |
| ---------------------------------------- | ---------------------------------------- |
| (p, lst2)                                | p是一个指向lst中元素的迭代器，或一个指向flst首前位置的迭代器。函数将lst2的所有元素移动到lst中p之前位置或是flst中p之后位置。将元素从lst2中删除。lst2的类型必须与lst或flst相同，且不能是同一个链表 |
| (p, lst2, p2)                            | p2是一个指向lst2中位置的有效迭代器。将p2指向的元素移动到lst中，或将p2之后的元素移动到flst中。lst2可以是与lst或flst相同的链表 |
| (p, lst2, b, e)                          | b和e必须表示lst2中的合法范围。将给定范围中的元素从lst2移动到lst或flst。lst2与lst(或flst)可以是相同的链表，但p不能指向给定范围中元素 |

与通用算法不同，链表特有的操作会改变容器，因为它们不是依赖于iterator的。