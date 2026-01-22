---
title: C++开发规范
date: 2026-01-18 14:47:07
tags:
	- 笔记
categories:
	- C++
	- 开发笔记
	- C++开发规范
---

# 一、现代C++规范
## 1.汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118211230598.png)

## 2.使用示例
###（1）常量：constexpr > const > #define
```cpp
// ✅ 首选：constexpr（编译期常量）
constexpr int kMaxSize = 100;
constexpr double kPi = 3.14159265358979;
constexpr auto kAppName = "MyApp";

// constexpr 函数
constexpr int square(int x) { return x * x; }
constexpr int result = square(10);  // 编译期计算

// ✅ 运行时常量用 const
const std::string configPath = getConfigPath();  // 运行时确定

// ❌ 避免：宏定义常量
#define MAX_SIZE 100      // 无类型检查，无作用域
#define PI 3.14159        // 难以调
```

### （2）字符串：std::string_view（只读）
```cpp
// ✅ 正确：只读参数用 string_view
void log(std::string_view msg) {       // 零拷贝
    std::cout << msg << "\n";
}

log("hello");                          // const char* 直接传
log(std::string("world"));             // std::string 也可以
log(someStringView);

// ✅ 函数内只读访问
void process(std::string_view data) {
    if (data.starts_with("prefix")) {  // C++20
        // ...
    }
}

// ❌ 避免：不必要的拷贝
void log(const std::string& msg);      // 传 "hello" 会构造临时对象
void log(const char* msg);             // 无法直接传 std::string

// ⚠️ 注意：string_view 不拥有数据
std::string_view bad() {
    std::string s = "hello";
    return s;  // ❌ 危险！s 被销毁，view 悬空
}
```

### （3）初始化：统一初始化 {}
```cpp
// ✅ 正确：统一使用 {} 初始化
int x{10};
double d{3.14};
std::string s{"hello"};
std::vector<int> v{1, 2, 3, 4, 5};

MyClass obj{arg1, arg2};
auto ptr = std::make_unique<MyClass>(arg1, arg2);

// {} 防止窄化转换
int x{3.14};        // ❌ 编译错误，防止精度丢失
int y = 3.14;       // ⚠️ 编译通过，但丢失精度

// ❌ 避免：混用多种风格
int a = 1;
int b(2);
int c{3};           // 同一文件风格不一致

// ⚠️ 注意：vector 的特殊情况
std::vector<int> v1(5, 1);   // 5 个 1：{1, 1, 1, 1, 1}
std::vector<int> v2{5, 1};   // 2 个元素：{5, 1}
```

### （4）类型转换：static_cast 等
```cpp
// ✅ 正确：使用 C++ 风格转换
double d = 3.14;
int i = static_cast<int>(d);           // 数值转换

Base* base = &derived;
Derived* d = static_cast<Derived*>(base);  // 下行转换（确定类型时）

void* vp = &x;
int* ip = static_cast<int*>(vp);       // void* 转换

// ✅ dynamic_cast：安全的多态转换
Base* base = getObject();
if (auto* derived = dynamic_cast<Derived*>(base)) {
    derived->specialMethod();
}

// ✅ const_cast：移除 const（慎用）
const int* cp = &x;
int* p = const_cast<int*>(cp);

// ✅ reinterpret_cast：底层位转换（慎用）
uintptr_t addr = reinterpret_cast<uintptr_t>(ptr);

// ❌ 避免：C 风格转换
int i = (int)d;                        // 不安全，难以搜索
Derived* d = (Derived*)base;           // 危险
```

