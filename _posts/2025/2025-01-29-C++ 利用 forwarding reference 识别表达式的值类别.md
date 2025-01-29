---
title: C++ 利用 forwarding reference 识别表达式的值类别
date: 2025-01-29 13:03
---

forwarding reference 是 TAD(template argument deduction) 的一条特殊规则。当函数的模板参数是 `T&&` 时：
1. 如果传入的函数实参是 `lvalue`，则 `T` 为左值引用。
2. 如果传入的函数实参是 `rvalue`，则 `T` 为对应类型。
例如：

```cpp
template<class T>
void f(T&& v) {}

int main()
{
    f(11);
    int x = 1;
    f(x);
}
```

`f(11)` 的参数 `11` 是 rvalue，所以 `T` 直接为 `int`。这样 `T&&` 就恰为 `int&&`。右值引用 `int&&` 可以绑定右值 `11`。  
`f(x)` 的参数 `x` 是 lvalue，所以 `T` 为 `int&`。然后根据引用折叠原理，`T&&` 为 `int&`。左值引用 `int&` 可以绑定左值 `x`。

这样，在函数 `f(T&& v)` 内，`v` 的绑定的值类别可以根据 `T` 是否为左值引用进行判断。

```cpp
#include <type_traits>
#include <utility>

template<class T>
constexpr bool is_lvalue(T&& v)
{
    return std::is_lvalue_reference_v<T>;
}

template<class T>
constexpr bool is_rvalue(T&& v)
{
    return !is_lvalue(std::forward<T>(v));
}

int main()
{
    int x = 3;
    static_assert(is_lvalue(x));
    static_assert(is_rvalue(11));
}
```

这种方法只能区分 lvalue 和 rvalue，无法区分 xvalue。如果想还是[使用 decltype((x)) 来判断]({% post_url /2022/2022-01-20-C++ 利用模板偏特化和 decltype(()) 识别表达式的值类别 %})。