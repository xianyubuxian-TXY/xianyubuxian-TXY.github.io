---
title: sqlpp11的安装
date: 2026-01-12 11:24:32
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- sqlpp11（已不准备使用）
---

# 提要（请看完再决定是否继续看）
1. 安装时间： 2026年1月12日 （注意：如果你安装时间较晚，下述方法可能已不再适用）
2. 安装后版本： 0.65 + 11 commits（相较于0.65 有11 个额外的 commit ）——>（可切换为0.65）
- **注**：也许你安装后并不是这个版本，已经升级
3. **注意**：也许由于版本不同等问题，安装后进行的“测试代码”不再适用，语法可能也不再适用，请自己斟酌！ 

## 个人抱怨
##### 1.在网上没找到很完整的教程，中间尝试了不同的安装方式（vcpkg等），不同安装版本，总是出问题，安装都花了一天（也许是我个人能力太低😞），真的很无奈。
##### 2.如果在这个过程中，你遇到任何问题，请寻找其它参考教程以及AI的帮助



# 一、sqlpp11介绍与环境搭建

## 1.sqlpp11介绍
- SQLPP11是一款基于C++11及以上标准的类型安全嵌入式SQL库，核心价值在于让开发者用原生C++语法编写SQL语句，通过编译期检查规避语法错误、类型不匹配及SQL注入风险
- 兼容MySQL、SQLite、PostgreSQL等主流数据库，零运行时开销且符合C++开发习惯

**核心优势**：
- **类型安全**：编译期校验SQL语法、字段类型匹配性（如int字段赋值字符串直接编译失败）；
- **原生C++集成**：支持移动语义、lambda、类型推导等C++11+特性，无需手写SQL字符串；
- **多库兼容**：通过适配层对接主流数据库，核心代码无需修改即可切换数据库；
- **轻量高效**：模板元编程实现核心逻辑，编译期完成大部分计算，无额外运行时开销。



