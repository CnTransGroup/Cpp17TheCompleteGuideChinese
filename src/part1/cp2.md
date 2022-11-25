# 第二章 带初始化的if和switch

现在**if**和**switch**控制结构允许我们在普通的条件语句或者选择语句之外再指定一个初始化语句。

比如，你可以这样写：
```cpp
if (status s = check(); s != status::success) {
    return s;
}
```
其中初始化语句是：
```cpp
status s = check();
```
它初始化s，然后用if判断s是否是有效状态。

## 2.1 带初始化的if
任何在if语句内初始化的值的生命周期都持续到then代码块或者else代码块（如果有的话）的最后。比如：
```cpp
if (std::ofstream strm = getLogStrm(); coll.empty()) {
    strm << "<no data>\n"; 
}
else {
    for (const auto& elem : coll) {
        strm << elem << '\n'; 
    } 
}
// strm no longer declared
```
strm的析构函数回在then代码块或者else代码块的最后调用。

另一个例子是执行一些依赖某些条件的任务的时候使用锁：
```cpp
if (std::lock_guard<std::mutex> lg{collMutex}; !coll.empty()) {
    std::cout << coll.front() << '\n'; 
}
```
因为有**类模板参数推导**，也可以这样写：
```cpp
if (std::lock_guard lg{collMutex}; !coll.empty()) {
    std::cout << coll.front() << '\n'; 
}
```
任何情况下，上面的代码都等价于：
```cpp
{
    std::lock_guard<std::mutex> lg{collMutex};
    if (!coll.empty()) {
        std::cout << coll.front() << '\n'; 
    } 
}
```
区别在于lg是在if语句的作用域中定义的，因此与条件在相同的作用域（声明性区域）中，就像for循环中初始化的情况一样。

任何被初始化的对象都必须有一个名字。否则，初始化语句会长久一个立即销毁大的临时值。举个例子，初始化一个没有名字的lock guard，其后的条件检查不是在加锁环境下进行的：
```cpp
if (std::lock_guard<std::mutex>{collMutex}; // run-time ERROR:
        !coll.empty()) { // - no longer locked
    std::cout << coll.front() << '\n'; // - no longer locked
} 
```
一般来说，一个`_`作为名字也是可以的（一些程序员喜欢它，另一些讨厌它因为它污染全局命名空间）：
```cpp
if (std::lock_guard<std::mutex> _{collMutex}; // OK, but...
    !coll.empty()) {
  std::cout << coll.front() << '\n';
}
```
接下来是第三个例子，考虑一段代码，插入新元素到map或者unordered map。你可以检查操作是否成功，就像下面一样：
```cpp
std::map<std::string, int> coll;
... 
if (auto [pos, ok] = coll.insert({"new", 42}); !ok) {
  // if insert failed, handle error using iterator pos:
  const auto &[key, val] = *pos;
  std::cout << "already there: " << key << '\n';
}
```
这段代码还是用了结构化绑定，给返回值和元素插入的位置pos分别赋予了名字，而不是first和second。在C++17前，上面相应的检查必须像下面一样规范：
```cpp
auto ret = coll.insert({"new", 42});
if (!ret.second) {
  // if insert failed, handle error using iterator ret.first
  const auto &elem = *(ret.first);
  std::cout << "already there: " << elem.first << '\n';
}
```
注意这种带if的初始化也能用于编译时if特性。

## 2.2 带初始化的switch
使用带初始化的switch语句允许我们在检查条件并决定控制流跳转到哪个case执行之前初始化一个对象。

比如，我们可以先初始化一个**文件系统路径**，再根据路径的类型选择对应的处理方式：
```cpp
using namespace std::filesystem;
... 
switch (path p(name); status(p).type()) {
case file_type::not_found:
  std::cout << p << " not found\n";
  break;
case file_type::directory:
  std::cout << p << ":\n";
  for (auto &e : std::filesystem::directory_iterator(p)) {
    std::cout << "- " << e.path() << '\n';
  }
  break;
default:
  std::cout << p << " exists\n";
  break;
}
```
初始化的p能在整个switch语句中使用。

## 2.3 后记
带初始化的if和switch最初由Thomas Koppe在[https://wg21.link/p0305r0](https://wg21.link/p0305r0)中提出，当时只有带初始化的if没有带初始化的switch。最后这个特性的公认措辞是由Thomas Koppe在[https://wg21.link/p0305r1](https://wg21.link/p0305r1)中给出。