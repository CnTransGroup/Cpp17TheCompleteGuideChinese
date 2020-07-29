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

## 1.1 结构化绑定的细节
为了理解结构化绑定，了解其中设计的一个匿名变量是很重要的。结构化绑定引入的新名字都是指代的这个匿名变量的成员/元素的。

### 绑定到匿名变量
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

### 使用修饰符
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

### 修饰符并非修饰结构化绑定
如题，修饰符修饰的是匿名变量。它们没必要修饰结构化绑定。事实上：
```cpp
const auto& [u,v] = ms;  // a reference, so that u/v refer to ms.i/ms.s
```
u和v都没有声明为引用。上面只是对匿名变量e的引用。u和v的类型需要ms的成员一致。根据我们最开始的定义可以知道，`decltype(u)`是int，`decltype(v)`是`std::string`。

当指定对齐宽度的时候也有一些不同。
```cpp
alignas(16) auto [u,v] = ms;
```
在这里，我们将初始化后的匿名对象对齐而不是结构化绑定u和v。这意味着u作为第一个成员，被强制对齐到16位，而v不是。

同样的原因，尽管使用了auto，结构化绑定的类型也不会类型退化（术语退化（decay）描述的是当参数值传递的时候发生的类型转换，这意味着数组会转换为指针，最外面的修饰符如const和引用会被忽略）。例如，如果我们有一个包含多个原生数组的结构体：
```cpp
struct S{
    const char x[6];
    const char y[3];
};
```
然后
```cpp
S s1{};
auto [a, b] = s1; // a and b get the exact member types
```
a的类型仍然是`const char[6]`。原因仍然是修饰符并非修饰结构化绑定而是修饰初始化结构化绑定的对象。这一点和使用auto初始化新对象很不一样，它会发生类型退化：
```cpp
auto a2 = a;    // a2 gets decayed type of a
```

### 移动语义
即将介绍到，结构化绑定也支持移动语义。在下面的声明中：
```cpp
MyStruct ms = { 42, "Jim" };
auto&& [v,n] = std::move(ms);  // entity is rvalue reference to ms
```
结构化绑定v和n指向匿名变量中的成员，该匿名变量是ms的右值引用。ms仍然持有它的值:
```cpp
std::cout << "ms.s: " << ms.s << '\n'; // prints "Jim"
```
但是你可以移动赋值n，它与`ms.s`关联：
```cpp
std::string s = std::move(n); // moves ms.s to s
std::cout << "ms.s: " << ms.s << '\n'; // prints unspecified value
std::cout << "n: " << n << '\n'; // prints unspecified value
std::cout << "s: " << s << '\n'; // prints "Jim"
```
通常，移动后的对象的状态是有效的，只是包含了未指定的值（unspecified value）。因此，输出它的值是没有问题的，但是不能断言输出的东西一定是什么。

这一点和直接移动ms的值给匿名变量稍有不同：
```cpp
MyStruct ms = { 42, "Jim" };
auto [v,n] = std::move(ms); // new entity with moved-from values from ms
```
此时匿名对象是一个新对象，它用移动后的ms的值来初始化。所以ms失去了他们的值：
```cpp
std::cout << "ms.s: " << ms.s << '\n'; // prints unspecified value
std::cout << "n: " << n << '\n'; // prints "Jim"
```
你仍然可以移动n并赋值，或者用它赋予一个新的值，但是不会影响`ms.s`：
```cpp
std::string s = std::move(n); // moves n to s
n = "Lara";
std::cout << "ms.s: " << ms.s << '\n'; // prints unspecified value
std::cout << "n: " << n << '\n'; // prints "Lara"
std::cout << "s: " << s << '\n'; // prints "Jim"
```

