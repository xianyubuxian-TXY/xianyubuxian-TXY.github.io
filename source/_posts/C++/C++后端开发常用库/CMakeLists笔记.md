---
title: CMakeLists笔记
date: 2026-01-15 10:26:32
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- CMake
---

# 一、使用示例
## 1.顶层目录示例
```cpp
cmake_minimum_required(VERSION 3.16)
project(UserService
        VERSION 1.0.0
        LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)	# 指定使用 C++17 标准
set(CMAKE_CXX_STANDARD_REQUIRED ON) #强制要求编译器支持 C++17，若编译器不支持，直接报错
set(CMAKE_CXX_EXTENSIONS OFF) #禁用编译器扩展语法， 保证代码在不同编译器下行为一致

# 核心逻辑：若用户未指定 CMAKE_BUILD_TYPE（如编译时未加 -DCMAKE_BUILD_TYPE=Debug），
# 默认设置为 Release（优化编译，去除调试信息，适合部署）；
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the build type" FORCE)
endif()

option(BUILD_TESTS "Build unit tests" ON) # 控制是否编译单元测试代码（tests 子目录）
# 控制生成 “动态库（.so/.dll）” 还是 “静态库（.a/.lib）”，默认 OFF 即生成静态库；
option(BUILD_SHARED_LIBS "Build shared instead of static libs" OFF) 


# 输出目录
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)


# YAML 配置
find_package(yaml-cpp CONFIG REQUIRED)

# spdlog 日志
find_package(spdlog CONFIG REQUIRED)

# MySQL
find_package(PkgConfig REQUIRED)
pkg_check_modules(MYSQL REQUIRED mysqlclient)

# hiredis 库 
find_path(HIREDIS_HEADER hiredis)
find_library(HIREDIS_LIB hiredis)

# redis++ 库 
# 注意：这里是 sw，不是 redis++
find_path(REDIS_PLUS_PLUS_HEADER sw)
find_library(REDIS_PLUS_PLUS_LIB redis++)

add_subdirectory(src)


# GTest 包
find_package(GTest REQUIRED)
if(BUILD_TESTS)
    enable_testing()
    find_package(GTest CONFIG REQUIRED)
    add_subdirectory(tests)
endif()
```


## 2.子目录
```cpp
# 添加可执行文件
add_executable(config_test config_test.cpp)

# 指定可执行文件的“头文件目录”
target_include_directories(config_test PUBLIC 
    ${PROJECT_SOURCE_DIR}/src
)

# 指令可执行文件的“库文件目录”
target_link_directories(config_test PUBLIC
    ${PROJECT_SOURCE_DIR}/include/lib
)

# 让可执行文件链接库
target_link_libraries(config_test PRIVATE
    user_config
    GTest::GTest
    GTest::Main
    pthread
    yaml-cpp  # Config 模块需要 yaml-cpp 库
)

# 设置可执行文件的输出路径
# set_target_properties：CMake 内置命令，用于设置目标（target）的属性
# config_test：目标名称
# PROPERTIES：关键字，表示后面是要设置的属性及其值
# RUNTIME_OUTPUT_DIRECTORY：目标属性名
# ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/tests：属性值（路径）
set_target_properties(config_test PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/tests
)
```

- **PUBLIC 关键字**:
	- **作用**：指定了包含目录的可见性范围
		- 编译 config_test 目标本身时会使用这些包含目录
		- 任何链接到 config_test 的其他目标也会继承这些包含目录
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115103707646.png)

