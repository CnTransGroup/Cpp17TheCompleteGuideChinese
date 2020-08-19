# 第八章 其他语言特性
有一些小的C++核心语言特性改动，它们会在本章描述。

## 8.1 嵌套命名空间
最早这个提案是在2003年提出的，C++标准委员会现在终于最终接受了它：
```cpp
namespace A::B::C {
  ...
}
```
它等价于:
```cpp
namespace A {
  namespace B {
    namespace C {
      ...
    }
  }
}
```
嵌套的inline命名空间还不支持。这是因为如果用了inline就不知道到底inline是针对最后一个还是对所有命名空间使用。

## 8.2 定于表达式求值顺序
很多代码库和C++书籍包含的代码首先给出符合直觉的假设，然后代码上看起来是有效的，但是严格来讲，这些代码可能产生未定义行为。一个例子是使用寻找并替换子字符串：
```cpp
std::string s = "I heard it even works if you don't believe";
s.replace(0,8,"").replace(s.find("even"),4,"sometimes")
                 .replace(s.find("you don✬t"),9,"I");
```
直觉上看起来这段代码是有效的，它将前8个字符替换为空，“even”替换为“sometimes”，将“you don't”替换为“I”：
```cpp
it sometimes works if I believe
```
然而，在C++17之前，结果是不保证的，因为，虽然`find()`调用返回从何处开始替换，但是当整个语句执行并且在结果被需要之前，这个调用可能在任何时候执行。实际上，所有`find()`，即计算待替换的起始索引，都可能在任何替换发生前被执行，因此结果是：
```cpp
it sometimes works if I believe
```
其他结果也是可能的：
```cpp
it sometimes workIdon’t believe
it even worsometiIdon’t believe
it even worsometimesf youIlieve
```
另一个例子是使用输出运算符来打印计算后的表达式的值：
```cpp
std::cout << f() << g() << h();
```
通常的假设是`f()`在`g()`之前被调用，两者又都在`h()`之前被调用。然而，这个假设是错误的。`f()`，`g()`和`h()`可以按任意顺序调用，这可能导致一些奇怪的，甚至是糟糕的结果，尤其是当这些调用互相依赖时

具体来说，考虑下面的例子，在C++17之前，这段代码会产生未定义行为：
```cpp
i = 0;
std::cout << ++i << ' ' << --i << '\n';
```
在C++17之前，他可能输出`1 0`，也可能输出`0 -1`，甚至是`0 0`。不管i是int还是用户定义的类型，都可能这样。（对于基本类型，一些编译器至少会warning这个问题）。

要修复这个未定义行为，一些运算符/操作符的求值被挑战，因此现在它们有确定的求值顺序：
+ 对于
  + `e1 [ e2 ]`
  + `e1 . e2`
  + `e1 .* e2`
  + `e1 ->* e2`
  + `e1 << e2`
  + `e1 >> e2`
e1保证在e2之前求值，它们的求值顺序是从左至右。

然而，相同函数的不同实参的求值顺序仍然是未定义的。即：
```cpp
e1.f(a1,a2,a3)
```
e1保证在a1 a2 a3之前求值。但是a1 a2 a3的求职顺序仍然是未定义的。
+ 所有赋值运算符
  + `e2 = e1`
  + `e2 += e1`
  + `e2 *= e1`
  + `...`
右手边的e1会先于左手变的e2被求值。
+ 最后，new表达式中
  + `new Type(e)`
分配行为保证在e之前求值，初始化新的值保证在任何使用初始化的值之前被求值。

上述所有保证对基本类型和用户定义类型都有效。

这样做的效果是，C++17后：
```cpp
std::string s = "I heard it even works if you don't believe";
s.replace(0,8,"").replace(s.find("even"),4,"sometimes")
                 .replace(s.find("you don✬t"),9,"I");
```
保证会改变s的值，变成：
```
it always works if you use C++17
```
因此，每个`find()`之前的替换都会在`find()`之前被求值。

另一个结果是，下面的语句
```cpp
i = 0;
std::cout << ++i << ' ' << --i << '\n';
```
其输出保证是`1 0`。

