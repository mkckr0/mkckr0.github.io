---
title: C++ happens-before 关系是不可传递的
date: 2021-12-22 06:53
---

[P0668R4](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0668r4.html) 对此进行了解释

> The definition of plain happens-before became unpleasantly complicated with the introduction of `memory_order_consume`. And it is not transitive, which remains counterintuitive. This proposal changes neither of those. And if the user refrains from using `memory_order_consume` it can continue to be entirely ignored, as before. Until we have a usable version of `memory_order_consume`, I would expect teaching materials to ignore these issues, and pretend that happens-befors is defined as our simply-happens-before relation, which is clearly transitive. In the presence of `memory_order_consume`, this problem is unavoidable, since consume can order two accesses without also ordering the first with respect to an access that immediately follows the second; happens-before cannot compose with sequenced-before, and thus happens-before cannot be transitive.

在不使用 `memory_order_consume` 的情况下，happens-before 可以视为可传递，否则为不可传递。C++17 提出了 `strongly happens before`，排除了 `memory_order_consume`，是可传递的，可以和 sequenced-before 进行组合。