# 二、CMake 变量设置顺序规范
## （1）完整概览
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118163502364.png)
```cpp
cmake_minimum_required(VERSION 3.15)

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     1️⃣ 项目信息变量                               ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么最先？project() 定义 PROJECT_NAME、PROJECT_SOURCE_DIR 等基础变量
# 📌 后续所有配置都依赖这些变量

project(MyApp
    VERSION 1.2.3                              # → PROJECT_VERSION, PROJECT_VERSION_MAJOR/MINOR/PATCH
    DESCRIPTION "企业级应用程序"                 # → PROJECT_DESCRIPTION
    HOMEPAGE_URL "https://github.com/xxx"      # → PROJECT_HOMEPAGE_URL
    LANGUAGES CXX                              # 启用 C++ 编译器检测
)

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     2️⃣ 平台与系统变量                   ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排第二？平台决定：编译器选项、路径格式、依赖位置
# 📌 WIN32/APPLE/UNIX 是 CMake 内置变量，project() 后自动可用

if(WIN32)                                      # Windows（包括64位）
    set(PLATFORM_NAME "windows")
elseif(APPLE)                                  # macOS、iOS
    set(PLATFORM_NAME "macos")
elseif(UNIX)                                   # Linux、BSD 等
    set(PLATFORM_NAME "linux")
endif()

# 架构检测
if(CMAKE_SIZEOF_VOID_P EQUAL 8)                # 指针大小判断架构
    set(ARCH_NAME "x64")                       # 8字节 = 64位
else()
    set(ARCH_NAME "x86")                       # 4字节 = 32位
endif()

message(STATUS "平台: ${CMAKE_SYSTEM_NAME} (${PLATFORM_NAME})")
message(STATUS "架构: ${CMAKE_SYSTEM_PROCESSOR} (${ARCH_NAME})")

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     3️⃣ 编译器与工具链变量                          ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排这里？依赖平台检测结果（MSVC vs GCC 选项不同）
# 📌 必须在 add_executable/add_library 之前设置，否则不生效！

# C++ 标准
set(CMAKE_CXX_STANDARD 17)                     # 使用 C++17
set(CMAKE_CXX_STANDARD_REQUIRED ON)            # 强制要求，不满足则报错
set(CMAKE_CXX_EXTENSIONS OFF)                  # 禁用编译器扩展（如 GNU++17）

# 编译器选项
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)          # 生成 compile_commands.json（IDE/LSP 用）

# 平台相关编译选项
if(MSVC)                                       # Visual Studio 编译器
    add_compile_options(/W4 /utf-8)            # 警告级别4 + UTF-8源码
else()                                         # GCC / Clang
    add_compile_options(-Wall -Wextra -Wpedantic)  # 开启常用警告
endif()

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     4️⃣ 构建类型变量                                ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排这里？影响编译选项(-O0/-O3)和输出路径(Debug/Release子目录)
# 📌 仅对单配置生成器生效（Makefile/Ninja），VS/Xcode 忽略此变量


# 默认构建类型
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "构建类型" FORCE)    # 未指定时默认 Release

    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS        # 在 cmake-gui 中提供下拉选项
        "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

message(STATUS "构建类型: ${CMAKE_BUILD_TYPE}")

# 自定义各构建类型的编译选项（覆盖默认值）
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DDEBUG")    # 调试：无优化+调试符号
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG")    # 发布：最高优化+禁用assert

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     5️⃣ 项目路径变量                                ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排这里？依赖 PROJECT_SOURCE_DIR（由 project() 定义）
# 📌 统一管理路径，避免硬编码，方便后续引用

# 自定义目录结构
set(PROJECT_ROOT_DIR     ${PROJECT_SOURCE_DIR})            # 项目根目录
set(PROJECT_SRC_DIR      ${PROJECT_SOURCE_DIR}/src)        # 源码目录
set(PROJECT_INCLUDE_DIR  ${PROJECT_SOURCE_DIR}/include)    # 头文件目录
set(PROJECT_LIB_DIR      ${PROJECT_SOURCE_DIR}/lib)        # 预编译库目录
set(PROJECT_TEST_DIR     ${PROJECT_SOURCE_DIR}/tests)      # 测试目录
set(PROJECT_DOC_DIR      ${PROJECT_SOURCE_DIR}/docs)       # 文档目录
set(PROJECT_CMAKE_DIR    ${PROJECT_SOURCE_DIR}/cmake)      # CMake 模块目录
set(PROJECT_THIRD_PARTY  ${PROJECT_SOURCE_DIR}/third_party) # 第三方库目录

# 添加自定义模块搜索路径（用于 include() 和 find_package()）
list(APPEND CMAKE_MODULE_PATH "${PROJECT_CMAKE_DIR}")

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     6️⃣ 输出路径变量                                ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排这里？依赖 PLATFORM_NAME、ARCH_NAME（第2步）和 PROJECT_BINARY_DIR
# 📌 统一输出结构：build/output/linux/x64/bin/

# 统一输出目录（按平台/架构/构建类型组织）
set(OUTPUT_BASE_DIR ${PROJECT_BINARY_DIR}/output/${PLATFORM_NAME}/${ARCH_NAME})

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${OUTPUT_BASE_DIR}/bin)      # 可执行文件
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${OUTPUT_BASE_DIR}/lib)      # 动态库
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${OUTPUT_BASE_DIR}/lib)      # 静态库

# 多配置生成器（VS/Xcode）需要单独设置每种配置的输出路径
foreach(CONFIG_TYPE ${CMAKE_CONFIGURATION_TYPES})           # Debug;Release;...
    string(TOUPPER ${CONFIG_TYPE} CONFIG_TYPE_UPPER)        # DEBUG/RELEASE
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${CONFIG_TYPE_UPPER} ${OUTPUT_BASE_DIR}/${CONFIG_TYPE}/bin)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${CONFIG_TYPE_UPPER} ${OUTPUT_BASE_DIR}/${CONFIG_TYPE}/lib)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${CONFIG_TYPE_UPPER} ${OUTPUT_BASE_DIR}/${CONFIG_TYPE}/lib)
endforeach()

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     7️⃣ 安装路径变量                                ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排这里？install() 命令依赖这些路径
# 📌 GNUInstallDirs 提供跨平台标准目录（bin/lib/include）

# 安装前缀
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    # 默认安装到构建目录下的 install/，而非系统目录
    set(CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/install" CACHE PATH "安装路径" FORCE)
endif()

# GNUInstallDirs 标准目录
include(GNUInstallDirs)  # 定义 CMAKE_INSTALL_BINDIR (bin)
                         #      CMAKE_INSTALL_LIBDIR (lib/lib64)
                         #      CMAKE_INSTALL_INCLUDEDIR (include)

message(STATUS "安装路径: ${CMAKE_INSTALL_PREFIX}")

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     8️⃣ 查找包相关变量                              ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排这里？find_package() 依赖 CMAKE_PREFIX_PATH 搜索路径
# 📌 必须在 target_link_libraries() 之前找到依赖

# 第三方库搜索路径
set(CMAKE_PREFIX_PATH
    "${PROJECT_THIRD_PARTY}"      # 项目内第三方库
    "/opt/Qt6"                    # 系统安装的 Qt
    "$ENV{HOME}/libs"             # 用户自定义库目录
    ${CMAKE_PREFIX_PATH}          # 保留已有路径
)

# 查找必需依赖
find_package(Qt6 COMPONENTS Core Widgets REQUIRED)  # REQUIRED = 找不到则报错终止
find_package(OpenSSL REQUIRED)
find_package(Threads REQUIRED)

# 查找可选依赖
find_package(ZLIB QUIET)    # QUIET = 找不到不报错，通过变量判断
find_package(Doxygen QUIET)

# 打印依赖状态
message(STATUS "========== 依赖状态 ==========")
message(STATUS "Qt6:     ${Qt6_FOUND} (${Qt6_VERSION})")
message(STATUS "OpenSSL: ${OPENSSL_FOUND} (${OPENSSL_VERSION})")
message(STATUS "ZLIB:    ${ZLIB_FOUND}")
message(STATUS "==============================")

# ╔══════════════════════════════════════════════════════════════════╗
# ║                     9️⃣ 项目选项（可选功能开关）                     ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么排这里？用户可通过 -D 覆盖默认值，放在目标定义前可控制条件编译
# 📌 option() 定义 BOOL 类型缓存变量

option(BUILD_SHARED_LIBS "构建动态库" ON)       # ON=动态库, OFF=静态库
option(BUILD_TESTS       "构建测试"   ON)       # 控制是否编译测试
option(BUILD_DOCS        "构建文档"   OFF)      # 控制是否生成文档
option(ENABLE_COVERAGE   "启用覆盖率" OFF)      # 控制覆盖率编译选项

# 使用：cmake -DBUILD_TESTS=OFF ..
# ╔══════════════════════════════════════════════════════════════════╗
# ║                     🔟 子目录与目标定义                           ║
# ╚══════════════════════════════════════════════════════════════════╝
# 📌 为什么放最后？所有配置就绪，目标可正确继承编译选项、链接依赖

add_subdirectory(src)            # 主程序/库

if(BUILD_TESTS)                  # 条件编译测试
    enable_testing()             # 启用 CTest
    add_subdirectory(tests)
endif()

if(BUILD_DOCS AND Doxygen_FOUND) # 条件生成文档
    add_subdirectory(docs)
endif()
```

