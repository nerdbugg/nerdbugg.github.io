---
title: '一个vector遍历引发的惨案'
description: 在vector遍历代码中遇到的unsigned坑点
slug: cpp-unsigned-bug
date: 2023-05-04 14:36:07+08:00
# image: cover.jpg
categories:
    - cpp
tags:
    - cpp
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 起因

事情的起因是这样的：某日在用C++的特性写些小demo时，同组的同学看到我在遍历vector后，向我介绍了一个他曾经很费解的Bug。

## 问题

我写的遍历是这样的：

```cpp
std::vector<int> vec{1,2,3,4,5};
for(int i=0;i<vec.size();i++) {
        fmt::print("{} ", vec[i]);
}
```

而同学的写法是这样的：

```cpp
std::vector<int> vec{1,2,3,4,5};
for(int i=0;i<=vec.size()-1;i++) {
        fmt::print("{} ", vec[i]);
}
```

看上去没问题，只是比较条件略做了变化，而且输出也很正常。但是，略作改动之后：

```cpp
std::vector<int> vec{1,2,3,4,5};
vec.clear(); // set vec size to 0
for(int i=0;i<=vec.size()-1;i++) {
        fmt::print("{} ", vec[i]);
}
```

这时，程序的输出就很奇怪了：在我的电脑上，本应没有输出的屏幕出现了大段的输出，并最终因为段错误而终止。而原本的写法没有问题。

## 分析

所以，执行这段代码时究竟发生了什么？两段代码的改动仅在比较条件上，通过查询cppreference可知， `vec.size()` 的返回值类型为 `size_type`，其定义为g++内部定义的宏 `__SIZE_TYPE__`，再通过

```bash
echo | g++ -dM -E -x c++ -|nvim -
```

查看g++内部定义宏可看到其类型为 `long unsigned int` ，也就是这里对 `unsinged` 的减法操作，且理论上得到的结果-1是 `unsigned` 无法表示的，那C标准是如 何对这个行为定义的呢？

根据stackoverflow上的回答[1]：从C语言的语义角度理解，该算式的结果-1将会与 `UINT_MAX+1`进行取模，得到 `UINT_MAX` 。所以， `0≤UINT_MAX` 的条件满足，程序根据偏移进行了非法的内存访问，并最终因为访问了无权限的内存页触发段错误终止进程。

这段程序带来的安全隐患是切实存在的。想象一下在一个后端服务的处理函数中使用了这段程序，而栈上的变量里又存储了历史其他用户请求的数据。那么攻击者就可以构造一个使vector大小为0的输入触发该bug，进而泄漏栈上变量的值。最差的情况下，也能导致服务触发段错误进而不可用。

## 总结

随手一写的简单代码，也有可能在意想不到的地方违背上层的语义。为了避免这样的情况发生，遵循一些常见的准则或者best practice是必要的。

在这个例子中，我们至少应该避免对 `unsigned` 类型进行减法，以及避免 `signed` 和 `unsigned` 类型之间的比较。

对于vector的使用，我们可以采取更加不易出错的遍历方法，比如range based loop或者使用迭代器进行遍历，并使用 `vec.at(i)` 而不是 `vec[i]` 避免内存越界问 题。

```cpp
for(auto item:vec) {item...}
for(auto itr=vec.begin();itr!=vec.end();itr++) {*itr...}
for(long unsinged int i=0;i<vec.size();i++) {vec.at(i)...}
```

同时，在写出上述存在内存越界问题的程序的全过程中，编译器以及Lsp没有给出任 何警告和提示，C/C++确实容易写出内存不安全的程序，尤其是语言掌握并不熟练的 人。

但是，仍然有一些工具能够辅助程序员避免这些问题（只是语言不强制使用）

- -Wall, -Wextra提供了额外的warning，对可能存在问题的代码在编译阶段报warning
- Address santinizer，可以在运行时动态检查内存越界行为，在运行时报错提供信息并终止程序

[1] [https://stackoverflow.com/questions/7221409/is-unsigned-integer-subtraction-defined-behavior](https://stackoverflow.com/questions/7221409/is-unsigned-integer-subtraction-defined-behavior)

[2] [Range-based for loop - cppreference](https://en.cppreference.com/w/cpp/language/range-for)

