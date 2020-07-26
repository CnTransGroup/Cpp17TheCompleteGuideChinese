# 第一章  结构化绑定

结构化绑定允许你使用对象的成员或者说元素来初始化多个变量。

举个例子，假如你定义了一个包含两个不同成员的结构：
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
使用结构化绑定的好处是可以直接通过名字访问值，并且由于名字可以传递语义信息，使得代码可读性也大大提高。

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
我们可以直接使用每个元素的键和值，key和value清晰的表示了它们的语义。

### 1.1 结构化绑定的细节
为了理解结构化绑定，了解其中设计的一个匿名变量是很重要的。结构化绑定引入的新名字都是指代的这个匿名变量的成员/元素的。

#### 绑定到匿名变量
初始化代码的最精确的行为：
```cpp
auto [u,v] = ms;
```
可以看成我们初始化一个匿名变量e，然后让结构化绑定u和v成为这个新对象的别名，类似下面：
```cpp
auto e = ms;
aliasname u = e.i;
aliasname v = e.s;
```
注意u和v不是`e.i`和`e.s`的引用。它们只是这两个成员的别名。因此，`decltype(u)`的类型与成员i的类型一致，`decltype(v)`的类型与成员s的类型一致。因为匿名变量e没有名字，所以我们不能直接访问这个已经初始化的变量。所以
```cpp
std::cout << u << ' ' << v << ✬\n✬;
```
输出`e.i`和`e.s`的值，它们是`ms.i`和`ms.s`的一份拷贝。

e和结构化绑定的存活时间一样长，当结构化绑定离开作用域时，e也会析构。

这样做的后果，除非使用引用，否则修改通过结构化绑定的值不会影响到初始化它的对象（反之亦然）：
```cpp
MyStruct ms{42,"hello"};
auto [u,v] = ms;
ms.i = 77;
std::cout << u;    // prints 42
u = 99;
std::cout << ms.i; // prints 77
```
u和`ms.i`地址是不一样的。

当对返回值使用结构化绑定的时候，上面的规则一样成立。下面代码的初始化：
```cpp
auto [u,v] = getStruct();
```
和我们使用`getStruct()`的返回值初始化匿名变量e，然后用u和v作为e的成员别名效果一样，类似下面：
```cpp
auto e = getStruct();
aliasname u = e.i;
aliasname v = e.s;
```
换句话说，结构化绑定将绑定到一个新的对象，它由返回值初始化，而不是直接绑定到返回值本身。

对于匿名变量e，内存地址和对齐也是存在的，以至于如果成员有对齐，结构化绑定也会有对齐。比如：
```cpp
auto [u,v] = ms;
assert(&((MyStruct*)&u)->s == &v); // OK
```
`((MyStruct*)&u)`会产生一个指向匿名变量的指针。

#### 使用修饰符
我们在结构化绑定过程中使用一些修饰符，如const和引用。再次强调，这些修饰符修饰的是匿名变量e。虽说是对匿名变量使用修饰符，但是通常也可以看作对结构化绑定使用修饰符，尽管存在一些额例外。

下面的例子中，我们对结构化绑定使用const引用：
```cpp
const auto& [u,v] = ms; // a reference, so that u/v refer to ms.i/ms.s
```
这里，匿名变量被声明为const引用，这意味着对ms使用const引用修饰，然后再将u和v作为i和s的别名。后续对ms成员的修改会直接影响到u和v：
```cpp
ms.i = 77;      // affects the value of u
std::cout << u; // prints 77
```
如果使用非const引用，你甚至可以通过对结构化绑定的修改，影响到初始化它的对象：
```cpp
MyStruct ms{42,"hello"};
auto& [u,v] = ms;       // the initialized entity is a reference to ms
ms.i = 77;              // affects the value of u
std::cout << u;         // prints 77
u = 99;                 // modifies ms.i
std::cout << ms.i;      // prints 99
```
如果初始化对象是临时变量，对它使用结构化绑定，此时临时值的生命周期会扩展：
```cpp
MyStruct getStruct();
...
const auto& [a,b] = getStruct();
std::cout << "a: " << a << '\n'; // OK
```



