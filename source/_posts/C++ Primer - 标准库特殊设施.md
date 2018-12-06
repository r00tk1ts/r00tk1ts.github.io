---
title: C++ Primer - 标准库特殊设施
date: 2018-12-06 20:22:11
categories: programming-language
tags:
	- cpp
	- cpp-primer


---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十七章“标准库特殊设施”时所做的笔记。

<!--more-->

# C++ Primer - 标准库特殊设施

## tuple

tuple类似pair，pair只有两个成员，但tuple可以有任意数量的成员。tuple也是类模板。

| tuple支持的操作                               |                                          |
| ---------------------------------------- | ---------------------------------------- |
| `tuple<T1, T2, ... ,Tn> t;`              | t是一个tuple，成员数为n，第i个成员的类型为Ti。所有成员都进行值初始化。 |
| `tuple<T1, T2, ..., Tn> t(v1, v2, ..., vn);` | t是一个tuple，成员类型为T1..Tn,每个成员用对应的初始值vi进行初始化。此构造函数是explicit的。 |
| make_tuple(v1, v2, ..., vn)              | 返回一个用给定初始值初始化的tuple。tuple的类型从初始值的类型推断    |
| t1 == t2                                 | 当两个tuple具有相同数量的成员且成员对应相等时，两个tuple相等。     |
| t1 != t2                                 | 这两个操作使用成员的==运算符来完成。一旦某个成员不等，就比接着比下去了。    |
| t1 relop t2                              | tuple的关系运算使用字典序。两个tuple必须具有相同数量的成员。使用<运算符比较t1的成员和t2中的对应成员 |
| `get<i>(t)`                              | 返回t的第i个数据成员的引用；如果t是一个左值，结果是一个左值引用；否则，结果是一个右值引用。tuple的所有成员都是public的 |
| `tuple_size<tupleType>::value`           | 类模板，可以通过一个tuple类型来初始化。value是其public constexpr static数据成员，类型为size_t，表示给定tuple类型中成员的数量 |
| `tuple_element<i, tupleType>::type`      | 类模板，可以通过一个整型常量和一个tuple类型来初始化。它有一个名为type的public成员，表示给定tuple类型中指定成员的类型。 |

### 定义初始化tuple

```cpp
tuple<size_t, size_t, size_t> threeD;	//三个成员都被值初始化为0
tuple<string, vector<double>, int, list<int>> someVal("constants", {3.14, 2.718}, 42, {0,1,2,3,4,5});
```

因为`tuple<T1, T2, ..., Tn> t(v1, v2, ..., vn);`是explicit的，所以必须使用直接初始化语法：

```cpp
tuple<size_t, size_t, size_t> threeD = {1,2,3};	//错误
tuple<size_t, size_t, size_t> threeD{1,2,3};	//正确
```

按照惯例，有一个make_tuple：

```cpp
auto item = make_tuple("0-999-78345-X", 3, 20.00);
```

auto推测出为`tuple<const char*, int, double>`。

可以使用`get`访问tuple的成员。get是一个函数模板，使用时必须指定一个显式模板实参，表示要访问的成员索引。传递给get一个tuple实参后，会返回其指定成员的引用。

```cpp
auto book = get<0>(item);	//返回item的第一个成员
auto cnt = get<1>(item);	//返回item的第二个成员
auto price = get<2>(item);	//返回item的最后一个成员
get<2>(item) *= 0.8;		//打八折
```

tuple_size和tuple_element可以辅助查询tuple成员的数量和类型。

```cpp
typedef decltype(item) trans;	//使用decltype确定对象的类型
size_t sz = tuple_size<trans>::value;	//返回3
tuple_element<1, trans>::type cnt = get<1>(item);	//cnt是int类型
```

由于tuple定义了`<`和`==`运算符，因此tuple序列可以被传递给算法，无序容器的关键字也可以使用tuple类型。

tuple的一个用途是函数一次性返回多个值。

## bitset

bitset类用于处理位运算，能够处理超过最长整型类型大小的位集合。bitset定义在头文件bitset中。

### 定义和初始化bitset

