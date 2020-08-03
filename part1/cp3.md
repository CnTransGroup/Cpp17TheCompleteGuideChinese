# 第三章 内联变量
C++的一个优点是它支持header-only（译注：即只有头文件）的库。然而，截止C++17，header-only的库也不能有全局变量或者对象出现。

C++17后，你可以在头文件中使用inline定义变量，如果这个变量被多个翻译单元（translation unit）使用，它们都会指向相同对象：
```cpp
class MyClass {
  static inline std::string name = ""; // OK since C++17
  ...
};
inline MyClass myGlobalObj; // OK even if included/defined by multiple CPP files
```

## 3.1 内联变量的动机
C++不允许在class内部初始化非const静态成员：
```cpp
class MyClass {
  static std::string name = "";  // Compile-Time ERROR
  ...
};
```
在class外面定义这个变量定义这个变量，且变量定义是在头文件中，多个CPP文件包含它，仍然会引发错误：
```cpp
class MyClass {
  static std::string name; // OK
  ...
};
MyClass::name = ""; // Link ERROR if included by multiple CPP files
```
根据一处定义规则（one definition 入了，ODR），每个翻译单元只能定义变量最多一次。

即便有预处理保护（译注：也叫头文件保护，header guard）也没有用：
```cpp
#ifndef MYHEADER_HPP
#define MYHEADER_HPP
class MyClass {
  static std::string name; // OK
  ...
};
MyClass.name = ""; // Link ERROR if included by multiple CPP files
#endif
```
不是因为头文件可能被包含多次，问题是两个不同的CPP如果都包含这个头文件，那么`MyClass.name`可能定义两次。

同样的原因，如果你在头文件中定义一个变量，你会得到一个链接时错误：
```cpp
class MyClass {
  ...
};
MyClass myGlobalObject; // Link ERROR if included by multiple CPP files
```
#### 临时解决方案
这里有一些临时的应对措施：
+ 你可以在class/struct内初始化一个static const整型数据成员：
```cpp
class MyClass {
  static const bool trace = false;
  ...
};
```
+ 你可以定义一个返回局部static对象的内联函数：
```cpp
inline std::string getName() {
  static std::string name = "initial value";
  return name;
}
```
+ 你可以定义一个static成员函数返回它的值：
```cpp
std::string getMyGlobalObject() {
  static std::string myGlobalObject = "initial value";
  return myGlobalObject;
}
```
+ 你可以使用变量模板（C++14及以后）：
```cpp
template<typename T = std::string>
T myGlobalObject = "initial value";
```
+ 你可以继承一个包含static成员的类模板：
```cpp
template<typename Dummy>
class MyClassStatics
{
  static std::string name;
};
template<typename Dummy>
std::string MyClassStatics<Dummy>::name = "initial value";
class MyClass : public MyClassStatics<void> {
  ...
};
```
但是这些方法都有不小的负载，可读性也比较差，想要使用全局变量也比较困难。除此之外，全局变量的初始化可能会推迟到它第一次使用的时候，这使得应用程序不能在启动的时候把对象初始化好。（比如用一个对象监控进程）。

## 3.2 使用内联变量
现在，有了inline，你可以在头文件中定义一个全局可用的变量，它可以被多个CPP文件包含：
```cpp
class MyClass {
  static inline std::string name = ""; // OK since C++17
  ...
};
inline MyClass myGlobalObj; // OK even if included/defined by multiple CPP files
```
初始化发生在第一个包含该头文件的翻译单元。

形式化来说，在变量前使用inline和将函数声明为inline有相同的语义：
+ 如果每个定义都是相同的，那么它可以在多个翻译单元定义
+ 它必须在使用它的每个翻译单元中定义

两者都是通过包含来自同一头文件的定义来实现的。最终程序的行为就像是只有一个变量。

你甚至可以在头文件中定义原子类型的变量：
```cpp
inline std::atomic<bool> ready{false};
```
注意，对于`std::atomic`，通常在定义它的时候你还得初始化它。

这意味着，你仍然必须保证在你初始化它之前类型是完全的（complete）。比如，如果一个struct或者class有一个static成员，类型是自身，那么该成员只能在该类型被声明后才能使用。
```cpp
struct MyType {
  int value;
  MyType(int i) : value{i} {
  }
// one static object to hold the maximum value of this type:
  static MyType max; // can only be declared here
  ...
};
inline MyType MyType::max{0};
```
参见另一个使用内联变量的例子，它会**使用头文件跟踪所有new调用**。

