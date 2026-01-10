---
title: 16.std::variant<>
date: 2026-01-08 10:42:12
tags:
	- 笔记
categories:
    - C++
    - 《C++17完全指南》
    - 三、新的标准库组件
    - 16.std::variant<>
---

# 1.概述
1.std::variant<> 可以认为是c++中类型安全的 union
2.variant所占的内存大小等于所有可能的底层类型中**最大的**再加上一个记录当前选项的固定内存开销。不
会分配堆内存。
3.和std::optional<>、std::any一样，variant对象是值语义 ——> 拷贝被实现为**深拷贝**
4.拷贝std::variant<>的开销要比拷贝当前选项的**开销稍微大一点**，因为variant必须找出要拷贝哪个值。
5.variant也支持move语义。

---
# 2.常用API
## 0.头文件
**#include\<variant\>**

**注意**：两个不同选项的类型也有可能相同，这在多个类型相同的选项分别代表不同含义的时候很有用
- 例如：有两个选项类型都是字符串，分别代表数据库中不同列的名称，你可以知道当前的值代表哪一个列）

## 1.构造与赋值
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108110315903.png)

## 2.取值访问 API
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108110558319.png)

## 3.状态查询 API
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108110959759.png)

## 4.修改与交换 API
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108111102983.png)

## 5.编译期辅助 API
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108111233871.png)

---
# 3.std::variant中的类型要求
**std::variant 可以包含几乎任何类型，包括 std::optional、自定义类、容器等。**

## 1.基本要求
- **可析构**（Destructible）
- **不能是引用类型**（int& ❌）
- **不能是数组类型**（int[10] ❌）
- **不能是** void
```cpp
// ✅ 全部合法
std::variant<int, double, std::string> v1;                    // 基础类型 + string
std::variant<int, std::optional<int>> v2;                     // 包含 optional
std::variant<std::vector<int>, std::map<int, int>> v3;        // 包含容器
std::variant<std::monostate, int, std::string> v4;            // monostate 用于表示"空"

// 自定义类型
struct MyClass {
    int x;
    std::string name;
};
std::variant<int, MyClass, std::optional<MyClass>> v5;        // ✅ 自定义类型也可以

// 嵌套 variant
std::variant<int, std::variant<double, std::string>> v6;      // ✅ 套娃也行

// ❌ 这些是非法的
std::variant<int&, double> bad1;           // 引用类型不行
std::variant<int[10], double> bad2;        // C 数组不行
std::variant<void, int> bad3;              // void 不行
```


## 2.对自定义类型的要求
- **最低要求：可析构**
- **其它要求**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108141739842.png)

```cpp
#include <variant>

// ✅ 完整实用的类
struct Good {
    int x;
    Good() = default;						//默认构造函数		
    Good(int val) : x(val) {}
    Good(const Good&) = default;         	// 拷贝构造
    Good(Good&&) = default;              	// 移动构造
    Good& operator=(const Good&) = default;
    Good& operator=(Good&&) = default;
    ~Good() = default;
};

// ⚠️ 没有默认构造函数
struct NoDefault {
    int x;
    NoDefault(int val) : x(val) {}       // 只有带参构造
};

// 用法
std::variant<Good, int> v1;              // ✅ Good 可以默认构造
std::variant<NoDefault, int> v2;         // ❌ 编译错误！NoDefault 不能默认构造

// 解决方案：用 monostate 或把有默认构造的类型放前面
std::variant<std::monostate, NoDefault> v3;  // ✅
std::variant<int, NoDefault> v4;             // ✅ int 在前面，可以默认构造

```

## 3.用 std::monostate 表示"无值"
```cpp
// 如果 variant 的第一个类型没有默认构造函数，需要用 monostate
struct NoDefault {
    NoDefault(int x) : val(x) {}
    int val;
};

// std::variant<NoDefault, int> v;  // ❌ 编译错误，NoDefault 没有默认构造
std::variant<std::monostate, NoDefault, int> v;  // ✅ monostate 可以默认构造
```

---
# 4.std::visit 多类型分支处理详解
## 1.核心原理
- **访问器（visitor）**：可是 lambda 表达式、函数对象，要求能接收 variant 所有可能存储的类型作为参数；
- **匹配规则**：std::visit 会在编译期检查访问器是否支持 variant 的所有类型，运行时自动根据 variant实际存储类型调用对应重载的函数/分支；
- **优势**：类型安全（编译期校验）、无分支冗余（**无需手动判断 index()** 或 holds_alternative）。