| 初始化bitset的方法                          |                                          |
| ------------------------------------- | ---------------------------------------- |
| `bitset<n> b;`                        | b有n位；每一位均为0。此构造函数是一个constexpr            |
| `bitset<n> b(u);`                     | b是unsigned long long值u的低n位的拷贝。如果n大于unsigned long long的大小，则b中超出unsigned long long的高位被置为0。此构造函数是一个constexpr。 |
| `bitset<n> b(s, pos, m, zero, one);`  | b是string s从位置pos开始m个字符的拷贝。s只能包含字符zero或one；如果s包含任何其他字符，构造函数会抛出invalid_argument异常。字符在b中分别保存为zero和one。pos默认为0，m默认为string::npos，zero默认为'0'，one默认为'1' |
| `bitset<n> b(cp, pos, m, zero, one);` | 同上，但从cp指向的字符数组中拷贝字符。如果未提供m，则cp必须指向C风格字符串。如提供了m，则从cp开始必须至少有m个zero或one字符。 |

```cpp
bitset<13> bitvec1(0xbeef);	//二进制位序列为1111011101111
bitset<20> bitvec2(0xbeef);	//二进制位序列为00001011111011101111
//x64的long long是64位
bitset<128> bitvec3(~0ULL);	//0~63位为1,63~127位为0

bitset<32> bitvec4("1100");	//二进制位序列为0...01100
string str("1111111000000011001101");
bitset<32> bitvec5(str, 5, 4);           //str[5]开始的四个二进制位，1100
bitset<32> bitvec6(str, str.size()-4);   // 使用最后四个字符
```

### bitset操作

| bitset操作                        |                                          |
| ------------------------------- | ---------------------------------------- |
| b.any()                         | b中是否存在置位的二进制位                            |
| b.all()                         | b中所有位都置位了？                               |
| b.none()                        | b中不存在置位的二进制？                             |
| b.count()                       | b中置位的数量                                  |
| b.size()                        | constexpr函数，返回b的位数                       |
| b.test(pos)                     | 若pos位置的位置位，则返回true，否则返回false             |
| b.set(pos, v)                   | 将位置pos处的位设置为bool值v。v默认为true。如果未传递实参，则b中所有位置位 |
| b.reset(pos)                    | 位置pos处的位复位                               |
| b.reset()                       | b中所有位复位                                  |
| b.flip(pos)                     | 改变位置pos处的位状态                             |
| b.flip()                        | b中所有位翻转                                  |
| b[pos]                          | 访问b中位置pos处的位；如果b是const的，则当该位置位时b[pos]返回true，否则返回false |
| b.to_ulong()<br />b.to_ullong() | 返回一个unsigned long或一个unsigned long long值，其位模式与b相同。如果b中位模式不能放入指定的结果类型，则抛overflow_error异常。 |
| b.to_string(zero, one)          | 返回string，zero和one默认为0和1，表示b中的0和1         |
| os << b                         | b中二进制位打印为字符1或0，打印到流os                    |
| is >> b                         | 从is读取字符存入b。下一个字符不是1或0时，或是已经读入b.size()个位时，读取过程停止。 |

值得注意的坑：

bitset的下标运算符对const属性进行了重载。const版本的下标运算符在指定位置置位时返回true，否则返回false。非const版本返回bitset定义的一个特殊类型，用来控制指定位置的值。

## 正则表达式

C++11增加了正则表达式（RE）库，在头文件regex中定义。

| RE组件            |                                          |
| --------------- | ---------------------------------------- |
| regex           | 表示有一个正则表达式的类                             |
| regex_match     | 将一个字符序列与一个正则表达式匹配                        |
| regex_search    | 寻找第一个与正则表达式匹配的子序列                        |
| regex_replace   | 使用给定格式替换一个正则表达式                          |
| sregex_iterator | 迭代器适配器，调用regex_search来遍历一个string中所有匹配的子串 |
| smatch          | 容器类，保存在string中搜索的结果                      |
| ssub_match      | string中匹配的子表达式的结果                        |

用例：

```cpp
//查找不在字符c之后的字符串ei
string pattern("[^c]ei");
//需要包含pattern的整个单词
pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
regex r(pattern);	//构造一个用于查找模式的regex
smatch results;		//定义一个对象保存搜索结果
//定义一个string保存于模式匹配和不匹配的文本
string test_str = "receipt freind theif receive";
//用r在test_str中查找域pattern匹配的子串
if(regex_search(test_str, results, r))	//如果有匹配子串，打印匹配的单词
  	cout << results.str() << endl;		//regex_search匹配第一个之后会停止
```