## （2）使用精简版
```cpp
cmake_minimum_required(VERSION 3.15)

# ╔══════════════════════════════════════════════════════════════════╗
# ║  1️⃣ 项目信息                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

# ╔══════════════════════════════════════════════════════════════════╗
# ║  2️⃣ 编译器设置                                                   ║
# ╚══════════════════════════════════════════════════════════════════╝
set(CMAKE_CXX_STANDARD 17)              # C++17 标准
set(CMAKE_CXX_STANDARD_REQUIRED ON)     # 强制要求
set(CMAKE_CXX_EXTENSIONS OFF)           # 禁用扩展
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)   # 生成 compile_commands.json

# ╔══════════════════════════════════════════════════════════════════╗
# ║  3️⃣ 构建类型                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE Release CACHE STRING "" FORCE)  # 默认 Release
endif()

# ╔══════════════════════════════════════════════════════════════════╗
# ║  4️⃣ 路径定义                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
set(PROJECT_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include)   # 头文件目录

# ╔══════════════════════════════════════════════════════════════════╗
# ║  5️⃣ 输出路径                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)  # 可执行文件
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)  # 动态库
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)  # 静态库

# ╔══════════════════════════════════════════════════════════════════╗
# ║  6️⃣ 安装路径                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
include(GNUInstallDirs)  # 提供 CMAKE_INSTALL_BINDIR 等标准目录

# ╔══════════════════════════════════════════════════════════════════╗
# ║  7️⃣ 查找依赖                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
# find_package(spdlog CONFIG REQUIRED)

# ╔══════════════════════════════════════════════════════════════════╗
# ║  8️⃣ 项目选项                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
option(BUILD_TESTS "构建测试" ON)

# ╔══════════════════════════════════════════════════════════════════╗
# ║  9️⃣ 目标定义                                                     ║
# ╚══════════════════════════════════════════════════════════════════╝
add_subdirectory(src)

if(BUILD_TESTS)
   enable_testing()
   add_subdirectory(tests)
endif()

# ╔══════════════════════════════════════════════════════════════════╗
# ║  🔟 安装规则                                                      ║
# ╚══════════════════════════════════════════════════════════════════╝
install(TARGETS ${PROJECT_NAME}
   RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}   # bin/
   LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}   # lib/
   ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
install(DIRECTORY ${PROJECT_INCLUDE_DIR}/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
```
**构建命令**：
```cpp
cmake -S . -B build                 # 配置
cmake --build build                 # 编译
cmake --install build               # 安装
```

