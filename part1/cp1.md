# 第一章  结构化绑定

结构化绑定允许你使用对象的成员或者说元素来初始化多个实体。举个例子，加入你定义了一个包含两个不同成员的结构：
```cpp
struct MyStruct {
  int i = 0;
  std::string s;
};

MyStruct ms;
```
只需使用下面的声明，你就可以将这个结构的成员直接绑定到新名字上
```cpp
auto [u,v] = ms;
```
在这里，名字u和v就被成为结构化绑定（structured bindings）。在某种程度上，它们分解了对象并用来初始化自己（在有些地方它们也被称为分解声明（decompose  declarations））。

结构化绑定对于那些返回结构体或者数组的函数来说尤其有用。举个例子，假设你有一个返回结构体的函数：
```cpp
MyStruct getStruct() {
  return MyStruct{42, "hello"};
}
```
你可以直接将函数返回的数据成员赋予两个局部名字：
```cpp
auto[id,val] = getStruct(); // id and val name i and s of returned struct
```
在这里，id和val分别表示返回的数据成员i和s。它们的类型分别是int和`std::string`，可以当新变量使用。
```cpp
if (id > 30) {
  std::cout << val;
}
```



