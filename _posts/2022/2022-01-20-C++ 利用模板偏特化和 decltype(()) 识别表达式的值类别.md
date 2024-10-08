---
title: C++ 利用模板偏特化和 decltype(()) 识别表达式的值类别
date: 2022-01-20 00:43
---

刚刚看到一篇 C++ 博客，里面讲到用模板偏特化和 `decltype()` 识别值类别：`lvalue` `glvalue` `xvalue` `rvalue` `prvalue`。依照博客的方法试了一下，发现根本行不通。之后，我查阅了一下 [cppreference.com](https://en.cppreference.com/w/cpp/language/decltype) 关于 `decltype` 关键字的描述，发现了 `decltype((表达式))` 具有以下特性：
- 如果 表达式 的值类别是 `xvalue`，`decltype` 将会产生 `T&&`；
- 如果 表达式 的值类别是 `lvalue`，`decltype` 将会产生 `T&`；
- 如果 表达式 的值类别是 `prvalue`，`decltype` 将会产生 `T`。

也就是可以细分 `xvalue` 和 `lvalue`，于是尝试将模板偏特化和 `decltype(())` 结合，发现这种方法可行。
```cpp
#include <iostream>
#include <type_traits>

template<typename T> struct is_lvalue : std::false_type {};
template<typename T> struct is_lvalue<T&> : std::true_type {};

template<typename T> struct is_xvalue : std::false_type {};
template<typename T> struct is_xvalue<T&&> : std::true_type {};

template<typename T> struct is_glvalue : std::integral_constant<bool, is_lvalue<T>::value || is_xvalue<T>::value> {};
template<typename T> struct is_prvalue : std::integral_constant<bool, !is_glvalue<T>::value> {};
template<typename T> struct is_rvalue : std::integral_constant<bool, !is_lvalue<T>::value> {};

struct A
{
    int x = 1;
};

int main()
{
    A a;

    std::cout << std::boolalpha
    << is_lvalue<decltype(("abcd"))>::value << std::endl
    << is_glvalue<decltype(("abcd"))>::value << std::endl
    << is_xvalue<decltype(("abcd"))>::value << std::endl
    << is_rvalue<decltype(("abcd"))>::value << std::endl
    << is_prvalue<decltype(("abcd"))>::value << std::endl
    << std::endl
    << is_lvalue<decltype((a))>::value << std::endl
    << is_glvalue<decltype((a))>::value << std::endl
    << is_xvalue<decltype((a))>::value << std::endl
    << is_rvalue<decltype((a))>::value << std::endl
    << is_prvalue<decltype((a))>::value << std::endl
    << std::endl
    << is_lvalue<decltype((A()))>::value << std::endl
    << is_glvalue<decltype((A()))>::value << std::endl
    << is_xvalue<decltype((A()))>::value << std::endl
    << is_rvalue<decltype((A()))>::value << std::endl
    << is_prvalue<decltype((A()))>::value << std::endl
    << std::endl
    << is_lvalue<decltype((A().x))>::value << std::endl
    << is_glvalue<decltype((A().x))>::value << std::endl
    << is_xvalue<decltype((A().x))>::value << std::endl
    << is_rvalue<decltype((A().x))>::value << std::endl
    << is_prvalue<decltype((A().x))>::value << std::endl
    ;
}
```
输出
```sh
true
true
false
false
false

true
true
false
false
false

false
false
false
true
true

false
true
true
true
false
```
所有的输出结果都符合预期。