# 三、常用变量
## 1.项目信息变量
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118160323210.png)

### （2）使用示例
#### <1>常用精简版
```cpp
cmake_minimum_required(VERSION 3.15)
project(MyApp 
    VERSION 1.2.3
    LANGUAGES CXX
)
# 打印项目信息
message(STATUS "项目: ${PROJECT_NAME} v${PROJECT_VERSION}")

#                       ......

add_executable(${PROJECT_NAME} main.cpp)
```

#### <2>完整版
```cpp
cmake_minimum_required(VERSION 3.15)

# ==================== 项目定义 ====================
project(MyApp
    VERSION 1.2.3.4                           # 完整版本号（MAJOR.MINOR.PATCH.TWEAK）
    DESCRIPTION "一个示例应用程序"              # 项目描述
    HOMEPAGE_URL "https://github.com/user/myapp"  # 项目主页
    LANGUAGES CXX                              # 使用的语言
)

#                       ......

# ==================== 打印项目信息 ====================
message(STATUS "========== 项目信息 ==========")
message(STATUS "名称:     ${PROJECT_NAME}")
message(STATUS "版本:     ${PROJECT_VERSION}")
message(STATUS "  主版本: ${PROJECT_VERSION_MAJOR}")
message(STATUS "  次版本: ${PROJECT_VERSION_MINOR}")
message(STATUS "  补丁:   ${PROJECT_VERSION_PATCH}")
message(STATUS "  微调:   ${PROJECT_VERSION_TWEAK}")
message(STATUS "描述:     ${PROJECT_DESCRIPTION}")
message(STATUS "主页:     ${PROJECT_HOMEPAGE_URL}")
message(STATUS "==============================")

#                       ......

# ==================== 创建目标 ====================
add_executable(${PROJECT_NAME} main.cpp)

# 添加生成目录到头文件搜索路径
target_include_directories(${PROJECT_NAME} PRIVATE ${PROJECT_BINARY_DIR})
```
**配套模板文件 config.h.in**
```cpp
#ifndef CONFIG_H
#define CONFIG_H

// 项目信息（由 CMake 自动生成）
#define PROJECT_NAME "@PROJECT_NAME@"
#define PROJECT_VERSION "@PROJECT_VERSION@"
#define PROJECT_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define PROJECT_VERSION_MINOR @PROJECT_VERSION_MINOR@
#define PROJECT_VERSION_PATCH @PROJECT_VERSION_PATCH@
#define PROJECT_DESCRIPTION "@PROJECT_DESCRIPTION@"
#define PROJECT_HOMEPAGE_URL "@PROJECT_HOMEPAGE_URL@"

#endif // CONFIG_H
```
**配套 main.cpp**
```cpp
#include <iostream>
#include "config.h"

int main() {
    std::cout << PROJECT_NAME << " v" << PROJECT_VERSION << std::endl;
    std::cout << "描述: " << PROJECT_DESCRIPTION << std::endl;
    std::cout << "主页: " << PROJECT_HOMEPAGE_URL << std::endl;
    return 0;
}
```

## 2.平台与系统变量
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118164449876.png)

### （2）使用示例
#### <1>常用精简版
```cpp
#                       ......

if(WIN32)                                      # Windows（包括64位）
    set(PLATFORM_NAME "windows")
elseif(APPLE)                                  # macOS、iOS
    set(PLATFORM_NAME "macos")
elseif(UNIX)                                   # Linux、BSD 等
    set(PLATFORM_NAME "linux")
endif()

# 架构检测
if(CMAKE_SIZEOF_VOID_P EQUAL 8)                # 指针大小判断架构
    set(ARCH_NAME "x64")                       # 8字节 = 64位
else()
    set(ARCH_NAME "x86")                       # 4字节 = 32位
endif()

message(STATUS "平台: ${CMAKE_SYSTEM_NAME} (${PLATFORM_NAME})")
message(STATUS "架构: ${CMAKE_SYSTEM_PROCESSOR} (${ARCH_NAME})")

#                       ......
```
**已经用“CMAKE_SYSTEM_NAME变量”了，为什么还要自定义“PLATFORM_NAME”？**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118164758165.png)

