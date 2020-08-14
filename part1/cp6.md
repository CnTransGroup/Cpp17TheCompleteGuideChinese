# 第六章 Lambda扩展
C++11引入了lambda，C++14引入了泛型lambda，这是一个成功的故事。lambda允许我们将功能指定为参数，这让定制函数的行为变得更加容易。

C++ 17进一步改进，允许lambda用在更多的地方。

## 6.1 constexpr lambda
自C++17后，只要可能，lambda就隐式地用constexpr修饰。也就是说，任何lambda都可以用于编译时上下文，前提是它使用的特性对编译时上下文有效（例如，仅字符串字面值，无静态变量，无virutal变量，无try/catch，无new/delete）。

举个例子，你可以传一个值给lambda，然后用计算的结果作为编译时的`std::array<>`大小：
```cpp
auto squared = [](auto val) { // implicitly constexpr since C++17
  return val*val;
};
std::array<int,squared(5)> a; // OK since C++17 => std::array<int,25>
```
如果在不允许constexpr的上下文使用这个特性就不行，但是你仍然可以在运行时傻姑娘上下文使用lambda：
```cpp
he lambda in run-time contexts:
auto squared2 = [](auto val) {      // implicitly constexpr since C++17
  static int calls = 0;             // OK, but disables lambda for constexpr contexts
  ...
  return val*val;
};
std::array<int,squared2(5)> a;      // ERROR: static variable in compile-time context
std::cout << squared2(5) << '\n';   // OK
```
要知道是否一个lambda在一个编译时上下文有效，你可以将它声明为constexpr：
```cpp
auto squared3 = [](auto val) constexpr {    // OK since C++17
  return val*val;
};
```
还可以指定返回类型，语法如下：
```cpp
auto squared3i = [](int val) constexpr -> int { // OK since C++17
  return val*val;
};
```
constexpr对于函数的一般规则仍然有效：如果lambda在运行时上下文中使用，相应的功能在运行时执行。

然而，在不允许编译时上下文的地方使用constexpr lambda会得到一个编译时错误：
```cpp
auto squared4 = [](auto val) constexpr {
  static int calls=0; // ERROR: static variable in compile-time context
  ...
  return val*val;
};
```
如果lambda式显式或隐式的constexpr，那么函数调用操作符也会是constexpr。换句话说，下面的定义：
```cpp
auto squared = [](auto val) { // implicitly constexpr since C++17
  return val*val;
};
```
会转换为闭包类型：
```cpp
class CompilerSpecificName {
  public:
    ...
    template<typename T>
    constexpr auto operator() (T val) const {
      return val*val;
    }
};
```
生成的闭包类型的函数调用操作符是自动附加constexpr的。在C++17中，如果lambda显式定义为constexpr或者隐式定义为constexpr（就像这个例子），那么生成的函数调用运算符也会是constexpr。

## 6.2 传递this的拷贝到lambda
当在成员函数中使用lambda时，你不能隐式的访问调用这个成员函数的对象的成员。也就是说，在lambda内部，如果不捕获this，那么你不能使用这个对象的成员：
```cpp
class C {
private:
    std::string name;
public:
    ...
    void foo() {
        auto l1 = [] { std::cout << name << '\n'; }; // ERROR
        auto l2 = [] { std::cout << this->name << '\n'; }; // ERROR
        ...
    }
};
```
C++11和C++14中可以传this引用或者传this值：
```cpp
class C {
private:
    std::string name;
public:
    ...
    void foo() {
        auto l1 = [this] { std::cout << name << '\n'; }; // OK
        auto l2 = [=] { std::cout << name << '\n'; }; // OK
        auto l3 = [&] { std::cout << name << '\n'; }; // OK
        ...
    }
};
```
然而，问题是即使是传递this的值，其底层捕获的仍然是引自对象（即只有*指针*被拷贝）。如果lambda的生命周期超过了对象的生命周期，这就会出现问题。一个重要的例子是当用lambda为新线程定义task，它应该使用对象的拷贝来避免任何并发或者生命周期问题。另一个原因可能只是传递一个对象的副本当前状态。

C++14有一个临时的解决方案，但是它读起来不好，工作起来也不好：
```cpp
class C {
private:
    std::string name;
public:
    ...
    void foo() {
        auto l1 = [thisCopy=*this] { std::cout << thisCopy.name << '\n'; };
        ...
    }
};
```
举个例子，就算使用`=`或`&`捕获了对象，开发者仍然可能不小心用到`this`：
```cpp
auto l1 = [&, thisCopy=*this] {
            thisCopy.name = "new name";
            std::cout << name << '\n'; // OOPS: still the old name
};
```
C++17开始，你可以显式地通过`*this`说明你想捕获当前对象的复制：
```cpp
class C {
private:
    std::string name;
public:
    ...
    void foo() {
        auto l1 = [*this] { std::cout << name << '\n'; };
        ...
    }
};
```
捕获`*this`意味着当前对象的复制传递到了lambda。

在捕获了`*this`的情况下你仍然可以捕获其他this，只要没有与其他的发生冲突：
```cpp
auto l2 = [&, *this] { ... };     // OK
auto l3 = [this, *this] { ... };  // ERROR
```
这里一个完整的例子：
```cpp
// lang/lambdathis.cpp
#include <iostream>
#include <string>
#include <thread>

class Data {
private:
    std::string name;
public:
    Data(const std::string& s) : name(s) {
    }
    auto startThreadWithCopyOfThis() const {
        // start and return new thread using this after 3 seconds:
        using namespace std::literals;
        std::thread t([*this] {
            std::this_thread::sleep_for(3s);
            std::cout << name << '\n';
        });
        return t;
    }
};

int main()
{
    std::thread t;
    {
        Data d{"c1"};
        t = d.startThreadWithCopyOfThis();
    } // d is no longer valid
    t.join();
}
```
lambda用`*this`获取对象拷贝，即d。因此，即便是d的析构函数被调用后线程再使用传递的对象也没有问题。

如果我们使用`[this],[=]`或`[&]`捕获this，线程会产生未定义行为，因为在lambda打印name时，lambda使用的是已经析构后的对象的成员。

## 6.3 捕获引用
通过使用新的utility库函数，你现在可以**捕获const对象引用**。


## 6.4 后记
constexpr最初由 Faisal Vali, Ville Voutilainen和Gabriel Dos Reis在[https://wg21.link/n4487](https://wg21.link/n4487)中提出。最后这个特性的公认措辞是由Faisal Vali, Jens
Maurer和Richard Smith在[https://wg21.link/p0170r1](https://wg21.link/p0170r1)中给出。

捕获`*this`最初由H. Carter Edwards, Christian Trott, Hal Finkel, Jim Reus, Robin Maffeo和Ben Sander在[https://wg21.link/p0018r0](https://wg21.link/p0018r0)中提出。最后这个特性的公认措辞是由 H. Carter Edwards, Daveed Vandevoorde, Christian Trott, Hal Finkel,
Jim Reus, Robin Maffeo和Ben Sander在[https://wg21.link/p0180r3](https://wg21.link/p0180r3)中给出。
