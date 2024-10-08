---
title: C++ std::initializer_list 实现原理勘误
date: 2022-02-01 11:06
---

今天正在看侯捷《C++ 新标准 C++11-14》的视频，里面讲到 `std::initializer_list` 的实现原理，并且把源码贴出来。
```cpp
  /// initializer_list
  template<class _E>
    class initializer_list
    {
    public:
      typedef _E 		value_type;
      typedef const _E& 	reference;
      typedef const _E& 	const_reference;
      typedef size_t 		size_type;
      typedef const _E* 	iterator;
      typedef const _E* 	const_iterator;

    private:
      iterator			_M_array;
      size_type			_M_len;

      // The compiler can call a private constructor.
      constexpr initializer_list(const_iterator __a, size_type __l)
      : _M_array(__a), _M_len(__l) { }

    public:
      constexpr initializer_list() noexcept
      : _M_array(0), _M_len(0) { }

      // Number of elements.
      constexpr size_type
      size() const noexcept { return _M_len; }

      // First element.
      constexpr const_iterator
      begin() const noexcept { return _M_array; }

      // One past the last element.
      constexpr const_iterator
      end() const noexcept { return begin() + size(); }
    };
```
他认为，构造 `std::initializer_list` 之前编译器会先构造一个 `std::array`，然后使用 `std::array` 的 `begin()` 和 `size()` 构造 `std::initializer_list`。这种说法有一处错误。编译器不会构造 `std::array`，而是在栈上直接构造一个数组 `const T[N]`。在栈上构造的数组会像其他变量一样，在离开作用域时自动析构，不需要手动管理内存。`std::array` 也是如此，它仅在其基础之上做了一层包装，使数组的行为如同其它容器一样。所以根本没必要使用 `std::array`，直接使用数组就足够了。

这个是 [cppreference.com](https://en.cppreference.com/w/cpp/utility/initializer_list) 的描述：
> The underlying array is a temporary array of type `const T[N]`

明确地说是普通的 `array`。

这个是 [N3337](https://wg21.link/std11) 的描述：
> An object of type `initializer_list<E>` provides access to an array of objects of type `const E`.

并没有说是 `std::array`。
