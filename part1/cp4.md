# 第四章 聚合扩展

C++中有一种初始化对象的方式叫做聚合初始化（aggregate initialization），它允许用花括号聚集多个值来初始化。
```cpp
struct Data {
  std::string name;
  double value;
};

Data x{"test1", 6.778};
```
从C++17开始，聚合还支持带基类的数据结构，所以下面这种数据结构用列表初始化也是允许的：
```cpp
struct MoreData : Data {
  bool done;
};
MoreData y{{"test1", 6.778}, false};
```
正如你看到的，聚合初始化现在支持嵌套的花括号传给基类的成员来初始化。

对于带有成员的子对象的初始化，如果基类或子对象只有一个值，则可以跳过嵌套的大括号：
```cpp
MoreData y{"test1", 6.778, false};
```

## 4.1 扩展聚合初始化的动机
如果没有这项特性的话，继承一个类之后就不能使用聚合初始化了，需要你为新类定义一个构造函数：
```cpp
struct Cpp14Data : Data {
  bool done;
  Cpp14Data (const std::string& s, double d, bool b)
      : Data{s,d}, done{b} {
  }
};
Cpp14Data y{"test1", 6.778, false};
```
现在，有了这个特性我们可以自由的使用嵌套的花括号，如果只传递一个值还可以省略它：
```cpp
MoreData x{{"test1", 6.778}, false}; // OK since C++17
MoreData y{"test1", 6.778, false}; // OK
```
注意，因为它现在是聚合体，其它初始化方式也是可以的：
```cpp
MoreData u; // OOPS: value/done are uninitialized
MoreData z{}; // OK: value/done have values 0/false
```
如果这个看起来太危险了，你还是最好提供一个构造函数。


