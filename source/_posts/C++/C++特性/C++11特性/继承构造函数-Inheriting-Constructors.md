---
title: 继承构造函数(using Base::Base)
date: 2026-01-23 15:47:22
tags:
	- 笔记
categories:
    - C++
    - C++特性
    - C++11特性
    - 继承构造函数(using Base::Base)
---

# 1.示例
```cpp
// ==================== common/exception/base.h ====================
#pragma once
#include <stdexcept>
#include <string>

namespace common::exception {

// 所有业务异常的基类
class AppException : public std::runtime_error {
public:
    explicit AppException(const std::string& msg) : std::runtime_error(msg) {}
};

// 客户端异常基类（用户的问题，可以告诉用户具体原因）
class ClientException : public AppException {
public:
    using AppException::AppException;
};

// 服务端异常基类（我们的问题，只告诉用户"系统繁忙"）
class ServerException : public AppException {
public:
    using AppException::AppException;
};

} // namespace common::exception
```
# 2.using Base::Base 继承构造函数
- **using Base::Base; = "基类有什么构造函数，我就有什么构造函数"**

## （1）作用
using AppException::AppException; 的意思是：**把基类的所有构造函数"继承"到派生类中**，让派生类可以直接使用基类的构造函数。

## （2）对比理解
**不用 using（传统写法）**
```cpp
class ClientException : public AppException {
public:
    // 必须手动定义构造函数，逐个转发给基类
    explicit ClientException(const std::string& msg) : AppException(msg) {}
    explicit ClientException(const char* msg) : AppException(msg) {}
};
```

**用 using（简化写法）**
```cpp
class ClientException : public AppException {
public:
    // 一行搞定，自动继承基类的所有构造函数
    using AppException::AppException;
};
```