regex_search和regex_match的参数格式：

- (seq, m, r, mft)	
- (seq, r, mft)

字符序列seq中查找regex对象r中的正则表达式。seq可以是string、表示范围的一对迭代器、指向空字符结尾的字符数组的指针。m是一个match对象，用来保存匹配结果的相关细节。m和seq必须具有兼容的类型。mft是可选的`regex_constants::match_flag_type`的值，它们会影响匹配过程。

默认情况下，regex使用的正则表达式语言是ECMAScript。定义一个regex或者对一个regex调用assign为其赋新值时，可以指定一些标志来影响regex的操作。`ECMAScript`、`basic`、`extended`、`awk`、`grep`和`egrep`这六个标志指定编写正则表达式时所使用的语言。这六个标志中必须设置其中之一，且只能设置一个。默认设置`ECMAScript`（使用ECMA-262规范）。

| regex(和wregex)选项                |                                          |
| ------------------------------- | ---------------------------------------- |
| regex r(re)<br />regex r(re, f) | re表示一个正则表达式，可以是string、表示字符范围的一对迭代器、指向空字符结尾的字符数组的指针、字符指针和一个计数器、花括号包围的字符列表。f是指出对象如何处理的标志。f通过下面列出的值来设置。如果未指定f，默认值为ECMAScript |
| r1 = re                         | 将r1中的正则替换为re。                            |
| r1.assign(re, f)                | 与使用赋值运算符效果相同。                            |
| r.mark_count()                  | r中子表达式的数目                                |
| r.flags()                       | 返回r的标志集                                  |

构造和赋值可能会抛出regex_error异常。

定义regex时可指定的标志：定义在`regex`和`regex_constants::syntax_option_type`中

- `icase` - 匹配过程中忽略大小写
- `nosubs` - 不保存匹配的子表达式
- `optimize` - 执行速度优先于构造速度
- `ECMAScript` - 使用ECMA-262指定的语法
- `basic` - 使用POSIX基本的正则表达式语法
- `extended` - 使用POSIX扩展的正则表达式语法
- `awk` - 使用POSIX版本的awk语言的语法
- `grep` - 使用POSIX版本的grep语法
- `egrep` - 使用POSIX版本的egrep语法

正则表达式的语法是否正确是在运行期间解析的。如果正则表达式存在错误，标准库会抛出类型为`regex_error`的异常。`regex_error`有一个what操作来描述发生了什么错误。除了what操作外，regex_error还有一个名为`code`的成员，用来返回错误类型对应的数值编码。code返回的值是由具体实现定义的。RE库能抛出的标准错误如下，code返回对应错误的编号（从0开始）。

正则表达式错误类型：(定义在`regex`和`regex_constants::error_type`中)