#### <2>完整版
```cpp
#                       ......

# ══════════════════════════════════════════════════════════════════
# 系统信息打印
# ══════════════════════════════════════════════════════════════════
message(STATUS "系统名称:   ${CMAKE_SYSTEM_NAME}")        # Linux/Windows/Darwin
message(STATUS "系统版本:   ${CMAKE_SYSTEM_VERSION}")     # 5.15.0
message(STATUS "处理器:     ${CMAKE_SYSTEM_PROCESSOR}")   # x86_64/aarch64
message(STATUS "主机系统:   ${CMAKE_HOST_SYSTEM_NAME}")   # 交叉编译时有用

# ══════════════════════════════════════════════════════════════════
# 平台检测（布尔变量）
# ══════════════════════════════════════════════════════════════════
if(WIN32)
    set(PLATFORM_NAME "windows")
    message(STATUS "检测到 Windows")
elseif(APPLE)
    set(PLATFORM_NAME "macos")
    message(STATUS "检测到 macOS (Darwin)")
elseif(UNIX)
    set(PLATFORM_NAME "linux")
    message(STATUS "检测到 Linux/Unix")
endif()

# ══════════════════════════════════════════════════════════════════
# 架构检测
# ══════════════════════════════════════════════════════════════════
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(ARCH_NAME "x64")
else()
    set(ARCH_NAME "x86")
endif()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm|aarch64|ARM64")
    set(ARCH_NAME "arm64")
endif()

# ══════════════════════════════════════════════════════════════════
# 编译器检测
# ══════════════════════════════════════════════════════════════════
if(MSVC)
    message(STATUS "编译器: MSVC")
    add_compile_options(/W4 /utf-8)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    message(STATUS "编译器: GCC")
    add_compile_options(-Wall -Wextra)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    message(STATUS "编译器: Clang")
    add_compile_options(-Wall -Wextra)
endif()

# ══════════════════════════════════════════════════════════════════
# 交叉编译检测
# ══════════════════════════════════════════════════════════════════
if(NOT CMAKE_SYSTEM_NAME STREQUAL CMAKE_HOST_SYSTEM_NAME)
    message(STATUS "交叉编译: ${CMAKE_HOST_SYSTEM_NAME} → ${CMAKE_SYSTEM_NAME}")
endif()

message(STATUS "最终: ${PLATFORM_NAME}/${ARCH_NAME}")

#                       ......
```

## 3.编译器与工具链变量
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118153916817.png)

### （2）使用示例
```cpp
cmake_minimum_required(VERSION 3.15)
project(MyProject LANGUAGES CXX)

# ==================== C++ 标准设置 ====================
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ==================== 编译选项 ====================
# 第3类：定义编译选项的内容
set(CMAKE_CXX_FLAGS         "-Wall -Wextra")      # 所有模式都用
set(CMAKE_CXX_FLAGS_DEBUG   "-g -O0")             # Debug 专用
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG")       # Release 专用

# ==================== 条件编译（利用只读变量） ====================
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    message(STATUS "使用 GCC 编译器，版本: ${CMAKE_CXX_COMPILER_VERSION}")
    add_compile_options(-Wno-unused-parameter)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    message(STATUS "使用 Clang 编译器")
    add_compile_options(-Wno-unused-lambda-capture)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    message(STATUS "使用 MSVC 编译器")
    add_compile_options(/W4 /utf-8)
endif()

# ==================== 版本检查示例 ====================
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.0)
    message(WARNING "GCC 版本过低，建议升级到 9.0+")
endif()

add_executable(main main.cpp)
```
```bash
# 1. 创建构建目录并进入
mkdir build && cd build

# 2. 配置（选择构建类型）
cmake -DCMAKE_BUILD_TYPE=Debug ..    # Debug 模式
# 或
cmake -DCMAKE_BUILD_TYPE=Release ..  # Release 模式

# 3. 编译
make

# 运行（在 build 目录下）
./main

# GDB 调试（需要 Debug 构建）
gdb ./main
```

## 4.构建类型变量
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118202232268.png)

### （2）使用示例

