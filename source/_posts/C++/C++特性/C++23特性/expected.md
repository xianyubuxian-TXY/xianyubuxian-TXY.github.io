---
title: std::expected<T, E>
date: 2026-01-19 17:34:03
tags:
	- 笔记
categories:
	- C++
	- C++特性
	- C++23特性
	- std::expected<T, E>
---

# 一、概述
- **std::expected<T, E>** 是 C++23 标准库提供的带错误处理的返回值类型，定义在 <expected> 头文件中，核心目的是：
	- 替代传统的 “返回值 + 错误码”“异常” 等错误处理方式，在不抛异常的前提下，优雅地封装 “成功结果” 或 “错误信息”；
	- 强制开发者显式处理错误（避免忽略错误码），同时保持代码的可读性和类型安全。
- 简单理解：std::expected<T, E> 实例**要么包含一个有效的 T 类型成功值，要么包含一个E 类型的错误值，二者只能存其一**（类似 Rust 的 Result<T, E>）。

# 二、核心语法与基础用法
## 1.基础声明与初始化
```cpp
#include <expected>   // 必须包含的头文件
#include <string>     // 示例用：错误类型常用std::string/枚举/自定义错误

// 声明：T=成功值类型，E=错误类型（推荐E为可拷贝/可移动的类型）
std::expected<int, std::string> func(int a);

// 初始化方式1：返回成功值（用std::expected构造或std::make_expected）
std::expected<int, std::string> success() {
    return 100;  // 隐式构造：成功值
    // 等价写法：return std::make_expected<int, std::string>(100);
}

// 初始化方式2：返回错误值（用std::unexpected包裹）
std::expected<int, std::string> fail() {
    return std::unexpected("参数错误");  // 错误值需用std::unexpected包裹
    // 等价写法：return std::make_unexpected(std::string("参数错误"));
}
```


## 2.核心成员函数（判断+取值）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119173916337.png)

## 3.基础使用示例
```cpp
#include <expected>
#include <string>
#include <iostream>

// 示例函数：除法运算，成功返回int，失败返回错误信息
std::expected<int, std::string> divide(int a, int b) {
    if (b == 0) {
        return std::unexpected("除数不能为0");
    }
    return a / b;
}

int main() {
    // 调用函数并处理结果
    auto res1 = divide(10, 2);
    if (res1.has_value()) {
        std::cout << "成功：" << res1.value() << std::endl;  // 输出：成功：5
    } else {
        std::cout << "失败：" << res1.error() << std::endl;
    }

    auto res2 = divide(10, 0);
    // 安全取值：失败时返回默认值-1
    std::cout << "结果：" << res2.value_or(-1) << std::endl;  // 输出：结果：-1

    return 0;
}
```

## 4.自定义错误类型（更丰富的错误信息）
```cpp
// 自定义错误类型（包含错误码和消息）
struct ServiceError {
    ErrorCode code;
    std::string message;
    
    static ServiceError UserNotFound(const std::string& msg = "用户不存在") {
        return {ErrorCode::UserNotFound, msg};
    }
    
    static ServiceError DatabaseError(const std::string& msg = "数据库错误") {
        return {ErrorCode::DatabaseError, msg};
    }
};

// 使用
std::expected<User, ServiceError> GetUser(int64_t id) {
    if (id <= 0) {
        return std::unexpected(ServiceError::UserNotFound("无效的用户ID"));
    }
    // ...
}
```