## 3.3 constexpr隐式包含inline
对于static数据成员，constexpr现在隐式包含inline的语义，所以下面的声明在C++17后会**定义**static数据成员n：
```cpp
struct D {
  static constexpr int n = 5; // C++11/C++14: declaration
                              // since C++17: definition
};
```
换句话说，它与下面的代码一样：
```cpp
struct D {
  inline static constexpr int n = 5;
};
```
在C++17之前,有时候你也可以只声明不定义。考虑下面的声明：
```cpp
struct D {
  static constexpr int n = 5;
};
```
如果不需要`D::n`的定义，这就足够了，例如，`D::n`只通过值传递的话：
```cpp
std::cout << D::n; // OK (ostream::operator<<(int) gets D::n by value)
```
如果`D::n`是传引用到非内联函数，并且/或者函数调用没有优化，那么就是无效的。比如：
```cpp
int inc(const int& i);
std::cout << inc(D::n); // usually an ERROR
```
这段代码违背了一处定义规则（ODR）。当使用带优化的编译器构建时，它可能正常工作，或者抛出链接时错误指出缺少定义。当使用不带优化的编译器时，几乎可以确定这段代码会由于缺少`D::n`的定义而拒绝编译：

因此，在C++17前，你不得不在相同的翻译单元定义`D::n`：
```cpp
constexpr int D::n; // C++11/C++14: definition
                    // since C++17: redundant declaration (deprecated)
```
当使用C++17构建，在class中的声明本身就是一个定义，所以这段代码就算没有前面的定义也是有效的。前面的定义也是可以的，但是已经废弃。

## 3.4 内联变量和thread_local
使用thread_local你可以让每个线程拥有一个内联变量：
```cpp
struct ThreadData {
  inline static thread_local std::string name; // unique name per thread
...
};

inline thread_local std::vector<std::string> cache; // one cache per thread
```
为了演示一个完整的例子，考虑下面的头文件：
```cpp
// lang/inlinethreadlocal.hpp
#include <string>
#include <iostream>

struct MyData {
  inline static std::string gName = "global"; // unique in program
  inline static thread_local std::string tName = "tls"; // unique per thread
  std::string lName = "local"; // for each object
  ...
  void print(const std::string& msg) const {
    std::cout << msg << '\n';
    std::cout << "- gName: " << gName << '\n';
    std::cout << "- tName: " << tName << '\n';
    std::cout << "- lName: " << lName << '\n'; }
};

inline thread_local MyData myThreadData; // one object per thread
```
你可以在有`main()`的翻译单元使用它：
```cpp
// lang/inlinethreadlocal1.cpp
#include "inlinethreadlocal.hpp" 
#include <thread>
void foo();

int main()
{
  myThreadData.print("main() begin:");
  myThreadData.gName = "thread1 name";
  myThreadData.tName = "thread1 name";
  myThreadData.lName = "thread1 name";
  myThreadData.print("main() later:");
  std::thread t(foo);
  t.join();
  myThreadData.print("main() end:");
}
```
你可以在另一个定义`foo()`的翻译单元使用头文件，其中`foo()`被不同的线程调用：
```cpp
// lang/inlinethreadlocal2.cpp
#include "inlinethreadlocal.hpp"

void foo()
{
  myThreadData.print("foo() begin:");
  myThreadData.gName = "thread2 name";
  myThreadData.tName = "thread2 name";
  myThreadData.lName = "thread2 name";
  myThreadData.print("foo() end:");
}
```
程序的输出如下：
```text
main() begin:
- gName: global
- tName: tls
- lName: local
main() later:
- gName: thread1 name
- tName: thread1 name
- lName: thread1 name
foo() begin:
- gName: thread1 name
- tName: tls
- lName: local
foo() end:
- gName: thread2 name
- tName: thread2 name
- lName: thread2 name
main() end:
- gName: thread2 name
- tName: thread1 name
- lName: thread1 name
```

## 3.5 后记
David Krauss的文档[https://wg21.link/n4147]( https://wg21.link/n4147)是内联变量产生的动机。内联变量最初由Hal Finkel和Richard Smith在[https://wg21.link/n4424](https://wg21.link/n4424)中提出。最后这个特性的公认措辞是由Hal Finkel和Richard Smith在[ https://wg21.link/p0386r2]( https://wg21.link/p0386r2)中给出。