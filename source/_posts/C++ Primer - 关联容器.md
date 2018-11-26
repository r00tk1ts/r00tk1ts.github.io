---
title: C++ Primer - 关联容器
date: 2018-11-26 19:49:11
categories: programming-language
tags:
	- cpp
	- cpp-primer





---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十一章“关联容器”时所做的笔记。

<!--more-->

# C++ Primer - 关联容器

关联容器和顺序容器虽然都是容器，但有着本质的不同。关联容器中的元素是按关键字来保存和访问的，而顺序容器的元素是按顺序来访问和保存的。

关联容器支持高效的关键字查找和访问。两个主要的关联容器类型是map和set。map是键值对，set则是集合，每个元素仅仅是一个关键字。因此，set支持高效的关键字查询操作，检查给定关键字是否在set中，而map则可以用作字典。

标准库提供8种关联容器。

| 关联容器类型             |                    |
| ------------------ | ------------------ |
| map                | 关联数组；保存键值对         |
| set                | 关键字即值，即只保存关键字的容器   |
| multimap           | 关键字可重复出现的map       |
| multiset           | 关键字可重复出现的set       |
| 无序集合               |                    |
| unordered_map      | 用哈希函数组织的map        |
| unordered_set      | 用哈希函数组织的set        |
| unordered_multimap | 哈希组织的map；关键字可以重复出现 |
| unordered_multiset | 哈希组织的set；关键字可以重复出现 |

## 使用关联容器

关联容器也是类模板。使用起来和顺序容器很相似。

### 使用map

经典的单词计数程序：

```cpp
//统计每个单词在输入中出现的次数
map<string, size_t> word_count;
string word;
while(cin >> word)
    ++word_count[word];	//提取word的计数器并将其加1
for(const auto &w : word_count)
    cout << w.first << " occurs " << w.second 
    	<< ((w.second > 1) ? " times" : " time") << endl;
```

### 使用set

利用set来保存想忽略的单词，再统计次数：

```cpp
map<string, size_t> word_count;
set<string> exclude = {"The", "But", "And", "Or", "An", "A", "the", "but", "and", "or", "an", "a"};//列表初始化

string word;
while(cin >> word)
    if(exclude.find(word) == exclude.end())
        ++word_count[word];
```

## 关联容器概述



| 容器操作一览                         |                                 |
| ------------------------------ | ------------------------------- |
| **类型别名**                       |                                 |
| iterator                       | 此容器类型的迭代器类型                     |
| const_iterator                 | 可以读取元素，但不能修改元素的迭代器类型            |
| size_type                      | 无符号整数类型，足够保存此种容器类型最大可能容器的大小     |
| difference_type                | 带符号整型，足够保存两个迭代器之间的距离            |
| value_type                     | 元素类型                            |
| reference                      | 元素的左值类型；与value_type&含义相同        |
| const_reference                | 元素的const左值类型(const value_type&) |
| **构造函数**                       |                                 |
| C c;                           | 默认构造函数，构造空容器                    |
| C c1(c2);                      | 构造c2的拷贝c1                       |
| C c(b, e);                     | 构造c，将迭代器b和e指定的范围内的元素拷贝到c        |
| C c{a, b, c...};               | 列表初始化c                          |
| **赋值与swap**                    |                                 |
| c1 = c2                        | 将c1中的元素替换为c2中元素                 |
| c1 = {a, b, c...}              | 将c1中的元素替换为列表中元素（不适用于array）      |
| a.swap(b)                      | 交换a和b的元素                        |
| swap(a, b)                     | 与a.swap(b)等价                    |
| **大小**                         |                                 |
| c.size()                       | c中元素的数目(不支持forward_list)        |
| c.max_size()                   | c可保存的最大元素数目                     |
| c.empty()                      | 若c中存储了元素，返回false，否则返回true       |
| **添加/删除元素 (不适用于array)**        | 注：不同容器中，这些操作的接口都不同              |
| c.insert(args)                 | 将args中元素拷贝到c                    |
| c.emplace(inits)               | 使用inits构造c中的一个元素                |
| c.erase(args)                  | 删除args指定的元素                     |
| c.clear()                      | 删除c中的所有元素，返回void                |
| **关系运算符**                      |                                 |
| ==, !=                         | 所有容器都支持相等(不等)运算符                |
| <, <=, >, >=                   | 关系运算符(无序关联容器不支持)                |
| **获取迭代器**                      |                                 |
| c.begin(), c.end()             | 返回指向c的首元素和尾元素之后位置的迭代器           |
| c.cbegin(). c.cend()           | 返回const_iterator                |
| **反向容器的额外成员(不支持forward_list)** |                                 |
| reverse_iterator               | 按逆序寻址元素的迭代器                     |
| const_reverse_iterator         | 不能修改元素的逆序迭代器                    |
| c.rbegin(), c.rend()           | 返回指向c的尾元素和首元素之前位置的迭代器           |
| c.crbegin(), c.crend()         | 返回const_reverse_iterator        |

