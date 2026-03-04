---
title: mutable
date: 2026-02-06 14:55:02
tags:
	- 笔记
categories:
	- C++
	- C++特性
	- C++98基础特性
	- mutable
---

# 一、mutable 核心定义
- mutable 是 C++ 关键字，专门用来突破 **const成员函数**的 “只读限制” —— 标记为 mutable 的成员变量，即使在 const 成员函数中，也可以被修改。
- 简单说：
	- **普通成员变量**：在 const 成员函数中不能改（编译器强制只读）；
	- **mutable 成员变量**：在 const 成员函数中可以改（唯一例外）。

**反例（无 mutable，编译报错）**
```cpp
class KafkaProducer {
private:
    // 无 mutable 修饰
    std::atomic<uint64_t> messages_sent_{0};

public:
    // 这个函数标记为 const（表示“不修改对象核心状态”）
    uint64_t GetMessagesSent() const {
        // ❌ 编译报错：const 函数中不能修改普通成员变量
        // 即使是 atomic 的自增，也会被编译器判定为“修改”
        messages_sent_++; 
        return messages_sent_;
    }
};
```

**正例（加 mutable，编译通过）**
```cpp
class KafkaProducer {
private:
    // 加 mutable 修饰：允许 const 函数中修改
    mutable std::atomic<uint64_t> messages_sent_{0};
    mutable std::atomic<uint64_t> messages_failed_{0};
    mutable std::atomic<uint64_t> bytes_sent_{0};

public:
    // const 函数：表示“不修改业务状态，但可以修改计数器”
    uint64_t GetMessagesSent() const {
        // ✅ 合法：mutable 变量不受 const 限制
        return messages_sent_;
    }

    // 即使是 const 函数，也能更新计数器（比如统计发送结果）
    void OnMessageSent(size_t bytes) const {
        messages_sent_++;       // 记录发送成功数
        bytes_sent_ += bytes;   // 记录发送字节数
    }

    void OnMessageFailed() const {
        messages_failed_++;     // 记录发送失败数
    }
};
```

# 二、mutable 的核心规则
- **仅适用于类的非静态成员变量**：
	- 不能修饰全局变量、静态变量、函数、参数等；
- **突破 const 函数的只读限制**：
	- 这是它的核心用途，也是唯一合法用途；
- **不改变对象的 “逻辑常量性”**：
	- mutable 用于修改 “不影响对象逻辑状态” 的变量（比如计数器、缓存、锁），而非对象的核心业务数据；
- **和 atomic 结合是常见最佳实践**：
	- 代码里的 mutable std::atomic<> 是工业级写法。
- **mutable 的限制**:
```cpp
// ❌ 不能与 const 一起用
mutable const int x;  // 矛盾，编译错误

// ❌ 不能与 static 一起用  
mutable static int y; // 编译错误

// ❌ 不能修饰引用
mutable int& ref;     // 编译错误
```


# 三、mutable 的典型使用场景
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260206150147454.png)

# 四、mutable + atomic
1. **atomic 保证线程安全**：std::atomic 是原子类型，自增/赋值等操作是线程安全的，适合多线程环境下的计数器（Kafka 生产者通常是多线程的）；
2. **mutable 保证 const 兼容**：计数器的更新通常在 const 成员函数中（比如统计类的查询接口），mutable 允许这种修改；
3. **符合“逻辑常量性”**：这些计数器是“辅助数据”，修改它们不会改变 KafkaProducer 对象的核心配置（比如 broker 地址、topic 名称），符合 mutable 的设计初衷。

# 五、mutable 在 Lambda 中的用法（C++11）
```cpp
// Lambda 默认情况下，值捕获的变量是 const 的
int count = 0;
auto f1 = [count]() {
    // count++;  // ❌ 编译错误：count 在 lambda 内是 const
};

// 加 mutable 后可以修改（注意：修改的是副本，不影响外部 count）
auto f2 = [count]() mutable {
    count++;      // ✅ 合法
    return count;
};

f2();  // 返回 1
f2();  // 返回 2
// 外部 count 仍然是 0
```