然而，对于其他大多数运算符而言，求值顺序仍然未定义。举个例子：
```cpp
i = i++ + i; // still undefined behavior
```
这里右手变的i可能在递增之前或者递增之后传递给左手变。

另一个使用new表达式求值顺序的例子是**在传值之前插入空格的函数**。

#### 向后兼容
新的求值顺序的保证可能影响既有程序的输出。这不是理论上可能，是真的。考虑下面的代码：
```cpp
#include <iostream>
#include <vector>

void print10elems(const std::vector<int>& v) 
{
    for (int i=0; i<10; ++i) {
        std::cout << "value: " << v.at(i) << '\n';
    }
}

int main()
{
    try {
        std::vector<int> vec{7, 14, 21, 28};
        print10elems(vec);
    }
    catch (const std::exception& e) { // handle standard exception
        std::cerr << "EXCEPTION: " << e.what() << '\n'; }
    catch (...) { // handle any other exception
        std::cerr << "EXCEPTION of unknown type\n"; 
    } 
}
```
因为这里的`vector<>`只有4个元素，程序会在`print10elems()`的循环中，调用`at()`时遇到无效索引抛出异常：
```cpp
std::cout << "value: " << v.at(i) << "\n";
```
在C++17之前，可能输出：
```
value: 7
value: 14
value: 21
value: 28
EXCEPTION: ...
```
因为`at()`可以在"value "输出之前求值，所以对于错误的索引可能直接跳过不输出"value "。

自C++17之后，保证输出：
```
value: 7
value: 14
value: 21
value: 28
value: EXCEPTION: ...
```
因为"value "一定在`at()`调用之前执行。

## 8.3 宽松的基于整数的枚举初始化
对于有固定基本类型的枚举，C++17允许你使用带数值的列表初始化。
```cpp
// unscoped enum with underlying type:
enum MyInt : char { };
MyInt i1{42};     // C++17 OK (C++17之前错误)
MyInt i2 = 42;    // 仍然错误
MyInt i3(42);     // 仍然错误
MyInt i4 = {42};  // 仍然错误

enum class Weekday { mon, tue, wed, thu, fri, sat, sun };
Weekday s1{0};    // C++17 OK (C++17之前错误)
Weekday s2 = 0;   // 仍然错误
Weekday s3(0);    // 仍然错误
Weekday s4 = {0}; // 仍然错误
```
类似的，如果Weekday有基本类型：
```cpp
// scoped enum with specified underlying type:
enum class Weekday : char { mon, tue, wed, thu, fri, sat, sun };
Weekday s1{0};    // C++17 OK (C++17之前错误)
Weekday s2 = 0;   // 仍然错误
Weekday s3(0);    // 仍然错误
Weekday s4 = {0}; // 仍然错误
```
对于没有指定基本类型的未限域枚举（不带class的enum），你仍然不能使用带数值的列表初始化：
```cpp
enum Flag { bit1=1, bit2=2, bit3=4 };
Flag f1{0}; // 仍然错误
```
注意，列表初始化还是不允许变窄（narrowing），因此你不能传递浮点值：
```cpp
enum MyInt : char { };
MyInt i5{42.2}; // 仍然错误
```
之所以提出这个特性，是想实现一种技巧，即基于原有的整数类型定义另一种新的枚举类型，就像上面MyInt一样。

实际上，C++17的标准库中的`std::byte`也提供这个功能，它直接使用了这个特性。

## 8.4 修复带auto和直接列表初始化一起使用产生的矛盾行为
C++11引入了统一初始化后，结果证明它和auto搭配会不幸地产生反直觉的矛盾行为：
```cpp
int x{42};      // initializes an int
int y{1,2,3};   // ERROR
auto a{42};     // initializes a std::initializer_list<int>
auto b{1,2,3};  // OK: initializes a std::initializer_list<int>
```
这些使用直接列表初始化（direct list initialization，不带`=`的花括号）造成的前后不一致行为已经得到修复，现在程序行为如下：
```cpp
int x{42};      // initializes an int
int y{1,2,3};   // ERROR
auto a{42};     // initializes an int now
auto b{1,2,3};  // ERROR now
```
注意这是一个非常大的改变，甚至可能悄悄的改变程序的行为。出于这个原因，编译器接受这个改变，但是通常也提供C++11版本的模式。对于主流编译器，比如Visual Studio 2015，g++5和clang3.8同时接受两种模式。

