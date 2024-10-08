---
title: C++ 内存模型 write_x_read_y 试例构造
date: 2022-07-29 18:32
---

之前一段时间偶然在 B 站上刷到了南京大学蒋炎岩（jyy）老师在直播操作系统网课。点进直播间看了一下发现这个老师实力非凡，上课从不照本宣科，而且旁征博引又不吝于亲自动手演示，于是点了关注。后来开始看其网课录播，其中一节的标题吸引了我，[多处理器编程：从入门到放弃 (线程库；现代处理器和宽松内存模型)](https://www.bilibili.com/video/BV13u411X72Q/?t=4429.0)。“多处理器编程”这个词让我联想到去年看的《The Art of Multiprocessor Programming》，于是仔细看了一下这节网课。里面介绍到了一个试例 write_x_read_y，它是用 C 语言和内联汇编写的，它用来说明运行期指令重排。这个试例能够成功观测到运行期指令重排现象。这让我不得不佩服 jyy 的实践精神。之前看了一些介绍 C++ 内存模型的文章，没有一个能用可复现的完整代码说明问题的，全部都是说这段代码可能出现 xx 结果，没有实际的执行结果。在 C++ 内存模型中，这个测试用例除了能够说明运行期指令重排，也能用于说明 happens-before consistency 和 sequential consistency 的差别。于是尝试用 C++ Atomic 来实现这段代码，看看能不能观测到预期结果。

首先线程库 pthread 替换为 std::thread，内联汇编替换为 std::atomic，且 load 和 store 操作全部使用最弱的 `std::memory_order_relaxed` 内存序。完整的代码如下：

```cpp
// write_x_read_y.cpp

#include <atomic>
#include <thread>
#include <stdio.h>

static std::atomic_int flag{0};

inline void wait_flag(int id)
{
    while (!(flag & (0x1 << id))) {}
}

inline void clear_flag(int id)
{
    flag.fetch_and(~(0x1 << id));
}

std::atomic_int x{0}, y{0};

void write_x_read_y()
{
    while (true) {
        wait_flag(0);

        x.store(1, std::memory_order_relaxed);    // t1.1
        int v = y.load(std::memory_order_relaxed); // t1.2
        printf("%d ", v);

        clear_flag(0);
    }
}

void write_y_read_x()
{
    while (true) {
        wait_flag(1);

        y.store(1, std::memory_order_relaxed);    // t2.1
        int v = x.load(std::memory_order_relaxed); // t2.2
        printf("%d ", v);

        clear_flag(1);
    }
}

int main()
{
    std::thread t1(write_x_read_y), t2(write_y_read_x);

    while (true) {
        x = 0, y = 0;
        flag = 0b11;

        while (flag) {}

        printf("\n");
        fflush(stdout);
    }

    t1.join();
    t2.join();
}
```

注意这段代码要开启代码优化才能观测到运行期指令重排，这里选择 O2
```bash
g++ -o write_x_read_y.out -O2 -pthread -std=c++11 -Wall -Wextra write_x_read_y.cpp
```

然后使用 jyy 视频里使用的 Unix 命令进行测试并整理结果

```bash
./write_x_read_y.out | head -n1000000 | sort | uniq -c
```

以下结果是在虚拟机环境中执行得到的。宿主机 CPU 型号为 AMD Ryzen 7 5800X，OS 为 Windows 10 x64，虚拟机是 Rocky Linux 8.6。

```
 948739 0 0 
  50150 0 1 
   1109 1 0 
      2 1 1
```

成功观测到“0 0”。假设程序按照简单交叉执行，执行结果只可能是“0 1”、“1 0”、“1 1”这三种，不可能出现“0 0”。也就是说发生了运行期指令重排。

接下来，将 `std::memory_order_relaxed` 替换为 `std::memory_order_release` 和 `std::memory_order_acquire`，再测一遍

```cpp
x.store(1, std::memory_order_release);    // t1.1
int v = y.load(std::memory_order_acquire); // t1.2
printf("%d ", v);

y.store(1, std::memory_order_release);    // t2.1
int v = x.load(std::memory_order_acquire); // t2.2
printf("%d ", v);
```

测试结果为：

```
 613684 0 0 
 360557 0 1 
  25757 1 0 
      2 1 1
```

又出现了“0 0”，也就说明这个试例无法区分 relaxed memory model 和 happens-before consistency。这也与理论相符，虽然 t1.1 happens-before t2.2、t2.1 happens-before t1.2，但是却无法借此推导出约束关系来限制执行结果。“0 0”依然有可能出现。

接下来替换为 `std::memory_order_seq_cst`

```cpp
x.store(1, std::memory_order_seq_cst);    // t1.1
int v = y.load(std::memory_order_seq_cst); // t1.2
printf("%d ", v);

y.store(1, std::memory_order_seq_cst);    // t2.1
int v = x.load(std::memory_order_seq_cst); // t2.2
printf("%d ", v);
```

测试结果为：

```
 132394 0 1 
    151 1 0 
 867455 1 1
```

这次“0 0”并没有出现，运行期指令重排没有被观测到。这与理论相符，使用 `std::memory_order_seq_cst` 的所有原子操作可以视为简单交叉执行，也就是 sequential consistency。“0 0”不可能出现。

write_x_read_y 这个试例很好地说明了 C++ 内存模型中的 happens-before consistency 和 sequential consistency 的区别。它的代码片段常见于各种相关文章中，却没有完整的代码和实际的测试结果。这下也算补全了 C++ 内存模型知识的一块拼图。
