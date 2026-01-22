---
title: requires
date: 2026-01-19 14:46:52
tags:
	- 笔记
categories:
	- C++
	- C++特性
	- C++20特性
    - requires
---

# 1.函数级别的 requires 约束
## （1）核心语法（3种）
### <1>基础格式（直接跟约束表达式）
这是最常用的写法，适用于普通函数 / 模板函数 / 成员函数
```cpp
// 模板函数 + 函数级requires
template <typename T>
返回值类型 函数名(参数列表) requires 编译期布尔表达式 {
    // 函数体
}

// 类模板的成员函数 + 函数级requires
template <typename T>
class ClassName {
public:
    返回值类型 成员函数名(参数列表) requires 编译期布尔表达式;
};

// 普通函数（非模板）也可加requires（C++20支持，但仅当表达式为true时函数存在）
返回值类型 普通函数名(参数列表) requires 编译期布尔表达式 {
    // 函数体
}
```

### <2>精简格式（模板参数列表后 + 函数级双重约束）
若模板本身有约束，可在模板参数后加 requires，函数级再叠加约束（逻辑与）
```cpp
template <typename T> requires 约束1  // 模板级约束
T add(T a, T b) requires 约束2 {     // 函数级约束（需同时满足约束1+约束2）
    return a + b;
}
```

### <3>括号格式（复杂约束用括号提升可读性）
多个约束组合时，用括号包裹表达式，逻辑更清晰
```cpp
template <typename T>
void print(T val) requires (约束1) && (约束2) {
    std::cout << val << std::endl;
}
```

## （2）常用的 “编译期布尔表达式”（约束条件）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119161844015.png)

## （3）实战基础示例
### <1>模板函数的函数级 requires
```cpp
#include <iostream>
#include <type_traits>

// 约束：T必须是算术类型（int/float/double等）
template <typename T>
T sum(T a, T b) requires std::is_arithmetic_v<T> {
    return a + b;
}

int main() {
    // 合法：int是算术类型
    std::cout << sum(10, 20) << std::endl;  // 输出30
    
    // 合法：double是算术类型
    std::cout << sum(3.14, 2.5) << std::endl;  // 输出5.64
    
    // 编译报错：std::string不是算术类型，约束不满足
    // std::string s1 = "hello", s2 = "world";
    // std::cout << sum(s1, s2) << std::endl;
    
    return 0;
}
```

### <2>类模板成员函数的函数级 requires
```cpp
#include <iostream>
#include <type_traits>
#include <string>

template <typename T>
class Wrapper {
private:
    T value;
public:
    Wrapper(T val) : value(val) {}

    // 约束：T是void类型时，才允许调用这个无参数print
    void print() requires std::is_same_v<T, void> {
        std::cout << "Value: void (no data)" << std::endl;
    }

    // 约束：T不是void且可打印（支持<<），才允许调用这个print
	void print() requires (!std::is_same_v<T, void>) && 
	    requires(std::ostream& os, T val) { os << val; }  // 这整个是约束表达式
	{  // 这才是函数体的开始
	    std::cout << "Value: " << value << std::endl;
	}
};

int main() {
    // T=void，调用第一个print
    Wrapper<void> w1;
    w1.print();  // 输出：Value: void (no data)

    // T=int，调用第二个print
    Wrapper<int> w2(100);
    w2.print();  // 输出：Value: 100

    // T=string，调用第二个print
    Wrapper<std::string> w3("hello");
    w3.print();  // 输出：Value: hello

    return 0;
}
```

### <3>普通函数的函数级 requires（少见但合法）
```cpp
#include <type_traits>

// 仅当编译期条件为true时，该函数才存在
void func() requires (sizeof(int) == 4) {
    std::cout << "int is 4 bytes" << std::endl;
}

int main() {
    // 若当前平台int是4字节，调用成功；否则编译报错
    func();
    return 0;
}
```