# 二、项目组织
## 1.项目目录结构
```
project_name/
│
├── CMakeLists.txt                    # 顶层构建文件
├── README.md                         # 项目说明
├── LICENSE                           # 开源协议
├── .gitignore
├── .clang-format                     # 代码格式化配置
├── .clang-tidy                       # 静态分析配置
│
├── cmake/                            # CMake 模块和工具
│   ├── FindXxx.cmake                 # 自定义 Find 模块
│   ├── ProjectConfig.cmake.in        # 导出配置模板
│   └── utils.cmake                   # 通用 CMake 函数
│
├── include/                          # 公开头文件（对外 API）
│   └── project_name/
│       ├── module_a.h
│       ├── module_b.h
│       └── types.h
│
├── src/                              # 源码实现
│   ├── CMakeLists.txt
│   ├── module_a.cpp
│   ├── module_b.cpp
│   │
│   └── detail/                       # 内部实现（不对外暴露）
│       ├── internal_utils.h
│       ├── internal_utils.cpp
│       └── impl_helper.h
│
├── test/                             # 单元测试
│   ├── CMakeLists.txt
│   ├── module_a_test.cpp
│   ├── module_b_test.cpp
│   └── test_utils.h                  # 测试辅助工具
│
├── benchmark/                        # 性能测试（可选）
│   ├── CMakeLists.txt
│   └── module_a_bench.cpp
│
├── examples/                         # 使用示例（可选）
│   ├── CMakeLists.txt
│   ├── basic_usage.cpp
│   └── advanced_usage.cpp
│
├── docs/                             # 文档
│   ├── design.md                     # 设计文档
│   ├── api.md                        # API 文档
│   └── Doxyfile                      # Doxygen 配置
│
├── scripts/                          # 脚本工具
│   ├── build.sh
│   ├── format.sh                     # 代码格式化脚本
│   └── install_deps.sh
│
├── third_party/                      # 第三方依赖（或用 git submodule）
│   └── json/
│       └── json.hpp
│
└── docker/                           # 容器化（可选）
    ├── Dockerfile
    └── docker-compose.yml

```

## 2.CMakeLists.txt模板
### （1）顶层CMakeLists.txt
```CMakeLists
cmake_minimum_required(VERSION 3.16)

project(project_name
    VERSION 1.0.0
    DESCRIPTION "Project description"
    LANGUAGES CXX
)

# ============ 编译选项 ============
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)  # 生成 compile_commands.json

# ============ 构建选项 ============
option(BUILD_TESTS "Build unit tests" ON)
option(BUILD_BENCHMARKS "Build benchmarks" OFF)
option(BUILD_EXAMPLES "Build examples" ON)
option(BUILD_DOCS "Build documentation" OFF)

# ============ 输出目录 ============
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# ============ 自定义 CMake 模块路径 ============
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

# ============ 第三方依赖 ============
# find_package(Boost 1.70 REQUIRED COMPONENTS system filesystem)
# find_package(spdlog REQUIRED)

# ============ 子目录 ============
add_subdirectory(src)

if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(test)
endif()

if(BUILD_BENCHMARKS)
    add_subdirectory(benchmark)
endif()

if(BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

# ============ 安装配置 ============
include(GNUInstallDirs)

install(
    DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
```

### （2）src/CMakeLists.txt 模板
```CMakeLists
# 收集源文件
set(SOURCES
    module_a.cpp
    module_b.cpp
    detail/internal_utils.cpp
)

# 创建库
add_library(${PROJECT_NAME} ${SOURCES})
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

# 头文件路径
target_include_directories(${PROJECT_NAME}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}          # src/ 目录
        ${CMAKE_CURRENT_SOURCE_DIR}/detail   # detail/ 目录
)

# 链接依赖
# target_link_libraries(${PROJECT_NAME}
#     PUBLIC
#         Boost::system
#     PRIVATE
#         spdlog::spdlog
# )

# 编译选项
target_compile_options(${PROJECT_NAME} PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

# 安装目标
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
```

## （2）test/CMakeLists.txt 模板
```CMakeLists
# 查找测试框架
find_package(GTest REQUIRED)
# 或使用 FetchContent 自动下载
# include(FetchContent)
# FetchContent_Declare(googletest
#     GIT_REPOSITORY https://github.com/google/googletest.git
#     GIT_TAG release-1.12.1
# )
# FetchContent_MakeAvailable(googletest)

# 测试源文件
set(TEST_SOURCES
    module_a_test.cpp
    module_b_test.cpp
)

# 创建测试可执行文件
add_executable(${PROJECT_NAME}_test ${TEST_SOURCES})

target_link_libraries(${PROJECT_NAME}_test
    PRIVATE
        ${PROJECT_NAME}
        GTest::gtest
        GTest::gtest_main
)

# 注册测试
include(GoogleTest)
gtest_discover_tests(${PROJECT_NAME}_test)
```



# 三、命名规范
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118151618231.png)

