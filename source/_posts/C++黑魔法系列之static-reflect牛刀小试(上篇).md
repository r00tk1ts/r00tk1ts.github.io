---
title: C++黑魔法系列之static-reflect牛刀小试(上篇)
date: 2024-07-20 10:30:50
tags: [cpp, cpp-templates]
category: 源码剖析
---

近日有位道友发了我个基于C++14的静态反射库[reflect-hpp](https://github.com/archibate/reflect-hpp)，表示奇技淫巧完全看不懂，丢给我想让我锐评一下。我一看，好家伙，这不是小彭老师的屠龙宝术嘛，我哪敢吱声，学吧，学无止境。。。

<!-- more -->

# C++黑魔法系列之static-reflect牛刀小试(上篇)

众所周知，C++没有反射，一些被Go,Java的反射惯坏了的框架开发者，上手C++难免水土不服。实际上说C++没有反射也不尽然，毕竟这玩意实现起来也没啥难度，且不说市面上存在大把反射库，就像是QT、protobuf这种赫赫有名的框架/库也都实现了反射，手法与隔壁也都大同小异，它们都是运行时反射，或者叫动态反射。

然而对于C++这门语言来说，标准一向追求的是零开销，动态反射这种拉胯的性能表现根本入不了委员会的法眼，标准C++想要提供的是编译期反射，即静态反射。实际上这一议题早就有提案了[Reflect TS](https://github.com/boostcon/cppnow_presentations_2019/blob/master/05-08-2019_wednesday/The_Cpp_Reflection_TS__David_Sankel__cppnow_05082019.pdf)，原本计划赶在C++23前板上钉钉，甚至某些编译期厂商都按捺不住开始实现了，奈何委员会进度感人。嗯，然后不出意外的跳票了，哦，因为疫情，疫情（~~在跳票这方面，还得看C++委员会，不愧是你，DDDD~~）。

那还能咋办，三年又三年呗，~~要隐忍~~。C++26未来可期（~~别骂了，别骂了，已经是C++26三个大目标之一了~~），最新的提案详见：[Reflection for C++26](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2996r0.html#ref-P2670R1)。

完整的编译期反射能力还是相当繁杂的，就不在此长篇大论了，有兴趣的前辈、道友~~和蝼蚁~~可以点开上面的传送门，自行钻研。

## 编译期反射

想要实现反射，首要任务是要记录Class Type的成员信息，对于运行时反射来说，开辟一个额外的空间予以记录即可，反射实际上就是个按图索骥，没图就遍历的操作。但是对于编译期反射来说，要考虑的事情就很多了，没有原生语法糖的支持，往往只能用大量的模板和宏去创造（~~猪脑过载警告~~）。

[reflect-hpp](https://github.com/archibate/reflect-hpp)正是如此，它在实现上使用了大量的模板和宏，通过元编程的方式在编译期生成辅助桩代码，用以支撑反射。当然，这个库只是一个最小的demo实现，作者为了向前兼容C++14，甚至舍弃了一些C++17,20的新特性，沿用了C++14之前的一些晦涩蹩脚的写法。

reflect-hpp分别用JSON和pybind做了示例，其中JSON借助了jsoncpp库做序列化和反序列化，通过静态反射桥接，可以非常方便的实现只用一句代码让任意Class类型与JSON对象/字符串之间相互转换。

整体的实现较为复杂，由于是最小实现demo作者写得也比较随意凌乱，我就不一行行展开了，而是通过roadmap的方式去打谱，一步一步逼近最终实现。

## 传统JSON库的任意类型转换

任意类型转换(Arbitrary Type Conversion)，对于JSON库来说是非常重要的能力，大部分JSON库都是通过编写converter代码的方式来支持，而如果能够使用反射的能力，那么甚至可以省去converter的编写，代码改动成本降到最低。

事实上配合reflect可以选用任意的JSON库，这里我就不用reflect-hpp所举例的jsoncpp了，换成我个人最熟悉的开源库：[nlohmann/json](https://github.com/nlohmann/json)来继续讲解，nlohmann/json是个非常modern的JSON库，我们先看看在没有反射之前，它对于任意类型转换是如何支撑的。

```cpp
#include <iostream>
#include <string>
#include "nlohmann/json.hpp"

struct Person {
  std::string name;
  int age;
  std::string address;
};

// 对Person类重载to_json方法，编写p到JSON obj的converter
void to_json(nlohmann::json& j, const Person& p) {
  j["name"] = p.name;
  j["age"] = p.age;
  j["address"] = p.address;
}

// 对Person类重载from_json方法，编写JSON obj到p的converter
void from_json(const nlohmann::json& j, Person& p) {
  j.at("name").get_to(p.name);
  j.at("age").get_to(p.age);
  j.at("address").get_to(p.address);
}

int main() {
  Person person = {.name = "Alice", .age = 25, .address = "Beijing, China:"};
  nlohmann::json j = person;
  // 按indent=4对齐输出JSON字符串
  std::cout << j.dump(4) << "\n";

  Person person2 = j.get<Person>();
  std::cout << person2.name << " " << person2.age << " " << person2.address << "\n";

  return 0;
}
```

执行结果：

```shell
 qichencao@QICHENCAO-MB1  ~/workspace/static-reflect  g++ main.cpp -o main -std=c++20
 qichencao@QICHENCAO-MB1  ~/workspace/static-reflect  ./main
{
    "address": "Beijing, China:",
    "age": 25,
    "name": "Alice"
}
Alice 25 Beijing, China:
```

显然，实现这样的转换逻辑，需要库使用者自行实现重载的`to_json`, `from_json`供nlohmann/json在做类型转换时调用。这种设计当然合理，但有个问题就在于：每次`Person`类有字段改动，开发者都得修改`to_json`和`from_json`。**那么，有没有什么办法可以自动获取`Person`的所有类成员变量，自动生成这一行行的字段填空代码呢？**

答案就是**反射**，而且，这次我们来实现一个**编译期反射**。

## 成员遍历：`iterate_members`

对于每个类来说，它都得定义个`iterate_members`可以遍历所有的成员，这个方法既可以定义在类外作为普通函数，也可以定义在类内，不论何种，我们先用trait大法封装下：

```cpp
namespace reflect {

// 主模板，实现体留空
template <class T>
struct reflect_trait {};

}  // namespace reflect

// 业务代码对Person类进行模板特化
namespace reflect {

template <>
struct reflect_trait<Person> {
  template <class Func>
  static constexpr void iterate_members(const Person& person, Func&& func) {
    func("name", person.name);
    func("age", person.age);
    func("address", person.address);
  }
};

}  // namespace reflect
```

有了`iterate_members`，就可以对JSON的操作做统一封装，无非就是在Class Type Object, JSON Object和字符串之间做转换：

```cpp
namespace reflect_json {

using json = nlohmann::json;

// 任意类型T obj转json obj
template <class T>
json serialize(const T& obj) {
  json root;
  // 传入的Func是个lambda函数，iterate_members会把成员变量的名字和值传给lambda执行
  // 如此就不需要写一行行的j["name"] = p.name;
  reflect::reflect_trait<T>::iterate_members(obj, [&](const char* key, auto& value) { root[key] = value; });
  return root;
}

// 与serialize同理
template <class T>
T deserialize(const json& j) {
  T obj;
  reflect::reflect_trait<T>::iterate_members(obj, [&](const char* key, auto& value) {
    if (j.count(key)) {
      value = j[key].get<typename std::decay<decltype(value)>::type>();
    }
  });
  return obj;
}

// json obj转字符串，json库备的原生能力
std::string to_json(const json& j, int indent = -1) { return j.dump(indent); }

// 字符串转json obj，json库必备的原生能力
json from_json(const std::string& s) { return json::parse(s); }

}  // namespace reflect_json
```

现在我们封装了`reflect_json`命名空间，可以对任意Class Type实现序列化/反序列化了，但要求这个`class`类型必须得业务自实现一个`reflect_trait`的模板特化。细心的读者会发现，到此为止我们只是对JSON接口与`iterate_members`能力做了解耦，而在特化的`iterate_members`实现中，我们还是得老老实实的写：

```cpp
    func("name", person.name);
    func("age", person.age);
    func("address", person.address);
```

对于业务来说，这是完全不能接受的：你这绕了一大圈，到头来还是得自己挨个一行行写成员处理，这不是脱裤子放屁吗？

我知道你很急，但你先别急，接下来，就是展现魔法的时刻。如果，只需要业务写一行代码，就可以自动生产出上面的模板特化代码，是不是就足够优雅了呢？

```cpp
struct Person {
  std::string name;
  int age;
  std::string address;
};

// 通过宏预编译期自动生成reflect_trait桩代码
REFLECT_EXTERNAL(Person, name, age, address)
```

这个`REFLECT_EXTERNAL`要怎么实现呢，其实也不难，我们先把自己实现的代码照搬过来：

```cpp
namespace reflect {

template <class T>
struct reflect_trait {};

#define REFLECT_EXTERNAL(Type, ...)                                       \
  template <>                                                             \
  struct reflect::reflect_trait<Type> {                                            \
    template <class Func>                                                 \
    static constexpr void iterate_members(const Type& obj, Func&& func) { \
      func("name", obj.name);                                             \
      func("age", obj.age);                                               \
      func("address", obj.address);                                       \
    }                                                                     \
  };

}  // namespace reflect
```

接下来，就得想办法处理`func("name", obj.name);`这些格格不入的东西了，怎么才能自动生成呢？显然这里的`name, age, address`已经通过宏的可变参数`__VA_ARGS__`传进来了，只是我们还没用上，假如我们可以从`__VA_ARGS__`里挨个取出来，那么就只需要生成多行：`func(#ARG, obj.ARG)`就行了。

```cpp
#define REFLECT_PER_MEMBER(x) \
    func(#x, obj.x);

#define REFLECT_EXTERNAL(Type, ...)                                       \
  template <>                                                             \
  struct reflect::reflect_trait<Type> {                                   \
    template <class Func>                                                 \
    static constexpr void iterate_members(const Type& obj, Func&& func) { \
      PP_FOREACH(REFLECT_PER_MEMBER, __VA_ARGS__)                         \
    }                                                                     \
  };
```

如何去遍历宏可变参数`__VA_ARGS__`，然后对每一个参数都“调用”某个宏呢？这里用到的是一个经典的`PP_FOREACH`实现手法：

```cpp
// 套娃是为了让宏能继续展开，防止中道崩殂，如果不套娃，a##b拼接成的宏不会进一步展开
#define PP_CONCAT_IMPL(a, b) a##b
#define PP_CONCAT(a, b) PP_CONCAT_IMPL(a, b)

// 由于不知道__VA_ARGS__的size，只好对每一种可能性都定义一个宏
#define PP_FOREACH_1(f, _1) f(_1)
#define PP_FOREACH_2(f, _1, _2) PP_FOREACH_1(f, _1) f(_2)
#define PP_FOREACH_3(f, _1, _2, _3) PP_FOREACH_2(f, _1, _2) f(_3)
// 更多参数的你自己搞个脚本生成吧 ...

// 预编译期计算__VA_ARGS__的size：本质上是通过反转占位法
#define PP_NARGS_IMPL(_1, _2, _3, N, ...) N
#define PP_NARGS(...) PP_NARGS_IMPL(__VA_ARGS__, 3, 2, 1)

// 比如对于上例，最终传递的就是PP_FOREARCH_3(REFLECT_PER_MEMBER, name, age, address)
// 进一步展开成func("name", obj.name);func("age", obj.age);func("address", obj.address);
#define PP_FOREACH(f, ...) PP_CONCAT(PP_FOREACH_, PP_NARGS(__VA_ARGS__))(f, __VA_ARGS__)
```


看看替换的结果是否符合预期：

![](2024-07-20-13-15-37.png)

没毛病！

> PP_FOREACH的手法源于boost/describe/pp_for_each.hpp，这里是简化后的实现。

### 类内也能定义`iterate_members`

稍加调整，也可以支持在类内定义`iterate_members`：

```cpp
// 这个时候由于iterate_members是类成员函数，所以直接用this->x访问成员就行了
#define REFLECT_PER_MEMBER_THIS(x) func(#x, this->x);

#define REFLECT(...)                                   \
  template <class Func>                                \
  constexpr void iterate_members(Func&& func) const { \
    PP_FOREACH(REFLECT_PER_MEMBER_THIS, __VA_ARGS__)   \
  }
```

业务代码可以这样写，显得更加内聚：

```cpp
struct Person {
  std::string name;
  int age;
  std::string address;

  REFLECT(name, age, address)
};
```

![](2024-07-20-14-39-30.png)

然后，再对`reflect_trait`做一下小小的升级，让它能够找到在类内定义的成员函数`iterate_members`：

```cpp
// 只需要主模板实现个默认的iterate_members函数，它会转发调用T类型的iterate_members成员函数
template <class T>
struct reflect_trait {
  template<class Func>
  static constexpr void iterate_members(const T &obj, Func&& func) {
    obj.template iterate_members(std::forward<Func>(func));
  }
};
```

如此，对于使用`REFLECT`宏在类内定义`iterate_members`成员函数的类型`T`，在json的`serialize`方法里，通过`reflect_trait`拿到的就是主模板里实现的`iterate_members`，而对于使用`REFLECT_EXTERNAL`在类外定义模板特化实现体的类`T`来说，`reflect_trait`拿到的是类外普通函数(传递`T`对象参数)。

## 嵌套与递归

到目前为止，我们的`reflect_json`在处理对象时，只考虑传递进来的是个基础的JSON对象型，比如我们的`Person`，其内都是平铺了JSON基础型(整数、字符串等)，然而，真实的业务代码类往往比较复杂，存在嵌套结构，比如我们对上面的`Person`做一下调整：

```cpp
struct Address {
  std::string country;
  std::string city;

  REFLECT(country, city)
};

struct Person {
  std::string name;
  int age;
  Address address;

  REFLECT(name, age, address)
};

int main() {
  Person person = {.name = "Alice", .age = 25, .address = {.country = "China", .city = "Shenzhen"}};

  auto bytes = reflect_json::to_json(reflect_json::serialize(person), 4);
  std::cout << bytes << "\n";

  return 0;
}
```

![](2024-07-20-14-56-42.png)

这个时候生成的反射桩代码，对于咱们的`serialize`就不灵了：

```shell
./nlohmann/json.hpp:20534:17: note: candidate function not viable: no known conversion from 'const Address' to 'basic_json<>' for 1st argument
    basic_json& operator=(basic_json other) noexcept (

```

显然，咱们没法把`Address`对象直接通过赋值的方式转成子对象塞到`json`对象中，这个过程显然要递归处理，`serialize`的写法应该像这样：

```cpp
template <class T>
json serialize(const T& obj) {
  json root;
  reflect::reflect_trait<T>::iterate_members(obj, [&](const char* key, auto& value) {
    // 递归处理每个成员Type，进而找到Address的iterate_members方法
    root[key] = serialize(value);
  });
  return root;
}
```

但这样写却又不够，因为我们需要分辨出哪些是基础类型，可以直接作为json对象塞进`root[key] = value`的，哪些又是对象类型，需要递归处理，

在C++20之前，我们需要通过函数模板重载+SFINAE大法来控制重载决议后最终选择某一个`serialize`，C++20之后可以使用Concept来处理：

```cpp
// 定义concept，判定是否是对象型
template<class T>
concept is_json_object = requires(T t) {
  t.iterate_members([](const char* key, auto& value) {});
};

// 基础型直接转
template<class T> requires (!is_json_object<T>)
json serialize(const T& obj) {
  return json(obj);
}

// 对象型需要递归
template <class T> requires (is_json_object<T>)
json serialize(const T& obj) {
  json root;
  reflect::reflect_trait<T>::iterate_members(const_cast<T&>(obj), [&](const char* key, auto& value) {
    root[key] = serialize(value);
  });
  return root;
}

template<class T> requires (!is_json_object<T>)
T deserialize(const json& j) {
  return j.get<T>();
}

template<class T> requires (is_json_object<T>)
T deserialize(const json& j) {
  T obj;
  reflect::reflect_trait<T>::iterate_members(obj, [&](const char* key, auto& value) {
    if (j.count(key)) {
      value = deserialize<std::decay_t<decltype(value)>>(j[key]);
    }
  });
  return obj;
}
```

### 数组的处理

除了对象结构外，JSON还有个麻烦的数组结构，上述的代码对于`std::vector<int>`类型的成员是可以正确处理的，比如：

```cpp
struct Address {
  std::string country;
  std::string city;
  std::vector<int> numbers;

  REFLECT(country, city, numbers)
};

int main() {
  Person person = {
      .name = "Alice", .age = 25, .address = {.country = "China", .city = "Shenzhen", .numbers = {1, 2, 3}}};

  auto bytes = reflect_json::to_json(reflect_json::serialize(person), 4);
  std::cout << bytes << "\n";

  Person person2 = reflect_json::deserialize<Person>(reflect_json::from_json(bytes));
  std::cout << person2.name << " " << person2.age << " " << person2.address.city << "\n";

  return 0;
}
```

```shell
qichencao@QICHENCAO-MB1  ~/workspace/static-reflect  ./main
{
    "address": {
        "city": "Shenzhen",
        "country": "China",
        "numbers": [
            1,
            2,
            3
        ]
    },
    "age": 25,
    "name": "Alice"
}
Alice 25 Shenzhen
```

这得益于nlohmann/json库可以对`std::vector<int>`这种基础型数组直接转json对象，但如果`vector`中类型是个对象型的话，上面的代码就行不通了。

我们得想办法对数组型做一下处理，而不是简单的`return json(obj);`：

```cpp
// 是否有一个iterate_members([](const char* key, auto& value) {})成员函数
template <class T>
concept is_json_object = requires(T t) {
  t.iterate_members([](const char* key, auto& value) {});
};

// 是不是vector类型
template <class T>
concept is_vector = std::is_same_v<std::vector<typename T::value_type>, T>;

// 基础型直接转
template <class T>
requires(!is_json_object<T> && !is_vector<T>) json serialize(const T& obj) { return json(obj); }

// 前置声明：让第二个重载能看见第三个重载模板，不然在实例化时不会进入候选集
template <class T>
requires(is_json_object<T>) json serialize(const T& obj);

// 数组型要特化处理
template <class T>
requires(is_vector<T>) json serialize(const T& obj) {
  json root;
  for (const auto& value : obj) {
    root.push_back(reflect_json::serialize(value));
  }
  return root;
}

// 对象型递归处理
template <class T>
requires(is_json_object<T>) json serialize(const T& obj) {
  json root;
  reflect::reflect_trait<T>::iterate_members(
      const_cast<T&>(obj), [&](const char* key, auto& value) { root[key] = reflect_json::serialize(value); });
  return root;
}

// 基础型
template <class T>
requires(!is_json_object<T> && !is_vector<T>) T deserialize(const json& j) { return j.get<T>(); }

// 前置声明：让第二个重载能看见第三个重载模板，不然在实例化时不会进入候选集
template <class T>
requires(is_json_object<T>) T deserialize(const json& j);

// 数组型
template <class T>
requires(is_vector<T>) T deserialize(const json& j) {
  T obj;
  for (const auto& value : j) {
    obj.push_back(deserialize<typename T::value_type>(value));
  }
  return obj;
}

// 对象型
template <class T>
requires(is_json_object<T>) T deserialize(const json& j) {
  T obj;
  reflect::reflect_trait<T>::iterate_members(obj, [&](const char* key, auto& value) {
    if (j.count(key)) {
      value = deserialize<std::decay_t<decltype(value)>>(j[key]);
    }
  });
  return obj;
}
```

到此，我们就可以对数组和对象进行随意嵌套了：

```cpp
struct Address {
  std::string country;
  std::string city;

  REFLECT(country, city)
};

struct Person {
  std::string name;
  int age;
  std::vector<Address> address;

  REFLECT(name, age, address)
};


int main() {
  Person person = {.name = "Alice",
                   .age = 25,
                   .address = {{.country = "China", .city = "Shenzhen"}, {.country = "China", .city = "Beijing"}}};

  auto bytes = reflect_json::to_json(reflect_json::serialize(person), 4);
  std::cout << bytes << "\n";

  Person person2 = reflect_json::deserialize<Person>(reflect_json::from_json(bytes));
  std::cout << person2.name << " " << person2.age << " address size:" <<  person2.address.size() << "\n";

  return 0;
}
```

```shell
qichencao@QICHENCAO-MB1  ~/workspace/static-reflect  ./main
{
    "address": [
        {
            "city": "Shenzhen",
            "country": "China"
        },
        {
            "city": "Beijing",
            "country": "China"
        }
    ],
    "age": 25,
    "name": "Alice"
}
Alice 25 address size:2
```

> 实际上除了对`std::vector`做特化处理外，还应该对nlohmann/json所默认支持的容器型做特化逻辑处理，比如它支持标准库`std::map`作为天然的对象型，那么我们可以写一个is_map的concept，然后重载serialize和deserialize的函数模板，这里限于篇幅，就不展开了。

### 对`REFLECT_EXTERNAL`的修正

上面的代码在做`serialize`,`deserialize`的拆分时，主要是根据`has_iterate_members`这个concept作为是否是对象类型的判定，细心的读者应该已经发现了，这对于类外使用的`REFLECT_EXTERNAL`来说是致命的，我们并没有照顾到使用这种方式的`class`。

> 当然了，一些读者会认为都有`REFLECT`了，既生瑜何生亮啊。但实际上，`REFLECT_EXTERNAL`更加灵活，毕竟不是所有的`class`你都能改。

`REFLECT_EXTERNAL`和`REFLECT`的本质区别在于，前者是对类型`T`生成了`reflect_trait`的特化实例，后者是生成一个类内成员模板函数。那么我们可以只需要改动一下对对象类型的判定条件即可，再追加一组对`reflect_trait`的某个特化是否有实例化的判断即可：

```cpp
// 为了方便做SFINAE，我们直接定义个_is_json_object静态成员常量
#define REFLECT_EXTERNAL(Type, ...)                                 \
  template <>                                                       \
  struct reflect::reflect_trait<Type> {                             \
    static constexpr bool _is_json_object = true;                   \
                                                                    \
    template <class Func>                                           \
    static constexpr void iterate_members(Type& obj, Func&& func) { \
      PP_FOREACH(REFLECT_PER_MEMBER, __VA_ARGS__)                   \
    }                                                               \
  };




// 辅助模板，用于检测特化实例，SFINAE fallback, 默认继承std::false_type
template <typename, typename = std::void_t<>>
struct _has_external_iterate_members : std::false_type {};

// 模板参数推导时，如果成功，就实例化这个更特殊的版本，继承std::true_type
template <typename T>
struct _has_external_iterate_members<T, std::void_t<decltype(reflect::reflect_trait<T>::_is_json_object)>> : std::true_type {};

template <typename T>
concept has_external_iterate_members = _has_external_iterate_members<T>::value;




// 是否有一个iterate_members([](const char* key, auto& value) {})成员函数
template <class T>
concept has_iterate_members = requires(T t) {
  t.iterate_members([](const char* key, auto& value) {});
};

// 把这两个条件组和一下，变成一个新concept，它用来整体判定是否是对象型
template <class T>
concept is_json_object = has_iterate_members<T> || has_external_iterate_members<T>;
```

到此，即使我们这样编写业务代码：

```cpp
struct Address {
  std::string country;
  std::string city;
};
REFLECT_EXTERNAL(Address, country, city)

struct Person {
  std::string name;
  int age;
  std::vector<Address> address;

  REFLECT(name, age, address)
};

int main() {
  Person person = {.name = "Alice",
                   .age = 25,
                   .address = {{.country = "China", .city = "Shenzhen"}, {.country = "China", .city = "Beijing"}}};

  auto bytes = reflect_json::to_json(reflect_json::serialize(person), 4);
  std::cout << bytes << "\n";

  Person person2 = reflect_json::deserialize<Person>(reflect_json::from_json(bytes));
  std::cout << person2.name << " " << person2.age << " address size:" <<  person2.address.size() << "\n";

  return 0;
}
```

代码依然可以成功运行：

```shell
qichencao@QICHENCAO-MB1  ~/workspace/static-reflect  ./main 
{
    "address": [
        {
            "city": "Shenzhen",
            "country": "China"
        },
        {
            "city": "Beijing",
            "country": "China"
        }
    ],
    "age": 25,
    "name": "Alice"
}
Alice 25 address size:2
```

## 【选修】没有concept时期的妥协

自C++11开始，标准就一直在扩充模板元编程的能力，陆陆续续纳入了可变模板参数、trait、if constexpr、concept等。C++20所引入的concept极大的简化了模板元编程的代码（~~你看看人家Rust含着金汤匙Trait出生~~），在没有concept的时代，我们只能通过传统的SFINAE机制来做各种type trait, value trait，去基于tag做元函数转发，因此，在这一时期有相当多的代码，它们编写得相当晦涩，通过各种迂回的手法去达成目标（三元运算符、-> decltype等等）。

> 上古时代更是如此，在模板能力还不完备的时代，元编程就已经被玩坏了，大佬们为了达成目标不得不滥用`enum`, `sizeof(int[0])`等机制，导致代码男默女泪。后之视今，亦如今之视昔（~~C++一直以来就是个残废语言~~）。

我们把上面的代码改一改，不用concept，来实现一遍:

