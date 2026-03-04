---
title: constexpr
date: 2026-02-06 10:16:09
tags:
	- 笔记
categories:
    - C++
    - C++特性
    - C++11特性
    - constexpr
---


# 一、constexpr介绍
****

## 1.constexpr 核心定义
- constexpr 是 C++11 引入的关键字，核心作用是**强制表达式 / 对象在编译期计算 / 初始化**，生成 “**编译期常量**”，而非运行期常量
	- 表示**编译期常量表达式**，要求**值在编译时就能确定**。
	- 编译期计算的优势：
		- 减少运行时开销
		- 支持编译期分支、类型推导
		- 可用于模板参数、数组大小
		- 等 **“必须编译期确定” 的场景**。
```cpp
constexpr int square(int x) { return x * x; }

constexpr int a = 10;           // ✅ 编译期确定
constexpr int b = square(5);    // ✅ 编译期计算，b = 25
constexpr int c = a + b;        // ✅ 编译期计算，c = 35
```

## 2. 典型应用场景（为什么要用 constexpr）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260206103428911.png)

# 二、constexpr 与 const 的核心区别
- 很多人误以为 constexpr 是 const 的“升级版”，但**二者设计目标完全不同**：
- **一句话总结区别**:
	- **const**：告诉程序“这个变量**只读**，不许改”（运行时约束）；
	- **constexpr**：告诉编译器“**这个值必须在编译期算出来**，我要拿它当常量用”（编译期约束）。
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260206104020760.png)
```cpp
// ============================== 核心目标差异 ==============================
// constexpr: 编译期常量
constexpr int SIZE = 100;      // 编译器在编译时就确定值为 100

// const: 运行期只读
int getValue();
const int READONLY = getValue();  // 运行时才知道值，但之后不可修改


// ============================== 是否编译期确定 ==============================
constexpr int a = 10;          // ✅ 必须编译期确定
// constexpr int b = rand();   // ❌ 编译错误！编译器强制检查

const int c = 10;              // ✅ 编译期常量
const int d = rand();          // ✅ 运行期只读变量（也合法）



// ============================== 可推导性 ==============================
constexpr int x = 5 + 3;       // 编译器知道 x = 8，可优化

const int y = someFunc();      // 编译器只知道 y 只读
                               // 不知道具体值，无法优化


// ============================== 反向兼容 ==============================
constexpr int ce = 42;
const int c1 = ce;             // ✅ constexpr → const 可以

const int c2 = 42;
// constexpr int ce2 = c2;     // ❌ const → constexpr 不行
                                // 即使 c2 实际是编译期常量！

// 原因：编译器对 const 变量不做"编译期常量"保证
//       它可能是运行期初始化的
```

# 三、constexpr 函数
- **热点函数**用 constexpr，把计算提前到编译期，降低运行时开销
- **constexpr 不是“性能银弹”**：
	- 只有频繁调用的小函数/常量用 constexpr 才有意义
	- 复杂函数编译期计算会增加编译时间
```cpp
// ==================== 场景：频繁调用的小函数 ====================

// ❌ 普通函数：每次运行时计算
int power_runtime(int base, int exp) {
    int result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    return result;
}

// ✅ constexpr 函数：编译期参数时直接内嵌结果
constexpr int power_constexpr(int base, int exp) {
    int result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    return result;
}

int main() {
    // 场景1：常量参数 → 编译期计算（零运行时开销）
    constexpr int SIZE = power_constexpr(2, 10);  // 编译期 = 1024
    int arr[SIZE];  // 数组大小编译期确定
    
    // 场景2：循环中使用常量
    for (int i = 0; i < 1000000; ++i) {
        // ❌ 每次循环都计算
        // int val = power_runtime(2, 16);
        
        // ✅ 编译期已算好，直接用 65536
        constexpr int val = power_constexpr(2, 16);
        // ...
    }
}
```
```cpp
// ==================== ❌ 不适合：复杂/大型函数 ====================
constexpr std::string processData(/* 大量参数 */) {
    // 复杂逻辑...
    // 编译期计算会导致：
    // 1. 编译时间大幅增加
    // 2. 编译器内存占用增加
    // 3. 可能超出编译器递归/计算限制
}

// ==================== ❌ 不适合：运行时才知道的参数 ====================
constexpr int compute(int x) { return x * x; }

int userInput;
std::cin >> userInput;
int result = compute(userInput);  // 运行时计算，constexpr 没意义

// ==================== ✅ 适合场景 ====================
// 1. 小型、频繁调用的计算函数
// 2. 配置常量、数组大小
// 3. 编译期查找表
// 4. 模板元编程辅助函数
```