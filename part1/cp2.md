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
