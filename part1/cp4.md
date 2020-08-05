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

## 4.2 使用扩展的聚合初始化
关于这个特性的常见用法是列表初始化一个C风格的数据结构，该数据结构继承自一个类，然后添加了一些数据成员或者操作。比如：
```cpp
struct Data {
  const char* name;
  double value;
};
struct PData : Data {
  bool critical;
  void print() const {
    std::cout << ✬[✬ << name << ✬,✬ << value << "]\n"; }
};

PData y{{"test1", 6.778}, false};
y.print();
```
这里里面的花括号会传递给基类Data的数据成员。

你可以跳过一些初始值。这种情况下这些元素是零值初始化（zero initalized）（调用默认构造函数或者将基本数据类型初始化为0，false或者nullptr）。比如：
```cpp
PData a{};          // zero-initialize all elements
PData b{{"msg"}};   // same as {{"msg",0.0},false}
PData c{{}, true};  // same as {{nullptr,0.0},true}
PData d;            // values of fundamental types are unspecified
```
注意使用空的花括号和不使用花括号的区别。
+ a零值初始化所有成员，所以name被默认构造，double value被初始化为0.0，bool flag被初始化为false。
+ d只调用name的默认构造函数。所有其它的成员都没用被初始化，所以值是未指定的（unspecified）。

你也可以继承非聚合体来创建一个聚合体。比如：
```cpp
struct MyString : std::string {
  void print() const {
    if (empty()) {
      std::cout << "<undefined>\n"; }
    else {
      std::cout << c_str() << '\n'; } }
};

MyString x{{"hello"}};
MyString y{"world"};
```
甚至还可以继承多个非聚合体：
```cpp
template<typename T>
struct D : std::string, std::complex<T>
{
  std::string data;
};
```
然后使用下面的代码初始化它们：
```cpp
D<float> s{{"hello"}, {4.5,6.7}, "world"};        // OK since C++17
D<float> t{"hello", {4.5, 6.7}, "world"};         // OK since C++17
std::cout << s.data;                              // outputs: ”world”
std::cout << static_cast<std::string>(s);         // outputs: ”hello”
std::cout << static_cast<std::complex<float>>(s); // outputs: (4.5,6.7)
```
内部花括号的值（initializer_lists）会传递给基类，其传递顺序遵循基类声明的顺序。

这项新特性还有助于用很少的代码定义**lambdas重载**。

## 4.3 聚合体定义
总结一下，C++17的聚合体（aggregate）定义如下：
+ 是个数组
+ 或者是个类类型（class，struct，union），其中
  + 没有用户声明的构造函数或者explicit构造函数
  + 没有使用using声明继承的构造函数
  + 没有private或者protected的非static数据成员
  + 没有virtual函数
  + 没有virtual，private或者protected基类

为了让聚合体可以使用，还要求聚合体没有private或者protected基类成员或者构造函数在初始化的时候使用。

C++17还引入了一种新的type trait即`is_aggregate<>`来检查一个类型是否是聚合体：
```cpp
template<typename T>
struct D : std::string, std::complex<T> {
  std::string data;
};
D<float> s{{"hello"}, {4.5,6.7}, "world"}; // OK since C++17
std::cout << std::is_aggregate<decltype(s)>::value; // outputs: 1 (true)
```

## 4.4 向后不兼容
注意，下面示例中的代码将不再能通过编译：
```cpp
// lang/aggr14.cpp
struct Derived;
struct Base {
  friend struct Derived;
private:
  Base() {
  }
};
struct Derived : Base {
};
int main()
{
  Derived d1{}; // ERROR since C++17
  Derived d2; // still OK (but might not initialize)
}
```
C++17之前，Derived不是一个聚合体，所以：
```cpp
Derived d1{};
```
调用Derived隐式定义的默认构造函数，它默认调用基类Base的默认构造函数。虽然基类的默认构造函数是private，但是通过子类的默认构造函数调用它是有效的，因为子类被声明为一个friend类。

C++17开始，Derived是一个聚合体，没有隐式的默认构造函数。所以这个初始化被认为是聚合初始化，聚合初始化不允许调用基类的默认构造函数。不管基类是不是friend都不行。

## 4.5 后记
内联变量最初由Oleg Smolsky在[https://wg21.link/n4404](https://wg21.link/n4404)中提出。最后这个特性的公认措辞是由Oleg Smolsky在[ https://wg21.link/p0017r1](https://wg21.link/p0017r1)中给出。

新的type trait即`std::is_aggregate<>`最初作为美国国家机构对C++ 17标准化的评论而引入。（参见[https://wg21.link/lwg2911](https://wg21.link/lwg2911)）