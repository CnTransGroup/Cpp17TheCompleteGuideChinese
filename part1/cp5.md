# 第五章 强制拷贝消除或传递未具体化对象

本章的主题可以从两个角度来看：

+ C++17引入了新的规则，在确定条件下可以强制消除拷贝：以前临时对象传值或者返回临时对象期间发生的拷贝操作的消除是可选的，现在是强制的。
+ 因此，我们处理传递未具体化对象的值以进行初始化。
  我将从技术上介绍这个特性，然后讨论具体化（materialization）的效果和相关术语。

## 5.1 强制消除临时量拷贝的动机

标准伊始，C++就明确允许一些拷贝操作可以被省略（消除），不调用拷贝构造函数会失去可能存在的副作用，从而可能影响程序的行为，即便这样也在所不惜。强制拷贝消除的场景之一是使用临时对象初始化新对象。这个情况经常发生，尤其是以值传递方式将临时对象传递给一个函数，或者函数返回临时对象。举个例子：

```cpp
class MyClass
{
  ...
};
void foo(MyClass param) { // param is initialized by passed argument
  ...
}
MyClass bar() {
  return MyClass(); // returns temporary
}

int main()
{
  foo(MyClass());       // pass temporary to initialize param
  MyClass x = bar();    // use returned temporary to initialize x
  foo(bar());           // use returned temporary to initialize param
}
```

但是，由于这些编译器优化不是强制的，要拷贝的对象必须提供隐式或显式的拷贝或移动构造函数。也就是说，尽管拷贝/移动构造函数一般不会被调用，但是也必须存在。如果没有定义拷贝/移动构造函数，那么代码不能通过编译。

因此，下面MyClass的定义的代码编译不了：

```cpp
class MyClass
{
public:
  ...
  // no copy/move constructor defined:
  MyClass(const MyClass&) = delete;
  MyClass(MyClass&&) = delete;
  ...
};
```

实际上，这里没有拷贝构造函数就足够了，因为只有当不存在用户声明的拷贝构造函数（或者拷贝赋值运算符）时，移动构造函数才隐式可用。

C++17后，通过临时对象初始化新对象期间发生的拷贝是强制消除的。事实上，在后面我们会看到，我们简单地传值以初始化实参，或者返回一个值，该值都将会用于具体化（materalize）一个新对象。

这意味着就算MyClass类完全没有启用拷贝操作也能通过编译，就如同上面这个例子。

然而，请注意其他可选的拷贝消除仍然是可选的，仍然要求一个可调用的拷贝或者移动构造函数才能通过编译，比如：

```cpp
MyClass foo()
{
  MyClass obj;
  ...
  return obj;    // still requires copy/move support
}
```

在这里，`foo()`里面的obj是一个带名字的变量（即左值（lvalue））。所以会发生**具名的**返回值优化（named return value optimization，NRVO），它要求类型具备拷贝或者移动行为。即便obj是一个参数也仍然如此：

```cpp
MyClass bar(MyClass obj) // copy elision for passed temporaries
{
  ...
  return obj; // still requires copy/move support
}
```

传递一个临时量（即纯右值（prvalue））到函数作为实参，不会发生拷贝/移动操作，但是返回这个参数仍然需要拷贝/移动操作，因为返回的对象有名字，即obj。

作为这一改变的部分，值范畴（value categories）修改和新增了很多术语。

## 5.2 强制消除临时量拷贝的好处

强制拷贝消除的一个好处是，很明显，避免一个开销较大的拷贝操作会获得更好的性能。虽然移动语义显著减少了拷贝开销，但如果能完全避免拷贝操作将会是一次关键的性能提升，尽管这些拷贝操作本身也很轻量（例如，这些对象的若干成员仅为基础数据类型）。这可能会减少出参的使用（译注：所谓出参即"out parameter"，是指使用参数来传递返回信息，通常是一个指针或者非const的引用），转而让函数直接返回一个值（假设这个值是由返回语句创建的）。

另一个好处是现在只要定义一个工厂函数，它总是能工作。因为现在的工厂函数可以返回对象，即便该对象不允许拷贝/移动。比如，考虑下面的泛型工厂函数：

```cpp
// lang/factory.hpp
#include <utility>

template <typename T, typename... Args>
T create(Args&&... args)
{
  ...
  return T{std::forward<Args>(args)...};
}
```

