---
title: C++关键字
date: 2026-01-18 09:54:09
tags:
	- 笔记
categories:
	- C++
	- C++特性
	- C++关键字
---

# 1.explicit
## （1）核心作用
- **阻止隐式类型转换**，强制要求显式调用，避免编译器"自作聪明"地进行意外转换。
- explicit 最初是为**单参数构造函数**设计的，但 C++11 之后**扩展了它的用途**。

## （2）使用场景汇总
### <1>单参数构造函数（最经典）
```cpp
class String {
public:
    // ❌ 不加 explicit
    String(int size);  // 可能导致：String s = 42; 意外编译通过
    
    // ✅ 加 explicit
    explicit String(int size);  // 必须：String s(42); 或 String s{42};
};

// 隐患示例
void print(const String& s);
print(42);  // ❌ 不加 explicit：编译通过，42 被隐式转成 String
print(42);  // ✅ 加 explicit：编译错误，必须 print(String(42))
```
### <2>带默认参数的多参数构造函数
```cpp
class Buffer {
public:
    // 本质上可以只传一个参数调用
    explicit Buffer(size_t size, char fill = '\0');
};

Buffer buf = 1024;       // ❌ explicit 阻止
Buffer buf(1024);        // ✅ OK
Buffer buf{1024, 'x'};   // ✅ OK
```

### <3>类型转换运算符（C++11）
```cpp
class SmartPtr {
    void* ptr_;
public:
    // ❌ 不加 explicit —— 危险
    operator bool() const { return ptr_ != nullptr; }
    
    // ✅ 加 explicit —— 安全
    explicit operator bool() const { return ptr_ != nullptr; }
    // explicit operator int() const {...}

};

SmartPtr p;

// 不加 explicit 的隐患
int x = p + 1;      // ❌ 编译通过！p → bool → int
if (p == 1) {}      // ❌ 编译通过！意外的比较
std::cout << p;     // ❌ 编译通过！输出 0 或 1

// 加 explicit 后
int x = p + 1;      // ✅ 编译错误
if (p) {}           // ✅ 布尔上下文允许
bool b = static_cast<bool>(p);  // ✅ 显式转换
```

### <4>初始化列表构造函数（C++11）
```cpp
class Vector {
public:
    explicit Vector(std::initializer_list<int> list);
};

Vector v1 = {1, 2, 3};   // ❌ explicit 阻止拷贝初始化
Vector v2{1, 2, 3};      // ✅ 直接初始化 OK
Vector v3({1, 2, 3});    // ✅ OK
```