## 2.安装教程（Linux，sqlpp11+mysql为例）
**注**：如果你在下面任何步骤出现问题无法解决且不想继续，可以查看[卸载/清理教程](#clean)进行清除

**以下库都放置在 “~/thirdparty” 目录下**

### （1）系统依赖
确保你已经安装了：
```text
g++ (>= 5.0) 或 clang++
cmake (>= 3.10)
git
make
libmysqlclient-dev   # MySQL C API 库头文件和链接库
```
以Ubuntu为例
```bash
sudo apt update
sudo apt install g++ cmake git make libmysqlclient-dev
```

### （2）安装sqlpp11
```bash
cd ~/thirdparty

# 克隆 sqlpp11
git clone https://github.com/rbock/sqlpp11.git
cd sqlpp11

# 创建构建目录
mkdir build && cd build

# 查看所有可用的 CMake 选项（-L 列出缓存变量）
# grep -i mysql 过滤出与 MySQL 相关的选项
# 目的：确认启用 MySQL connector 的选项名称（不同版本可能不同）
cmake .. -L | grep -i mysql 
# 预期输出类似：BUILD_MYSQL_CONNECTOR:BOOL=OFF

# 清空构建目录，准备正式编译
rm -rf *

# 正式编译
# -DBUILD_MYSQL_CONNECTOR=ON  : 启用 MySQL connector（上一步查到的选项）
# -DCMAKE_INSTALL_PREFIX=/usr/local : 指定安装路径
cmake .. \
  -DBUILD_MYSQL_CONNECTOR=ON \
  -DCMAKE_INSTALL_PREFIX=/usr/local

# 并行编译（-j$(nproc) 使用所有 CPU 核心加速）
make -j$(nproc)

# 安装到系统目录（需要 sudo 权限）
sudo make install
```

**如果连接github失败，且你的宿主机上有VPN，可以看我另一篇笔记[Linux设置VPN代理（解决github连接问题）](/categories/Linux/Linux的使用/Linux设置VPN代理（解决github连接问题）/)**

### （3）测试 sqlpp11 MySQL 连接
**个人认为还是通过“代码测试”比较靠谱，毕竟能用才是“目标”**

#### <1>创建测试项目
```bash
mkdir -p ~/test_sqlpp11
cd ~/test_sqlpp11
```

**main.cpp**：
- **注意**：
	- mysql连接配置换成你自己的！！
	- test数据库需要先创建，或换为你已有的数据库
```cpp
#include <sqlpp11/mysql/mysql.h>
#include <iostream>

int main()
{
    try
    {
        sqlpp::mysql::global_library_init();

        // 使用 shared_ptr
        auto config = std::make_shared<sqlpp::mysql::connection_config>();
        config->host = "localhost";
        config->port = 3306;
        config->user = "root";
        config->password = "password";
        config->database = "test";
        config->debug = true;

        // 直接传 shared_ptr
        sqlpp::mysql::connection db{config};

        db.execute("SELECT 1");

        std::cout << "✅ 连接成功！" << std::endl;
    }
    catch (const std::exception& e)
    {
        std::cerr << "❌ 失败: " << e.what() << std::endl;
        return 1;
    }
    return 0;
}
```

**CMakeLists.txt**
```cmake
cmake_minimum_required(VERSION 3.14)
project(test_sqlpp11)

set(CMAKE_CXX_STANDARD 17)

# 用 pkg-config 找 MySQL
find_package(PkgConfig REQUIRED)
pkg_check_modules(MYSQL REQUIRED IMPORTED_TARGET mysqlclient)

# 创建 MySQL::MySQL alias（sqlpp11 需要这个名字）
if(NOT TARGET MySQL::MySQL)
    add_library(MySQL::MySQL ALIAS PkgConfig::MYSQL)
endif()

find_package(Sqlpp11 REQUIRED)
find_package(Threads REQUIRED)

add_executable(test_mysql main.cpp)

target_link_libraries(test_mysql 
    PRIVATE 
    sqlpp11::sqlpp11
    sqlpp11::mysql
    Threads::Threads
)
```

#### <2>编译运行：
```bash
cd ~/test_sqlpp11
mkdir build && cd build

cmake .. -DCMAKE_PREFIX_PATH=/usr/local
make

./test_mysql
```

#### <3>查看安装版本的方法
```bash
cd ~/thirdparty/sqlpp11
git describe --tags --always
# 输出类似: 0.65-11-gabcdef （0.65 版本 + 11 个 commit）
```


## 3.卸载/清理教程 <a id="clean"></a>
##### 如果你在上面步骤中出现任何问题，不想继续，可按下面方式进行清除
```bash
# ==========================================
# sqlpp11 完整卸载脚本
# ==========================================

# 1. 尝试官方卸载（如果 build 目录还在）
cd ~/thirdparty/sqlpp11/build 2>/dev/null && sudo make uninstall

# 2. 删除头文件
sudo rm -rf /usr/local/include/sqlpp11

# 3. 删除 cmake 配置文件
sudo rm -rf /usr/local/lib/cmake/Sqlpp11
sudo rm -rf /usr/local/lib/cmake/sqlpp11
sudo rm -rf /usr/local/share/sqlpp11

# 4. 删除库文件（如果有编译出 .a 或 .so）
sudo rm -f /usr/local/lib/libsqlpp11*
sudo rm -f /usr/local/lib/libsqlpp-mysql*

# 5. 删除 pkg-config 文件（如果有）
sudo rm -f /usr/local/lib/pkgconfig/sqlpp11*.pc

# 6. 删除可能安装到 /usr 的文件
sudo rm -rf /usr/include/sqlpp11
sudo rm -rf /usr/lib/cmake/Sqlpp11
sudo rm -rf /usr/lib/cmake/sqlpp11

# 7. 删除源码目录
rm -rf ~/thirdparty/sqlpp11

# 8. 更新链接器缓存
sudo ldconfig

# 9. 验证是否清理干净
echo "=== 检查残留文件 ==="
find /usr/local -name "*sqlpp*" 2>/dev/null
find /usr -name "*sqlpp*" 2>/dev/null

```