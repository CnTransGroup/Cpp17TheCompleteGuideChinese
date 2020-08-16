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