此前顺序容器支持的这些操作，对于关联容器来说也是支持的。它们是通用的。

关联容器不支持顺序容器的位置相关的操作，比如push_front, push_back。因为关联容器的元素是根据关键字进行存储，没有存储顺序的概念。关联容器也不支持构造函数或插入操作这些接受一个元素值和一个数量值得操作。

关联容器也有他自己专属的类型别名和操作。

### 定义

前面看过了map和set的定义以及列表初始化set的方式。

map的初始化也类似：

```cpp
map<string, string> authors = {{"Joyce", "James"},
                               {"Austen", "Jane"},
                               {"Dickens", "Charles"}};
```

每个花括号中，第一个是键，第二个是值。

multimap和multiset的关键字可以重复。

```cpp
vector<int> ivec;
for(vector<int>::size_type i = 0;i != 10; ++i){
    ivec.push_back(i);
    ivec.push_back(i);	//每个数重复保存一次
}

//iset包含来自ivec的不重复元素，miset包含所有20个元素
set<int> iset(ivec.cbegin(), ivec.cend());
multiset<int> miset(ivec.cbegin(), ivec.cend());
cout << ivec.size() << endl;	//20
cout << iset.size() << endl;	//10
cout << miset.size() << endl;	//20
```

### 关键字类型的要求

关联容器对关键字的类型是有限制的。对有序容器map,multimap,set和multiset来说，关键字类型定义元素比较的方法。默认情况下，标准库使用关键字类型的<运算符来比较两个关键字。在集合类型中，关键字类型就是元素类型。映射类型中，关键字类型是元素的第一部分的类型。

有序容器还支持一种特殊的构造方式：

```cpp
bool compareIsbn(const Sales_data &lhs, const Sales_data &rhs){
    return lhs.isbn() < rhs.isbn();
}

//Sales_data本身不支持<运算符，但是可以用另一种构造方式来构造，这种需要提供一个函数指针类型，在定义对象和具体的类模板类型时同时提供
multiset<Sales_data, decltype(compareIsbn)*> 
bookstore(compareIsbn);//也可以不用decltype，直接compareIsbn或&compareIsbn

```

> 再次强调，decltype获得函数指针类型时，需要加上*来表示是一个指针。有序容器会利用compareIsbn来对Sales_data进行排序。

### pair类型

定义在utility头文件中，pair对关联容器的操作至关重要。

pair也是类模板，创建pair时，必须提供两个类型名，pair的数据成员将具有对应的类型，两个类型可以不同。

```cpp
pair<string, string> anon;	//保存两个string
pair<string, size_t> word_count;	//保存一个string和一个size_t
pair<string, vector<int>> line;	//保存string和vector<int>
```

pair的默认构造函数对数据成员进行值初始化。

当然也可以提供初始化：

```cpp
pair<string, string> author{"James", "Joyce"};
```

pair的数据成员是public的，两个成员分别命名为first和second。

```cpp
cout << w.first << " occurs " << w.second << ((w.second > 1) ? " times" : " time") << endl;
//w是指向map中某个元素的引用。map的元素是pair。
```

| pair的操作                    |                                          |
| -------------------------- | ---------------------------------------- |
| pair<T1, T2> p;            | p是一个pair，两个类型分别为T1和T2的成员都进行了值初始化         |
| pair<T1, T2> p(v1, v2);    | p是一个成员类型为T1和T2的pair；first和second成员分别用v1和v2进行初始化 |
| pair<T1, T2> p = {v1, v2}; | 等价于p(v1, v2)                             |
| make_pair(v1, v2)          | 返回一个用v1和v2初始化的pair。pair的类型从v1和v2的类型推断出来  |
| p.first                    | 返回p的名为first的数据成员                         |
| p.second                   | 返回p的名为second的数据成员                        |
| p1 relop p2                | 关系运算符(<, <=, >, >=)按字典序定义：例如，当p1.first < p2.first或!(p2.first < p1.first) && p1.second < p2.second成立时，p1 < p2为true。关系运算利用元素的<运算符来实现 |
| p1 == p2<br />p1 != p2     | 当first和second成员分别相等时，两个pair相等。相等性判断利用元素的==运算符实现 |

### 创建pair对象的函数