## 2.使用示例（按场景分类）
### 场景1：单 variant 多类型分支（基础场景）
```cpp
#include <variant>
#include <string>
#include <iostream>

int main() {
    std::variant<int, std::string, double> var;

    // 1. 定义访问器（lambda 表达式，用 constexpr if 实现分支）
    auto type_visitor = [](auto&& val) {
        // 用 decay_t 获取val的原始类型（排除引用、右值引用）
        using T = std::decay_t<decltype(val)>;
        if constexpr (std::is_same_v<T, int>) {
            std::cout << "当前类型：int，值：" << val << "，执行int专属逻辑（加10）：" << val + 10 << std::endl;
        } else if constexpr (std::is_same_v<T, std::string>) {
            std::cout << "当前类型：string，值：" << val << "，执行string专属逻辑（拼接后缀）：" << val + "_suffix" << std::endl;
        } else if constexpr (std::is_same_v<T, double>) {
            std::cout << "当前类型：double，值：" << val << "，执行double专属逻辑（乘2）：" << val * 2 << std::endl;
        }
    };

    // 2. 切换 variant 存储类型，测试访问器
    var = 100;          // 存储 int
    std::visit(type_visitor, var); // 输出：当前类型：int，值：100，执行int专属逻辑（加10）：110

    var = "hello variant"; // 存储 string
    std::visit(type_visitor, var); // 输出：当前类型：string，值：hello variant，执行string专属逻辑（拼接后缀）：hello variant_suffix

    var = 3.14;         // 存储 double
    std::visit(type_visitor, var); // 输出：当前类型：double，值：3.14，执行double专属逻辑（乘2）：6.28

    return 0;
}
```
- 用 **constexpr if**实现编译期分支判断，避免运行时开销；
- 用 **std::decay_t** 处理 val 的引用属性，确保类型匹配准确；
- 若访问器遗漏 variant 支持的某类类型，编译期会直接报错（类型安全保障）。

### 场景2：多 variant 组合分支（进阶场景）
**适用于需要同时处理多个 variant 的场景，std::visit 会匹配所有 variant 的类型组合，执行对应逻辑。**
```cpp
#include <variant>
#include <string>
#include <iostream>

int main() {
    // 定义两个 variant（可存储不同类型列表）
    std::variant<int, std::string> var1;
    std::variant<double, std::string> var2;

    // 定义访问器（处理 var1 和 var2 的所有类型组合）
    auto combo_visitor = [](auto&& v1, auto&& v2) {
        using T1 = std::decay_t<decltype(v1)>;
        using T2 = std::decay_t<decltype(v2)>;

        // 组合分支1：var1=int，var2=double
        if constexpr (std::is_same_v<T1, int> && std::is_same_v<T2, double>) {
            std::cout << "组合类型：int + double，执行求和：" << v1 + v2 << std::endl;
        }
        // 组合分支2：var1=int，var2=string
        else if constexpr (std::is_same_v<T1, int> && std::is_same_v<T2, std::string>) {
            std::cout << "组合类型：int + string，执行拼接：" << std::to_string(v1) + "_" + v2 << std::endl;
        }
        // 组合分支3：var1=string，var2=double
        else if constexpr (std::is_same_v<T1, std::string> && std::is_same_v<T2, double>) {
            std::cout << "组合类型：string + double，执行拼接：" << v1 + "_" + std::to_string(v2) << std::endl;
        }
        // 组合分支4：var1=string，var2=string
        else if constexpr (std::is_same_v<T1, std::string> && std::is_same_v<T2, std::string>) {
            std::cout << "组合类型：string + string，执行拼接：" << v1 + v2 << std::endl;
        }
    };

    // 测试不同类型组合
    var1 = 10; var2 = 3.14;
    std::visit(combo_visitor, var1, var2); // 输出：组合类型：int + double，执行求和：13.14

    var1 = 20; var2 = "test";
    std::visit(combo_visitor, var1, var2); // 输出：组合类型：int + string，执行拼接：20_test

    var1 = "hello"; var2 = 6.28;
    std::visit(combo_visitor, var1, var2); // 输出：组合类型：string + double，执行拼接：hello_6.28xxxx（精度取决于实现）

    var1 = "hello"; var2 = "world";
    std::visit(combo_visitor, var1, var2); // 输出：组合类型：string + string，执行拼接：helloworld

    return 0;
}
```
- std::visit 支持传入多个 variant 参数（逗号分隔），访问器需接收对应数量的参数；
- 需覆盖所有可能的类型组合，否则编译报错；**若部分组合无需处理，可添加默认分支（else）**。

## 3.std::visit 关键注意事项
**1.类型安全校验**:访问器必须支持 variant 所有模板类型，否则编译失败（避免遗漏类型处理）；
**2.避免窄化转换**:若 variant 存储的类型可隐式转换（如 int 和 long），需明确区分，避免分支匹配歧义；
**3.右值引用处理**:若需修改 variant 中的值，访问器参数可声明为非 const 引用（如 auto& val）；若需转移所有权，可声明为右值引用（auto&& val）；
**4.返回值支持**:访问器可返回值，所有分支的返回值类型需一致（或可隐式转换为同一类型）
```cpp
// 访问器返回值示例
auto return_visitor = [](auto&& val) -> int {
    using T = std::decay_t<decltype(val)>;
    if constexpr (std::is_same_v<T, int>) return val;
    else if constexpr (std::is_same_v<T, std::string>) return val.size();
    else if constexpr (std::is_same_v<T, double>) return static_cast<int>(val);
};

std::variant<int, std::string, double> var = "return test";
int result = std::visit(return_visitor, var);
std::cout << "访问器返回值：" << result << std::endl; // 输出：10（字符串长度）
```