这个工厂函数现在甚至可以用于生产`std::atomic<>`这种类型的对象，该类型既没有定义拷贝构造函数也没有定义移动构造函数：

```cpp
// lang/factory.cpp
#include "factory.hpp" 
#include <memory>
#include <atomic>

int main() {
    int i = create<int>(42);
    std::unique_ptr<int> up = create<std::unique_ptr<int>>(new int{42});
    std::atomic<int> ai = create<std::atomic<int>>(42);
}
```

这个特性带来的另一个效果是，如果类有显式delete的移动构造函数，你现在可以返回临时值，然后用它初始化对象：

```cpp
class CopyOnly {
public:
    CopyOnly() {
    }
    CopyOnly(int) {
    }
    CopyOnly(const CopyOnly&) = default;
    CopyOnly(CopyOnly&&) = delete; // explicitly deleted
};

CopyOnly ret() {
    return CopyOnly{}; // OK since C++17
}

CopyOnly x = 42; // OK since C++17
```

对象x的初始化代码在C++17之前是无效的，因为拷贝初始化需要将42转换为一个CopyOnly类型的临时对象，然后该临时对象原则上需要提供一个移动构造函数，尽管用不到它。（事实上，仅当移动构造函数未由用户声明，拷贝构造函数才会作为移动构造函数的替代，用以构造对象x。）

## 5.3 值范畴的阐述

强制拷贝消除这一特性造成的额外影响，是需要对值范畴（value category）做出一些调整。

### 5.3.1 值范畴

在C++中的每个表达式都有一个值范畴。这个值范畴描述了表达式可以做什么。

#### 值范畴的历史

从C语言历史的角度来看，在赋值语句中只有左值（lvalue）和右值（rvalue）：

```cpp
  x = 42;
```

表达式x是lvalue，因为它可以出现在一条赋值语句的左边，表达式42是rvalue，因为它只能出现在一条赋值语句的右边。但是因为ANSI-C的出现，事情变得更复杂。因为x如果声明为`const int`就不能在赋值语句的左边了，但是它仍然是个（不具可修改性的）lvalue。

自C++11以来，我们有了可移动的对象，这些对象在语义上只能出现在赋值语句右边，但却可以被修改，因为一个赋值符号可以窃取它们的值（移动到左侧变量，并对其做出修改）。基于此，新的值范畴xvalue被引入，并且之前值范畴的rvalue被重新命名为prvalue。

#### C++11的值范畴

C++11后，值范畴如图5.1描述的那样：我们的核心值范畴是lvalue，prvalue（pure rvalue，纯右值），xvalue（eXpiring value，将亡值）。组合得到的值范畴有：glvalue（generalized lvalue，泛化左值，是lvalue和xvalue的结合）以及rvalue（是xvalue和prvalue的结合）。

<img src="../public/fig5-1.jpg" align="center"/>
<p align="center">图5.1 C++11后的值范畴</p>

lvalue左值包括：

+ 一个只包含变量、函数或者成员的标识符的表达式；
+ 一个字符串字面值的表达式；
+ 语言内置的一元操作符`*`的计算结果（即对原生指针解引用的结果）；
+ 返回左值引用（`T&`）的函数的返回值；

prvalue纯右值包括：

+ 除字符串字面值外的其他字面值的表达式（或者用户定义的字面值，此类情况下由与之关联的字面值操作符的返回类型决定值范畴）；
+ 语言内置的一元操作符`&`的计算结果（即对表达式取地址的结果）；
+ 语言内置的算术运算符的计算结果；
+ 返回值（`T`）的函数的返回值；
+ 一条lambda表达式；

xvalue将亡值包括：

+ 返回右值引用（`T&&`，尤其通过`std::move()`返回）的函数的返回值；
+ 右值引用到对象类型的转换；

总的来说：

+ 所有直接使用标识符的表达式是lvalue；
+ 所有字符串字面值的表达式是lvalue；
+ 所有其他字面值（4.2，true，nullptr）的表达式是prvalue；
+ 所有临时变量（尤其是返回值的函数返回的对象）是prvalue；
+ `std::move()`的结果是xvalue；

举个例子：

