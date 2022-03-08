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

通过这种方式，我们可以计算传递给排序函数的sc的调用次数：
```cpp
std::sort(v.begin(), v.end(),
          td::ref(sc));
std::cout << "sorted with " << sc.count() << " calls\n";
```
包装后的lambda通过引用的方式传递给排序函数，因为如若不然`std::sort()`只会计算传递给他的lambda的拷贝的调用，毕竟是传值的方式。

然而，我没可以传递包装后的lambda给`std::for_each`，因为这个算法可以返回传递给他的callback的拷贝：
```cpp
auto fo = std::for_each(v.begin(), v.end(),
                        CountCalls([](auto i) {
                                      std::cout << "elem: " << i << '\n';
                        }));
std::cout << "output with " << fo.count() << " calls\n";
```

### 9.1.3 非部分类模板参数推导
不像函数模板那样，类模板参数不能部分推导（显示模板参数的一部分）。比如：
```cpp
template<typename T1, typename T2, typename T3 = T2>
class C {
public:
  C (T1 x = T1{}, T2 y = T2{}, T3 z = T3{}) {
    ...
  }
  ...
};
// all deduced:
C c1(22, 44.3, "hi"); // OK: T1 is int, T2 is double, T3 is const char*
C c2(22, 44.3); // OK: T1 is int, T2 and T3 are double
C c3("hi", "guy"); // OK: T1, T2, and T3 are const char*
// only some deduced:
C<string> c4("hi", "my"); // ERROR: only T1 explicitly defined
C<> c5(22, 44.3); // ERROR: neither T1 not T2 explicitly defined
C<> c6(22, 44.3, 42); // ERROR: neither T1 nor T2 explicitly defined
// all specified:
C<string,string,int> c7; // OK: T1,T2 are string, T3 is int
C<int,string> c8(52, "my"); // OK: T1 is int,T2 and T3 are strings
C<string,string> c9("a", "b", "c"); // OK: T1,T2,T3 are strings
```
因为第三个模板参数类型有默认值，所以如果已经指定了第二个就可以省略第三个。

如果i想知道为什么不支持偏特化，下面是造成这个抉择的原因：
```cpp
std::tuple<int> t(42, 43); // still ERROR
```
`std::tuple`是一个可变参数模板，所以你可以指定任意数量的参数。在这种情况下，到底是认为这是只指定了一个类型的而导致的错误还是有意为之很难说清。看起来是有问题的。后期有更多考量后，偏特化也有可能加入C++标准。尽管目前没有。


不幸的是，缺少部分特化就不能解决一个常见代码需求。对于关联容器的排序规则，或者无序容器的hash函数，我们仍然不能简单的传一个lambda：
```cpp
std::set<Cust> coll([](const Cust& x, const Cust& y) { // still ERROR
  return x.name() > y.name();
});
```
我们还是得指定lambda的类型，因此需要像下面这样写：
```cpp
auto sortcrit = [](const Cust& x, const Cust& y) {
  return x.name() > y.name();
};
std::set<Cust, decltype(sortcrit)> coll(sortcrit); // OK
```

### 9.1.4 类模板参数推导代替便捷的工具函数。
有了类模板参数推导，我们可以不再使用那些目的仅是推导传的参数的类型的便捷工具函数。

最明显的是`make_pair`，他允许我们不指定传的参数的类型。比如，对于v：
```cpp
std::vector<int> v;
```
我们可以使用
```cpp
auto p = std::make_pair(v.begin(), v.end());
```
来代替
```cpp
std::pair<typename std::vector<int>::iterator,typename std::vector<int>::iterator> p(v.begin(), v.end());
```
现在，`make_pair()`不再需要了，可以直接这么写：
```cpp
std::pair p(v.begin(), v.end());
```
然而，`std::make_pair()`也是一个很好的例子，它说明了有时候工具函数不只是做模板参数推导一件事情。事实上，`std::make_pair()`也会类型退化，这意味着传入的string字面值会转换为`const char*`：
```cpp
auto q = std::make_pair("hi", "world"); // pair of pointers
```
在这个例子中，q的类型是`std::pair<const char*, const char*>`。