#### <1>常用精简版
```cpp
# 单配置生成器（Makefile/Ninja）设置默认构建类型
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "" FORCE)
endif()

```
#### <2>完整版
```cpp
# ══════════════════════════════════════════════════════════════════
# 构建类型设置
# ══════════════════════════════════════════════════════════════════

# 📌 判断条件说明：
#    - CMAKE_BUILD_TYPE 空 → 单配置生成器未指定类型
#    - CMAKE_CONFIGURATION_TYPES 空 → 不是多配置生成器（VS/Xcode）
if(NOT CMAKE_BUILD_TYPE)
    # 📌 CACHE STRING：写入缓存，cmake-gui 可见
    # 📌 FORCE：强制覆盖已有值
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "构建类型" FORCE)
    
    # 📌 cmake-gui 中显示下拉选项
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
        "Debug" "Release" "RelWithDebInfo" "MinSizeRel")
endif()

message(STATUS "当前构建类型: ${CMAKE_BUILD_TYPE}")

# ══════════════════════════════════════════════════════════════════
# 多配置生成器处理（VS/Xcode）
# ══════════════════════════════════════════════════════════════════

# 📌 CMAKE_CONFIGURATION_TYPES 仅多配置生成器有值
if(CMAKE_CONFIGURATION_TYPES)
    # 📌 限制只支持 Debug 和 Release
    set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "" FORCE)
    message(STATUS "多配置类型: ${CMAKE_CONFIGURATION_TYPES}")
endif()

# ══════════════════════════════════════════════════════════════════
# 按构建类型条件编译（生成器表达式，推荐）
# ══════════════════════════════════════════════════════════════════

add_executable(myapp main.cpp)

# 📌 $<CONFIG:xxx> 在生成时根据实际配置展开
# 📌 单配置和多配置生成器都适用
target_compile_definitions(myapp PRIVATE
    $<$<CONFIG:Debug>:DEBUG_MODE>      # Debug 时定义 DEBUG_MODE
    $<$<CONFIG:Release>:NDEBUG>        # Release 时定义 NDEBUG
)

target_compile_options(myapp PRIVATE
    $<$<CONFIG:Debug>:-g -O0>          # Debug：无优化+调试符号
    $<$<CONFIG:Release>:-O3>           # Release：最高优化
)
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118203157380.png)

## 拓：编译器变量 vs 构建类型变量
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118203245613.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118203314548.png)


## 5.项目路径变量
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118105157843.png)
```
myproject/                      ← CMAKE_SOURCE_DIR / PROJECT_SOURCE_DIR
├── CMakeLists.txt
├── build/                      ← CMAKE_BINARY_DIR / PROJECT_BINARY_DIR
├── src/
│   └── module/
│       └── CMakeLists.txt      ← CMAKE_CURRENT_SOURCE_DIR (处理此文件时)
└── cmake/
    └── utils.cmake             ← CMAKE_CURRENT_LIST_DIR (include 此文件时)
```

### （2）使用示例
**以（1）中的目录为例**
#### <1>常用精简版
```cpp
cmake_minimum_required(VERSION 3.15)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

# 加载自定义模块
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

# 头文件 & 源文件
include_directories(${PROJECT_SOURCE_DIR}/include)
add_subdirectory(src/module)

add_executable(${PROJECT_NAME} src/main.cpp)
```

#### <2>完整版
```cpp
cmake_minimum_required(VERSION 3.15)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ==================== 目录变量演示 ====================
message(STATUS "========== 顶层 CMakeLists.txt ==========")
message(STATUS "CMAKE_SOURCE_DIR:         ${CMAKE_SOURCE_DIR}")
#                                          → /home/user/myproject
message(STATUS "CMAKE_BINARY_DIR:         ${CMAKE_BINARY_DIR}")
#                                          → /home/user/myproject/build
message(STATUS "PROJECT_SOURCE_DIR:       ${PROJECT_SOURCE_DIR}")
#                                          → /home/user/myproject（与 CMAKE_SOURCE_DIR 相同）
message(STATUS "PROJECT_BINARY_DIR:       ${PROJECT_BINARY_DIR}")
#                                          → /home/user/myproject/build
message(STATUS "CMAKE_CURRENT_SOURCE_DIR: ${CMAKE_CURRENT_SOURCE_DIR}")
#                                          → /home/user/myproject（当前处理的文件所在目录）
message(STATUS "CMAKE_CURRENT_BINARY_DIR: ${CMAKE_CURRENT_BINARY_DIR}")
#                                          → /home/user/myproject/build
message(STATUS "=============================================")

# ==================== 加载自定义模块 ====================
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")
include(utils)  # 加载 cmake/utils.cmake

# ==================== 输出目录配置 ====================
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

# ==================== 添加子目录 ====================
add_subdirectory(src/module)

