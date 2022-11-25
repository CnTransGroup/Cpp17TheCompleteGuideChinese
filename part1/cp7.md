# 第七章 新属性和属性相关特性

C++11开始，你可以指定属性（attribute，一种规范的注解，可以启用或者禁用一些警告）。C++17还引入了新的属性。此外，属性现在可以在更多的地方使用，并且有一些额外的便利。

## 7.1 `[[nodiscard]]`属性

新属性`[[nodiscard]]`用于鼓励编译器，当发现函数返回值没有被使用的时候，产生一个警告。

通常，这个属性可以用于通知一些返回值没有使用的错误行为。错误行为可能是：

+ **内存泄漏**，比如没有使用已经分配并返回的内存；
+ **不符合预期或者非直观的行为**，比如没有使用返回值时候可能产生的一些不同寻常/不符合期望的行为；
+ **不必要的开销**，比如如果没有使用返回值，这个调用过程相当于无操作。

下面一些例子展示了`[[nodiscard]]`属性的应用场景：

+ 分配的资源必须由另一个函数释放的函数应当标记为``[[nodiscard]]``。 一个典型的例子是分配内存的函数，例如`malloc()`或分配器的成员函数`allocate()`。
  但是请注意，某些函数可能会返回一个值，且后续无需再针对这个值做其他调用。 例如，程序员调用大小为零字节的C函数`realloc(0)`释放内存，这个函数的返回值就不必保存以便后续调用`free()`。

+ 一个关于若不使用返回值则函数的行为将会改变的例子是`std::async`（由C++11引入）。它的作用是启动异步任务，并返回一个句柄以等待任务结束（并使用任务的返回结果）。如果`std::async`的返回值未被使用，那么这个异步调用会演变为同步调用，因为未使用的返回值的析构函数会立即被调用，而这个值本该等待任务结束后再析构。 因此，不使用返回值会与`std::async()`的设计目的相矛盾。这种情况下用`[[nodiscard]]`让编译器对此发出警告。

+ 另一个例子是成员函数`empty()`，它检查对象（容器、字符串等）是否没有元素。程序员有时候可能错误的调用这个函数来清空容器（译注：即误以为empty是写操作）。
  
  ```cpp
  cont.empty();
  ```
  
  这种对`empty()`的误用可以被检查出来，因为它的返回值没有被使用。将成员函数标注`[[nodiscard]]`属性即可：
  
  ```cpp
  class MyContainer {
  ...
  public:
  [[nodiscard]] bool empty() const noexcept;
  ...
  };
  ```
  
  尽管这个属性是C++17引入的，但是标准库至今都没有使用它。对于C++17来说，应用此功能的建议来得太晚了。因此关于这个特性的关键动机，即为`std::async()`的声明添加`[[nodiscard]]`属性，至今都还未完成。对于上述所有示例，下一个C++标准将附带相应的修复程序（具体参见已经接受的提案[https://wg21.link/p0600r1](https://wg21.link/p0600r1)）。
  
  为了使代码更具可移植性，你应该使用它，而不是使用不可移植的方式（比如gcc或者clang的`[[gnu:warn_unused_result]]`属性）来标注函数。
  
  当定义`operator new()`时，你应该为函数标记`[[nodiscard]]`。典型地，定义一个头文件来追踪所有对`operator new()`的调用。

## 7.2 `[[maybe_unused]]`属性

新属性`[[maybe_unused]]`可以用来避免编译器对未被使用的标识符或对象发出警告。

这个属性可以用于：类的声明、类型的定义（通过`typedef`或`using`）、变量、非静态数据成员、函数、枚举类型或者枚举值。

该属性的一个应用场景是标记那些非必要的参数：

```cpp
void foo(int val, [[maybe_unused]] std::string msg)
{
#ifdef DEBUG
  log(msg);
#endif
  ...
}
```

另一个场景是标记可能不会使用的成员：

```cpp
class MyStruct {
  char c;
  int i;
  [[maybe_unused]] char makeLargerSize[100];
  ...
};
```

注意，你不能为一个语句标注`[[maybe_unused]]`。基于此，你不能用`[[maybe_unused]]`修饰使用了`[[nodiscard]]`的函数的调用语句：

```cpp
int main()
{
  foo(); // WARNING: return value not used
  [[maybe_unused]] foo(); // ERROR: attribute not allowed here
  [[maybe_unused]] auto x = foo(); // OK
}
```

## 7.3 `[[fallthrough]]`属性

新属性`[[fallthrough]]`可以让编译器不警告如下情况：在一条switch语句中，某个case子句缺少break子句，从而导致后续case子句相继执行。

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

此处将会输出：

```
very well
```

因为switch语句同时执行了case 1子句和case 2子句。

注意这个属性必须被用在空语句中。因此，你需要在其末尾添加一个分号。

另外，不允许在switch语句的最后一条子句使用这个属性（不论是case子句还是default子句）。

## 7.4 通用属性扩展

下面的特性在C++17中被启用：

1. 现在允许为namespace标记属性。比如，你可以像下面代码一样弃用一个namespace：
   
   ```cpp
   namespace [[deprecated]] DraftAPI {
   ...
   }
   ```
   
   `[[deprecated]]`属性也可用于内联namespace和匿名namespace。

2. 枚举值现在也可以标注`[[deprecated]]`属性。比如，你可以引入新的枚举值代替原有的枚举值，然后弃用原有枚举值：

```cpp
enum class City { Berlin = 0,
                  NewYork = 1,
                  Mumbai = 2, Bombay [[deprecated]] = Mumbai,
                  ... };
```

Mumbai和Bombay都表示相同的city枚举值，但是Bombay已经弃用，所以不会发生枚举值的冲突。注意标记枚举值时，语法上需要将属性放到枚举值标识符的后面。

3. 用户自定义的属性通常应该定义在自己的namespace中，现在可以使用新的using语法来避免重复书写namespace。例如，以前写法是：
   
   ```cpp
   [[MyLib::WebService, MyLib::RestService, MyLib::doc("html")]] void foo();
   ```
   
   现在你可以这么写：
   
   ```cpp
   [[using MyLib: WebService, RestService, doc("html")]] void foo();
   ```
   
   注意，用了using之后再书写namespace前缀会出错：
   
   ```cpp
   [[using MyLib: MyLib::doc("html")]] void foo(); // ERROR
   ```

## 7.5 后记

这三个属性最初由Andrew Tomazos在[https://wg21.link/p0068r0](https://wg21.link/p0068r0)中提出。最后`[[nodiscard]]`的公认措辞是由Andrew Tomazos在[https://wg21.link/p0189r1](https://wg21.link/p0189r1)中给出。`[[maybe_unused]]`的公认措辞是由Andrew Tomazos在[https://wg21.link/p0212r1](https://wg21.link/p0212r1)中给出。`[[fallthrough]]`的公认措辞是由Andrew Tomazos在[https://wg21.link/p0188r1](https://wg21.link/p0188r1)中给出。

允许namespace和枚举值标注属性这个特性最初由 Richard Smith在[https://wg21.link/n4196](https://wg21.link/n4196)中提出。最后的公认措辞是由 Richard Smith在[https://wg21.link/n4266](https://wg21.link/n4266)中给出。

属性允许使用using这个特性最初由J. Daniel Garcia, Luis M. Sanchez, Massimo
Torquati, Marco Danelutto和Peter Sommerlad在[https://wg21.link/p0028r0](https://wg21.link/p0028r0)中提出。最后的公认措辞是由J. Daniel Garcia and Daveed Vandevoorde在[https://wg21.link/P0028R4](https://wg21.link/P0028R4)中给出。