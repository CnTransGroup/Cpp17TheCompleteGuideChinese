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
c3和c4的初始化方式是可行的