```cpp
class X {
};

X v;
const X c;

void f(const X&);   // accepts an expression of any value category
void f(X&&);        // accepts prvalues and xvalues only, but is a better match

f(v);               // passes a modifiable lvalue to the first f()
f(c);               // passes a non-modifiable lvalue to the first f()
f(X());             // passes a prvalue to the second f()
f(std::move(v));    // passes an xvalue to the second f()
```

值得强调的是，严格来说，glvalue，prvalue和xvalue是针对“表达式”的术语， 不是针对值的术语（这意味着这些术语可能是误称）。举个例子，一个变量本身不是一个lvalue，只有一个变量放到表达式里才表明这个变量是lvalue：

```cpp
int x = 3; // x here is a variable, not an lvalue
int y = x; // x here is an lvalue
```

第一句中3是prvalue，它用来初始化变量x（此时x不是lvalue）。第二个语句中x是lvalue（对它求值会会发现它包含值3）。然后作为lvallue的x转换为prvalue，用来初始化变量y。

### 5.3.2 C++17的值范畴

C++17没有改变既有的值范畴，但是进一步阐述了它们的语义含义（如图5.2所示）。

<img src="../public/fig5-2.jpg" align="center"/>
<p align="center">图5.1 C++17后的值范畴</p>

现在解释值范畴的关键方式是认为我们有两类表达式：

+ glvalue：用于描述对象或函数的定位的表达式；（译注，即用于定位对象的内存地址的表达式。实际上lvalue是"locator value"，locator即指定位内存地址。）
+ prvalue：用于初始化的表达式；
  此外，xvalue被认为是用来定位一块特殊内存的表达式，它表示有一个变量的资源可以重用（通常因为已接近该变量的生命周期的末期）。

C++17引入了一个新术语，称为“具体化（materialization）”，表示在某个时刻一个prvalue成为临时对象。因此，临时变量具体化转换（temporary materialization conversion）是指prvalue到xvalue的转换。

任何时刻，期望出现glvalue（lvalue或xvalue）的地方出现prvalue都是有效的，创建一个临时对象并通过prvalue初始化（上面提到prvalue就是用于初始化的），随后prvalue被替换为xvalue，并委派给该临时对象。因此在上面的例子中，严格来说：

```cpp
void f(const X& p); // accepts an expression of any value category,
                    // but expects a glvalue
f(X());             // passes a prvalue materialized as xvalue
```

因为例子中的`f()`有一个引用参数，它期望一个glvalue实参。然而，表达式`X()`是一个prvalue。临时量的具体化规则因而生效，表达式`X()`“转换”为一个xvalue委派给临时对象，在此期间缺省的无参构造函数被调用。

注意到具体化过程并不意味着我们创建了一个新的/不同的对象。lvalue引用仍然绑定到xvalue和prvalue，虽然后者总是先转换为xvalue（即上图中的虚线箭头）。

在这些改变后（从此prvalue不再是对象本身，而是能用来初始化对象的表达式），拷贝消除的意义就完美地体现出来了，因为prvalue不再要求可移动，而此前被要求可移动是为了通过赋值语句来初始化左侧的变量。现在我们只需要传递一个初始值，这个值迟早会具体化然后用来初始化一个对象。

## 5.4 未具体化的返回值传递

未具体化返回值传递是指所有形式的返回临时对象（prvalue）的值：

+ 当返回一个不是字符串字面值的字面值：
  
  ```cpp
  int f1() {    // return int by value
  return 42;
  }
  ```

+ 当返回类型为临时变量的值或者使用auto：
  
  ```cpp
  auto f2() {   // return deduced type by value
  ...
  return MyType{...};
  }
  ```

+ 当返回临时对象，并且类型用`decltype(auto)`推导：
  
  ```cpp
  decltype(auto) f3() {   // return temporary from return statement by value
  ...
  return MyType{...};
  }
  ```
  
  记住如果用于初始化的表达式（这里是返回语句）会创建一个临时变量（prvalue），那么用`decltype(auto)`声明的类型是值。

上述所有形式我们都返回一个prvalue的值，我们不需要任何拷贝/移动操作的支持。

## 5.5 后记

强制拷贝消除最初由Richard Smith在[https://wg21.link/p0135r0](https://wg21.link/p0135r0)中提出。最后这个特性的公认措辞是由Richard Smith在[https://wg21.link/p0135r1](https://wg21.link/p0135r1)中给出。