## 1.2 结构化绑定可以在哪使用
原则上，结构化绑定可以用于公有成员，原始C-style数组，以及“似若tuple”的对象：
+ 如果结构体或者类中，所有非静态数据成员都是public，那么你可以使用结构化绑定来绑定非静态数据成员
+ 对于原生数组，你可以使用结构化绑定来绑定每个元素
+ 对于任何类型，你都可以使用似若tuple的API来进行绑定。对于类型type，API可以粗糙的概括为下列内容：
  + `std::tuple_size<type>::value`返回元素数量
  + `std::tupel_element<idx,type>::type`返回第idx个元素的类型
  + 一个全局的或者成员函数`get<idx>()`返回第idx个元素的值

如果结构体或者累提供这些似若tuple的API，那么就可以使用它们。

任何情况下都要求元素或者数据成员的数量必须匹配结构化绑定的名字的个数。你不能跳过任何一个元素，也不能使用同一个名字两次。但是你可以看使用非常段的名字如"_"（很多程序员倾向于用下划线，但是也有些人讨厌它，不允许它出现在全局命名空间中），但是在一个作用域它也只能出现一次：
```cpp
auto [_,val1] = getStruct(); // OK
auto [_,val2] = getStruct(); // ERROR: name _ already used
```
嵌套或者非平坦的对象分解是不支持的。（译注：指的是形如OCaml等语言的这种`let a,(b,c) = (3,(4,2));;`模式匹配能力）

接下来的章节讨论本节列表提到的各种情况。

### 1.2.1 结构体和类
到目前为止，已经演示了很多关于结构体和类的简单示例了。

如果类和结构体用到了继承，那么结构化绑定的使用就很受限了。所有非静态数据成员必须出现在同一个类。（换句话说，这些数据成员要么全是该类的，要么全是基类的）。
```cpp
struct B {
  int a = 1;
  int b = 2;
};

struct D1 : B {
};
auto [x, y] = D1{}; // OK

struct D2 : B {
  int c = 3;
};

auto [i, j, k] = D2{}; // Compile-Time ERROR
```

### 1.2.1 原生数组
下面的代码使用有两个元素的C-style数组初始化x和y：
```cpp
int arr[] = { 47, 11 };
auto [x, y] = arr; // x and y are ints initialized by elems of arr
auto [z] = arr; // ERROR: number of elements doesn’t fit
```
这种方式只能出现在数组长度已知的情况下。如果将数组作为参数传递，这样写就行不通，因为数组作为参数传递会发生类型退化，变成指针类型。

C++允许我们返回带长度的数组引用，如果有函数返回这种带长度的数组引用，那么也可以使用结构化绑定：
```cpp
auto getArr() -> int(&)[2]; // getArr() returns reference to raw int array
...
auto [x, y] = getArr(); // x and y are ints initialized by elems of returned array
```
你也可以对`std::array`使用结构化绑定，但是这需要使用似若tuple的API，这也是下一节的内容。

### 1.2.3 `std::paor`,`std::tuple`和`std::array`
结构化绑定是可扩展的，你可以为任何类型添加结构化绑定机制。标准库为`std::paor`,`std::tuple`和`std::array`都添加了该机制。

#### `std::array`
举个例子，下面的`getArray()`将返回四个元素的`std::array<>`，并用它初始化i，j，k和l。
```cpp
std::array<int,4> getArray();
...
auto [i,j,k,l] = getArray(); // i,j,k,l name the 4 elements of the copied return value
```
i，j，k和l分别绑定到`getArray()`返回的四个元素上。

写操作也是支持的，但这要求用来初始化结构化绑定的值不是一个临时的返回值：
```cpp
std::array<int,4> stdarr { 1, 2, 3, 4 };
...
auto& [i,j,k,l] = stdarr;
i += 10; // modifies std::array[0]
```

#### `std::tuple`
下面的代码使用`getTuple()`返回有三个元素的`std::tuple<>`来初始化a，b和c：
```cpp
std::tuple<char,float,std::string> getTuple();
...
auto [a,b,c] = getTuple();    // a,b,c have types and values of returned tuple
```