# 四、代码风格规范

## 1.头文件规范
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118205115805.png)

### （2）示例
#### <1>用 #pragma once
```cpp
// ❌ 传统方式（冗长，容易出错）
#ifndef USER_SERVICE_USER_H_
#define USER_SERVICE_USER_H_
// ...
#endif

// ✅ 推荐方式
#pragma once
// ...
```

#### <2>头文件自包含
```cpp
// ❌ 错误：依赖外部先 include <string>
// user.h
#pragma once
class User {
    std::string name;  // 编译失败，std::string 未定义
};

// ✅ 正确：头文件自己包含所有依赖
// user.h
#pragma once
#include <string>
class User {
    std::string name;  // ✅ 可独立编译
};
```

#### <3>include 顺序
```cpp
// user_service.cpp

#include "user_service.h"      // ① 对应的 .h（验证头文件自包含）

#include <cstdio>              // ② C 标准库
#include <cstring>

#include <string>              // ③ C++ 标准库
#include <vector>
#include <memory>

#include <spdlog/spdlog.h>     // ④ 第三方库
#include <nlohmann/json.hpp>

#include "common/utils.h"      // ⑤ 项目头文件
#include "models/user.h"
```

#### <4> 组间空行
```cpp
// ✅ 正确：不同类别之间空一行
#include "order_service.h"

#include <cstdlib>

#include <map>
#include <string>

#include <grpcpp/grpcpp.h>

#include "common/config.h"
#include "models/order.h"

// ❌ 错误：全部挤在一起
#include "order_service.h"
#include <cstdlib>
#include <map>
#include <grpcpp/grpcpp.h>
#include "common/config.h"
```

#### <5>前向声明优先
```cpp
// ❌ 不推荐：不必要的 include（增加编译依赖）
// user_service.h
#pragma once
#include "models/user.h"       // 完整定义
#include "repositories/user_repository.h"

class UserService {
    User* getUser(int id);     // 只用指针，不需要完整定义
    UserRepository* repo_;
};

// ✅ 推荐：使用前向声明
// user_service.h
#pragma once

class User;                    // 前向声明
class UserRepository;

class UserService {
    User* getUser(int id);     // ✅ 指针/引用只需前向声明
    UserRepository* repo_;
};

// user_service.cpp
#include "user_service.h"
#include "models/user.h"       // 实现文件才 include 完整定义
#include "repositories/user_repository.h"
```
**何时必须 include（不能前向声明）**
```cpp
class Foo;  // 前向声明

class Bar {
    Foo* ptr;      // ✅ 指针 - 可以
    Foo& ref;      // ✅ 引用 - 可以
    Foo obj;       // ❌ 对象 - 需要完整定义
    Foo func();    // ✅ 返回值 - 可以（声明时）
    void f(Foo);   // ✅ 参数 - 可以（声明时）
};
```

## 2.注释规范
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118210302886.png)

### （2）使用示例
#### <1>文件头注释
```cpp
/**
 * @file user_service.h
 * @brief 用户服务模块，提供用户的增删改查功能
 * 
 * @author zhangsan
 * @date 2024-01-15
 * @version 1.0.0
 * 
 * @copyright Copyright (c) 2024 MyCompany
 */

#pragma once

#include <string>
// ...
```

#### <2>类注释
```cpp
/**
 * @brief 用户服务类，封装用户相关的业务逻辑
 * 
 * 负责用户的创建、查询、更新、删除等操作，
 * 通过 UserRepository 与数据层交互。
 * 
 * @note 线程安全：此类非线程安全，多线程使用需外部同步
 * 
 * @see UserRepository
 * @see User
 */
class UserService {
public:
    // ...
};

/**
 * @brief 简单类可以用单行注释
 */
class Config {
    // ...
};
```