```cpp
pair<string, int> process(vector<string> &v)
{
    // 处理v
    if(!v.empty())
        return {v.back(), v.back().size()};// 列表初始化
    else
        return pair<string, int>();	// 隐式构造返回值
}
```

C++11允许直接返回花括号列表来初始化返回值对象。

## 关联容器操作

| 关联容器额外的类型别名 |                                          |
| ----------- | ---------------------------------------- |
| key_type    | 此容器类型的关键字类型                              |
| mapped_type | 每个关键字关联的类型；只适用于map                       |
| value_type  | 对于set，与key_type相同；<br />对于map，为pair<const key_type, mapped_type> |

### 关联容器迭代器

解引用关联容器迭代器会获得容器的value_type类型的值的引用。

```cpp
auto map_it = word_count.begin();
// *map_it是指向一个pair<const string, size_t>对象的引用
cout << map_it->first;
cout << " " << map_it->second;
map_it->first = "new key";	//错误：关键字是const的
++map_it->second;	//正确：可以通过迭代器改变元素
```

set的迭代器是const的。尽管set定义了iterator和const_iterator两种类型，但它们都只允许只读访问set中的元素。

```cpp
set<int> iset = {0,1,2,3,4,5,6,7,8,9};
set<int>::iterator set_it = iset.begin();
if(set_it != iset.end()){
    *set_it = 42;				//错误：set中的关键字是只读的
  	cout << *set_it << endl;	//正确：可以读关键字
}
```

通常不对关联容器使用泛型算法，因为关联容器的const特性导致修改或重排容器元素的算法是不可用的，关联容器能够应用那些只读算法。但另一方面，由于关联容器中的元素不能通过它们的关键字进行（快速）查找，因此对其使用泛型搜索算法并不高效。关联容器定义了一个名为find的成员，它通过一个给定的关键字直接获取元素。这一成员find方法比泛型算法find快得多。

> 因为map和set是红黑树组织的，这是一种平衡二叉搜索树，泛型的find算法去逐个遍历成员显然要比内置的搜索树find策略慢得多。

| 关联容器的insert操作                         |                                          |
| ------------------------------------- | ---------------------------------------- |
| c.insert(v)                           | v是value_type类型的对象；args用来构造一个元素           |
| c.emplace(args)                       | 对于map和set，只有当元素的关键字不在c中时才插入(或构造)元素。函数返回一个pair，包含一个迭代器，指向具有指定关键字的元素，以及一个指示插入是否成功的bool值。对于multimap和multiset，总会插入（或构造）给定元素，并返回一个指向新元素的迭代器。 |
| c.insert(b, e)                        | b和e是迭代器，表示一个c::value_type类型值的范围；         |
| c.insert(il)                          | il是这种值的花括号列表。返回void。对于map和set，只插入关键字不在c中的元素。对multimap和multiset，则会插入范围中的每个元素。 |
| c.insert(p,v)<br />c.emplace(p, args) | 类似insert(v)（或emplace(args)），但将迭代器p作为一个提示，指出从哪里开始搜索新元素应该存储的位置。返回一个迭代器，指向具有给定关键字的元素 |
| 删除操作                                  |                                          |
| c.erase(k)                            | 从c中删除每个关键字为k的元素。返回一个size_type值，指出删除的元素的数量 |
| c.erase(p)                            | 从c中删除迭代器p指定的元素。p必须指向c中一个真实元素，不能等于c.end()。返回一个指向p之后元素的迭代器，若p指向c中的尾元素，则返回c.end() |
| c.erase(b, e)                         | 删除迭代器对b和e所表示范围的所有元素。返回e                  |

map和unordered_map支持下标运算符和at函数：

- c[k]: 返回关键字为k的元素；如果k不在c中，添加一个关键字为k的元素，进行值初始化（这一点提供了简洁性）。
- c.at(k): 访问关键字为k的元素，带参数检查；若k不在c中，抛出一个out_of_range异常

一般来说，下标运算符返回类型和迭代器解引用返回的类型是一致的，但对map来说，下标操作获得的是一个mapped_type对象，而解引用map迭代器则获得的是value_type对象。

map的下标运算符返回一个左值，所以可以进行写元素。

| 关联容器查找元素         | lower_bound和upper_bound不适用于无序容器。下标和at操作只适用于非const的map和unordered_map。 |
| ---------------- | ---------------------------------------- |
| c.find(k)        | 返回一个迭代器，指向第一个关键字为k的元素，若k不在容器，则返回尾后迭代器    |
| c.count(k)       | 返回关键字等于k的元素的数量。对于不允许重复关键字的容器，返回值永远是0或1   |
| c.lower_bound(k) | 返回一个迭代器，指向第一个关键字不小于k的元素                  |
| c.upper_bound(k) | 返回一个迭代器，指向第一个关键字大于k的元素                   |
| c.equal_range(k) | 返回一个迭代器pair，表示关键字等于k的元素的范围。若k不存在，pair的两个成员均等于c.end() |

