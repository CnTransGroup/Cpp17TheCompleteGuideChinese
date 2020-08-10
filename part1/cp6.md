# 第六章 Lambda扩展
C++11引入了lambda，C++14引入了泛型lambda，这是一个成功的故事。lambda允许我们将功能指定为参数，这让定制函数的行为变得更加容易。

C++ 17进一步改进，允许lambda用在更多的地方。

## 6.1 constexpr lambda
自C++17后，只要可能，lambda就隐式地用constexpr修饰。也就是说，任何lambda都可以用于编译时上下文，前提是它使用的特性对编译时上下文有效（例如，仅字符串字面值，无静态变量，无virutal变量，无try/catch，无new/delete）。
