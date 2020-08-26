# 第九章 类模板参数推导
C++17之前，你必须显式指定类模板的所有模板参数类型。比如，你不能忽略这里的double：
```cpp
std::complex<double> c{5.1,3.3};
```
也不能忽略第二次的`std::mutex`：
```cpp
std::mutex mx;
std::lock_guard<std::mutex> lg(mx);
```
C++17开始，必须显式指定类模板的所有模板参数类型这个限制变得宽松了。有了类模板参数推导（class template argument deduction，CTAD）技术，如果构造函数可以推导出所有模板参数，那么你可以跳过显式指定模板实参。

比如：
+ 你可以这样声明：
```cpp
std::complex c{5.1,3.3}; // OK: std::complex<double> deduced
```
+ 你可以这样实现：
```cpp
std::mutex mx;
std::lock_guard lg{mx}; // OK: std::lock_guard<std_mutex> deduced
```
+ 你甚至可以让容器推导其元素的类型：
```cpp
std::vector v1 {1, 2, 3} // OK: std::vector<int> deduced
std::vector v2 {"hello", "world"}; // OK: std::vector<const char*> deduced
```

## 9.1 使用类模板参数推导
只要传给构造函数的实参可以用来推导类型模板参数，那么就可以使用类模板参数推导技术。该技术支持所有初始化方式：
```cpp
std::complex c1{1.1, 2.2}; // deduces std::complex<double>
std::complex c2(2.2, 3.3); // deduces std::complex<double>
std::complex c3 = 3.3; // deduces std::complex<double>
std::complex c4 = {4.4}; // deduces std::complex<double>
````
c3和c4的初始化方式是可行的，因为你可以传递一个值来初始化`std::complex<>`，这对于推导出模板参数T来说足够了，它会被用于实数和虚数部分：
```cpp
namespace std {
  template<typename T>
  class complex {
    constexpr complex(const T& re = T(), const T& im = T());
  ...
  }
};
```
假设有如下声明
```cpp
std::complex c1{1.1, 2.2};
```
编译器会在调用的地方找到构造函数
```cpp
constexpr complex(const T& re = T(), const T& im = T());
```
因为两个参数T都是double，所以编译器推导出T是double，然后编译下面的代码：
```cpp
complex<double>::complex(const double& re = double(),
                         const double& im = double());
```
注意模板参数必须是无歧义、可推导的。因此，下面的初始化是有问题的：
```cpp
std::complex c5{5,3.3}; // ERROR: attempts to int and double as T
```
对于模板来说，不会在推导模板参数的时候做类型转换。

对于可变参数模板的类模板参数推导也是支持的。比如，`std::tuple<>`定义如下：
```cpp
namespace std {
  template<typename... Types>
  class tuple;
    public:
    constexpr tuple(const Types&...);
    ...
  };
};
```
这个声明：
```cpp
std::tuple t{42, 'x', nullptr};
```
推导出的类型是`std::tuple<int, char, std::nullptr_t>`。

你也可以推导出非类型模板参数。举个例子，像下面例子中传递一个数组，在推导模板参数的时候可以同时推导出元素类型和数组大小：
```cpp
template<typename T, int SZ>
class MyClass {
public:
  MyClass (T(&)[SZ]) {
    ...
  }
};
MyClass mc("hello"); // deduces T as const char and SZ as 6
```
SZ推导为6，因为模板参数类型传递了一个六个字符的字符串字面值。

你甚至可以推导出**用作基类的lambda**的类型，或者推导出**auto模板参数**类型。

### 9.1.1 默认拷贝
如果类模板参数推导发现一个行为更像是拷贝初始化，它就倾向于这么认为。比如，在用一个元素初始化`std::vector`后：
```cpp
std::vector v1{42}; // vector<int> with one element
```
用这个vector去初始化另一个vector：
```cpp
std::vector v2{v1}; // v2 also is vector<int>
```
v2会被解释为`vector<int>`而不是`vector<vector<int>>`

又比如，这个规则适用于下面所有初始化形式：
```cpp
std::vector v3(v1); // v3 also is vector<int>
std::vector v4 = {v1}; // v4 also is vector<int>
auto v5 = std::vector{v1}; // v5 also is vector<int>
```
如果传递多个元素时，就不能被解释为拷贝初始化，此时initializer list的类型会成为新vector的元素类型：
```cpp
std::vector vv{v, v}; // vv is vector<vector<int>>
```
那么问题来了，如果传递可变参数模板，那么类模板参数推导会发生什么：
```cpp
template<typename... Args>
auto make_vector(const Args&... elems) {
  return std::vector{elems...};
}

std::vector<int> v{1, 2, 3};
auto x1 = make_vector(v, v); // vector<vector<int>>
auto x2 = make_vector(v); // vector<int> or vector<vector<int>> ?
```
当前，不同的编译器有不同的处理方式，这个问题还在讨论中。

### 9.1.2 推导lambda的类型
有了类模板参数推导，我们现在终于可以用lambda的类型实例化类模板类。举个例子，我们可以提供一个泛型类，然后包装一下callback，并统计调用了多少次callback：
```cpp
// tmpl/classarglambda.hpp
#include <utility> // for std::forward()

template<typename CB>
class CountCalls
{
private:
  CB callback; // callback to call
  long calls = 0; // counter for calls
public:
  CountCalls(CB cb) : callback(cb) {
  }
  template<typename... Args>
  auto operator() (Args&&... args) {
    ++calls;
    return callback(std::forward<Args>(args)...);
  }
  long count() const {
    return calls;
  }
};
```
这里，构造函数接受一个callback，然后包装一下，用它的类型来推导出模板参数CB。比如，我们可以传一个lambda：
```cpp
CountCalls sc([](auto x, auto y) {
                   return x > y;
             });
```
这意味着sc的类型被推导为`CountCalls<TypeOfTheLambda>`。

