---
title: 19.std::string_view
date: 2026-01-18 21:20:03
tags:
    - 笔记
categories:
    - C++
    - 《C++17完全指南》
    - 三、新的标准库组件
    - 19.std::string_view
---

# 一、std::string_view 的介绍、核心特点、优势
## 1.核心介绍
- std::string_view 是 C++17 引入的标准库类型，定义在 <string_view> 头文件中，本质是字符序列的**轻量级引用**
	- 它**不持有字符数据**，仅通过「**起始指针 + 长度**」的方式引用外部字符序列（如字符串字面量、std::string、字符数组等），无需分配内存，也不管理字符序列的生命周期。

## 2.核心特点
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118213057119.png)

## 3.核心优势
- **性能优化**：
	- 避免不必要的内存分配（如字符串字面量传参给 const std::string& 时的临时对象创建）、减少拷贝，尤其适合处理子字符串、内存映射文件等场景；
- **接口友好**：
	- 提供与 std::string 几乎一致的只读接口，无需改变使用习惯；
- **编译期支持**：
	- 可通过 constexpr 初始化，支持编译期字符序列操作；
- **多字符类型适配**：
	- 提供 u16string_view/u32string_view/wstring_view 特化版本，适配宽字符场景。

# 二、与 std::string、const std::string& 的区别
## 1.对比表
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118213442111.png)

## 2.std::string_view 何时代替 const std::string& ？
### （1）核心区别
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118213703783.png)

### （2）必须使用 const std::string& 的场景（无法用std::string_view 代替）
```cpp
┌─ 需要存储字符串？
│   └─ 是 → std::string (参数可以是 string_view，内部转换存储)
│
├─ 需要传给 C API / 使用 c_str()？
│   └─ 是 → const std::string&
│
├─ 需要修改字符串？
│   └─ 是 → std::string&
│
├─ 是函数返回值？
│   ├─ 返回局部变量 → std::string
│   └─ 返回成员/参数 → std::string_view 或 const std::string&
│
├─ 数据会跨越作用域/线程？
│   └─ 是 → std::string
│
└─ 只读参数，短期使用？
    └─ 是 → std::string_view ✅
```

#### <1>需要传给 C API（要求 null 结尾）
```cpp
// ❌ 错误：string_view 不保证 null 结尾
void openFile(std::string_view path) {
    fopen(path.data(), "r");  // ❌ 危险！
}

// ✅ 正确
void openFile(const std::string& path) {
    fopen(path.c_str(), "r");  // ✅ 保证 null 结尾
}

// 常见 C API：fopen, open, printf, 系统调用, 数据库接口等
```

#### <2>需要使用 c_str() 方法
```cpp
// ❌ string_view 没有 c_str()
void callLegacy(std::string_view s) {
    legacy_api(s.c_str());  // ❌ 编译错误
}

// ✅ 正确
void callLegacy(const std::string& s) {
    legacy_api(s.c_str());  // ✅
}
```

#### <3>接口与旧代码兼容
```cpp
// 旧接口设计
class OldLibrary {
public:
    void process(const std::string& data);  // 已有大量调用
};

// 如果改成 string_view，所有返回 c_str() 的地方都要改
```

#### <4>需要延长临时对象生命周期
```cpp
// ✅ const 引用可以延长临时对象生命周期
const std::string& ref = createString();  // 临时对象生命周期延长
use(ref);  // ✅ 安全

// ❌ string_view 不能延长
std::string_view view = createString();  // 临时对象立即销毁
use(view);  // ❌ 悬空引用！
```

### （3）⚠️ string_view 的危险场景
#### <1>存储 string_view（生命周期陷阱）
```cpp
class Config {
    std::string_view name_;  // ❌ 危险！

public:
    void setName(std::string_view name) {
        name_ = name;  // 如果 name 来自临时对象，会悬空
    }
};

// 错误使用
Config c;
c.setName(std::string("temp"));  // 临时 string 销毁，name_ 悬空！

// ✅ 正确：存储 std::string
class Config {
    std::string name_;
public:
    void setName(std::string_view name) {
        name_ = name;  // 隐式转换，拷贝数据
    }
};
```

#### <2>返回局部变量的 string_view
```cpp
// ❌ 错误
std::string_view process() {
    std::string result = compute();
    return result;  // ❌ result 销毁，view 悬空
}

// ✅ 正确：返回 string
std::string process() {
    std::string result = compute();
    return result;  // ✅ 移动语义，高效
}
```

