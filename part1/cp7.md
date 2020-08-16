# 第七章 新属性和属性相关特性
C++11开始，你可以指定属性（attribute，一种规范的注解，可以启用或者禁用一些warning）。C++17还引入了新的属性。此外，属性现在可以在更多的地方使用，并且有一些额外的便利。

## 7.1 `[[nodiscard]]`属性
新属性`[[nodiscard]]`用于鼓励编译器，当发现函数返回值没有被使用的时候，产生一个warning。

通常，这个属性可以用于通知一些返回值没有使用的错误行为。错误行为可能是：
+ **内存泄漏**，比如没有使用已经分配并返回的内存
+ **不符合期望，或者非直观行为**，比如没有使用返回值时候可能产生的一些不同寻常/不符合期望的行为
+ **不必要的负载**，比如如果没有使用返回值，这个调用过程相当于无操作。

这是一些例子，它们展示了这个属性的是有用的：
+ 分配资源必须由另一个函数释放的函数应标记为
``[[nodiscard]]``。 一个典型的例子是分配内存的函数，例如`malloc()`或分配器的成员函数`allocate()`。
但是请注意，某些函数可能会返回一个值，后续无需再针对这个值做其他调用。 例如，程序员调用大小为零字节的C函数`realloc(0`以释放内存，这个函数的返回值就不必保存以后再调用`free()`
+ 一个关于不使用返回值那么函数的行为将会改变的例子是`std::async`（由C++11引入）。它的目的是异步启动任务，并返回一个句柄以等待其结束（并使用结果）。当返回值没使用时，这个调用会成为同步调用，因为未使用的返回值的析构函数会立即调用，即立刻开始等待任务结束。 因此，不使用返回值会与`std::async()`的设计目的相矛盾。 这种情况下用`[[nodiscard]]`让编译器对此发出警告。
+ 另一个例子是成员函数`empty()`，它检查对象是否没有元素。程序员有时候可能错误的调用这个函数来清空容器（译注：即误以为empty做动词）
```cpp
cont.empty();
```
这种对`empty()`的误用可以被检查出来，因为它的返回值没有被使用。将成员函数标注这个属性即可：
```cpp
class MyContainer {
  ...
public:
  [[nodiscard]] bool empty() const noexcept;
  ...
};
```
尽管这个是C++17引入的，但是标准库至今都没有使用它。对于C++17来说，应用此功能的建议来得太晚了。因此关于这个特性的关键动机，即为`std::async()`的声明添加现在都没有完成。对于上述所有示例，下一个C++标准将附带相应的修复程序（具体参见已经接受的提案[https://wg21.link/p0600r1](https://wg21.link/p0600r1)）。为了使代码更具可移植性，你应该使用它，而不是使用不可移植的方式（比如gcc或者clang的`[[gnu:warn_unused_result]]`）来标注函数。当定义`operator new()`时你应该为函数标记`[[nodiscard]]`。

## 7.2 `[[maybe_unused]]`属性
新属性`[[maybe_unused]]`可以用来避免编译器为未被使用的名字或者对象发出警告。

这个属性可以用在类声明上、类型定义`typedef`或者`using`上、变量、非静态数据成员、函数、枚举类型或者枚举值。

这个属性的一个应用是标记那些不是必要的参数：
```cpp
void foo(int val, [[maybe_unused]] std::string msg)
{
#ifdef DEBUG
  log(msg);
#endif
  ...
}
```
另一个例子是标记可能不会使用的成员
```cpp
class MyStruct {
  char c;
  int i;
  [[maybe_unused]] char makeLargerSize[100];
  ...
};
```
注意，你不能为一个语句标注`[[maybe_unused]]`。基于这个原因，你不能使用让`[[maybe_unused]]`与`[[nodiscard]]`相见：
```cpp
int main()
{
  foo(); // WARNING: return value not used
  [[maybe_unused]] foo(); // ERROR: attribute not allowed here
  [[maybe_unused]] auto x = foo(); // OK
}
```

## 7.3 `[[fallthrough]]`属性
新属性`[[fallthrough]]`可以让编译器不警告那些switch中的某个case没有break，导致其他case被相继执行的情况。

比如：
```cpp
void commentPlace(int place)
{
  switch (place) {
    case 1:
      std::cout << "very ";
      [[fallthrough]];
    case 2:
      std::cout << "well\n";
      break;
    default:
      std::cout << "OK\n";
      break; 
  } 
}
```
传递1会输出
```
very well
```
同时执行了case 1和case 2。

注意这个属性必须被用在空语句中。因此，你需要在它尾巴上加个分号。

在switch的最后一条语句使用这个属性是不允许的。

## 7.4 通用属性扩展
下面的特性在C++17zhong被启用：
1. 现在允许为namespace标记属性。比如，你可以像下面代码一样弃用一个命名空间：
```cpp
namespace [[deprecated]] DraftAPI {
  ...
}
```
也可以用于inline namespace和匿名namespace。
2. 枚举值现在也可以标注属性。

比如，你可以引入新的枚举值代替原有的枚举值，然后弃用原有枚举值：
```cpp
enum class City { Berlin = 0,
                  NewYork = 1,
                  Mumbai = 2, Bombay [[deprecated]] = Mumbai,
                  ... };
```
Mumbai和Bombay都表示相同的city数值，但是Bombay已经弃用。注意标记枚举值时，语法上需要将属性放到枚举值名字的后面。

3. 用户定义的属性它们通常在自己的namespace定义，你现在可以使用using来避免重复书写namespace。换句话说，以前写法是：
```cpp
[[MyLib::WebService, MyLib::RestService, MyLib::doc("html")]] void foo();
```
现在你可以这么写：
```cpp
[[using MyLib: WebService, RestService, doc("html")]] void foo();
```
注意用了using之后再书写namespace前缀会出错的：
```cpp
[[using MyLib: MyLib::doc("html")]] void foo(); // ERROR
```

## 7.5 后记
这三个属性最初由Andrew Tomazos在[https://wg21.link/p0068r0](https://wg21.link/p0068r0)中提出。最后`[[nodiscard]]`的公认措辞是由Andrew Tomazos在[https://wg21.link/p0189r1](https://wg21.link/p0189r1)中给出。`[[maybe_unused]]`的公认措辞是由Andrew Tomazos在[https://wg21.link/p0212r1](https://wg21.link/p0212r1)中给出。`[[fallthrough]]`的公认措辞是由Andrew Tomazos在[https://wg21.link/p0188r1](https://wg21.link/p0188r1)中给出。

允许namespace和枚举值标注属性这个特性最初由 Richard Smith在[https://wg21.link/n4196](https://wg21.link/n4196)中提出。最后的公认措辞是由 Richard Smith在[https://wg21.link/n4266](https://wg21.link/n4266)中给出。

属性允许使用using这个特性最初由J. Daniel Garcia, Luis M. Sanchez, Massimo
Torquati, Marco Danelutto和Peter Sommerlad在[https://wg21.link/p0028r0](https://wg21.link/p0028r0)中提出。最后的公认措辞是由J. Daniel Garcia and Daveed Vandevoorde在[https://wg21.link/P0028R4](https://wg21.link/P0028R4)中给出。