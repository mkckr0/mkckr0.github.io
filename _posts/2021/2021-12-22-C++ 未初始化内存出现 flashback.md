---
title: C++ 未初始化内存出现 flashback
date: 2021-12-22 09:09
---

在 C++ 中分配一个未初始化内存，然后读取它，会读取到这块内存之前被使用所留下的值，这种现象我称之为 flashback。

1. 栈内存很容易出现这种现象，而且很容易观测出某种规律。
```cpp
for (int i = 0; i < 10; ++i) {
    int a;
    std::cout << a << " ";
    a = i;
}
```
这段代码可能输出
```
0 0 1 2 3 4 5 6 7 8 
```
除了第一个 0，其余的 0 1 2 3 4 5 6 7 8 都是 flashback 的结果

2. 堆内存也会出现这种现象，但是很难观测出规律。
```cpp
struct A
{
    int8_t m1[13];
    int x;
};

for (int i = 0; i < 10; ++i) {
    A* a = new A;
    std::cout << a->x << " ";
    a->x = i;
    delete a;
}
std::cout << std::endl;
```
这段代码仍然可能输出
```
0 0 1 2 3 4 5 6 7 8 
```
除了第一个 0，其余的 0 1 2 3 4 5 6 7 8 都是 flashback 的结果。

在实际的业务逻辑代码中，new 操作可能深埋在复杂代码之中，并且不同对象的 new 操作也会相互影响。

```cpp
struct A
{
    int8_t m1[13];
    int x;
};

struct B
{
    int8_t m1[13];
    int x;
};

// cs1
A* a1 = new A;
a1->x = 66;
delete a1;

// cs2
/*
B* b1 = new B;
b1->x = 22;
delete b1;
*/

// cs3
A* a2 = new A;
delete a2;

std::cout << a1 << " " << a2 << " " << std::boolalpha << (a1 == a2) << " " << a2->x << std::endl;
```

这段代码可能输出
```
0x1b05eb0 0x1b05eb0 true 66
```
成功观测到了 flashback

把 cs2 的注释解开，可能输出
```
0x1718eb0 0x1718eb0 true 22
```

假设 cs2 的执行次数是随机的，或者 `b1->x = 22` 的 22 是随机的，并且只观测 a1 和 a2 的关系，那么观测到 flashback 的次数也是随机的。