#### <3>跨越异步边界
```cpp
// ❌ 错误：异步任务中使用
void async(std::string_view data) {
    threadPool.submit([=] {
        use(data);  // ❌ 原始数据可能已销毁
    });
}

// ✅ 正确：捕获 string
void async(std::string data) {
    threadPool.submit([data = std::move(data)] {
        use(data);  // ✅ 拥有数据
    });
}
```

# 三、核心 API (与std::string相似)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118214152000.png)

# 四、使用示例
## 1.作为函数参数优化性能（替代 const std::string&）
```cpp
#include <string_view>
#include <iostream>
#include <vector>

// 优化前：传 const std::string& 会导致字符串字面量创建临时对象
// 优化后：传 string_view 无内存分配
void printElements(const auto& coll, std::string_view prefix = {}) {
    for (const auto& elem : coll) {
        // 排除 nullptr 情况
        if (!prefix.empty()) {
            std::cout << prefix << ": " << elem << "\n";
        } else {
            std::cout << elem << "\n";
        }
    }
}

int main() {
    std::vector<int> vec = {10, 20, 30};
    // 传递字符串字面量，无临时 string 创建
    printElements(vec, "Number");
    // 传递 std::string，隐式转换为 string_view
    std::string str = "Value";
    printElements(vec, str);
    return 0;
}
```

## 2.字符串转整数（高效处理字符序列）
```cpp
#include <string_view>
#include <optional>
#include <charconv>
#include <iostream>

std::optional<int> asInt(std::string_view sv) {
    int val;
    // 直接传递字符范围，无需分配内存
    auto [ptr, ec] = std::from_chars(sv.data(), sv.data() + sv.size(), val);
    if (ec != std::errc{}) {
        return std::nullopt;
    }
    return val;
}

int main() {
    for (std::string_view s : {"42", "  77", "hello", "0x33"}) {
        if (auto res = asInt(s)) {
            std::cout << "Convert '" << s << "' to int: " << *res << "\n";
        } else {
            std::cout << "Can't convert '" << s << "' to int\n";
        }
    }
    return 0;
}
```

## 3.处理子字符串（无内存分配）
```cpp
#include <string_view>
#include <iostream>
#include <algorithm>
#include <vector>

int main() {
    std::vector<std::string> coll = {"apple123", "banana456", "cherry789"};
    // 按子串（从第3位开始）排序，无内存分配
    std::sort(coll.begin(), coll.end(), [](const auto& a, const auto& b) {
        return std::string_view{a}.substr(3) < std::string_view{b}.substr(3);
    });

    for (const auto& s : coll) {
        std::cout << s << "\n";
    }
    return 0;
}
```

# 五、使用注意事项（核心避坑点）
## 1.生命周期管理（最核心）
### （1）禁止绑定临时字符串
std::string_view 不会延长外部字符序列的生命周期，绑定临时 std::string 会导致悬空引用
```cpp
// 错误示例：临时 string 销毁后，sv 引用无效
std::string_view sv = std::string("hello");
std::cout << sv; // 未定义行为！
```

### （2）禁止返回 string_view 指向局部变量
函数返回的 string_view 不能引用函数内的局部字符序列
```cpp
// 错误示例
std::string_view badFunc() {
    std::string s = "local";
    return s; // 返回后 s 销毁，视图悬空
}
```

### （3）允许返回的场景
仅当 string_view 转发函数输入参数，或引用全局/静态字符序列时可返回。


## 2.空字符与 nullptr 处理
### （1）必须用 size() 确认长度
```cpp
// 正确示例
void safeAccess(std::string_view sv) {
    if (sv.size() > 0) {
        std::cout << sv.data(); // 避免 nullptr 访问
    }
}
```

### （2）避免直接传给要求 \0 终止的函数
如 strlen()、printf("%s")，需先转换为 std::string
```cpp
// 错误：sv 可能无 \0 结尾
printf("%s", sv.data());
// 正确
printf("%s", std::string(sv).c_str());
```

## 3.接口设计避坑
### （1）不要用 string_view 初始化 string 成员
会破坏 std::move 优化，导致额外内存分配
```cpp
// 错误设计
class Person {
    std::string name;
public:
    Person(std::string_view n) : name(n) {} // 传入 move 的 string 也会拷贝
};
// 正确设计
Person(std::string n) : name(std::move(n)) {}
```

### （2）函数模板返回 auto 而非 T
```cpp
// 错误示例：T 为 string_view 时，返回值悬空
template<typename T>
T concat(const T& a, const T& b) {
    return std::string(a) + std::string(b);
}
// 正确示例：auto 推导为 std::string
template<typename T>
auto concat(const T& a, const T& b) {
    return std::string(a) + std::string(b);
}
```