#### `std::pair`
另一个例子是处理关联型/无序型容器的`insert()`调用的返回值，使用结构化绑定使代码可读性更强，可以清晰的表达自己的意图，而不是依赖于`std::tuple`通用的first和second：
```cpp
std::map<std::string, int> coll;
...
auto [pos,ok] = coll.insert({"new",42});
if (!ok) {
  // if insert failed, handle error using iterator pos:
  ...
}
```
在C++17之前，必须使用下面的代码检查返回数据：
```cpp
auto ret = coll.insert({"new",42});
if (!ret.second){
  // if insert failed, handle error using iterator ret.first
  ...
}
```
注意，在这个例子中，C++17甚至还提供一种表达力更强的带初始化的if：

#### 为pair和tuple的结构化绑定赋值
在声明了结构化绑定之后，通常你不能一次性修改全部结构化绑定，因为结构化绑定是一次性声明所有而不是一次性使用所有。然而，如果重新赋的值是`std::pair<>`或者`std::tuple<>`那么你可以使用`std::tie()`。

也就是说，你可以写出下面的代码：
```cpp
std::tuple<char,float,std::string> getTuple();
...
auto [a,b,c] = getTuple(); // a,b,c have types and values of returned tuple
...
std::tie(a,b,c) = getTuple(); // a,b,c get values of next returned tuple
```
这种方式在实现循环调用且每次循环赋予一对返回值的过程中尤其有用，比如下面子啊循环中使用searcher的代码：
```cpp
std::boyer_moore_searcher bm{sub.begin(), sub.end()};
for (auto [beg, end] = bm(text.begin(), text.end());
  beg != text.end();
  std::tie(beg,end) = bm(end, text.end())) {
  ...
}
```

## 1.3
前面提到过，只要你的类型实现了似若tuple的API，那么就可以针对该类型使用结构化绑定，就和标准库的`std::pair<>`,`std::tuple<>`和`std::array<>`意义。

#### 只读结构化绑定
下面的代码展示了如何为类型Customer添加结构化绑定功能，Customer的定义如下
```cpp
// lang/customer1.hpp
#include <string>
#include <utility> // for std::move()
class Customer {
private:
  std::string first;
  std::string last;
  long val;
public:
  Customer (std::string f, std::string l, long v)
      : first(std::move(f)), last(std::move(l)), val(v) {
  }
  std::string getFirst() const {
    return first;
  }
  std::string getLast() const {
    return last;
  }
  long getValue() const {
    return val;
  }
};
```
我们可以提供似若tuple的API：
```cpp
// lang/structbind1.hpp
#include "customer1.hpp" #include <utility> // for tuple-like API
// provide a tuple-like API for class Customer for structured bindings:
template<>
struct std::tuple_size<Customer> {
  static constexpr int value = 3; // we have 3 attributes
};
template<>
struct std::tuple_element<2, Customer> {
  using type = long; // last attribute is a long
};
template<std::size_t Idx>
struct std::tuple_element<Idx, Customer> {
  using type = std::string; // the other attributes are strings
};
// define specific getters:
template<std::size_t> auto get(const Customer& c);
template<> auto get<0>(const Customer& c) { return c.getFirst(); }
template<> auto get<1>(const Customer& c) { return c.getLast(); }
template<> auto get<2>(const Customer& c) { return c.getValue(); }
```
代码Customer有三个成员，还有为三个成员准备的getter：
+ 表示first name的成员，`std::string`类型
+ 表示last nane的成员，`std::string`类型
+ 表示value的成员，long类型

获取Customer成员个数的函数是`std::tuple_size`的特化：
```cpp
template<>
struct std::tuple_size<Customer> {
  static constexpr int value = 3; // we have 3 attributes
};
```
获取成员类型的函数是`std::tuple_element`的特化：
```cpp
template<>
struct std::tuple_element<2, Customer> {
  using type = long; // last attribute is a long
};
template<std::size_t Idx>
struct std::tuple_element<Idx, Customer> {
  using type = std::string; // the other attributes are strings
};
```
第三个成员类型是long，需要为它（index 2）编写全特化代码。其它成员是`std::stinrg`类型，部分特化（比全特化优先级低）即可。