还请注意拷贝列表初始化（copy list initialization，带`=`的花括号）的行为是不变的，当使用auto时初始化一个`std::initializer_list<>`：
```cpp
auto c = {42}; // still initializes a std::initializer_list<int>
auto d = {1,2,3}; // still OK: initializes a std::initializer_list<int>
```
因此，现在的直接列表初始化（不带`=`）和拷贝列表初始化（带`=`）有另一个显著区别：
```cpp
auto a{42}; // initializes an int now
auto c = {42}; // still initializes a std::initializer_list<int>
```
推荐的方式是总是使用直接列表初始化（不带`=`的花括号）来初始化变量和对象。

## 8.5 十六进制浮点字面值
C++17标准化了十六进制的浮点值字面值（有些编译器早已在C++17之前就支持了）。这种方式尤其适用于要求精确的浮点表示（对于双精度浮点值，没法保证精确值的存在）。

举个例子：
```cpp
// lang/hexfloat.cpp
#include <iostream>
#include <iomanip>

int main() {
  // init list of floating-point values:
  std::initializer_list<double> values{
      0x1p4, // 16
      0xA, // 10
      0xAp2, // 40
      5e0, // 5
      0x1.4p+2, // 5
      1e5, // 100000
      0x1.86Ap+16, // 100000
      0xC.68p+2, // 49.625
  };
  
  // print all values both as decimal and hexadecimal value:
  for (double d : values) {
    std::cout << "dec: " << std::setw(6) << std::defaultfloat << d
              << " hex: " << std::hexfloat << d << '\n';
  }
}
```
这个程序使用不同的方式定义了不同的浮点值，其中包括使用十六进制浮点记法。新的记法是base为2的科学表示法：
+ significant/mantissa写作十六进制方式
+ exponent写作数值方式，解释为base为2

比如说，`0xAp2`是指定数值40（10乘以2的次方）。这个值也可以表示为`0x1.4p+5`，表示1.25乘以32（0.4是十六进制的四分之一，2的5次方是32）。

程序输出如下：
```
dec: 16     hex: 0x1p+4
dec: 10     hex: 0x1.4p+3
dec: 40     hex: 0x1.4p+5
dec: 5      hex: 0x1.4p+2
dec: 5      hex: 0x1.4p+2
dec: 100000 hex: 0x1.86ap+16
dec: 100000 hex: 0x1.86ap+16
dec: 49.625 hex: 0x1.8dp+5
```
如你说见，这个例子的浮点记法早已在C++11的`std::hexfloat`操作符上就已经支持了。

## 8.9 预处理条件`__has_include`
C++17扩展了预处理起，可以检查一个特定的头文件是否被include。比如：
```cpp
#if __has_include(<filesystem>)
# include <filesystem>
# define HAS_FILESYSTEM 1
#elif __has_include(<experimental/filesystem>)
# include <experimental/filesystem>
# define HAS_FILESYSTEM 1
# define FILESYSTEM_IS_EXPERIMENTAL 1
#elif __has_include("filesystem.hpp") # include "filesystem.hpp" # define HAS_FILESYSTEM 1
# define FILESYSTEM_IS_EXPERIMENTAL 1
#else
# define HAS_FILESYSTEM #if __has_include(<filesystem>)
# include <filesystem>
# define HAS_FILESYSTEM 1
#elif __has_include(<experimental/filesystem>)
# include <experimental/filesystem>
# define HAS_FILESYSTEM 1
# define FILESYSTEM_IS_EXPERIMENTAL 1
#elif __has_include("filesystem.hpp") # include "filesystem.hpp" # define HAS_FILESYSTEM 1
# define FILESYSTEM_IS_EXPERIMENTAL 1
#else
# define HAS_FILESYSTEM 0
#endif0
#endif
```
如果`#include`成功则`__has_include(...)`会求值为1(true)。如果不成功则没有什么影响。