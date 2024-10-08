---
title: longjmp 使 C++ RAII 失效
date: 2022-04-22 11:05
---

C 语言的 `longjmp` 没有进行栈展开，而是直接跳转。从 `longjmp` 到 `setjmp` 之间的所有析构函数都没有调用，也就是 `RAII` 失效。
```cpp
#include <setjmp.h>
#include <iostream>

jmp_buf buf;

class A
{
public:
    A(int x) : x(x) {}
    ~A() { std::cout << "~A x=" << x << std::endl; }
private:
    int x;
};

void f1(), f2(), f3();

void f1()
{
    A a(1);
    if (setjmp(buf) == 0) {
        f2();
    }
}

void f2()
{
    A a(2);
    f3();
}

void f3()
{
    A a(3);
    longjmp(buf, 1);
}

int main()
{
    A a(0);
    f1();
}
```
这段代码输出
```
~A x=1
~A x=0
```