#### <3>函数注释
```cpp
/**
 * @brief 根据 ID 查找用户
 * 
 * @param id 用户 ID，必须为正整数
 * @return User* 找到返回用户指针，未找到返回 nullptr
 * @throw std::invalid_argument 当 id <= 0 时抛出
 * 
 * @note 返回的指针由 Repository 管理，调用者不应删除
 * 
 * @code
 * auto* user = service.findById(123);
 * if (user) {
 *     std::cout << user->name << std::endl;
 * }
 * @endcode
 */
User* findById(int id);

/**
 * @brief 创建新用户
 * 
 * @param[in] name 用户名，不能为空
 * @param[in] email 邮箱地址
 * @param[out] errorMsg 失败时的错误信息
 * @return int 成功返回用户 ID，失败返回 -1
 */
int createUser(const std::string& name, const std::string& email, std::string& errorMsg);

// 简单函数可用单行注释
/// @brief 获取用户数量
int getUserCount();
```
#### <4>行内注释
```cpp
// ✅ 正确：// 后空一格，与代码间隔至少 2 空格
int maxRetry = 3;  // 最大重试次数
int timeout = 5000;  // 超时时间（毫秒）

// ✅ 复杂逻辑前单独一行注释
// 使用二分查找优化性能，时间复杂度 O(log n)
auto it = std::lower_bound(vec.begin(), vec.end(), target);

// ❌ 错误：无空格或间隔不足
int x = 1;// 说明
int y = 2; //说明
```

#### <5>TODO 注释
```cpp
// TODO(zhangsan): 优化数据库查询性能，考虑添加缓存
User* findById(int id) {
    return repo_->findById(id);
}

// TODO(zhangsan): 2024-Q2 支持批量查询接口
// TODO: 添加分页功能（无作者时可省略）

void process() {
    // TODO(lisi): 这里需要添加输入校验
    doSomething(input);
}
```

#### <6>FIXME 注释
```cpp
// FIXME(lisi): 修复当 name 为空时的崩溃问题
void setName(const std::string& name) {
    this->name = name;  // 未检查空值
}

// FIXME(wangwu): 边界情况：当列表为空时返回值不正确
int getFirst() {
    return items[0];
}

// FIXME: 内存泄漏，需要在析构函数中释放
```

#### <7>HACK 注释
```cpp
// HACK: 临时绕过第三方库的 bug，等待 v2.0 修复后移除
void connect() {
    sleep(100);  // 延迟避免竞态条件
    client.connect();
}

// HACK: Windows 下的特殊处理，Linux 不需要
#ifdef _WIN32
    SetConsoleOutputCP(65001);  // HACK: 强制 UTF-8 输出
#endif

// HACK: 硬编码魔数，后续需要改为配置项
constexpr int MAGIC_BUFFER_SIZE = 4096;
```

#### <8>废弃标记
```cpp
/**
 * @brief 获取用户名（已废弃）
 * @deprecated 使用 getDisplayName() 替代，将在 v2.0 移除
 */
[[deprecated("Use getDisplayName() instead, will be removed in v2.0")]]
std::string getName() const {
    return name_;
}

// 新接口
std::string getDisplayName() const {
    return displayName_;
}

// 废弃整个类
class [[deprecated("Use NewUserManager instead")]] OldUserManager {
    // ...
};
```

#### <9>禁止：无意义的注释
```cpp
// ❌ 错误：显而易见的注释
int count = 0;  // 定义 count 变量
i++;  // i 加 1
// 循环遍历数组
for (int i = 0; i < n; ++i) {
    // 调用处理函数
    process(arr[i]);  // 处理 arr[i]
}

// ✅ 正确：注释解释"为什么"，而非"是什么"
// 从 1 开始，跳过已处理的头元素
for (int i = 1; i < n; ++i) {
    process(arr[i]);
}

// ❌ 错误：装饰性注释
/*************************************/
/*          这是注释                  */
/*************************************/

// ✅ 正确：简洁的分隔符（如果需要）
// ═══════════════════════════════════
// 用户相关函数
// ═══════════════════════════════════
```

## 3.代码组织规范
### （1）表格汇总
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118210850782.png)

### （2）使用示例
#### <1>类成员顺序：public → protected → private
```cpp
// ✅ 正确：按访问权限从宽到严排列
class UserService {
public:
    // 公开接口在最前，使用者最关心
    void createUser();
    User* findById(int id);

protected:
    // 子类可能用到的
    void validateUser();

private:
    // 内部实现细节
    UserRepository* repo_;
};

// ❌ 错误：顺序混乱
class UserService {
private:
    UserRepository* repo_;
public:
    void createUser();
private:
    int cache_;
public:
    User* findById(int id);
};
```

