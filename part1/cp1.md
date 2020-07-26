# 第一章  结构化绑定

结构化绑定允许你使用对象的成员或者说元素来初始化多个实体。举个例子，假如你定义了一个包含两个不同成员的结构：
```cpp
struct MyStruct {
  int i = 0;
  std::string s;
};

MyStruct ms;
```
只需使用下面的声明，你就可以将这个结构体的成员直接绑定到新名字上
```cpp
auto [u,v] = ms;
```
在这里，名字u和v就被称为结构化绑定（structured bindings）。在某种程度上，它们分解了对象并用来初始化自己（在有些地方它们也被称为分解声明（decompose  declarations））。

结构化绑定对于那些返回结构体或者数组的函数来说尤其有用。举个例子，假设你有一个返回结构体的函数：
```cpp
MyStruct getStruct() {
  return MyStruct{42, "hello"};
}
```
你可以直接为函数返回的数据成员赋予两个局部名字：
```cpp
auto[id,val] = getStruct(); // id and val name i and s of returned struct
```
在这里，id和val分别表示返回的数据成员i和s。它们的类型分别是int和`std::string` ，可以当新变量使用。
```cpp
if (id > 30) {
  std::cout << val;
}
```
使用结构化绑定的好处是可以直接通过名字访问值，并且由于名字可以传递信息，使得代码可读性也大大提高。
下面的示例展示了结构化绑定如何改善代码可读性。在没有结构化绑定的时候，要想迭代处理`std::map<>`的所有元素，需要这么写：
```cpp
for (const auto& elem : mymap) {
  std::cout << elem.first << ": " << elem.second << '\n'; 
}
```
代码中的elem是表示键和值的`std::pair`，它们在`std::pair`中分别用first和second表示，你可以使用这两个名字去访问键和值。使用结构化绑定后，代码可读性大大提高：
```cpp
for (const auto& [key,val] : mymap) {
  std::cout << key << ": " << val << '\n'; 
}
```
我们可以直接使用每个元素的键和值，名字（译注：即key和value）清晰的表示了它们的语义。

### 1.1 结构化绑定的细节