multimap和multiset中查找元素可能对一个给定关键字有多个结果，所以要配合count：

```cpp
string search_item("Alain de Botton");
auto entries = authors.count(search_item);	//元素的数量
auto iter = authors.find(search_item);	//此作者的第一本书

while(entries){
    cout << iter->second << endl;
  	++iter;
  	--entries;
}
```

也可以：

```cpp
for(auto beg = authors.lower_bound(search_item), end = authors.upper_bound(search_item);beg != end; ++beg){
    cout << beg->second << endl;
}
```

还可以：

```cpp
for(auto pos=authors.equal_range(search_item);pos.first != pos.second; ++pos.first)
  	cout << pos.first->second << endl;
```

equal_range接受一个关键字，返回一个迭代器pair。若关键字存在，则第一个迭代器指向第一个与关键字匹配的元素，第二个迭代器指向最后一个匹配元素之后的位置。若未找到匹配元素，则两个迭代器都指向关键字可以插入的位置。

## 无序容器

无序容器不适用比较运算符来组织元素，而是用哈希函数和关键字类型的==运算符。

应用于有序容器的操作也都可以用于无序容器(find, insert等)。因此，通常可以用一个无序容器替换有序容器，反之亦然。

用unordered_map重写计数程序：

```cpp
unordered_map<string, size_t> word_count;
string word;
while(cin >> word)
  	++word_count[word];
for(const auto &w : word_count)
  	cout << w.first << " occurs " << w.second << ((w.second > 1) ? " times" : " time") << endl;
```

无序容器在存储上组织为一组桶，每个桶保存零到多个元素。无序容器使用一个哈希函数将元素映射到桶。为了访问一个元素，容器首先计算元素的哈希值，指出应该搜索哪个桶。容器将具有同一哈希值的所有元素保存在相同的桶中。

因此，无序容器的性能依赖于哈希函数的质量和桶数量的大小。

> 其实就是最普通的桶式哈希表。

| 无序容器管理操作               |                                          |
| ---------------------- | ---------------------------------------- |
| 桶接口                    |                                          |
| c.bucket_count()       | 正在使用的桶的数目                                |
| c.max_bucket_count()   | 容器能容纳的最多的桶的数量                            |
| c.bucket_size(n)       | 第n个桶中有多少个元素                              |
| c.bucket(k)            | 关键字为k的元素在哪个桶中                            |
| 桶迭代                    |                                          |
| local_iterator         | 可以用来访问桶中元素的迭代器类型                         |
| const_local_iterator   | 桶迭代器的const版本                             |
| c.begin(n), c.end(n)   | 桶n的首元素迭代器和尾后迭代器                          |
| c.cbegin(n), c.cend(n) | 与前两个函数类似，但返回const_local_iterator         |
| 哈希策略                   |                                          |
| c.load_factor()        | 每个桶的平均元素数量，返回float值                      |
| c.max_load_factor()    | c试图维护的平均通大小，返回float值。c会在需要时添加新的桶，以使得load_factor <= max_load_factor |
| c.rehash(n)            | 重组存储，使得bucket_count >= n且bucket_count > size/max_load_factor |
| c.reserve(n)           | 重组存储，使得c可以保存n个元素且不必rehash                |

默认情况，无序容器使用关键字类型的==来比较元素，它们还使用一个`hash<key_type>`类型的对象来生成每个元素的哈希值。标准库为内置类型（包括指针）提供了hash模板。还为一些标准库类型，如string和智能指针类型定义了hash。因此，我们可以直接定义关键字是内置类型（包括指针类型）、string和智能指针类型的无序容器。

如果想要定义自定义类型的无序容器，就不能直接使用哈希模板，而必须提供自己的hash模板版本。

```cpp
size_t hasher(const Sales_data & sd){
    return hash<string>()(sd.isbn());
}
bool eqOp(const Sales_data &lhs, const Sales_data &rhs){
    return lhs.isbn() == rhs.isbn();
}
using SD_multiset = unordered_multiset<Sales_data, decltype(hasher)*, decltype(eqOp)*>;
//参数是桶大小，哈希函数指针和==运算符指针
SD_multiset bookstore(42, hasher, eqOp);
```

具体是如何做到的，以后会晓得。

如果类定义了==运算符，则可以只重载哈希函数：

```cpp
//使用FooHash生成哈希值；Foo必须有==运算符
unordered_set<Foo, decltype(FooHash)*> fooSet(10, FooHash);
```

