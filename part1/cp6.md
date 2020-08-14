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
对于显示或隐示的constexpr lambda，函数调用操作符是constexpr。换句话说，下面的定义：
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