# ==================== 主程序 ====================
add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE module)
```
**子目录 myproject/src/module/CMakeLists.txt**
```cpp
# ==================== 子目录中的目录变量 ====================
message(STATUS "========== src/module/CMakeLists.txt ==========")
message(STATUS "CMAKE_SOURCE_DIR:         ${CMAKE_SOURCE_DIR}")
#                                          → /home/user/myproject（不变，始终是顶层）
message(STATUS "CMAKE_BINARY_DIR:         ${CMAKE_BINARY_DIR}")
#                                          → /home/user/myproject/build（不变）
message(STATUS "PROJECT_SOURCE_DIR:       ${PROJECT_SOURCE_DIR}")
#                                          → /home/user/myproject（没有新 project()，沿用父级）
message(STATUS "CMAKE_CURRENT_SOURCE_DIR: ${CMAKE_CURRENT_SOURCE_DIR}")
#                                          → /home/user/myproject/src/module ⭐ 变了
message(STATUS "CMAKE_CURRENT_BINARY_DIR: ${CMAKE_CURRENT_BINARY_DIR}")
#                                          → /home/user/myproject/build/src/module ⭐ 变了
message(STATUS "================================================")

# ==================== 创建模块库 ====================
add_library(module STATIC
    ${CMAKE_CURRENT_SOURCE_DIR}/module.cpp
)

target_include_directories(module PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

## 6.输出路径变量
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118152755840.png)

### （2）使用示例
```cpp
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
```

## 7.安装路径
你的项目编译产物的安装路径，即执行 cmake --install 或 make install 时，文件被复制到的目标位置。
```
源码目录          构建目录              安装目录
src/main.cpp  →  build/myapp      →  /usr/local/bin/myapp
src/mylib.cpp →  build/libmy.so   →  /usr/local/lib/libmy.so
include/*.h   →  （不变）          →  /usr/local/include/*.h

    编译                 安装
  cmake --build .    cmake --install .
```

### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118203635317.png)

### （2）使用示例
#### <1>常用精简版
```cpp
# 设置安装前缀
set(CMAKE_INSTALL_PREFIX "/opt/myapp" CACHE PATH "" FORCE)

# 安装可执行文件和库
install(TARGETS myapp mylib
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

# 安装头文件
install(DIRECTORY include/ DESTINATION include)
```
#### <2>完整版
```cpp
# ══════════════════════════════════════════════════════════════════
# 引入 GNUInstallDirs 模块（提供标准安装目录变量）
# ══════════════════════════════════════════════════════════════════

# 📌 自动设置 CMAKE_INSTALL_BINDIR、CMAKE_INSTALL_LIBDIR 等
# 📌 会根据系统自动处理 lib vs lib64
include(GNUInstallDirs)

# ══════════════════════════════════════════════════════════════════
# 自定义安装前缀（可选）
# ══════════════════════════════════════════════════════════════════

# 📌 仅在未指定时设置默认值
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "/opt/${PROJECT_NAME}" CACHE PATH "安装路径" FORCE)
endif()

message(STATUS "安装前缀: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "  bin:     ${CMAKE_INSTALL_BINDIR}")
message(STATUS "  lib:     ${CMAKE_INSTALL_LIBDIR}")
message(STATUS "  include: ${CMAKE_INSTALL_INCLUDEDIR}")

# ══════════════════════════════════════════════════════════════════
# 定义目标
# ══════════════════════════════════════════════════════════════════

add_executable(myapp src/main.cpp)
add_library(mylib SHARED src/mylib.cpp)
add_library(mylib_static STATIC src/mylib.cpp)

# ══════════════════════════════════════════════════════════════════
# 安装可执行文件
# ══════════════════════════════════════════════════════════════════

install(TARGETS myapp
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}   # 📌 → prefix/bin
)

# ══════════════════════════════════════════════════════════════════
# 安装库文件
# ══════════════════════════════════════════════════════════════════

install(TARGETS mylib mylib_static
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}   # 📌 .so → prefix/lib
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}   # 📌 .a  → prefix/lib
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}   # 📌 .dll（Windows）
)

# ══════════════════════════════════════════════════════════════════
# 安装头文件
# ══════════════════════════════════════════════════════════════════

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}       # 📌 → prefix/include
    FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"  # 📌 只复制头文件
)

# 或安装单个头文件
install(FILES include/mylib.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/mylib
)

# ══════════════════════════════════════════════════════════════════
# 安装数据/资源文件
# ══════════════════════════════════════════════════════════════════

install(DIRECTORY resources/
    DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}  # 📌 → prefix/share/myapp
)

# ══════════════════════════════════════════════════════════════════
# 安装配置文件
# ══════════════════════════════════════════════════════════════════

install(FILES config/myapp.conf
    DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/${PROJECT_NAME}  # 📌 → prefix/etc/myapp
)
```
**安装命令**
```cpp
# 配置
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=/opt/myapp

# 编译
cmake --build build

# 安装（可能需要 sudo）
cmake --install build

# 安装到临时目录（打包用）
cmake --install build --prefix ./package
```