- `error_collate` - 无效的元素校对请求
- `error_ctype` - 无效的字符类
- `error_escape` - 无效的转义字符或无效的尾置转义
- `error_backref` - 无效的向后引用
- `error_brack` - 不匹配的方括号
- `error_paren` - 不匹配的小括号
- `error_brace` - 不匹配的花括号
- `error_badbrace` - {}中无效的范围
- `error_range` - 无效的字符范围
- `error_space` - 内存不足，无法处理此RE
- `error_badrepeat` - 重复字符(* ? + {)之前没有有效的RE
- `error_complexity` - 要求的匹配过于复杂
- `error_stack` - 栈空间不足，无法处理匹配

>  正则表达式在程序运行时才编译，这是一个非常慢的操作。因此构造一个regex对象或者给一个已经存在的regex赋值是很耗时间的。为了最小化这种开销，应该尽量避免创建不必要的regex。特别是在循环中使用正则表达式时，应该在循环体外部创建regex对象。

### 正则表达式类与输入序列类型

RE库为不同的输入序列都定义了对应的类型。使用时RE库类型必须与输入类型匹配。

- regex类保存char类型的正则表达式；`wregex`保存wchar_t类型的正则表达式。
- smatch表示string类型的输入序列；`cmatch`表示字符数组类型的输入序列；`wsmatch`表示wstring类型的输入序列；`wcmatch`表示宽字符数组类型的输入序列。

### 匹配与Regex迭代器类型

regex迭代器是一种迭代器适配器，它被绑定到一个输入序列和一个regex对象上，每种输入类型都有对应的迭代器类型。

| sregex_iterator操作          |                                          |
| -------------------------- | ---------------------------------------- |
| sregex_iterator it(b,e,r); | 遍历迭代器b和e表示的string。它调用sregex_search(b,e,r)将it定位到输入中第一个匹配的位置 |
| sregex_iterator end;       | sregex_iterator的尾后迭代器                    |
| *it<br />it->              | 根据最后一个调用regex_search的结果，返回一个smatch对象的引用或一个指向smatch对象的指针 |
| ++it<br />it++             | 从输入序列当前匹配位置开始调用regex_search。前置版本返回递增后迭代器，后置版本返回旧值 |
| it1 == it2<br />it1 != it2 | 如果两个sregex_iterator都是尾后迭代器，则它们相等；两个非尾后迭代器是从相同的输入序列和regex对象构造，则它们相等 |

用例：

```cpp
string pattern("[^c]ei");
pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
regex r(pattern, regex::icase);
for(sregex_iterator it(file.begin(), file.end(), r), end_it;it!=end_it;++it)
  	cout << it->str() << endl;
```

利用迭代器就可以处理多个匹配项了。

输出freind和theif（不会止于freind）。

匹配类型有两个名为`prefix`和`suffix`的成员，分别返回表示输入序列中当前匹配之前和之后部分的`ssub_match`对象。一个ssub_match对象有两个名为`str`和`length`的成员，分别返回匹配的string和该string的长度。

```cpp
for (sregex_iterator it(file.begin(), file.end(), r), end_it;
    it != end_it; ++it)
{
    auto pos = it->prefix().length();    // 前缀的大小
    pos = pos > 40 ? pos - 40 : 0;       // 想要最多40个字符
    cout << it->prefix().str().substr(pos)          // 前缀的最后一部分
        << "\n\t\t>>> " << it->str() << " <<<\n"    // 匹配的单词
        << it->suffix().str().substr(0, 40)         // 后缀的第一部分
        << endl;
}
```

| smatch操作                                 |                                          |
| ---------------------------------------- | ---------------------------------------- |
| m.ready()                                | 如果已经通过调用regex_search或regex_match设置了m，则返回true；否则返回false。如果ready返回false，则对m进行操作是未定义的 |
| m.size()                                 | 如果匹配失败，则返回0；否则返回最近一次匹配的正则表达式中子表达式的数目     |
| m.empty()                                | 若m.size()为0，则返回true                      |
| m.prefix()                               | 一个ssub_match对象，表示当前匹配之前的序列               |
| m.suffix()                               | 一个ssub_match对象，表示当前匹配之后的部分               |
| m.format(...)                            | ...                                      |
| m.length(n)                              | 第n个匹配的子表达式的大小                            |
| m.position(n)                            | 第n个子表达式距序列开始的距离                          |
| m.str(n)                                 | 第n个子表达式匹配的string                         |
| m[n]                                     | 对应第n个子表达式的ssub_match对象                   |
| m.begin(),m.end()<br />m.cbegin(), m.cend() | m中sub_match元素范围的迭代器。与往常一样，cbegin和cend返回const_iterator |

### 使用子表达式

正则表达式中的模式通常包含一个或多个子表达式。子表达式是模式的一部分，本身也有意义。正则表达式语法通常用括号表示子表达式。

```cpp
// r有两个子表达式：第一个是点之前表示文件名的部分，第二个表示文件扩展名
regex r("([[:alnum:]]+)\\.(cpp|cxx|cc)$",regex::icase);
```

前者匹配一个或多个字符的序列，后者匹配三种文件扩展名。

> 因为反斜线`\`是C++中的特殊字符，所以在模式中使用`\`时，需要一个额外的反斜线进行转义。

匹配对象除了提供匹配整体的相关信息外，还可以用来访问模式中的每个子表达式。子匹配是按位置来访问的，第一个子匹配位置为0，表示整个模式对应的匹配，随后是每个子表达式对应的匹配。

子表达式的一个常见用途是验证必须匹配特定格式的数据，如电话号码和电子邮箱地址。

ECMAScript正则表达式语言的一些特性：

- 模式`[[:alnum:]]`匹配任意字母。
- 符号`+`表示匹配一个或多个字符。
- 符号`*`表示匹配零个或多个字符。
- `\{d}`表示单个数字，`\{d}{n}`表示一个n个数字的序列。
- 在方括号中的字符集合表示匹配这些字符中的任意一个。
- 后接`?`的组件是可选的。
- 类似C++，ECMAScript使用反斜线进行转义。由于模式包含括号，而括号是ECMAScript中的特殊字符，因此需要用`\(`和`\)`来表示括号是模式的一部分。

| 子匹配操作             |                                          |
| ----------------- | ---------------------------------------- |
| matched           | 一个public bool数据成员，指出此ssub_match是否匹配了     |
| first<br />second | public数据成员，指向匹配序列首元素和尾后位置的迭代器。如果未匹配，则first和second相等 |
| length()          | 匹配的大小。如果matched为false，则返回0               |
| str()             | 返回一个包含输入中匹配部分的string。如果matched为false，则返回空string |
| s = ssub          | 将ssub_match对象ssub转化为string对象s。等价于s = ssub.str()。转换运算符不是explicit的 |

### regex_replace

| RE的替换操作                                  |                                          |
| ---------------------------------------- | ---------------------------------------- |
| m.format(dest, fmt, mft)<br />m.format(fmt, mft) | 使用格式字符串fmt生成格式化输出，匹配在m中，可选的match_flag_type标志在mft中。第一个版本写入迭代器dest指向的目的位置并接受fmt参数，第二个版本返回string，保存输出，并接受fmt参数。mft的默认值为format_default |
| regex_replace(dest,seq,r,fmt,mft)<br />regex_replace(seq,r,fmt,mft) | 遍历seq，用regex_search查找与regex对象r匹配的子串。使用格式字符串fmt和可选的match_flag_type标志来生成输出。第一个版本将输出写入到迭代器dest指定的位置，并接受一对迭代器seq表示范围；第二个版本返回string保存输出。mft默认值为match_default |

匹配标志：(在`regex_constants::match_flag_type`中)

- match_default	- 等价于format_default
- match_not_bol - 不将首字符作为行首处理
- match_not_eol - 不将尾字符作为行尾处理
- match_not_bow - 不将首字符作为单词首处理
- match_not_eow - 不将尾字符作为单词尾处理
- match_any - 如果存在多于一个匹配，则可返回任意一个匹配
- match_not_null - 不匹配任何空序列
- match_continuous - 匹配必须从输入的首字符开始
- match_prev_avail - 输入序列包含第一个匹配之前的内容
- format_default - 用ECMAScript规则替换字符串
- format_sed - 用POSIX sed规则替换字符串
- format_no_copy - 不输出输入序列中未匹配的部分
- format_first_only - 只替换子表达式的第一次出现

默认情况下，`regex_replace`输出整个输入序列。未与正则表达式匹配的部分会原样输出，匹配的部分按照格式字符串指定的格式输出。使用`format_no_copy`标志可以只输出匹配部分。

## 随机数

在新标准出现之前，C和C++都依赖于一个简单的C库函数`rand`来生成随机数。该函数生成均匀分布的伪随机整数，每个随机数的范围在0和一个系统相关的最大值（至少为32767）之间。

头文件*random*中的随机数库定义了一组类来解决rand函数的一些问题：随机数引擎类（random-number engines）可以生成unsigned随机数序列；随机数分布类（random-number distribution classes）使用引擎类生成指定类型、范围和概率分布的随机数。

随机数库的组成:

- 引擎 - 类型，生成随机unsigned整数序列
- 分布 - 类型，使用引擎返回服从特定概率分布的随机数

C++程序不应该使用rand函数，而应该使用`default_random_engine`类和恰当的分布类对象。

### 随机数引擎和分布

随机数引擎是函数对象类，定义了一个不接受参数的调用运算符，返回一个随机unsigned整数。调用一个随机数引擎对象可以生成原始随机数。

```cpp
default_random_engine e;	//生成随机无符号数
for(size_t i=0;i<10;i++)
  	cout << e() << " ";	//调用对象e()生成下一个随机数
```

| 随机数引擎操作              |                                 |
| -------------------- | ------------------------------- |
| Engine e;            | 默认构造函数；使用该引擎类型默认的种子             |
| Engine e(s);         | 使用整型值s作为种子                      |
| e.seed(s)            | 使用种子s重置引擎状态                     |
| e.min()<br />e.max() | 此引擎可生成的最小最大值                    |
| Engine::result_type  | 此引擎生成的unsigned整型类型              |
| e.discard(u)         | 将引擎推进u步，u的类型为unsigned long long |

使用分布类对象可以得到指定范围的随机数。新标准库的`uniform_int_distribution<unsigned>`类型生成均匀分布的unsigned值。

```cpp
//生成0到9之间的均匀分布的随机数
uniform_int_distribution<unsigned> u(0,9);
default_random_engine e;    // 生成无符号随机整数
for (size_t i = 0; i < 10; ++i)
    // u作为随机数源
    // 每个调用返回在指定范围内并服从均匀分布的值
    cout << u(e) << " ";
```

分布类型也是函数对象，定义了一个调用运算符，它接受一个随机数引擎作为参数。分布对象用引擎参数生成随机数，并映射到指定的分布。

通常所说的随机数发生器，是指分布对象和引擎对象的组合。

rand函数的生成范围在0到`RAND_MAX`之间，随机数引擎生成的unsigned整数在一个系统定义的范围内。一个引擎类型的范围可以通过调用该类型对象的`min`和`max`成员来获得。

即使随机数发生器生成的数看起来是随机的，但对于一个给定的发生器，每次运行程序时它都会返回相同的数值序列（所谓的伪随机）。

如果函数需要局部的随机数发生器，应该将其（包括引擎和分布对象）定义为static对象，这样随机数发生器就能在函数调用期间保持状态。否则每次调用函数都会生成相同的序列。

可以为引擎提供一个seed，让引擎在程序每次运行时生成不同的序列。种子是一个数值，引擎利用它从序列中的一个新位置重新开始生成随机数。

为引擎设置种子有两种方式：

- 在创建对象时提供种子。
- 调用引擎的`seed`成员设置种子。

选择种子的常用方法是调用系统函数`time`。该函数定义在头文件*ctime*中，返回从一个特定时刻到当前经过的秒数。time函数接受单个指针参数，指向用于写入时间的数据结构。如果指针为空，则函数简单地返回时间。

```cpp
default_random_engine eq(time(0));	
```

由于time函数返回以秒计算的时间，因此用time返回值作为种子的方式只适用于生成种子的间隔为秒级或更长时间的应用。另外如果程序作为一个自动过程的一部分反复运行，这种方式也会无效，可能多次使用的是相同的种子。

### 其他随机数分布

随机数引擎生成的unsigned数，范围内生成的数是均匀分布的。可以通过定义不同分布类型来满足其他分布的要求。

| 分布类型操作              |                                          |
| ------------------- | ---------------------------------------- |
| Dist d;             | 默认构造；其他构造依赖于Dist的类型，explicit             |
| d(e)                | 用相同的e连续调用d的话，会根据d的分布式类型生成一个随机数序列；e是一个随机数引擎对象 |
| d.min()<b />d.max() | 返回d(e)能生成的最小值和最大值                        |
| d.reset()           | 重建d状态，使得随后对d的使用不依赖于d已经生成的值               |

#### 随机浮点数

从rand函数获得随机浮点数的一个常用但不正确的方法是用rand()的结果除以*RAND_MAX*。但因为随机整数的精度通常低于随机浮点数，所以使用这种方法时，有一些浮点值永远不会被生成。使用新标准库的`uniform_real_distribution`类型可以获得随机浮点数。

```cpp
default_random_engine e;
uniform_real_distribution<double> u(0,1);
//uniform_real_distribution<> u(0,1)也行，因为默认模板形参是double
cout << u(e);
```

#### 非均匀分布的随机数

标准库支持20种分布，不一一展开了。

## IO库再探

待续。。。