使用类模板参数推导，情况变得更复杂。让我们看看一个简单的类声明，它有点像`std::pair`：
```cpp
template<typename T1, typename T2>
struct Pair1 {
  T1 first;
  T2 second;
  Pair1(const T1& x, const T2& y) : first{x}, second{y} {
  }
};
```
重点是元素通过引用传递。根据语言规则，当使用引用传递一个模板类型的实参时，形参不会_类型退化_，所谓类型退化是指将原生数组类型转换为原生指针类型这样一种机制。所以，当这样调用的时候：
```cpp
Pair1p1{"hi","world"};//deduces pair of arrays of different size, but...
```
T1被推导为`char[3]`，T2被推导为`char[6]`。基本上，这个推导是有效地。然而，当我们使用T1和T2类型去声明成员**first**和**second**时，结果是，它们被声明为：
```cpp
char first[3];
char second[6];
```
并且从一个左值数组初始化出新数组是不被允许的。编译过程就像这样：
```cpp
const char x[3] = "hi";
const char y[6] = "world";
char first[3] {x}; // ERROR
char second[6] {y}; // ERROR
```
注意如果直接使用值传递，然后用它的类型来声明成员就不会有这个问题：
```cpp
template<typename T1, typename T2>
struct Pair2 {
  T1 first;
  T2 second;
  Pair2(T1 x, T2 y) : first{x}, second{y} {
  }
};
```
如果我们这样调用：
```cpp
Pair2 p2{"hi", "world"}; // deduces pair of pointers
```
T1和T2将会被推导为`const char*`。
因为类`std::pair<>`被声明，所以构造函数使用引用传参，你可能期望看到下面的初始化代码不会被编译：
```cpp
std::pair p{"hi", "world"}; // seems to deduce pair of arrays of different size, but...
```
但是它可以编。原因是我们用到了**推导规则**

## 9.2 推导规则
你可以定义特定的**推导规则**来提供额外的类模板实参推导能力，或者修复已经存在的、由构造函数定义的推导。举个例子，你可以定义一个规则，无论何时Pair3被推导，类型推导看起来就像在操作值传递的参数一样：
```cpp
template<typename T1, typename T2>
struct Pair3 {
  T1 first;
  T2 second;
  Pair3(const T1& x, const T2& y) : first{x}, second{y} {
  }
};
// deduction guide for the constructor:
template<typename T1, typename T2>
Pair3(T1, T2) -> Pair3<T1, T2>;
```
这里`->`左边是什么我们想推导什么。在这里例子中，我们想推导一个构造函数，其参数是值传递，类型是任意T1和T2。在`->`右边是我们想定义的推导结果。本例中，Pair3进行实例化。

你可能争辩说，这不就是构造函数做的事情吗。然而，构造函数的参数是引用传递，这里是值传递，不一样。通常来说，即使在模板外面，值传递的实参也会_类型退化_，而引用传递的实参不会退化。_类型退化_意味着原始数组转换为指针，顶级修饰符，比如const和引用符号，会被忽略。

没有类型推导规则，下面的声明：
```cpp
Pair3 p3{"hi", "world"};
```
x的类型，即T1是`const char[3]`，y的类型，即T2是`const char[6]`。

因为有类型推导，模板参数会类型华为，意味着传递的数组或者字符串字面值会退化为对应的指针类型。现在当我们声明：
```cpp
Pair3 p3{"hi", "world"};
```
推导规则被应用，两个参数类型都是`const char*`。推导后的类型就像我们直接这样写：
```cpp
Pair3<const char*, const char*> p3{"hi", "world"};
```
注意，此时构造函数仍然是引用传参。推导规则只影响模板类型的推导，不影响T1、T2被推导后的构造函数调用。

### 9.2.1 使用推导规则强制类型退化
正如上面例子演示的那样，通常，这些重载规则的一个常见用途是确保一个模板参数T在推导过程中类型进行退化。考虑一个传统的类模板：
```cpp
template<typename T>
struct C {
  C(const T&) {
  }
  ...
};
```
如果我们这里传一个字符串字面值"hello"，T被推导为字符串字面值的类型，即`const char[6]`：
```cpp
C x{"hello"}; // T deduced as const char[6]
```
原因是当引用传值时，模板类型推导不会将它退化成对应的指针类型。
带上一个简单的推导规则：
```cpp
template<typename T> C(T) -> C<T>;
```
我们就修复了这个问题：
```cpp
C x{"hello"}; // T deduced as const char*
```
现在，因为推导规则是值传递，它的类型发生退化，所以"hello"的类型T最终是`const char*`。
出于这个原因，对于任意类模板，其构造函数带引用传递的参数，都给出这样一个推导规则是很合理的。C++标准库为pair和tuple提供了对应的推导规则（参见9.2.6）。

### 9.2.2 非模板推导规则
to translate