#### <2>成员内部顺序
```cpp
class UserService {
public:
    // ① 类型别名
    using Ptr = std::shared_ptr<UserService>;
    using UserList = std::vector<User*>;

    // ② 静态成员
    static constexpr int kMaxUsers = 1000;
    static UserService& getInstance();

    // ③ 构造/析构
    explicit UserService(std::shared_ptr<UserRepository> repo);
    ~UserService();
    UserService(const UserService&) = delete;
    UserService& operator=(const UserService&) = delete;

    // ④ 普通函数
    User* findById(int id);
    UserList findAll();
    bool createUser(const std::string& name);
    bool deleteUser(int id);

private:
    // ① 类型别名
    using CacheMap = std::unordered_map<int, User*>;

    // ② 静态成员
    static int instanceCount_;

    // ④ 普通函数（私有辅助函数）
    void refreshCache();
    bool validateInput(const std::string& name);

    // ⑤ 数据成员（最后）
    std::shared_ptr<UserRepository> repo_;
    CacheMap cache_;
    int maxCacheSize_ = 100;
};
```

#### <3>参数数量：不超过 5 个
```cpp
// ❌ 错误：参数过多（7 个）
User* createUser(
    const std::string& name,
    const std::string& email,
    const std::string& password,
    int age,
    const std::string& phone,
    const std::string& address,
    bool isAdmin);

// ✅ 正确：使用结构体封装
struct CreateUserOptions {
    std::string name;
    std::string email;
    std::string password;
    int age = 0;
    std::string phone;
    std::string address;
    bool isAdmin = false;
};

User* createUser(const CreateUserOptions& options);

// 调用时更清晰
auto user = createUser({
    .name = "张三",
    .email = "zhangsan@example.com",
    .password = "123456",
    .age = 25
});

// ✅ 另一种方式：Builder 模式
auto user = UserBuilder()
    .name("张三")
    .email("zhangsan@example.com")
    .password("123456")
    .age(25)
    .build();
```

## 4.格式化规范
### （1）表格汇总
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118205735765.png)

####（2）使用示例
#### <1>大括号风格
```cpp
// ✅ K&R 风格（推荐）：左括号不换行
if (condition) {
    doSomething();
} else {
    doOther();
}

void func() {
    // ...
}

// ✅ Allman 风格：左括号换行
if (condition)
{
    doSomething();
}
else
{
    doOther();
}

// ❌ 错误：混用
if (condition) {
    doSomething();
}
else                    // 风格不一致
{
    doOther();
}
```

#### <2>指针/引用位置
```cpp
// ✅ 风格 A：靠近类型（推荐，强调类型）
int* ptr;
const std::string& name;
std::vector<int>* vec;

// ✅ 风格 B：靠近变量
int *ptr;
const std::string &name;

// ❌ 错误：混用
int* ptr1;
int *ptr2;              // 同一文件风格不一致

// ⚠️ 注意：多变量声明时的陷阱
int* a, b;              // a 是指针，b 是 int！
int *a, *b;             // 都是指针（风格 B 更清晰）
int* a;                 // ✅ 推荐：每行只声明一个变量
int* b;
```

#### <3>函数参数换行
```cpp
// ✅ 短参数：一行
void init(int x, int y, int z);

// ✅ 超过 120 字符：每个参数独占一行，对齐
void createUserWithFullInfo(
    const std::string& username,
    const std::string& email,
    const std::string& password,
    const Address& address,
    const std::vector<std::string>& roles);

// ✅ 函数调用同理
auto result = sendHttpRequest(
    "https://api.example.com/users",
    HttpMethod::POST,
    headers,
    requestBody,
    timeoutMs);

// ✅ 链式调用
auto query = db.select("users")
    .where("age", ">", 18)
    .where("status", "=", "active")
    .orderBy("created_at", "DESC")
    .limit(100);
```

#### <4>行尾空格 & 文件末尾
```cpp
// ❌ 错误：行尾有空格（不可见但存在）
int x = 1;   ␣␣␣        // ← 行尾空格

// ✅ 正确：无行尾空格
int x = 1;

// ✅ 文件末尾：有且仅有一个空行
void lastFunction() {
    // ...
}
⏎                       // ← 文件最后一行是空行，然后 EOF
```

