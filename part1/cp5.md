# 第五章 强制拷贝消除或者传递unmaterialized对象
本章的主题可以从两个角度来看：
+ C++17引入了新的规则，在确定条件下可以强制消除拷贝：以前临时对象传值或者返回临时对象期间发生的拷贝操作的消除是可选的，现在是强制的。
+ 因此，我们处理传递未具体化对象的值以进行初始化
我将从技术上介绍这个特性，然后讨论具体化（materialization）的效果和相关术语。

## 5.1 临时量强制拷贝消除的动机
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
但是，由于这些拷贝消除优化不是强制的，要拷贝的对象必须提供隐式或显式的拷贝或移动构造函数。也就是说，尽管拷贝/移动构造函数一般不会调用，但是也必须存在。如果没有定义拷贝/移动构造函数，那么代码不能通过编译。

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
这里没有拷贝构造函数就足够了，因为仅当没有用户声明的拷贝构造（或者拷贝赋值运算符）时移动构造函数才隐式可用。

C++17后，临时变量初始化新对象期间发生的拷贝是强制消除的。事实上，在后面我们会看到，我们简单的传值作为实参初始化或者返回一个值，该值会接下来用于具体化（materalize）一个新对象。

这意味着就算MyClass类完全没有表示启用拷贝操作，上面的例子也能通过编译。

然而，请注意其他可选的拷贝消除仍然是可选的，仍然要求一个可调用的拷贝或者移动构造函数，比如：
```cpp
MyClass foo()
{
  MyClass obj;
  ...
  return obj;    // still requires copy/move support
}
```
在这里，`foo()`里面的obj是一个带名字的变量（即左值（lvalue））。所以会发生命名的返回值优化（named return value optimization，NRVO），它要求类型支持拷贝或者移动操作。即便obj是一个参数也仍然如此：
```cpp
MyClass bar(MyClass obj) // copy elision for passed temporaries
{
  ...
  return obj; // still requires copy/move support
}
```
传递一个临时量（即纯右值（prvalue））到函数作为实参，不会发生拷贝/移动操作，但是返回这个参数仍然需要拷贝/移动操作，因为返回的对象有名字。

作为这一改变的部分，值分类表（value categories）修改和新增了很多术语。

## 5.2 临时量强制拷贝消除的好处