## 8.查包相关变量
### （1）汇总表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118155211465.png)

### （2）使用示例
#### <1>常用精简版
```cpp
cmake_minimum_required(VERSION 3.15)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ==================== 搜索路径 ====================
set(CMAKE_PREFIX_PATH "/opt/Qt6" "/usr/local")

# ==================== 查找包 ====================
find_package(Qt6 COMPONENTS Core Widgets REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(ZLIB QUIET)  # 可选

# ==================== 条件判断 ====================
if(ZLIB_FOUND)
    message(STATUS "ZLIB 版本: ${ZLIB_VERSION_STRING}")
endif()

# ==================== 创建目标 ====================
add_executable(myapp main.cpp)

# ==================== 链接库（现代方式） ====================
target_link_libraries(myapp PRIVATE
    Qt6::Core
    Qt6::Widgets
    OpenSSL::SSL
    OpenSSL::Crypto
    $<$<BOOL:${ZLIB_FOUND}>:ZLIB::ZLIB>  # 条件链接
)
```

#### <2>完整版
```cpp
cmake_minimum_required(VERSION 3.15)
project(FindPackageDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ==================== 搜索路径配置 ====================
# 指定第三方库安装位置
set(CMAKE_PREFIX_PATH 
    "/opt/Qt6"
    "/usr/local"
    "$ENV{HOME}/libs"
)

# 添加自定义 CMake 模块路径
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

# ==================== 查找包 ====================
# 查找 Qt6（CONFIG 模式）
find_package(Qt6 COMPONENTS Core Widgets REQUIRED)

# 查找 OpenSSL（MODULE 模式）
find_package(OpenSSL REQUIRED)

# 查找 ZLIB（可选）
find_package(ZLIB QUIET)

# ==================== 条件判断：_FOUND ====================
if(Qt6_FOUND)
    message(STATUS "✅ Qt6 找到")
    message(STATUS "   版本: ${Qt6_VERSION}")
    message(STATUS "   配置文件: ${Qt6_CONFIG}")
else()
    message(FATAL_ERROR "❌ Qt6 未找到")
endif()

if(OpenSSL_FOUND)
    message(STATUS "✅ OpenSSL 找到")
    message(STATUS "   版本: ${OPENSSL_VERSION}")
    message(STATUS "   头文件: ${OPENSSL_INCLUDE_DIR}")
    message(STATUS "   库文件: ${OPENSSL_LIBRARIES}")
endif()

if(ZLIB_FOUND)
    message(STATUS "✅ ZLIB 找到，版本: ${ZLIB_VERSION_STRING}")
else()
    message(STATUS "⚠️ ZLIB 未找到，将使用内置实现")
endif()

# ==================== 版本检查：_VERSION_* ====================
if(Qt6_VERSION_MAJOR GREATER_EQUAL 6 AND Qt6_VERSION_MINOR GREATER_EQUAL 5)
    message(STATUS "Qt6 版本 >= 6.5，启用新特性")
    add_compile_definitions(QT_NEW_FEATURES)
endif()

# 精确版本比较
if(Qt6_VERSION VERSION_LESS "6.2.0")
    message(WARNING "Qt6 版本过低，建议升级到 6.2.0+")
endif()

# ==================== 创建目标 ====================
add_executable(myapp main.cpp)

# ==================== 使用 _INCLUDE_DIRS ====================
# 方式1：现代 CMake（推荐，自动传递）
target_link_libraries(myapp PRIVATE 
    Qt6::Core 
    Qt6::Widgets
)

# 方式2：传统方式（显式指定）
if(OpenSSL_FOUND)
    target_include_directories(myapp PRIVATE ${OPENSSL_INCLUDE_DIR})
    target_link_libraries(myapp PRIVATE ${OPENSSL_LIBRARIES})
endif()

# ==================== 使用 _LIBRARIES ====================
# 现代导入目标（推荐）
# Qt6::Core, Qt6::Widgets 自动包含头文件路径和依赖

# 传统库变量
if(ZLIB_FOUND)
    target_include_directories(myapp PRIVATE ${ZLIB_INCLUDE_DIRS})
    target_link_libraries(myapp PRIVATE ${ZLIB_LIBRARIES})
endif()

# ==================== 打印所有变量（调试用） ====================
message(STATUS "========== 包信息汇总 ==========")
message(STATUS "Qt6_FOUND:         ${Qt6_FOUND}")
message(STATUS "Qt6_VERSION:       ${Qt6_VERSION}")
message(STATUS "Qt6_VERSION_MAJOR: ${Qt6_VERSION_MAJOR}")
message(STATUS "Qt6_VERSION_MINOR: ${Qt6_VERSION_MINOR}")
message(STATUS "Qt6_CONFIG:        ${Qt6_CONFIG}")
message(STATUS "================================")
```