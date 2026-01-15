---
title: redis-plus-plus使用教程
date: 2026-01-12 22:15:28
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- redis-plus-plus
---

[附录1：Redis++ 核心场景](#use)
[附录2：Redis++ 重要机制](#mechanism)
[附录3：Redis++ 安装教程](#install)

# 一、Redis++概述
## 1.Redis++ 适配模式介绍
Redis++ 支持多种部署模式的适配，不同模式对应不同的生产场景需求，核心模式如下：
- **单机模式**：
	- 适配小型应用、开发/测试环境，直接连接单个Redis节点，部署简单但无高可用保障。
	- 生产环境仅推荐用于**非核心业务**，需配合数据备份策略使用。​
- **主从模式**：
	- 适配**读多写少**的生产场景
	- 包含一个主节点（写入）和多个从节点（读取）
	- Redis++可通过配置指定读请求路由到从节点，提升读吞吐量，主从故障切换需依赖哨兵或手动操作。​
- **哨兵模式**：
	- 在**主从模式基础上**增加哨兵节点，负责监控主从节点健康状态
	- 当主节点故障时自动完成故障切换，保障高可用性
	- Redis++通过连接哨兵节点获取主从信息，无需手动维护节点地址。​
- **集群模式（Cluster）**：
	- 适配**海量数据、高并发**的核心生产场景，将数据分片存储在多个主节点，每个主节点对应多个从节点。
	- Redis++提供RedisCluster客户端，自动适配数据分片、节点发现和故障切换，API与单机模式兼容，生产可无缝切换。

**注意！！**：
- 主从复制、哨兵模式、集群模式都是**由“Redis服务端”手动配置**
- Redis++是Redis客户端库
	- **不负责配置**“主从复制”、“哨兵模式”、“集群模式”
	- 只负责**连接到Redis服务器**
- ![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113171238862.png)

## 2.Redis++核心配置介绍
Redis++的配置核心围绕**连接参数**和**池化参数**，不同模式的配置略有差异

### （1）基础连接配置
- **host/port**：
	- Redis节点的**ip地址 和 端口port**
		- 单机模式：填写单个节点信息
		- 集群/哨兵模式：填写任意一个节点（客户端自动发现其他节点）
	- 生产环境建议使用内网IP或域名，避免公网传输风险。
- **password**：
	- **Redis访问密码**，防未授权访问
	- **生产环境必须强制设置**：
		- 长度建议不小于8位
		- 包含字母、数字和特殊字符
- **db**：
	- **Redis逻辑数据库编号**（0-15默认）
	- 生产环境建议按业务模块隔离数据库
		- 例如用户模块用db1、订单模块用db2，避免键名冲突和数据混乱。

- **socket_timeout**：
	- **套接字读写超时时间**
	- 生产环境建议设置500ms-1s，避免因网络阻塞导致应用线程挂起。

- **connect_timeout**：
	- **连接建立超时时间**
	- 生产环境建议设置1s-2s，防止因Redis节点故障导致连接等待过久。

### （2）连接池配置
- **size**：
	- **连接池大小**
	- 需根据业务QPS调整，生产环境建议20-50个连接（高QPS场景可增至100）
		- 连接过多：导致Redis资源耗尽
		- 连接过少：导致请求排队。
- **wait_timeout**：
	- **获取连接的等待超时时间**
	- 生产环境建议设置200-500ms，超时后抛出异常，避免应用线程无限等待连接。

- **connection_lifetime**：
	- **连接最大生存时间**
	- 生产环境建议设置5-10分钟，自动关闭长时间空闲的连接，避免无效连接占用资源。

## 3.核心依赖与编译规范
### （1）核心依赖与编译规范
- **编译选项**：
	- 必须指定 -std=c++17
	- 链接 -lredis++ -lhiredis -pthread（线程库）；
- **编译优化**：
	- 添加 -O2 提升性能
	- -Wall -Werror 严格检查代码；
- **头文件**：
	- #include <sw/redis++/redis++.h>（核心）
	- #include <sw/redis++/cluster.h>（集群）。

### （2）CMake 项目集成
```cmake
cmake_minimum_required(VERSION 3.10)
project(MyRedisProject)

# ============ 1. 设置 C++ 标准（必须与 redis++ 编译时一致） ============
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ============ 2. 查找 hiredis 库 ============
find_path(HIREDIS_HEADER hiredis)
find_library(HIREDIS_LIB hiredis)

# ============ 3. 查找 redis++ 库 ============
# 注意：这里是 sw，不是 redis++
find_path(REDIS_PLUS_PLUS_HEADER sw)
find_library(REDIS_PLUS_PLUS_LIB redis++)

# ============ 4. 创建可执行文件 ============
add_executable(myapp main.cpp)

# ============ 5. 添加头文件路径 ============
target_include_directories(myapp PRIVATE 
    ${HIREDIS_HEADER}
    ${REDIS_PLUS_PLUS_HEADER}
)

# ============ 6. 链接库 ============
target_link_libraries(myapp PRIVATE
    ${REDIS_PLUS_PLUS_LIB}
    ${HIREDIS_LIB}
    pthread  # 必须链接线程库
)
```


### （3）连接配置示例
- 生产环境**禁止使用单连接**，必须用**连接池**
- 同时添加超时、密码等生产必备配置
- **注**：
	- 如果业务需要，可在业务层添加“**手动重试封装**”
```cpp
#include <sw/redis++/redis++.h>
#include <iostream>
#include <string>
#include <chrono>
#include <stdexcept>

using namespace sw::redis;
using namespace std;

// ═══════════════════════════════════════════════════════════
// Redis 连接创建函数（带连接池）
// ═══════════════════════════════════════════════════════════
Redis create_prod_redis_conn() {
    // 1. 基础连接配置
    ConnectionOptions opts;
    opts.host = "127.0.0.1";
    opts.port = 6379;
    opts.password = "your_redis_pwd";  // 无密码则注释掉
    opts.db = 0;
    opts.socket_timeout = chrono::milliseconds(500);
    opts.connect_timeout = chrono::milliseconds(1000);
    // 注意：redis++ 默认会自动重连，无需手动配置重连参数

    // 2. 连接池配置
    ConnectionPoolOptions pool_opts;
    pool_opts.size = 20;
    pool_opts.wait_timeout = chrono::milliseconds(200);
    pool_opts.connection_lifetime = chrono::minutes(5); 

    try {
        return Redis(opts, pool_opts);
    } catch (const Error& e) {
        cerr << "Redis 连接失败：" << e.what() << endl;
        throw runtime_error("Redis 初始化失败");
    }
}

// ═══════════════════════════════════════════════════════════
// 带重试的执行封装（解决重连问题）
// ═══════════════════════════════════════════════════════════
template<typename Func>
auto redis_exec_with_retry(Redis& redis, Func&& func, int max_retry = 3) 
    -> decltype(func(redis)) 
{
    for (int i = 0; i < max_retry; ++i) {
        try {
            return func(redis);
        } catch (const TimeoutError& e) {
            cerr << "Redis 超时，重试 " << (i + 1) << "/" << max_retry << endl;
            if (i == max_retry - 1) throw;
        } catch (const IoError& e) {
            cerr << "Redis IO 错误，重试 " << (i + 1) << "/" << max_retry << endl;
            if (i == max_retry - 1) throw;
        }
        this_thread::sleep_for(chrono::milliseconds(100 * (i + 1)));  // 退避
    }
    throw runtime_error("Redis 重试耗尽");
}

// ═══════════════════════════════════════════════════════════
// 使用示例
// ═══════════════════════════════════════════════════════════
void example_usage() {
    auto redis = create_prod_redis_conn();
    
    // 普通调用
    try {
        redis.set("key", "value");
    } catch (const Error& e) {
        cerr << "操作失败: " << e.what() << endl;
    }
    
    // 带重试的调用
    auto result = redis_exec_with_retry(redis, [](Redis& r) {
        return r.get("key");
    });
}

```

# 二、官方文档提示事项
**建议先阅读之后的小节，这里放在前面是为了“强调它的重要性”**

## 1.Connection连接
### （1）尽可能复用 Redis 对象
- **你应该尽可能复用 Redis 对象**
	- 因为**创建 Redis 对象的开销并不小**，因为它会创建到 Redis 服务器的新连接。
	- ![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114000730301.png)
- **Redis对象是多线程安全的**
	- 你可以在多个线程中共享同一个 Redis 对象。
	- ![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114000814117.png)

### （2）自动重连机制
- Redis 类维护一个连接到 Redis 服务器的连接池。
	- 如果连接断开，Redis 会**自动重新连接**到 Redis 服务器。
- **不需要手动检查 Redis 对象是否成功连接到服务器**
	- 如果 Redis 无法创建到 Redis 服务器的连接，或者连接在某个时刻断开，当你尝试发送命令时，它会**抛出 Error 类型的异常**
	- **即使你收到异常（即连接断开），你也不需要创建新的 Redis 对象！！**
		- 你可以复用同一个 Redis 对象继续发送命令，Redis 对象会自动尝试重连服务器。
		- 如果重连成功，它会发送命令到服务器；
		- 否则，它会再次抛出异常。
	- ![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114000348588.png)


### （3）“阻塞命令”的使用
- 当使用**阻塞命令**时，socket_timeout **必须大于阻塞命令的超时时间**，否则会导致误报超时和数据丢失。
	- ![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113234947400.png)

### （4）连接池的懒加载机制
- 连接池中的连接是**懒加载（延迟创建）**的
	- 当连接池初始化时（即 Redis 构造函数执行时），Redis 不会立即连接到服务器
	- 只有在你尝试**发送命令时才会建立连接**
		- 通过这种方式，我们可以**避免不必要的连接**
	- 因此，如果连接池大小设置为 5，但实际最大并发连接数只有 3，那么连接池中实际只会有 3 个连接。

### （5）URI 参数区分大小写（要求全小写）
```cpp
// ✅ 正确：全小写
auto redis = Redis("tcp://127.0.0.1:6379?socket_timeout=100ms&keep_alive=true");

// ❌ 错误：大小写混用
auto redis = Redis("tcp://127.0.0.1:6379?Socket_Timeout=100ms");  // 无法识别
```

### （6）Pipeline / Transaction 注意事项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114084518210.png)

### （7）Subscriber 订阅者注意事项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114084903240.png)

### （8）Redis Cluster 集群注意事项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114084633220.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114084947303.png)

### （9）异步接口注意事项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114084713792.png)


### （10）异常处理
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114084746832.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114085208456.png)

# 三、配置结构体
<a id="ConnectionOptions"></a>
## 1.ConnectionOptions结构体
- **注**：内部成员变量顺序我做了调整，其它都未改动
```cpp
struct ConnectionOptions {
    // ============ 默认构造/拷贝/移动/析构 ============
    ConnectionOptions() = default;
    ConnectionOptions(const ConnectionOptions &) = default;
    ConnectionOptions& operator=(const ConnectionOptions &) = default;
    ConnectionOptions(ConnectionOptions &&) = default;
    ConnectionOptions& operator=(ConnectionOptions &&) = default;
    ~ConnectionOptions() = default;

    // ============ 连接目标 ============
    ConnectionType type = ConnectionType::TCP;  // 连接类型: TCP 或 UNIX
    std::string host;                           // Redis 服务器ip地址 (TCP模式)
    int port = 6379;                            // 端口号 (TCP模式)
    std::string path;                           // Unix Socket 路径 (UNIX模式)

    // ============ 认证相关 ============
    std::string user = "default";               // ACL 用户名 (Redis 6.0+ 支持)
    std::string password;                       // 认证密码

    // ============ 数据库选择 ============
    int db = 0;                                 // 数据库索引 (0-15)

    // ============ 超时设置 ============
    std::chrono::milliseconds connect_timeout{0};  // 连接超时 (0=系统默认)
    std::chrono::milliseconds socket_timeout{0};   // 读写超时 (0=无限等待)

    // ============ TCP 保活 ============
    bool keep_alive = false;                    // 是否启用 TCP KeepAlive（心跳机制）

#ifdef REDIS_PLUS_PLUS_HAS_redisEnableKeepAliveWithInterval
    std::chrono::seconds keep_alive_s{0};       // KeepAlive 探测（心跳）间隔 (需要 hiredis 支持)
#endif

    // ============ TLS/SSL ============
    tls::TlsOptions tls;                        // TLS 加密配置


    // ============ 协议版本 ============
    int resp = 2;                               // RESP 协议版本 (2 或 3)

    // ============ 连接名称 ============
    std::string name;                           // CLIENT SETNAME 设置的连接名

    // ============ 集群专用（与我们无关，不用看） ============
    // 是否允许从 Redis Cluster 的从节点（slave/replica）读取数据。
    // 客户端不应手动设置，RedisCluster 类会自动管理这个标志（未来可能移除）
    bool readonly = false;	

    // 内部使用，返回服务器信息字符串
    std::string _server_info() const;
};

```
**keep_alive 和 keep_alive_s 的关系**
- keep_alive = true 但 keep_alive_s = 0 时，使用系统默认的 KeepAlive 间隔（通常是 7200 秒）
- 若要自定义间隔，需要 hiredis 版本支持 redisEnableKeepAliveWithInterval


<a id="ConnectionPoolOptions"></a>
## 2.ConnectionPoolOptions结构体
```cpp
struct ConnectionPoolOptions {
    // 连接池最大连接数
    std::size_t size = 1;

    // 获取连接时的最大等待时间
    // 当连接池耗尽时，调用者最多阻塞等待这么久
    // 0 = 无限等待（不推荐，可能导致线程死锁）
    std::chrono::milliseconds wait_timeout{0};

    // 连接的最大存活时间
    // 超过此时间的连接会被销毁并重建
    // 用于防止长连接腐化、应对服务端重启等场景
    // 0 = 永不过期
    std::chrono::milliseconds connection_lifetime{0};
    
    // 连接的最大空闲时间
    // 超过此时间未被使用的连接会被回收
    // 用于释放闲置资源、避免空闲连接被防火墙/代理断开
    // 0 = 不回收空闲连接
    std::chrono::milliseconds connection_idle_time{0};
};
```

<a id="SentinelOptions"></a>
## 3.SentinelOptions结构体
- **注**：内部成员变量顺序我做了调整，其它都未改动
```cpp
struct SentinelOptions {
    /// 哨兵节点列表：每个元素为<哨兵节点IP/域名, 哨兵端口>
    /// 示例：{{"127.0.0.1", 26379}, {"127.0.0.1", 26380}}
    std::vector<std::pair<std::string, int>> nodes;

    /// 连接哨兵节点的超时时间，默认100毫秒
    /// 超过该时间未建立连接则视为连接失败
    std::chrono::milliseconds connect_timeout{100};

    /// 与哨兵节点通信的Socket超时时间，默认100毫秒
    /// 超过该时间未收到哨兵响应则视为通信失败
    std::chrono::milliseconds socket_timeout{100};

    /// 连接哨兵失败后的重试间隔，默认100毫秒
    std::chrono::milliseconds retry_interval{100};

    /// 连接哨兵的最大重试次数，默认2次
    /// 达到该次数仍失败则整体连接流程终止
    std::size_t max_retry = 2;

    /// 哨兵认证的用户名，默认值为"default"（Redis 6.0+ 新增的ACL用户认证）
    std::string user = "default";

    /// 哨兵认证的密码（如果哨兵节点配置了密码验证，需填写此项）
    std::string password;

    /// 是否启用TCP Keep-Alive机制，默认true
    /// 作用：检测无效的连接，避免长时间空闲的连接被断开
    bool keep_alive = true;

    /// TLS/SSL连接配置选项（如果哨兵节点启用了TLS加密，需配置此项）
    tls::TlsOptions tls;

    /// 指定与哨兵通信使用的Redis RESP协议版本，默认2（RESP2）
    /// 可选值：2（RESP2）、3（RESP3），需与哨兵节点的协议版本匹配
    int resp = 2;
};
```

# 四、Redis 类（单机/主从/哨兵模式核心客户端）
- 作为 Redis++ 最基础且核心的“**客户端类**”，封装了所有 Redis 原生命令的调用接口
- 支持**单机、主从、哨兵**三种部署模式的连接与操作，是生产环境中最常用的类

**注！！**：单机、主从、哨兵等部署模式都是**由“服务端”配置**，Redis类只负责连接到已存在的部署模式

## 1.Redis类介绍
### （1）Redis类的关键实现细节
```cpp
class Redis {
public:

	// 1. 基础构造：通过连接选项 + 连接池选项初始化（最常用）
	/// @param connection_opts 连接选项
    /// @param pool_opts 连接池选项
    explicit Redis(const ConnectionOptions &connection_opts,
           		   const ConnectionPoolOptions &pool_opts = {})
           	 	:_pool(std::make_shared<ConnectionPool>(pool_opts, connection_opts)) 
           	 	{}

    // 2. 简化构造：通过 URI 字符串直接初始化（快速配置场景）
    /// @brief 用 URI 字符串构造 Redis 实例（简化配置，无需手动组装 ConnectionOptions）
    /// @param uri URI 格式说明：
    /// 	- TCP 连接：tcp://[[username:]password@]host[:port][/db]
    ///     	- 示例："tcp://127.0.0.1"、"tcp://user:pass@127.0.0.1:6379/0"（连接 0 号库）
    ///     - Unix 套接字连接：unix://[[username:]password@]path-to-unix-domain-socket[/db]
    ///         - 示例："unix:///tmp/redis.sock"
    explicit Redis(const std::string &uri) : Redis(Uri(uri)) {}  // 内部转发给 Uri 解析后的构造函数

    // 3. 哨兵模式构造：通过 Redis Sentinel 自动发现节点（高可用场景）
    /// @brief 基于 Redis 哨兵模式构造 Redis 实例（自动获取主从节点信息，支持故障自动切换）
    /// @param sentinel Sentinel 实例的智能指针（用于与哨兵集群通信）
    /// @param master_name 主节点名称（哨兵集群监控的主节点标识）
    /// @param role 连接角色：
    /// 	- Role::MASTER: 连接主节点（用于写操作）
    ///     - Role::SLAVE: 连接从节点（用于读操作，实现读写分离）
    /// @param connection_opts 连接选项（同基础构造，可配置 KeepAlive 等）
    /// @param pool_opts 连接池选项（同基础构造，默认使用默认配置）
    Redis(const std::shared_ptr<Sentinel> &sentinel,
          const std::string &master_name,
          Role role,
          const ConnectionOptions &connection_opts,
          const ConnectionPoolOptions &pool_opts = {})
          : _pool(std::make_shared<ConnectionPool>(SimpleSentinel(sentinel, master_name, role),
                                                   pool_opts,
                                                   connection_opts)) 
          {}

    Redis(const Redis &) = delete;
    Redis& operator=(const Redis &) = delete;

    Redis(Redis &&) = default;
    Redis& operator=(Redis &&) = default;

private:
    std::shared_ptr<ConnectionPool> _pool;  // 核心成员：连接池智能指针（管理 Redis 连接）
};
```

### （2）数据类型 & 操作对照表
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113145948394.png)


### （3）key、管道、事务、发布订阅等功能
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113150625783.png)


## 2.Redis类 适配“单机模式”
### （1）介绍
- Redis类直接连接单个Redis节点，部署简单但无高可用保障
- **服务端部署！！**：
	- Linux上安装Redis后，直接启动即可

### （2）代码实现
[ConnectionOptions结构体](#ConnectionOptions)
[ConnectionPoolOptions结构体](#ConnectionPoolOptions)
```cpp
#include <sw/redis++/redis++.h>
#include <iostream>
#include <thread>
#include <vector>

using namespace sw::redis;

void worker(Redis &redis, int id) {
    try {
        for (int i = 0; i < 10; ++i) {
            std::string key = "key_" + std::to_string(id) + "_" + std::to_string(i);
            redis.set(key, "value_" + std::to_string(i));
            
            auto val = redis.get(key);
            if (val) {
                std::cout << "线程 " << id << ": " << key << " = " << *val << std::endl;
            }
        }
    } catch (const Error &e) {
        // ✅ 线程内部捕获异常
        std::cerr << "✗ 线程 " << id << " 错误: " << e.what() << std::endl;
    }
}

int main() {
    ConnectionOptions conn_opts;
    conn_opts.host = "127.0.0.1";
    conn_opts.port = 6379;
    conn_opts.socket_timeout = std::chrono::milliseconds(100);
    
    ConnectionPoolOptions pool_opts;
    pool_opts.size = 5;
    pool_opts.wait_timeout = std::chrono::milliseconds(100);
    pool_opts.connection_lifetime = std::chrono::minutes(10);
    
    // 创建 Redis 对象（可能失败） 
    std::unique_ptr<Redis> redis;
    try {
        redis = std::make_unique<Redis>(conn_opts, pool_opts);
        std::cout << "✓ Redis 连接成功" << std::endl;
    } catch (const Error &e) {
        std::cerr << "✗ Redis 连接失败: " << e.what() << std::endl;
        return 1;  // 连接失败，直接退出
    }
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(worker, std::ref(*redis), i);
    }
    
    for (auto &t : threads) {
        t.join();
    }
    
    std::cout << "✓ 所有线程完成" << std::endl;
    return 0;
}
```

## 3.Redis类 适配“主从复制模式”

### （1）介绍
- 适配读多写少的生产场景，包含一个主节点（写入）和多个从节点（读取）
- 分别创建Redis类，按“单机模式“连接方式分别连接“主节点、从节点”即可（通过**ip：port**）
- **服务端部署！！**：
	- 服务端需**手动“配置主从”**

### （2）代码实现
[ConnectionOptions结构体](#ConnectionOptions)
[ConnectionPoolOptions结构体](#ConnectionPoolOptions)
**假设主从复制模式配置如下**：
```plaintext
# 主节点
127.0.0.1：6370

# 从节点1
127.0.0.1：6380

# 从节点2
127.0.0.1:6381

```
```cpp
#include <sw/redis++/redis++.h>
#include <iostream>
#include <vector>
#include <string>
#include <random>

using namespace sw::redis;

int main() {
        // ==================== 配置参数 ====================
        
        // 主节点配置
        std::string master_host = "127.0.0.1";
        int master_port = 6379;
        
        // 从节点配置
        std::vector<std::pair<std::string, int>> slaves = {
            {"127.0.0.1", 6380},  // 从节点1
            {"127.0.0.1", 6381}   // 从节点2
        };
        
        std::string password = "";  // 如果有密码，填写在这里
        size_t pool_size = 3;       // 连接池大小
        
        
        // ==================== 1. 连接主节点 ====================
        
        std::cout << "=== 连接主节点 ===" << std::endl;
        
        ConnectionOptions master_opts;
        master_opts.host = master_host;
        master_opts.port = master_port;
        master_opts.password = password;
        master_opts.connect_timeout = std::chrono::milliseconds(100);
        master_opts.socket_timeout = std::chrono::milliseconds(100);
        
        ConnectionPoolOptions pool_opts;
        pool_opts.size = pool_size;
        pool_opts.wait_timeout = std::chrono::milliseconds(100);
        
        auto master = Redis(master_opts, pool_opts);
        std::cout << "✓ 主节点连接成功：" << master_host << ":" << master_port << std::endl;
        
        
        // ==================== 2. 连接从节点 ====================
        
        std::cout << "\n=== 连接从节点 ===" << std::endl;
        
        std::vector<Redis> slave_clients;
        
        for (const auto& [host, port] : slaves) {
            ConnectionOptions slave_opts;
            slave_opts.host = host;
            slave_opts.port = port;
            slave_opts.password = password;
            slave_opts.connect_timeout = std::chrono::milliseconds(100);
            slave_opts.socket_timeout = std::chrono::milliseconds(100);
            
            slave_clients.emplace_back(Redis(slave_opts, pool_opts));
            std::cout << "✓ 从节点连接成功：" << host << ":" << port << std::endl;
        }

    return 0;
}
```

## 4.Redis类 适配“哨兵模式”
### （1）介绍
- “哨兵模式”为Redis提供了高可用性，但相较于前两中模式，要复杂一点
- “哨兵模式”需要用到“**Sentinel类**”（哨兵类）
- **服务端部署！！**：
	- 服务端需手动**“配置主从”**（**哨兵模式的基础**）
	- 服务器序手动创建“**哨兵节点**”

### （2）代码实现
[ConnectionOptions结构体](#ConnectionOptions)
[ConnectionPoolOptions结构体](#ConnectionPoolOptions)
**推荐先看**：[Sentinel类](#Sentinel)
**假设哨兵模式配置如下**：
```plaintext
# 主节点
127.0.0.1：6370

# 从节点
127.0.0.1：6380
127.0.0.1：6381

# 哨兵节点
127.0.0.1：9000
127.0.0.1：9001
127.0.0.1：9002

```

```cpp
#include <sw/redis++/redis++.h>
#include <iostream>
#include <memory>

using namespace sw::redis;

int main() {

    // ============ 1. 哨兵配置 ============
    SentinelOptions sentinel_opts;
    sentinel_opts.nodes = {
	    {"127.0.0.1", 9000},
	    {"127.0.0.1", 9001},
	    {"127.0.0.1", 9002}
    };
    sentinel_opts.connect_timeout = std::chrono::milliseconds(100);
    sentinel_opts.socket_timeout = std::chrono::milliseconds(100);
    
    auto sentinel = std::make_shared<Sentinel>(sentinel_opts);
    
    
    // ============ 2. 连接配置 ============
    ConnectionOptions conn_opts;
    conn_opts.connect_timeout = std::chrono::milliseconds(100);
    conn_opts.socket_timeout = std::chrono::milliseconds(100);
    
    ConnectionPoolOptions pool_opts;
    pool_opts.size = 3;
    
    // ============ 3. 创建主节点 ============
    auto master = Redis(sentinel, 
        				"mymaster",  	// 这是在哨兵配置文件中定义的主节点名称
        				Role::MASTER,   // 角色
        				conn_opts, 
        				pool_opts);

    // ============ 4. 创建从节点 ============
    auto slave = Redis(sentinel, 
    					"mymaster", 	// 这是在哨兵配置文件中定义的主节点名称
    					Role::SLAVE, 	// 角色
                        conn_opts, 
                        pool_opts);
    
    std::cout << "✅ 从节点连接成功" << std::endl;
        

    return 0;
}
```
- **不需要手动指定“主节点”、“从节点”的ip:port**
	- 节点的分配都有“**Sentinel类**”（哨兵类）完成

### （3）为什么需要“哨兵模式”？
#### <1>核心问题
**❌ 主从复制模式的致命缺陷**：
```cpp
主从复制模式 = 手动管理 + 单点故障

客户端连接：
master_redis = Redis("127.0.0.1:6379")
slave_redis  = Redis("127.0.0.1:6380")

问题1：主节点宕机
→ master_redis 连接失效
→ 抛出异常：Connection refused
→ 写操作全部失败
→ 需要人工介入：
   1. 手动提升从节点（SLAVEOF NO ONE）
   2. 修改代码中的主节点地址
   3. 重启应用程序

问题2：从节点宕机
→ slave_redis 连接失效
→ 抛出异常：Connection refused
→ 读操作失败
→ 需要人工切换到其他从节点
```

**核心矛盾**：
```Markdown
客户端 `Redis` 对象 = 静态连接（固定 `IP:Port`）
服务端节点状态   = 动态变化（宕机/切换/新增）

→ 两者无法自动同步
→ 需要人工干预
→ 高可用性差
```

#### <2>哨兵模式的解决方案
**核心思想**:
```plaintext
引入中间层：Sentinel
作用：动态感知节点变化，自动更新客户端连接
```
**框架设计**：
```Markdown
传统模式（静态连接）：
应用程序 → Redis 对象 → 固定节点 (`127.0.0.1:6379`)
              ↓ 节点宕机
         连接失效，抛出异常

哨兵模式（动态连接）：
应用程序 → Sentinel → 哨兵集群 (监控节点状态)
              ↓                  ↓
         Redis 对象 ←─── 动态获取可用节点
              ↓
         当前主节点 (`127.0.0.1:6379`)
              ↓ 主节点宕机
         哨兵检测 → 选举新主节点 (`127.0.0.1:6380`)
              ↓
         Sentinel 自动通知 Redis 对象
              ↓
         Redis 对象自动重连新主节点
              ↓
         应用程序无感知，继续正常工作
```
#### <3>Redis++ 的实现细节（简要版）
1. **传统模式**
```cpp
// ❌ 静态连接
auto redis = Redis("127.0.0.1:6379");

// 主节点宕机
redis.set("key", "value");
// → 抛出异常：Connection refused
// → 无法自动恢复
```
**问题**：
- Redis 对象只知道创建时的固定地址
- 节点变化时，无法自动更新连接
- 需要手动修改代码并重启

2. **哨兵模式（Sentinel + Redis）**
```cpp
// ✅ 动态连接
SentinelOptions sentinel_opts;
sentinel_opts.nodes = {
    {"127.0.0.1", 26379},  // 哨兵1
    {"127.0.0.1", 26380},  // 哨兵2
    {"127.0.0.1", 26381}   // 哨兵3
};
auto sentinel = std::make_shared<Sentinel>(sentinel_opts);

// 连接配置
ConnectionOptions conn_opts;
conn_opts.connect_timeout = std::chrono::milliseconds(100);
conn_opts.socket_timeout = std::chrono::milliseconds(100);

ConnectionPoolOptions pool_opts;
pool_opts.size = 3;

// 通过 Sentinel 创建 Redis 对象
    auto master = Redis(sentinel, 
        "mymaster",  	// 这是在哨兵配置文件中定义的主节点名称
        Role::MASTER,   // 角色
        conn_opts, 
        pool_opts);

// 内部机制：
// 1. Redis 对象持有 Sentinel 的引用
// 2. 发现连接失效时，调用 Sentinel 查询新主节点
// 3. 自动重连新主节点
// 4. 重试失败的操作
```
**内部流程**（伪代码）：
```cpp
class Redis {
private:
    std::shared_ptr<Sentinel> sentinel_;  // 持有 Sentinel 引用
    std::string master_name_;
    Connection* conn_;
    
public:
    // 通过 Sentinel 创建
    Redis(std::shared_ptr<Sentinel> sentinel, 
          const std::string& master_name) 
        : sentinel_(sentinel), master_name_(master_name) {
        
        // 初始连接
        auto [host, port] = sentinel_->getMasterAddr(master_name_);
        conn_ = connect(host, port);
    }
    
    // 写操作
    void set(const std::string& key, const std::string& value) {
        try {
            conn_->send("SET", key, value);
            
        } catch (const ConnectionError& e) {
            // 连接失效，自动重连
            reconnect();
            // 重试操作
            conn_->send("SET", key, value);
        }
    }
    
    // 自动重连
    void reconnect() {
        // 从 Sentinel 查询新主节点
        auto [host, port] = sentinel_->getMasterAddr(master_name_);
        
        // 销毁旧连接
        delete conn_;
        
        // 重连新主节点
        conn_ = connect(host, port);
    }
};
```

#### <4>“主从复制模式” vs “哨兵模式”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113224050272.png)

## 5.注意事项！！
- Redis 类对象是**线程安全**的，生产环境可全局共享一个实例（基于连接池实现线程隔离），无需每个线程创建单独对象；
- 所有Redis命令调用可能会抛出sw::redis::Error异常，应当捕获并妥善处理（如重连、告警、日志记录等）；

- 哨兵模式下，只需在 conn_opts.host 填写任意一个哨兵节点地址，客户端会自动发现主从节点；

- 避免长时间持有 Redis 对象的引用，使用完毕后及时释放，让连接池回收连接。


<a id="Sentinel"></a>
# 五、Sentinel
Sentinel 类是 Redis++ 中专门用于与 **Redis Sentinel 哨兵集群** 通信的组件，功能：
- 服务发现 
	- 从哨兵集群获取当前主库/从库的地址
- 故障透明 
	- 故障切换时**自动获取新主库地址**
- 连接中介 
	- 作为 Redis 对象和实际 Redis 实例之间的桥梁

## 1.Sentinel类的职责
- **使用 Role::MASTER**：
	- redis-plus-plus **总是连接到当前的master**（即使发生故障切换）。
	- 每次 redis-plus-plus 需要“为master创建一个新的连接”，或者“连接断开并且需要重新连接到master”时
		- redis-plus-plus 会向 Redis Sentinel 询问master地址，并连接到当前的master。
	- **发生故障切换**
		- redis-plus-plus 可以自动获取新master的地址，并**刷新底层连接池中的所有连地址接（连接到新的master）**。
- **使用 Role::SLAVE**
	- redis-plus-plus **总是连接到slave**。
	-  一个主节点可能有多个slave，redis-plus-plus 会**随机选一个并连接到它**
		- **底层连接池中的所有连接都连接到同一个slave**
	- **如果连接断开，且该slave仍为活体**
		- redis-plus-plus 将重新连接该从属实例
	- **如果该salve宕机，或者被提升为master**
		- redis-plus-plus 会随机连接到另一个slave。
		- 如果没有活着的slave，它会抛出例外。

## 2.Sentinel类的关键实现细节
```cpp
class Sentinel {
public:
    explicit Sentinel(const SentinelOptions &sentinel_opts);

    Sentinel(const Sentinel &) = delete;
    Sentinel& operator=(const Sentinel &) = delete;

    Sentinel(Sentinel &&) = delete;
    Sentinel& operator=(Sentinel &&) = delete;

    ~Sentinel() = default;
}；
```

## 3.Sentinel 类的使用方式
[SentinelOptions结构体](#SentinelOptions)
### （1）创建 Sentinel 对象
```cpp
#include <sw/redis++/redis++.h>

using namespace sw::redis;
// 配置哨兵选项
SentinelOptions sentinel_opts;

// 【必需】哨兵节点列表（至少配置一个，建议三个以上保证高可用）
sentinel_opts.nodes = {
    {"127.0.0.1", 26379},
    {"127.0.0.1", 26380},
    {"127.0.0.1", 26381}
};

// 【可选】连接哨兵的超时时间，默认 100ms ——>注：不能为0
sentinel_opts.connect_timeout = std::chrono::milliseconds(200);

// 【可选】请求/响应超时时间，默认 100ms  ——>注：不能为0
sentinel_opts.socket_timeout = std::chrono::milliseconds(200);

// 创建 Sentinel 共享指针（必须用 shared_ptr）
auto sentinel = std::make_shared<Sentinel>(sentinel_opts);
```

### （2）通过 Sentinel 创建 Redis 连接
```cpp
// =============== 1.创建哨兵 ===============
// 配置哨兵选项
SentinelOptions sentinel_opts;

// 【必需】哨兵节点列表（至少配置一个，建议三个以上保证高可用）
sentinel_opts.nodes = {
    {"127.0.0.1", 26379},
    {"127.0.0.1", 26380},
    {"127.0.0.1", 26381}
};

// 【可选】连接哨兵的超时时间，默认 100ms ——>注：不能为0
sentinel_opts.connect_timeout = std::chrono::milliseconds(200);

// 【可选】请求/响应超时时间，默认 100ms  ——>注：不能为0
sentinel_opts.socket_timeout = std::chrono::milliseconds(200);

// 创建 Sentinel 共享指针（必须用 shared_ptr）
auto sentinel = std::make_shared<Sentinel>(sentinel_opts);


// =============== 2.创建Redis连接配置 ===============
// 连接配置
ConnectionOptions conn_opts;
conn_opts.host = "127.0.0.1";
conn_opts.port = 6379;
conn_opts.connect_timeout = std::chrono::milliseconds(100);   // 必须设置！不能为 0
conn_opts.socket_timeout = std::chrono::milliseconds(100);    // 必须设置！不能为 0

// 连接池配置
ConnectionPoolOptions pool_opts;
pool_opts.size = 5;  // 连接池大小：5个连接
pool_opts.wait_timeout = std::chrono::milliseconds(100);  // 等待连接超时
pool_opts.connection_lifetime = std::chrono::minutes(10); // 连接生命周期


// =============== 3.创建Redis主节点 ===============
// 创建连接主库的 Redis 对象
auto master = Redis(sentinel,
					"mymaster",	 	// Name of master node
					Role::MASTER, 	// 角色
					conn_opts, 
					pool_opts);

// =============== 3.创建Redis从节点 ===============
// 创建连接从库的 Redis 对象
auto slave = Redis(sentinel, 
				   "mymaster",		// Name of master node 
				   Role::SLAVE, 	// 角色
				   conn_opts, 
				   pool_opts);
```
- **不需要手动指定“主节点/从节点”的ip：port**，由Sentinel自动分配



# 六、RedisCluster（集群模式客户端）
- RedisCluster类是集群模式下“客户端”连接“服务端”的句柄，向服务端发送请求
	- redis-plus-plus 客户端库**不负责创建集群**，它只负责**连接到已存在的集群**。
- **接口与 Redis 类类似！！**

## 1.RedisCluster类 的构造
### （1）构造函数
#### <1>基本构造函数
```cpp
RedisCluster(const ConnectionOptions &connection_opts,
             const ConnectionPoolOptions &pool_opts = {},	//默认为每个主节点维护单一连接
             Role role = Role::MASTER,
             const ClusterOptions &cluster_opts = {})

```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113163903839.png)

#### <2>URL构造函数
```cpp
//如： "tcp://127.0.0.1" or "tcp://127.0.0.1:6379"
explicit RedisCluster(const std::string &uri);
```
**限制**：
- ❌ 无法指定密码
- ❌ 只能使用默认连接池配置（size=1）

### （2）配置选项
#### <1>ConnectionOptions（连接配置）
```cpp
ConnectionOptions connection_options;
connection_options.host = "127.0.0.1";  // 必需：任意一个主节点
connection_options.port = 7000;         // 可选：默认 6379
connection_options.password = "auth";   // 可选：集群密码（所有节点相同）
connection_options.connect_timeout = std::chrono::milliseconds(100);
connection_options.socket_timeout = std::chrono::milliseconds(100);
connection_options.keep_alive = true;

// ❌ 集群模式下被忽略
connection_options.db = 0;  // Redis Cluster 不支持多数据库
```
**注意事项**：
- ⚠️ 只能使用 TCP 连接，**不支持 Unix Domain Socket**

- ⚠️ 所有节点**必须使用相同的密码**

- ⚠️ db 参数被忽略（集群不支持多数据库）

**自动发现机制**：
```plaintext
只需配置一个主节点
        │
        ▼
RedisCluster 发送 CLUSTER SLOTS 命令
        │
        ▼
自动获取所有节点信息并建立连接
```

#### <2>ConnectionPoolOptions（连接池配置）
```cpp
ConnectionPoolOptions pool_options;
pool_options.size = 3;  // 每个节点的连接数
pool_options.wait_timeout = std::chrono::milliseconds(100);
pool_options.connection_lifetime = std::chrono::minutes(10);
pool_options.connection_idle_time = std::chrono::milliseconds(0);
```
**工作原理**：
```plaintext
假设集群有 3 个主节点，且pool_options.size = 3

节点1 (127.0.0.1:7000) → 连接池 [conn1, conn2, conn3]
节点2 (127.0.0.1:7001) → 连接池 [conn1, conn2, conn3]
节点3 (127.0.0.1:7002) → 连接池 [conn1, conn2, conn3]

总连接数 = 3 节点 × 3 连接 = 9 个连接
```

#### <3>ClusterOptions（集群配置）
```cpp
struct ClusterOptions {
	//刷新“槽位-节点”映射的时间
    std::chrono::milliseconds slot_map_refresh_interval = std::chrono::seconds(10);
};
```
**作用**：
- 🔄 定期刷新槽位-节点映射（默认 10 秒）

- 🔄 同时刷新节点列表（自动发现新节点/移除下线节点）

**刷新触发条件**：

1. 定时刷新：每 10 秒（可配置）

2. 错误触发：收到 MOVED/ASK 重定向时立即刷新



### （3）创建实例的方式
#### <1>使用 URI（最简单）
```cpp
#include <sw/redis++/redis++.h>

// 只指定一个节点，自动发现其他节点
auto redis = sw::redis::RedisCluster("tcp://127.0.0.1:7000");

redis.set("key", "value");
```
**优点**：代码简洁  
**缺点**：无法配置密码、连接池大小

#### <2>标准配置（推荐）
```cpp
using namespace sw::redis;

// 1. 连接配置（任意一个主节点）
ConnectionOptions conn_opts;
conn_opts.host = "127.0.0.1";
conn_opts.port = 7000;
conn_opts.password = "your_password";
conn_opts.connect_timeout = std::chrono::milliseconds(100);
conn_opts.socket_timeout = std::chrono::milliseconds(100);

// 2. 连接池配置（每个节点 3 个连接）
ConnectionPoolOptions pool_opts;
pool_opts.size = 3;
pool_opts.wait_timeout = std::chrono::milliseconds(100);

// 3. 创建 RedisCluster
auto redis = RedisCluster(conn_opts, pool_opts);

// 使用
redis.set("key", "value");
auto val = redis.get("key");

```

#### <3>完整配置
```cpp
ConnectionOptions conn_opts;
conn_opts.host = "127.0.0.1";
conn_opts.port = 7000;
conn_opts.password = "your_password";

ConnectionPoolOptions pool_opts;
pool_opts.size = 3;

ClusterOptions cluster_opts;
cluster_opts.slot_map_refresh_interval = std::chrono::seconds(10);

auto redis = RedisCluster(
    conn_opts, 
    pool_opts, 
    Role::MASTER,      // 访问主节点
    cluster_opts
);
```


## 2.RedisCluster的连接机制
### （1）Role::MASTER 模式（默认）
```cpp
auto redis = RedisCluster(conn_opts, pool_opts, Role::MASTER);
```
#### <1>连接拓扑（连接所有主节点）
```markdown
Redis 集群：
┌─────────┐     ┌─────────┐     ┌─────────┐
│ 主节点1 │     │ 主节点2 │     │ 主节点3 │
│  :7000  │     │  :7001  │     │  :7002  │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
   从节点         从节点          从节点
   :7003          :7004          :7005

客户端连接：
┌──────────┐
│ 客户端   │
└────┬─────┘
     ├─────直连─────→ 主节点1 (:7000) [连接池 3个连接]
     ├─────直连─────→ 主节点2 (:7001) [连接池 3个连接]
     └─────直连─────→ 主节点3 (:7002) [连接池 3个连接]
```
- **注意**：Role::MASTER 连接**所有**主节点！！

#### <2>为什么 Role::MASTER 要连接所有主节点？
- **redis集群是对redis的水平扩容**：
	- 即启动N个redis节点，将整个数据分布存储在这个**N个节点**中，每个节
点**存储总数据的1/N**

##### 原因1：数据分片存储
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114091913625.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114091947050.png)

##### 原因2：写操作必须到主节点
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114092102449.png)

```plaintext
单主节点设计：
请求 → 主节点1 → MOVED → 重定向 → 主节点2/3
           ↓ 每次都要重定向
      性能损失：额外的网络往返

多主节点设计：
请求 → 计算槽位 → 直接找到正确节点 → 立即返回
           ↓ 零重定向
      性能最优：一次网络往返
```

### （2）Role::SLAVE 模式（连接每个主节点的一个随机从节点）
```cpp
auto redis = RedisCluster(conn_opts, pool_opts, Role::SLAVE);
```

#### <1>连接拓扑
```markdown
Redis 集群：
┌─────────┐     ┌─────────┐     ┌─────────┐
│ 主节点1 │     │ 主节点2 │     │ 主节点3 │
│  :7000  │     │  :7001  │     │  :7002  │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
   从节点1a        从节点2a        从节点3a
   :7003          :7004          :7005
     │               │
   从节点1b        从节点2b
   :7006          :7007

客户端连接（Role::SLAVE 模式）：
┌──────────┐
│ 客户端   │
└────┬─────┘
     ├─────直连─────→ 从节点1a (:7003) [连接池 3个连接]
     │                ↑ 随机选择主节点1的一个从节点
     │
     ├─────直连─────→ 从节点2a (:7004) [连接池 3个连接]
     │                ↑ 随机选择主节点2的一个从节点
     │
     └─────直连─────→ 从节点3a (:7005) [连接池 3个连接]
                      ↑ 随机选择主节点3的一个从节点
```
**注意**：Role::SLAVE 连接每个主节点的一个随机从节点


#### <2>为什么 Role::SLAVE 要连接每个主节点的一个随机从节点？
##### 原因1:保持槽位覆盖
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114092349865.png)

##### 原因2：负载均衡
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114092417497.png)

##### 为什么不是所有从节点？
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114092451027.png)

### （3）“Role::MASTER 模式” vs “Role::SLAVE 模式”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114091249622.png)

### （4）常见误解
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114091358906.png)


## 3.RedisCluster类的注意事项
### （1）无法直接发送“无key参数的命令”
RedisCluster 对象确实可以发送大多数请求，但“key参数命令”无法直接发送
#### <1>问题
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114093548922.png)
#### <2>解决方案
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114093630371.png)
#### <3>“RedisCluster” vs “RedisCluster::redis()”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114093926722.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114093942338.png)

**注意**：
- RedisCluster::redis() 返回的 Redis 对象不是线程安全的
- 一旦抛出异常，必须销毁并重新创建
```cpp
// ⚠️ 重要：返回的 Redis 对象不是线程安全的！
auto r = redis_cluster.redis("hash-tag");

// ⚠️ 如果抛出异常，必须销毁并重新创建
try {
    r.command("client", "setname", "name");
} catch (const Error& e) {
    // r 不再可用，必须重建
    r = redis_cluster.redis("hash-tag");
}
```

# 七、Redis++ 命令的发送
## 1.命令调用方式
Redis++ 为每个 Redis 命令提供同名（小写）方法，支持多种重载：
```cpp
// DEL 命令的 3 种重载
long long Redis::del(const StringView &key);                    // 单个 key
template <typename Input>
long long Redis::del(Input first, Input last);                  // 迭代器范围
template <typename T>
long long Redis::del(std::initializer_list<T> il);              // 初始化列表
```

## 2.参数类型详解
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114102612093.png)

**StringView 详解**:
	- C++17 下 StringView 就是 **std::string_view** 的别名
```cpp
// StringView 兼容多种字符串类型
redis.set("key", "value");                          // C 风格字符串

std::string key = "key", val = "value";
redis.set(key, val);                                // std::string

std::vector<char> data = {...};
redis.set("key", StringView(data.data(), data.size()));  // 避免拷贝大数据

```

## 3.返回值详解
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114102827965.png)

## 4.bool 返回值
- 某些方法的返回类型为bool，例如 EXPIRE、HSET
-  **千万不要用返回值来检查命令是否成功发送到 Redis 服务器！！**：
	- **如果方法返回 false**：
		- 并不意味着 Redis 未能将命令发送到服务器。相反，这意味着 Redis 服务器返回一个整数回复 ，且回复值为 0。
	- **如果方法返回为真**：
		- 意味着 Redis 服务器返回的是整数回复 ，且回复值为 1
- 如果 Redis 未能向服务器发送命令，就**会抛出 Error 类型的异常**
```cpp
// ⚠️ 重要：bool 返回值不代表命令是否发送成功！
// true  = Redis 返回整数 1
// false = Redis 返回整数 0

if (redis.expire("key", seconds(100))) {
    // 返回 1：超时设置成功
} else {
    // 返回 0：key 不存在（不是发送失败！）
}

if (redis.setnx("key", "val")) {
    // 返回 1：key 不存在，设置成功
} else {
    // 返回 0：key 已存在，设置失败
}

// 💡 命令发送失败会抛异常，不会返回 false
```

## 5.Optional 处理 NULL 回复
```cpp
// GET 可能返回 NULL（key 不存在）
OptionalString val = redis.get("key");

if (val) {
    std::cout << *val << std::endl;       // 解引用获取值
} else {
    std::cout << "key 不存在" << std::endl; // NULL 回复
}

// MGET 返回多个 Optional
std::vector<OptionalString> values;
redis.mget({"k1", "k2", "k3"}, std::back_inserter(values));

for (const auto& v : values) {
    if (v) {
        std::cout << *v << std::endl;
    }
}

// 常用 Optional 类型别名
using OptionalString = Optional<std::string>;
using OptionalLongLong = Optional<long long>;
using OptionalDouble = Optional<double>;
using OptionalStringPair = Optional<std::pair<std::string, std::string>>;
```

## 6. Variant 处理多类型回复（C++17）
- 如果**回复类型不同**，**std::variant** 是个不错的回复类型选择
```cpp
// MEMORY STATS 返回混合类型
// 值可能是 long long、double 或嵌套 map
using Var = Variant<double, long long, std::unordered_map<std::string, long long>>;

auto result = redis.command<std::unordered_map<std::string, Var>>("memory", "stats");

// 访问结果
for (const auto& [key, value] : result) {
    if (std::holds_alternative<long long>(value)) {
        std::cout << key << ": " << std::get<long long>(value) << std::endl;
    } else if (std::holds_alternative<double>(value)) {
        std::cout << key << ": " << std::get<double>(value) << std::endl;
    }
}
```
**⚠️ Variant 的限制**：
- **Variant 的类型参数不能有重复的项**（Redis++ 库限制）
	- 例如 Variant<double, long long, double> 无法工作。
- **double 必须放在 std::string 之前**：
	- 因为 double 回复实际是字符串格式
	- 在解析变体时，我们尝试将回复解析为第一个匹配的类型，由左到右的类型参数指定
	- 如果 double 放在 std::string 之后，回复总是会解析成 std::string

## 7.输出迭代器与 STL 容器
- 使用**command通用命令接口**时，你也可以把它解析成 STL 容器，而不是解析输出的回复。
```cpp
// 使用输出迭代器
std::vector<std::string> members;
redis.lrange("list", 0, -1, std::back_inserter(members));

std::unordered_map<std::string, std::string> hash;
redis.hgetall("hash", std::inserter(hash, hash.end()));

// 直接返回 STL 容器（通用命令接口）
auto config = redis.command<std::unordered_map<std::string, std::string>>("config", "get", "*");
```

## 8.各种参数类型
```cpp
// ***** StringView 类型的参数 *****

// 隐式通过 C 风格字符串构造 StringView
redis.set("key", "value");

// 隐式通过 std::string 构造 StringView
std::string key("key");
std::string val("value");
redis.set(key, val);

// 显式传递 StringView 作为参数
std::vector<char> large_data;
// 避免数据拷贝
redis.set("key", StringView(large_data.data(), large_data.size()));

// ***** long long 类型的参数 *****

// 用于索引参数
redis.bitcount(key, 1, 3);

// 用于数值参数
redis.incrby("num", 100);

// ***** double 类型的参数 *****

// 用于分数（score）参数
redis.zadd("zset", "m1", 2.5);
redis.zadd("zset", "m2", 3.5);
redis.zadd("zset", "m3", 5);

// 用于（经度，纬度）坐标参数
redis.geoadd("geo", std::make_tuple("member", 13.5, 15.6));

// ***** 时间相关参数 *****

using namespace std::chrono;

redis.expire(key, seconds(1000)); // 设置过期时间为 1000 秒

auto tp = time_point_cast<seconds>(system_clock::now() + seconds(100));
redis.expireat(key, tp); // 设置指定时间点过期

// ***** 命令的一些可选参数 *****

if (redis.set(key, "value", milliseconds(100), UpdateType::NOT_EXIST)) {
    std::cout << "set 操作成功" << std::endl;
}

redis.linsert("list", InsertPosition::BEFORE, "pivot", "val"); // 在指定元素前插入值

std::vector<std::string> res;

// (-无穷, +无穷) 区间
redis.zrangebyscore("zset", UnboundedInterval<double>{}, std::back_inserter(res));

// [3, 6] 闭区间
redis.zrangebyscore("zset",
    BoundedInterval<double>(3, 6, BoundType::CLOSED),
    std::back_inserter(res));

// (3, 6] 左开右闭区间
redis.zrangebyscore("zset",
    BoundedInterval<double>(3, 6, BoundType::LEFT_OPEN),
    std::back_inserter(res));

// (3, 6) 开区间
redis.zrangebyscore("zset",
    BoundedInterval<double>(3, 6, BoundType::OPEN),
    std::back_inserter(res));

// [3, 6) 左闭右开区间
redis.zrangebyscore("zset",
    BoundedInterval<double>(3, 6, BoundType::RIGHT_OPEN),
    std::back_inserter(res));

// [3, +无穷) 左闭右无界区间
redis.zrangebyscore("zset",
    LeftBoundedInterval<double>(3, BoundType::RIGHT_OPEN),
    std::back_inserter(res));

// (3, +无穷) 左开右无界区间
redis.zrangebyscore("zset",
    LeftBoundedInterval<double>(3, BoundType::OPEN),
    std::back_inserter(res));

// (-无穷, 6] 左无界右闭区间
redis.zrangebyscore("zset",
    RightBoundedInterval<double>(6, BoundType::LEFT_OPEN),
    std::back_inserter(res));

// (-无穷, 6) 左无界右开区间
redis.zrangebyscore("zset",
    RightBoundedInterval<double>(6, BoundType::OPEN),
    std::back_inserter(res));

// ***** 迭代器对参数 *****

std::vector<std::pair<std::string, std::string>> kvs = {{"k1", "v1"}, {"k2", "v2"}, {"k3", "v3"}};
redis.mset(kvs.begin(), kvs.end()); // 批量设置键值对

std::unordered_map<std::string, std::string> kv_map = {{"k1", "v1"}, {"k2", "v2"}, {"k3", "v3"}};
redis.mset(kv_map.begin(), kv_map.end());

std::unordered_map<std::string, std::string> str_map = {{"f1", "v1"}, {"f2", "v2"}, {"f3", "v3"}};
redis.hmset("hash", str_map.begin(), str_map.end()); // 批量设置哈希字段

std::unordered_map<std::string, double> score_map = {{"m1", 20}, {"m2", 12.5}, {"m3", 3.14}};
redis.zadd("zset", score_map.begin(), score_map.end()); // 批量添加有序集合元素

std::vector<std::string> keys = {"k1", "k2", "k3"};
redis.del(keys.begin(), keys.end()); // 批量删除键

// ***** initializer_list 类型的参数 *****

redis.mset({
    std::make_pair("k1", "v1"),
    std::make_pair("k2", "v2"),
    std::make_pair("k3", "v3")
}); // 用初始化列表批量设置键值对

redis.hmset("hash",
    {
        std::make_pair("f1", "v1"),
        std::make_pair("f2", "v2"),
        std::make_pair("f3", "v3")
    }); // 用初始化列表批量设置哈希字段

redis.zadd("zset",
    {
        std::make_pair("m1", 20.0),
        std::make_pair("m2", 34.5),
        std::make_pair("m3", 23.4)
    }); // 用初始化列表批量添加有序集合元素

redis.del({"k1", "k2", "k3"}); // 用初始化列表批量删除键
```

## 9.各种返回值类型
```cpp
// ***** 返回值为 void 类型 *****

redis.save();

// ***** 返回值为 std::string 类型 *****

auto info = redis.info();

// ***** 返回值为 bool 类型 *****

if (!redis.expire("nonexistent", std::chrono::seconds(100))) {
    std::cerr << "键不存在" << std::endl;
}

if (redis.setnx("key", "val")) {
    std::cout << "set 操作成功" << std::endl;
}

// ***** 返回值为 long long 类型 *****

auto len = redis.strlen("key");
auto num = redis.del({"a", "b", "c"});
num = redis.incr("a");

// ***** 返回值为 double 类型 *****

auto real = redis.incrbyfloat("b", 23.4);
real = redis.hincrbyfloat("c", "f", 34.5);

// ***** 返回值为 Optional<std::string>（即 OptionalString）类型 *****

auto os = redis.get("kk");
if (os) {
    std::cout << *os << std::endl;
} else {
    std::cerr << "键不存在" << std::endl;
}

os = redis.spop("set");
if (os) {
    std::cout << *os << std::endl;
} else {
    std::cerr << "集合为空" << std::endl;
}

// ***** 返回值为 Optional<long long>（即 OptionalLongLong）类型 *****

auto oll = redis.zrank("zset", "mem");
if (oll) {
    std::cout << "排名为 " << *oll << std::endl;
} else {
    std::cerr << "成员不存在" << std::endl;
}

// ***** 返回值为 Optional<double>（即 OptionalDouble）类型 *****

auto ob = redis.zscore("zset", "m1");
if (ob) {
    std::cout << "分数为 " << *ob << std::endl;
} else {
    std::cerr << "成员不存在" << std::endl;
}

// ***** 返回值为 Optional<pair<string, string>> 类型 *****

auto op = redis.blpop({"list1", "list2"}, std::chrono::seconds(2));
if (op) {
    std::cout << "键为 " << op->first << "，值为 " << op->second << std::endl;
} else {
    std::cerr << "超时" << std::endl;
}

// ***** 输出迭代器接收返回值 *****

std::vector<OptionalString> os_vec;
redis.mget({"k1", "k2", "k3"}, std::back_inserter(os_vec));

std::vector<std::string> s_vec;
redis.lrange("list", 0, -1, std::back_inserter(s_vec));

std::unordered_map<std::string, std::string> hash;
redis.hgetall("hash", std::inserter(hash, hash.end()));
// 也可以将结果保存到字符串键值对的向量中
std::vector<std::pair<std::string, std::string>> hash_vec;
redis.hgetall("hash", std::back_inserter(hash_vec));

std::unordered_set<std::string> str_set;
redis.smembers("s1", std::inserter(str_set, str_set.end()));
// 也可以将结果保存到字符串向量中
s_vec.clear();
redis.smembers("s1", std::back_inserter(s_vec));
```

# 八、Redis++ 的异常处理
## 1.Redis++的Exception
```plaintext
std::exception
    └── sw::redis::Error              // 基类：所有 Redis++ 异常的父类
            ├── IoError               // IO 错误（连接相关）
            │       └── TimeoutError  // 超时错误（读写超时）
            ├── ClosedError           // 服务器关闭连接
            ├── ProtoError            // 协议错误（命令/回复无效）
            ├── OomError              // hiredis 内存不足
            ├── ReplyError            // Redis 返回错误回复
            └── WatchError            // WATCH 的 key 被修改
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114095237003.png)
**注意！！**：
- **NULL REPLY 不是异常**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114095332179.png)

## 2.异常后的对象状态
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114095437891.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114095455051.png)

## 3.异常处理模板
```cpp
#include <sw/redis++/redis++.h>
using namespace sw::redis;

void safe_redis_operation(Redis& redis) {
    try {
        redis.set("key", "value");
        
    } catch (const TimeoutError& e) {
        // 超时：可重试
        std::cerr << "超时: " << e.what() << std::endl;
        
    } catch (const ReplyError& e) {
        // Redis 返回错误（如类型错误）：检查业务逻辑
        std::cerr << "Redis 错误: " << e.what() << std::endl;
        
    } catch (const IoError& e) {
        // IO 错误：连接问题，自动重连
        std::cerr << "IO 错误: " << e.what() << std::endl;
        
    } catch (const Error& e) {
        // 其他 Redis++ 错误
        std::cerr << "错误: " << e.what() << std::endl;
    }
    
    // redis 对象仍然可用，下次调用会自动重连
}
```

# 九、Redis::command（Redis++ 通用命令接口）
Redis++ 无法为所有 Redis 命令提供内置方法，Redis::command 允许你发送**任意 Redis 命令**
## 1.基本用法
### （1）直接指定返回类型（推荐）
```cpp
auto redis = Redis("tcp://127.0.0.1");

// 无返回值的命令
redis.command<void>("client", "setname", "my-connection");

// 返回字符串（可能为空）
auto val = redis.command<OptionalString>("client", "getname");
if (val) {
    std::cout << *val << std::endl;
}

// 返回整数
auto num = redis.command<long long>("incrby", "counter", 10);

// 返回浮点数
auto real = redis.command<double>("incrbyfloat", "price", 2.5);

// 返回数组 - 方式1：使用输出迭代器
std::vector<OptionalString> result;
redis.command("mget", "k1", "k2", "k3", std::back_inserter(result));

// 返回数组 - 方式2：直接解析为容器
auto result2 = redis.command<std::vector<OptionalString>>("mget", "k1", "k2", "k3");
```

### （2）参数类型灵活
```cpp
auto redis = Redis("tcp://127.0.0.1");

// Redis 类没有内置的 *CLIENT SETNAME* 方法。
// 不过，你可以使用 Redis::command 手动发送该命令。
redis.command<void>("client", "setname", "name");
auto val = redis.command<OptionalString>("client", "getname");
if (val) {
    std::cout << *val << std::endl;
}

// 注意：以下代码仅作示例使用。实际上，Redis 已为
// 下述命令提供了内置方法。

// 命令参数可以是字符串类型。
// 注意：对于 SET 命令，返回值并非始终为 void 类型，后续会详细说明。
redis.command<void>("set", "key", "100");

// 命令参数可以是字符串和整数的组合。
auto num = redis.command<long long>("incrby", "key", 1);

// 参数也可以是双精度浮点型（double）。
auto real = redis.command<double>("incrbyfloat", "key", 2.3);

// 甚至命令中的键（key）也可以是算术类型。
redis.command<void>("set", 100, "value");

val = redis.command<OptionalString>("get", 100);

// 如果命令返回元素数组。
std::vector<OptionalString> result;
redis.command("mget", "k1", "k2", "k3", std::back_inserter(result));

// 也可以直接解析为 vector 容器。
result = redis.command<std::vector<OptionalString>>("mget", "k1", "k2", "k3");

// 命令参数可以是字符串范围（迭代器）。
auto set_cmd_strs = {"set", "key", "value"};
redis.command<void>(set_cmd_strs.begin(), set_cmd_strs.end());

auto get_cmd_strs = {"get", "key"};
val = redis.command<OptionalString>(get_cmd_strs.begin(), get_cmd_strs.end());

// 如果返回的是元素数组。
result.clear();
auto mget_cmd_strs = {"mget", "key1", "key2"};
redis.command(mget_cmd_strs.begin(), mget_cmd_strs.end(), std::back_inserter(result));
```
## 2.重要注意事项
### （1）多词命令必须分开传递
```cpp
// ✅ 正确：分开传递
redis.command<void>("client", "setname", "name");
redis.command<void>("cluster", "info");
redis.command<void>("config", "get", "maxmemory");

// ❌ 错误：合并为一个字符串
redis.command<void>("client setname", "name");  // 失败！
```

### （2）返回类型不固定时的处理
```cpp
auto redis = Redis("tcp://127.0.0.1");

// 方式1：使用 OptionalString 处理可能的 NULL
auto r = redis.command<OptionalString>("set", "key", "val", "NX");
if (r) {
    // 返回 "OK"，设置成功
    std::cout << "设置成功: " << *r << std::endl;
} else {
    // 返回 NULL，key 已存在，设置失败
    std::cout << "key 已存在，设置失败" << std::endl;
}

// 方式2：使用 ReplyUPtr 手动解析
auto reply = redis.command("set", "key", "val", "NX");
if (reply->type == REDIS_REPLY_NIL) {
    // NULL 回复，设置失败
    std::cout << "key 已存在" << std::endl;
} else if (reply->type == REDIS_REPLY_STATUS) {
    // 状态回复 "OK"，设置成功
    std::cout << "设置成功: " << reply->str << std::endl;
}

// 方式3：封装成 bool（模拟内置 set() 的行为）
bool set_if_not_exist(Redis& redis, const std::string& key, const std::string& val) {
    auto r = redis.command<OptionalString>("set", key, val, "NX");
    return r.has_value();  // 有值 = 成功，空 = 失败
}

// 使用封装函数
if (set_if_not_exist(redis, "mykey", "myval")) {
    std::cout << "设置成功" << std::endl;
} else {
    std::cout << "key 已存在" << std::endl;
}

```
**常见陷阱**：
```cpp
// ⚠️ SET 命令的返回类型不是固定的！
// 普通 SET → 返回 "OK"（Status Reply）
// SET ... NX → 可能返回 NULL（key 已存在时）

// ❌ 可能出错
redis.command<void>("set", "key", "val", "NX");  // NX 失败时无法判断

// ✅ 正确处理
auto result = redis.command<OptionalString>("set", "key", "val", "NX");
if (result) {
    // 设置成功
} else {
    // key 已存在，设置失败
}
```


# 十、Redis++ Pub/Sub（发布/订阅）
## 1.概述
- 你可以使用 **Redis::publish方法** 向频道（channels）发布消息
	- Redis 会从底层连接池中**随机选择**一个连接，并通过该连接发布消息
	- 因此，你发布的两条消息可能会使用两个不同的连接
- 当你通过一个连接**订阅某个频道**时，发布到**该频道的所有消息**都会通过这个连接回传给客户端
	- Redis 类中并**没有Redis::subscribe方法**：
		- 你可以调用 **Redis::subscriber方法** 创建一个 Subscriber（订阅者）对象，该对象会单独维护一个与 Redis 服务器的连接
		- 这个底层连接是 **新建的连接**，**“并非”从连接池中选取**，且该新连接会 **“沿用”原 Redis 对象的 ConnectionOptions（连接配置）**。
```cpp
发布者                          Redis 服务器                      订阅者
┌─────────┐                    ┌─────────────┐                 ┌─────────────┐
│  Redis  │  ──publish()──►   │             │  ──推送消息──►  │ Subscriber  │
│ 对象    │                    │   Channel   │                 │ (独立连接)  │
│(连接池) │                    │             │                 │             │
└─────────┘                    └─────────────┘                 └─────────────┘
   │                                                                  │
   │ 随机选择连接发布                                          新建独立连接
   └──────────────────────────────────────────────────────────────────┘
```

📊 速查表
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114120517753.png)

## 2.Publish 发布消息
```cpp
auto redis = Redis("tcp://127.0.0.1:6379");

// 向频道发布消息（从连接池随机选择连接）
redis.publish("channel1", "hello");
redis.publish("news", "breaking news!");

// 返回值：收到消息的订阅者数量
long long receivers = redis.publish("channel1", "message");
std::cout << "收到消息的订阅者: " << receivers << std::endl;
```

## 3.Subscriber 订阅
### （1）创建Subscriber对象
```cpp
auto redis = Redis("tcp://127.0.0.1:6379");

// 创建订阅者（新建独立连接，不使用连接池）
auto sub = redis.subscriber();
```
**注意！！：**
- 如果你想有**不同的连接选项**，比如不同频道的 ConnectionOptions::socket_timeout ，你应该创建带有不同连接选项的 Redis 对象，然后你可以用这些 Redis 对象创建订阅对象
```cpp
// 频道1：100ms 超时
ConnectionOptions opts1;
opts1.host = "127.0.0.1";
opts1.port = 6379;
opts1.socket_timeout = std::chrono::milliseconds(100);
auto redis1 = Redis(opts1);
auto sub1 = redis1.subscriber();  // 继承 100ms 超时

// 频道2：300ms 超时
ConnectionOptions opts2;
opts2.host = "127.0.0.1";
opts2.port = 6379;
opts2.socket_timeout = std::chrono::milliseconds(300);
auto redis2 = Redis(opts2);
auto sub2 = redis2.subscriber();  // 继承 300ms 超时

// 注意⚠️：虽然创建了两个 Redis 对象，但没有性能损失
// 因为 Redis 对象是懒加载的，只有调用 subscriber() 时才创建连接
```

### （2）Subscriber的使用
### <1>订阅频道和模式
```cpp
// 订阅单个频道
sub.subscribe("channel1");

// 订阅多个频道
sub.subscribe({"channel2", "channel3", "channel4"});

// 订阅模式（支持通配符）
sub.psubscribe("news:*");      // 匹配 news:sports, news:tech 等
sub.psubscribe("user:*:msg");  // 匹配 user:123:msg, user:456:msg 等

// 取消订阅
sub.unsubscribe("channel1");
sub.unsubscribe({"channel2", "channel3"});

// 取消订阅模式
sub.punsubscribe("news:*");

// 取消所有订阅（无参数）
sub.unsubscribe();   // 取消所有频道订阅
sub.punsubscribe();  // 取消所有模式订阅
```

#### <2> Subscriber类 接收的6种消息类型
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114113148877.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114113623553.png)
- Subscriber 类是所有订阅消息（6 种类型）的**唯一接收主体**
- Subscriber类 通过消费方法（如 consume()）主动获取消息，且接收过程为**阻塞式**
	- 调用 consume() 后会**一直等待，直到有消息到达或超时**
	- 这也是为什么 Subscriber 对象需要独占一个连接（**不从连接池取**）、且**不建议多线程**共用的原因。

#### <3>设置回调函数（处理6种消息）
**为处理Subscriber类接收的6种消息，可以在 Subscriber 上设置“回调函数”**
- **Subscriber::on_message(MsgCallback)**：
	- **功能**：
		- 为 MESSAGE 类型消息设置回调函数
	- **回调接口定义**：
		- void (std::string channel, std::string msg)  
  	- 参数说明：
  			- channel：消息所属频道
  			- msg：实际消息内容

- **Subscriber::on_pmessage(PatternMsgCallback)**：
	- **功能**
		- 为 PMESSAGE 类型消息设置回调函数
	- **回调接口定义**：
		- void (std::string pattern, std::string channel, std::string msg) 
  	- 参数说明：
  			- pattern： 匹配的频道模式
  			- channel：消息实际所属频道
  			- msg：实际消息内容）

- **Subscriber::on_meta(MetaCallback)**：
	- **功能**：
		- 为元消息（META MESSAGE）类型设置回调函数
	- **回调接口定义**：
		- void (Subscriber::MsgType type, OptionalString channel, long long num)  
  	- 参数说明：
  			- type：消息类型枚举
  				- Subscriber::MsgType::SUBSCRIBE
  				- Subscriber::MsgType::UNSUBSCRIBE
  				- Subscriber::MsgType::PSUBSCRIBE
  				- Subscriber::MsgType::PUNSUBSCRIBE
  				- Subscriber::MsgType::MESSAGE（不会触发）
  				- Subscriber::MsgType::PMESSAGE（不会触发）
  			- channel：频道/模式名称（OptionalString 类型，即可选字符串）
  				- 若未订阅任何频道/模式，且调用无参数的 unsubscribe/punsubscribe（即取消所有订阅）时，该参数为 null；
  			- num：当前仍处于订阅状态的频道/模式总数。
```cpp
auto sub = redis.subscriber();

// 1️⃣ 普通消息回调（MESSAGE 类型）
sub.on_message([](std::string channel, std::string msg) {
    std::cout << "频道 [" << channel << "]: " << msg << std::endl;
    
    // ✅ 可以安全地 move 这些参数
    process(std::move(channel), std::move(msg));
});

// 2️⃣ 模式消息回调（PMESSAGE 类型）
sub.on_pmessage([](std::string pattern, std::string channel, std::string msg) {
    std::cout << "模式 [" << pattern << "] 频道 [" << channel << "]: " << msg << std::endl;
});

// 3️⃣ 元消息回调（SUBSCRIBE/UNSUBSCRIBE 等）
sub.on_meta([](Subscriber::MsgType type, OptionalString channel, long long num) {
    switch (type) {
        case Subscriber::MsgType::SUBSCRIBE:
            std::cout << "已订阅: " << *channel << ", 当前订阅数: " << num << std::endl;
            break;
        case Subscriber::MsgType::UNSUBSCRIBE:
            if (channel) {
                std::cout << "已取消订阅: " << *channel << std::endl;
            } else {
                std::cout << "已取消所有订阅" << std::endl;  // channel 可能为空
            }
            break;
        case Subscriber::MsgType::PSUBSCRIBE:
            std::cout << "已订阅模式: " << *channel << std::endl;
            break;
        case Subscriber::MsgType::PUNSUBSCRIBE:
            std::cout << "已取消订阅模式" << std::endl;
            break;
        default:
            break;
    }
});
```

#### <4>consume 消费消息

##### 1）consume函数执行特点
- 可以调用 **Subscriber::consume方法**，**消费**发布到该订阅者已订阅的频道 / 模式的消息
- Subscriber::consume 会**阻塞等待**底层连接传来消息：
	- 若达到 ConnectionOptions::socket_timeout 设定的超时时间，且无任何消息传入该连接
		- Subscriber::consume 会**抛出 TimeoutError 异常**；
	- 若 ConnectionOptions::socket_timeout 设为 0 毫秒
		- Subscriber::consume 会**一直阻塞，直到接收到消息为止**。
- 接收到消息后，Subscriber::consume 会**根据消息类型调用“对应的回调函数”处理消息**
	- 如果你未为某类消息设置回调函数，Subscriber::consume 会消费（读取）该消息并**直接丢弃**
	- 即 Subscriber::consume **不会执行回调，直接返回**
- 调用 consume() 后会**一直等待，直到有消息到达或超时**：
	- 这也是为什么 Subscriber 对象需要独占一个连接（不从连接池选取）、且不建议多线程共用的原因。
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114171934666.png)

##### 2）不带超时的模式（设为0毫秒）
```cpp
auto sub = redis.subscriber();

sub.on_message([](std::string channel, std::string msg) {
    std::cout << channel << ": " << msg << std::endl;
});

sub.subscribe("channel1");

// 消费消息（阻塞等待）
while (true) {
    try {
        sub.consume();  // 阻塞直到收到消息
    } catch (const Error& err) {
        std::cerr << "错误: " << err.what() << std::endl;
        break;
    }
}
```
##### 3）带超时的模式
```cpp
// 设置超时
ConnectionOptions opts;
opts.host = "127.0.0.1";
opts.port = 6379;
opts.socket_timeout = std::chrono::milliseconds(1000);  // 1秒超时

auto redis = Redis(opts);
auto sub = redis.subscriber();

sub.on_message([](std::string channel, std::string msg) {
    std::cout << channel << ": " << msg << std::endl;
});

sub.subscribe("channel1");

while (true) {
    try {
        sub.consume();
    } catch (const TimeoutError& e) {
        // 超时不是错误，继续等待
        std::cout << "等待消息中..." << std::endl;
        continue;
    } catch (const ReplyError& e) {
        // Redis 返回错误，可以继续使用 sub
        std::cerr << "Redis 错误: " << e.what() << std::endl;
        continue;
    } catch (const Error& e) {
        // ⚠️ 其他错误：必须销毁并重建 Subscriber
        std::cerr << "严重错误: " << e.what() << std::endl;
        break;
    }
}
```

##### 4）消息流向完整图示
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114143816994.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114143840833.png)

#### <5>完整示例
```cpp
#include <sw/redis++/redis++.h>
#include <iostream>

using namespace sw::redis;

int main() {
    try {
        ConnectionOptions opts;
        opts.host = "127.0.0.1";
        opts.port = 6379;
        opts.socket_timeout = std::chrono::milliseconds(1000);

        auto redis = Redis(opts);
        auto sub = redis.subscriber();

        // 设置所有回调
        sub.on_message([](std::string channel, std::string msg) {
            std::cout << "[MESSAGE] " << channel << ": " << msg << std::endl;
        });

        sub.on_pmessage([](std::string pattern, std::string channel, std::string msg) {
            std::cout << "[PMESSAGE] " << pattern << " -> " << channel << ": " << msg << std::endl;
        });

        sub.on_meta([](Subscriber::MsgType type, OptionalString channel, long long num) {
            std::cout << "[META] 类型=" << static_cast<int>(type) 
                      << ", 频道=" << (channel ? *channel : "null")
                      << ", 数量=" << num << std::endl;
        });

        // 订阅
        sub.subscribe("chat");
        sub.psubscribe("news:*");

        // 消费循环
        std::cout << "开始监听消息..." << std::endl;
        while (true) {
            try {
                sub.consume();
            } catch (const TimeoutError&) {
                continue;
            }
        }

    } catch (const Error& e) {
        std::cerr << "错误: " << e.what() << std::endl;
        return 1;
    }

    return 0;
}
```

### （3）Subscriber 使用注意事项
#### <1>非线程安全
- **Subscriber**（订阅者）对象是 **非线程安全**的。如果你希望在多线程环境中调用其成员函数，需要**手动实现线程间的同步**

#### <2>特定Exception后不可复用
- 如果 **Subscriber（订阅者）对象**的任意方法抛出了**“非”** ReplyError（回复错误）或 TimeoutError（超时错误）类型的异常，那么该对象**将无法再继续使用**。此时你**必须销毁**这个 Subscriber 对象，并**重新创建**一个新的实例。
```cpp
try {
    sub.consume();
} catch (const TimeoutError& e) {
    // ✅ 可以继续使用 sub
} catch (const ReplyError& e) {
    // ✅ 可以继续使用 sub
} catch (const Error& e) {
    // ❌ 必须销毁并重建 sub
    sub = redis.subscriber();
    sub.on_message(...);  // 重新设置回调
    sub.subscribe("channel1");  // 重新订阅
}
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114143913218.png)

#### <3>consume消费是“阻塞的”
- 调用 consume() 后会**一直等待，直到有消息到达或超时**
- 这也是为什么 Subscriber 对象需要独占一个连接（**不从连接池取**）、且**不建议多线程**共用的原因。

# 十一、Redis++ Pipeline（管道）
## 1.概述
- Pipeline 用于减少 RTT（往返时间），**加速 Redis 查询**。
- 通过将多个命令**批量发送**，一次性获取所有回复，避免每个命令都等待网络往返。

## 2.创建Pipeline
- 可以通过 `Redis::pipeline` 方法创建一个pipeline，它会返回一个**pipeline对象**
- 通过 `Redis::pipeline` 方法会创建的pipeline，是一个**新的Redis服务连接**，而不是直接使用连接池中的连接
	- 该连接与连接池中其它连接拥有相同的`ConnectionOptions`
- **最好尽可能复用同一个“pipeline对象”！！**
	- 因为创建一个管道对象并不便宜，因为它会创建一个新的连接

```cpp
ConnectionOptions connection_options;
ConnectionPoolOptions pool_options;

Redis redis(connection_options, pool_options);

auto pipe = redis.pipeline();
```

**补充：**
- 创建pipeline是，可以直接从连接池获取连接而不创建新连接
- [用连接池中连接创建pipeline](#pipeline_from_pool) 
	- ⚠️ 此方式有死锁风险

## 3.通过pipeline发送命令
- 通过 Pipeline 对象发送 Redis 命令 和 Redis 类一样。
- **Pipeline 的设计**:
	- 先批量添加命令，再一次性执行并获取结果
- **链式调用**：
	- **功能**：
		- **将命令批量添加到pipeline中**（未发送！）
	- 通过“Pipeline对象”调用方法，不会直接返回响应，而是**返回 Pipeline 对象本身**，这样你就可以**链式调用**这些方法	 
- **`Pipeline::exec`方法**：
	- **功能**：
		- **将pipeline中所有已缓存的命令发送到Redis，并返回批量执行的结果**
- **`Pipeline::discard`方法**：
	- **功能**：
		- **丢弃pipeline中所有缓存的命令**
```cpp
ConnectionOptions connection_options;
ConnectionPoolOptions pool_options;

Redis redis(connection_options, pool_options);

auto pipe = redis.pipeline();

// ================= 方式1：添加与执行链式调用 ===================
auto replies = pipe.set("key", "val")
    .incr("counter")
    .rpush("list", {1, 2, 3})
    .hset("hash", "field", "value")
    .command("client", "setname", "app")
    .exec();

// ================= 方式2： 添加与执行分开调用 ===================
// 链式调用：添加多个命令（还未执行）
pipe.set("key", "val")
    .incr("counter")
    .rpush("list", {1, 2, 3})
    .hset("hash", "field", "value")
    .command("client", "setname", "app");  // 通用命令接口

// 调用 exec() 发送命令并获取回复
auto replies = pipe.exec();

// ================= 丢弃管道中的命令 ===================
pipe.set("key", "val").incr("num");

pipe.discard();
```
**exec() 执行流程**
- 发送所有已缓存的命令到 Redis
- 接收所有命令的回复
- 返回 `QueuedReplies`对象

<a id="parse_pipeline_result"></a>

## 4.解析pipeline执行结果
- Pipeline::exec 返回一个 `QueuedReplylies`对象，该对象包含所有发送到 Redis 命令的回复
- 通过 `QueuedReplies::get` 方法来获取并解析回复
- `ueuedReplies` 提供 3 种 get() 方法：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114151808620.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114152209995.png)
```cpp
ConnectionOptions connection_options;
ConnectionPoolOptions pool_options;

Redis redis(connection_options, pool_options);

auto pipe = redis.pipeline();

auto replies = pipe.set("key", "val")      // 0: SET → bool
                   .incr("counter")        // 1: INCR → long long
                   .lrange("list", 0, -1)  // 2: LRANGE → vector
                   .get("key")             // 3: GET → OptionalString
                   .exec();

// 解析各个回复
bool set_ok = replies.get<bool>(0);
long long counter = replies.get<long long>(1);

std::vector<std::string> list_items;
replies.get(2, std::back_inserter(list_items));

auto val = replies.get<OptionalString>(3);
if (val) {
    std::cout << "key = " << *val << std::endl;
}
```

<a id="return_mechanism"></a>
## 5.连接归还机制
- 当pipeline执行exec、discard、对象析构、抛出异常时，会归还连接（主要针对“连接池”模式）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114170235009.png)

### （1）两种创建方式的区别
```cpp
// 方式1：默认创建（新建连接）
auto pipe = redis.pipeline();        // 创建新连接，不从连接池取

// 方式2：使用连接池
auto pipe = redis.pipeline(false);   // 从连接池借用连接
```
### （2）"归还连接"只对连接池模式有意义
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114170102888.png)

### （3）"归还"的具体含义
```cpp
ConnectionPoolOptions pool_opts;
pool_opts.size = 3;  // 连接池有 3 个连接

Redis redis(opts, pool_opts);

// 使用连接池模式创建 Pipeline
auto pipe = redis.pipeline(false);

// 此时：Pipeline 从连接池"借走"了一个连接
// 连接池状态：[连接1][连接2][ 空 ]  ← 只剩 2 个可用

pipe.set("key", "val").incr("num");

// 执行 exec()
auto replies = pipe.exec();

// exec() 完成后：连接"归还"到连接池
// 连接池状态：[连接1][连接2][连接3]  ← 恢复 3 个可用
```

### （4） 为什么"归还"很重要？
```cpp
// 连接池只有 3 个连接
Redis redis(opts, pool_opts);  // pool_opts.size = 3

auto pipe1 = redis.pipeline(false);  // 借走连接 #1
auto pipe2 = redis.pipeline(false);  // 借走连接 #2
auto pipe3 = redis.pipeline(false);  // 借走连接 #3

// 连接池已空！

pipe1.set("k1", "v1");
pipe2.set("k2", "v2");
pipe3.set("k3", "v3");

// ❌ 这里会阻塞或超时！因为连接池没有可用连接
redis.get("key");  // 等待连接...

// 必须先执行 exec() 归还连接
pipe1.exec();  // 归还连接 #1

// ✅ 现在可以了
redis.get("key");  // 获取到连接 #1
```

### （5）归还之后再次触发呢？
- 归还连接后，Pipeline对象仍然存在，可以继续使用！
- 调用 exec() 或 discard() 后：
	- 连接被归还到连接池
	- 下次调用命令时，会重新从连接池借用连接
```cpp
Redis redis(opts, pool_opts);

auto pipe = redis.pipeline(false);  // 从连接池借用连接 #1

// 第一批命令
pipe.set("k1", "v1").set("k2", "v2");
auto replies1 = pipe.exec();  // 执行后，连接 #1 归还到池

// ✅ Pipeline 对象仍可使用！
// 第二批命令（会重新从连接池借用连接，可能是 #1，也可能是 #2、#3）
pipe.set("k3", "v3").incr("counter");
auto replies2 = pipe.exec();  // 执行后，连接再次归还

// ✅ 可以一直这样复用
pipe.get("k1").get("k2");
auto replies3 = pipe.exec();
```

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114170533657.png)



## 6.pipeline 使用注意事项
### （1）特定Exception后不可复用
- 如果pipeline的任何方法抛出除 ReplyError 以外的异常， 管道对象将进入无效状态
	- 不能再使用它了，只能摧毁该物体，然后创建一个新的
```cpp
try {
    auto replies = pipe.set("k", "v").incr("n").exec();
} catch (const ReplyError& e) {
    // ✅ ReplyError：Pipeline 仍可使用
    std::cerr << "Redis 返回错误: " << e.what() << std::endl;
} catch (const Error& e) {
    // ❌ 其他异常：Pipeline 进入无效状态，必须销毁重建
    std::cerr << "严重错误: " << e.what() << std::endl;
    pipe = redis.pipeline();  // 重新创建
}
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114153043892.png)

### （2）非线程安全
- Pipeline **不是线程安全**的
- 如果你想在多线程环境中调用其成员函数，就需要在线程间手动同步

<a id="pipeline_from_pool"></a>
## 5.创建管道而不创建新连接
### （1）用连接池中的连接创建pipeline对象（有死锁风险）
- 可以创建一个带有底层连接池连接的管道对象，这样调用 Redis::pipeline 方法可以便宜得多（因为它不需要新建连接）
- **Pipeline pipeline(bool new_connection = true)**
	-  如果 new_connection 为假， 管道对象将与底层池的连接创建。
```cpp
ConnectionOptions connection_options;
ConnectionPoolOptions pool_options;

Redis redis(connection_options, pool_options);

// 传入 false，从连接池获取连接
//⚠️在这种情况下，你必须非常小心，否则可能会表现不佳甚至死锁
auto pipe = redis.pipeline(false);
```

<a id="dead_lock"></a>
### （2）死锁风险
- 当你用 Pipeline 对象执行命令时，它会保留连接，直到调用 Pipeline::exec、Pipeline::discard 或 Pipeline 的 destructor
	- 如果 Pipeline 的任何方法抛出异常 ，连接也会被释放
- 如果管道对象长时间保持连接，其他 Redis 方法可能无法从底层池获取连接。

**死锁示例（❌ 错误用法）**：
```cpp
// 默认只有 1 个连接，wait_timeout = 0（永久等待）
Redis redis("tcp://127.0.0.1");

auto pipe = redis.pipeline(false);  // 从连接池取连接

pipe.set("key1", "val");  // 占用连接
pipe.set("key2", "val");  // 继续占用

// ❌ 死锁！连接池为空，Redis::get 永久阻塞
redis.get("key");  // 等待连接池，但 pipe 还没释放

pipe.exec();  // 永远执行不到
```

### （3）最佳实践
- 始终将 ConnectionPoolOptions::wait_timeout 设置为大于 0 毫秒
	- 即当连接池为空时，避免调用者永久阻塞
- 最好将 Pipeline 相关代码放在代码块作用域内
- 避免在 Pipeline 方法调用之间执行耗时操作
- 尽量将 Pipeline 方法与 Pipeline::exec 链式调用在同一条语句中


✅ 正确用法
```cpp
ConnectionOptions opts;
opts.host = "127.0.0.1";
opts.port = 6379;
opts.socket_timeout = std::chrono::milliseconds(50);

ConnectionPoolOptions pool_opts;
pool_opts.size = 3;
// ✅ 最佳实践1： ConnectionPoolOptions::wait_timeout > 0ms
pool_opts.wait_timeout = std::chrono::milliseconds(50);

auto redis = Redis(opts, pool_opts);

// ✅ 最佳实践2：使用块作用域
{
    auto pipe = redis.pipeline(false);
    
    //✅ 最佳实践3：避免在 Pipeline 方法调用之间执行耗时操作
    // ❌ 不要在这里做耗时操作！
    
    auto replies = pipe.set("k1", "v1").set("k2", "v2").exec();
    
    // exec() 完成后，连接自动归还
}

// ✅ 最佳实践4：链式调用一气呵成
{
    auto pipe = redis.pipeline(false);
    auto replies = pipe.set("k1", "v1")
                       .set("k2", "v2")
                       .incr("counter")
                       .exec();  // 立即发送并释放连接
}

for (auto i = 0; i < 10; ++i) {
    // 这种操作（复用连接池连接创建 Pipeline）开销极低
    auto pipe = redis.pipeline(false);

    // 从底层连接池获取连接并持有
    pipe.set("key1", "val").set("key2", "val");

    // 即使未调用 Pipeline::exec 或 Pipeline::discard（即pipeline还缓存有命令）
    // 当 Pipeline 对象析构时，连接也会被自动归还到连接池
}
```

# 十二、Redis++ Transaction（事务）

## 1.概述
Transaction 用于确保多个命令**原子性执行**——要么全部成功，要么全部不执行。
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114155752421.png)
- 前面提过，Redis是线程安全的，为减小开销，尽量复用同一个Redis ——> 这也就可能导致一个线程通过Redis执行多个命令时，其它线程也通过同一个Redis执行命令，从而导致被打断

## 2.创建Transaction
### （1）默认方式（创建新连接）
```cpp
Redis redis("tcp://127.0.0.1:6379");

// 创建事务（默认创建新连接，不从连接池取）
auto tx = redis.transaction();
```
⚠️ 注意：默认创建 Transaction 开销较大（需要新建连接），应尽量复用。

### （2）使用连接池连接创建（有死锁风险）
```cpp
// 参数说明：transaction(piped, new_connection)
// piped = false（非管道模式），new_connection = false（从连接池取）
auto tx = redis.transaction(false, false);
```

### （3）创建管道化事务（性能更好）
[管带事务](#pipeline_transaction)

```cpp
// piped = true，命令通过 Pipeline 发送，减少 RTT
auto tx = redis.transaction(true);

// 或使用连接池 + 管道 () （有死锁风险）
auto tx = redis.transaction(true, false);
```

## 3.Transaction的使用
### （1）发送命令
Transaction 与 Pipeline 接口完全相同，支持链式调用：
```cpp
tx.set("key", "val")
  .incr("counter")
  .lpush("list", {1, 2, 3})
  .hset("hash", "field", "value")
  .command("client", "setname", "app");
```
📌 你不需要向 Redis 发送多重命令。 交易会自动帮你完成这件事。

### （2）执行事务
- 当你调用 Transaction::exec 时，你会明确要求 Redis 执行那些排队命令，并返回回复
	- 否则，这些命令不会被执行
- 你可以调用 Transaction::discard 来丢弃执行，也就是说不会执行任何命令。

**调用 exec() 执行**
```cpp
// 方式1：先发送，再执行
tx.set("key", "val").incr("num");
auto replies = tx.exec();

// 方式2：链式调用（推荐）
auto replies = tx.set("key", "val").incr("num").exec();
```

**放弃事务**
```cpp
tx.set("key", "val").incr("num");
tx.discard();  // 放弃执行，所有命令都不会执行

```

### （3）解析事务执行结果
与 Pipeline 完全相同：[解析回复](#parse_pipeline_result)
```cpp
auto replies = tx.set("key", "val")      // 0: SET → bool
                 .incr("counter")        // 1: INCR → long long
                 .lrange("list", 0, -1)  // 2: LRANGE → vector
                 .exec();

// 解析
bool set_ok = replies.get<bool>(0);
long long counter = replies.get<long long>(1);

std::vector<std::string> items;
replies.get(2, std::back_inserter(items));
```

<a id="pipeline_transaction"></a>
### （4）管道事务（Piped Transaction）
通常，一个事务会发送多条指令，为了提升性能，可以在pipeline中发送这些命令
```cpp
// Create a piped transaction
auto tx = redis.transaction(true);
```
- 通过这种piped transaction，所有命令都会通过pipeline发送到 Redis。

## 4. WATCH 乐观锁（CAS 机制）
### （1）为什么需要 WATCH？
**Transacation 并不安全于线程**
```markdown
场景：两个客户端同时对 counter 加 1

客户端A（事务1）              客户端B （事务2）
────────                    ────────
GET counter → 10            GET counter → 10
new_counter = 10 + 1        new_counter = 10 + 1
SET counter new_counter     SET counter new_counter

结果：counter = 11（错误！应该是 12）
```
### （2）WATCH 的作用
```markdown
客户端A（事务1）              客户端B（事务2）
────────                    ────────
WATCH counter  //监视 counter 是否被修改
GET counter → 10
                            SET counter 20（修改了 counter）
MULTI	//开启事务
SET counter 
EXEC → 失败！（因为 counter 被修改）

→ 客户端A 重试整个流程
```

### （3）Watch的使用
#### <1>概述
- **WATCH 命令必须与该交易在同一连接中发送**
- 通常在 WATCH 命令之后，我们还需要发送其他命令
	- 如：**执行事务前从 Redis 获取数据**
```markdown
WATCH key           // 监听指定的 key
val = GET key       // 获取该 key 对应的 value
new_val = val + 1   // 对 value 执行自增操作
MULTI               // 开启事务
SET key new_val     // 仅当该 key 的值未被其他客户端修改时，才执行此赋值操作
EXEC                // 尝试执行事务
                    // 若监听期间 val 已被修改，整个事务不会执行
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114163237592.png)

#### <2>使用 WATCH 的关键
- 使用 Transaction 对象时，你无法获得命令的结果，直到整个事务完成。
- 相反，你需要从事务对象创建一个 Redis 对象。创建的 Redis 对象与事务对象**共享连接**
- 有了这个创建的 Redis 对象，你可以向 Redis 服务器发送 WATCH 命令和其他 Redis 命令，并立即获得结果。
```cpp
auto tx = redis.transaction();

// 创建共享连接的 Redis 对象
auto r = tx.redis();

// r 和 tx 使用同一个连接
r.watch("key");           // 在共享连接上执行 WATCH
auto val = r.get("key");  // 可以立即获取结果
tx.set("key", new_val).exec();  // 在同一连接上执行事务
```

#### <3>WATCH 完整示例（CAS 实现）
```cpp
auto redis = Redis("tcp://127.0.0.1");

// 在循环外创建 Transaction（避免重复创建连接）
auto tx = redis.transaction();

// 被 WATCH 的 key 可能被其他客户端修改，需要重试
while (true) {
    try {
        // 创建共享连接的 Redis 对象
        auto r = tx.redis();
        
        // 监视 key
        r.watch("key");
        
        // 获取当前值
        auto val = r.get("key");
        int num = val ? std::stoi(*val) : 0;
        
        // 计算新值
        ++num;
        
        // 执行事务
        auto replies = tx.set("key", std::to_string(num)).exec();
        
        // 成功！验证并退出循环
        assert(replies.size() == 1 && replies.get<bool>(0) == true);
        break;
        
    } catch (const WatchError& e) {
        // key 被其他客户端修改，重试
        continue;
    } catch (const Error& e) {
        // 其他错误，Transaction 已失效
        throw;
    }
}
```

## 5.创建事务但不创建新连接
### （1）使用连接池内连接创建Transaction
- 事实上，可以创建一个带有底层连接池连接的事务对象，这样调用 Redis::transaction 方法会便宜得多（因为不需要新建连接）。
- Redis::transaction 的原型:
	- `Transaction transaction(bool piped = false, bool new_connection = true);`
		- 如果 new_connection 为假， 交易对象将通过底层池的连接创建
```cpp
ConnectionOptions connection_options;
ConnectionPoolOptions pool_options;

Redis redis(connection_options, pool_options);

// 创建一个事务（Transaction），但不新建连接。
auto tx = redis.transaction(false, false);
```
- 不过，在这种情况下，你必须非常小心
	- 否则：**可能会表现不佳甚至死锁!!**

**参考**：[pipeline的死锁风险](#dead_lock)

### （2）使用连接池连接的最佳实践
- 限制 由Transaction::Redis创建的Redis对象的 作用域（生命周期）
	- 即尽快销毁Redis对象。

**⚠️ 特别注意**：tx.redis() 创建的对象会持有连接
```cpp
auto redis = Redis(opts, pool_opts);

// 创建一个 Transaction 对象，且不新建连接（复用连接池中的现有连接）
auto tx = redis.transaction(false, false);
// 注：第一个 false = 不使用 Pipeline 模式执行事务；第二个 false = 不新建连接

// 创建一个 Redis 对象，共享同一个连接
auto r = tx.redis();

// 其他业务代码（⚠️ 若此处有耗时操作，会长期占用连接，导致连接池耗尽）

// Execute the transaction.
// 执行事务（提交事务队列中的所有命令）
auto replies = tx.set("key", "val").exec();

// 尽管已调用 Transaction::exec 执行事务，但连接并未归还到连接池
// 原因：由 tx.redis() 创建的 Redis 对象（即变量 r）仍持有该连接
```

**✅ 正确用法**：确保 r 和 tx 都尽快销毁
```cpp
ConnectionOptions opts;
opts.host = "127.0.0.1";
opts.port = 6379;

ConnectionPoolOptions pool_opts;
pool_opts.size = 3;
pool_opts.wait_timeout = std::chrono::milliseconds(100);  // ⚠️ 必须 > 0

auto redis = Redis(opts, pool_opts);

while (true) {
    try {
        // ✅ 在循环内创建 Transaction（使用连接池，开销很小）
        auto tx = redis.transaction(false, false);
        
        // ✅ r 和 tx 在同一作用域，同时销毁
        auto r = tx.redis();
        
        r.watch("key");
        auto val = r.get("key");
        int num = val ? std::stoi(*val) : 0;
        ++num;
        
        auto replies = tx.set("key", std::to_string(num)).exec();
        
        assert(replies.size() == 1 && replies.get<bool>(0));
        break;
        
        // ✅ 离开作用域时，r 和 tx 都销毁，连接归还
        
    } catch (const WatchError&) {
        continue;
    } catch (const Error&) {
        throw;
    }
}
```

## 6.连接归还机制
**参考**：[pipeline的连接归还机制](#return_mechanism)

## 7.Transaction 使用注意事项
### （1）特定Exception后不可复用
- 如果事务的任何方法抛出除 WatchError 或 ReplyError 以外的异常， 事务对象将进入无效状态
	- 你不能再使用它，只能摧毁该物体并创建一个新的
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260114161718422.png)

### （2）非线程安全
- Transacation 并不安全于线程。
- 如果你想在多线程环境中调用其成员函数，就需要在线程间手动同步。




<a id="use"></a>
# 附录1：Redis++ 核心场景
## 1.缓存场景（最核心场景）
### （1）适用场景
用户信息缓存、商品详情缓存、接口结果缓存、热点数据加速访问等，核心目标是减轻数据库压力，提升接口响应速度。

### （2）核心优势
- Redis 基于内存操作，响应速度快（毫秒级）；
- Redis++ 提供丰富的缓存操作 API，支持过期时间设置、批量操作，配合**连接池**可高效支撑高并发缓存访问。

### （3）生产实现要点（重点！）
- **缓存key命名规范**：
	- 采用“ **业务模块:数据类型:唯一标识** ”格式
		- 如： goods:detail:10086，避免键名冲突
- **过期策略**：
	- 必须设置合理的过期时间
		- 如：热点商品30分钟、用户token24小时
		- 避免内存泄漏：Redis 内存持续增长且无法被有效回收，最终导致内存耗尽
		- 可通过 **set(key, val, expire)** 实现
- **缓存更新**：
	- 采用“更新数据库+更新缓存”或“先删缓存+后更数据库”策略，避免缓存与数据库数据不一致；
	- 高并发场景推荐“先删缓存+延迟双删”；
- **缓存异常防护**：
	- **缓存穿透**（缓存和数据库均无数据）
		- 可**设置空值缓存**（短期过期）
	- **缓存击穿**（热点key过期瞬间高并发）
		- 可使用**互斥锁**或**热点key永不过期**
	- **缓存雪崩**（大量key同时过期）
		- 可设置**过期时间随机偏移**（如基础过期30分钟+随机0-5分钟）。

```cpp
#include <sw/redis++/redis++.h>
#include <random>
#include <sstream>

// 生成 UUID
std::string generate_uuid() {
    static std::random_device rd;
    static std::mt19937 gen(rd());
    static std::uniform_int_distribution<> dis(0, 15);
    static const char* hex = "0123456789abcdef";
    
    std::string uuid;
    for (int i = 0; i < 32; ++i) {
        uuid += hex[dis(gen)];
    }
    return uuid;
}

std::optional<std::string> get_goods_detail_cache_safe(
    Redis& redis, 
    const std::string& goods_id,
    int max_retries = 5  // ✅ 添加重试次数限制
) {
    const std::string cache_key = "goods:detail:" + goods_id;
    const std::string lock_key = "lock:goods:" + goods_id;
    
    // 1. 查缓存
    if (auto val = redis.get(cache_key)) {
        return (*val == "__EMPTY__") ? std::nullopt : val;
    }
   
    // 2. 尝试获取锁
    std::string lock_value = generate_uuid();  // ✅ 唯一标识
    
    bool got_lock = redis.set(
        lock_key, 
        lock_value,  // ✅ 用 UUID 防止误删
        std::chrono::seconds(10), 
        sw::redis::UpdateType::NOT_EXIST
    );
    
    if (got_lock) {
        try {
            // 双重检查（可能其他线程已写入）
            if (auto val = redis.get(cache_key)) {
                redis.del(lock_key);  // ✅ 释放锁
                return (*val == "__EMPTY__") ? std::nullopt : val;
            }
            
            // 查数据库
            std::string db_val = query_goods_detail_from_db(goods_id);
            
            if (db_val.empty()) {
                redis.set(cache_key, "__EMPTY__", std::chrono::minutes(5));
            } else {
                redis.set(cache_key, db_val, std::chrono::minutes(30));
            }
            
            // ✅ 安全释放锁（Lua 脚本，原子操作）
            const char* lua_script = R"(
                if redis.call("get", KEYS[1]) == ARGV[1] then
                    return redis.call("del", KEYS[1])
                else
                    return 0
                end
            )";
            redis.eval<long long>(lua_script, {lock_key}, {lock_value});
            
            return db_val.empty() ? std::nullopt : std::optional{db_val};
            
        } catch (...) {
            // ✅ 异常时也要释放锁
            redis.eval<long long>(
                R"(if redis.call("get", KEYS[1]) == ARGV[1] then 
                     return redis.call("del", KEYS[1]) 
                   else 
                     return 0 
                   end)",
                {lock_key}, {lock_value}
            );
            throw;
        }
    } else {
        // ✅ 改用循环代替递归
        for (int i = 0; i < max_retries; ++i) {
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            
            // 再次查缓存（可能已被其他线程写入）
            if (auto val = redis.get(cache_key)) {
                return (*val == "__EMPTY__") ? std::nullopt : val;
            }
        }
        
        // ✅ 超过重试次数，直接查数据库（降级）
        std::string db_val = query_goods_detail_from_db(goods_id);
        return db_val.empty() ? std::nullopt : std::optional{db_val};
    }
}

```

## 2.分布式锁场景
### （1）适用场景
秒杀活动、分布式事务、跨服务资源竞争、避免重复提交等，核心目标是保证分布式环境下资源访问的原子性。

### （2）核心优势
- Redis 单线程特性保证锁操作的原子性；
- Redis++ 支持 Lua 脚本，可将“加锁+过期”、“解锁+验证持有者”等逻辑封装为原子操作，避免锁漏洞。

### （3）生产实现要点
- **锁标识唯一性**：
	- 每个锁持有者需携带唯一标识（如 UUID+线程ID），避免误解锁其他线程的锁；
- **锁过期机制**：
	- 必须设置锁过期时间，**防止持有者崩溃导致锁永久占用**
		- 可通过 **Lua 脚本**原子设置“NX+EX”；

- **锁续约**：
	- 如果业务逻辑执行时间可能超过锁过期时间，需启动后台线程定期续约（如每10秒续期20秒）；
- **解锁原子性**：
	- 通过 Lua 脚本先验证锁持有者，再删除锁，避免锁过期后误删其他线程的锁。
```cpp
#include <atomic>
#include <thread>
#include <chrono>

class DistributedLock {
private:
    Redis& redis_;
    string lock_key_;      // 锁的 key，如 "lock:order:12345"
    string lock_val_;      // 锁的唯一标识，用于验证持有者身份
    int expire_seconds_;   // 锁过期时间（防止持有者崩溃导致死锁）
    
    atomic<bool> holding_{false};  // 原子标志：当前是否持有锁（线程安全）
    thread renew_thread_;          // 后台续约线程

    // ═══════════════════════════════════════════════════════════
    // 续约循环：防止业务执行时间超过锁过期时间导致锁被抢
    // ═══════════════════════════════════════════════════════════
    void renew_loop() {
        // Lua 脚本：原子操作 - 先验证是自己的锁，再续期
        // 避免续期了别人的锁（锁已过期被其他进程抢走的情况）
        const string renew_script = R"(
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('expire', KEYS[1], ARGV[2])
            end
            return 0
        )";
        
        // 续约频率：过期时间的 1/3（如30秒过期，每10秒续约一次）
        // 留足安全边际，避免网络延迟导致锁意外过期
        int sleep_ms = (expire_seconds_ * 1000) / 3;
        
        while (holding_.load()) {
            this_thread::sleep_for(chrono::milliseconds(sleep_ms));
            
            // 双重检查：sleep 期间可能已经 unlock 了
            if (!holding_.load()) break;
            
            try {
                auto result = redis_.eval<long long>(
                    renew_script,
                    {lock_key_},
                    {lock_val_, to_string(expire_seconds_)}
                );
                
                // result == 0 说明锁已不属于自己（被抢或过期）
                if (result == 0) {
                    holding_.store(false);
                    break;
                }
            } catch (const sw::redis::Error& e) {
                // 网络异常 → 保守处理：认为锁已丢失
                // 防止脑裂：宁可放弃锁，也不能两个进程同时认为自己持有锁
                holding_.store(false);
                break;
            }
        }
    }

public:
    DistributedLock(Redis& redis, const string& key, int expire_sec = 30)
        : redis_(redis), lock_key_(key), expire_seconds_(expire_sec) {
        // ═══════════════════════════════════════════════════════
        // 锁标识必须全局唯一：UUID + 线程ID
        // 作用：解锁时验证身份，防止 A 误删 B 的锁
        // ═══════════════════════════════════════════════════════
        lock_val_ = generate_uuid() + ":" + to_string(this_thread::get_id());
        // 注意：generate_uuid() 需要自行实现，可使用 boost::uuid 或其他 UUID 库
    }
    
    // RAII：析构时自动释放锁
    ~DistributedLock() {
        unlock();
    }

    bool try_lock() {
        // ═══════════════════════════════════════════════════════
        // SET key value EX seconds NX（原子操作）
        // NX = 仅当 key 不存在时才设置（互斥保证）
        // EX = 设置过期时间（防死锁保证）
        // ═══════════════════════════════════════════════════════
        bool acquired = redis_.set(
            lock_key_, 
            lock_val_,
            chrono::seconds(expire_seconds_),
            sw::redis::UpdateType::NOT_EXIST  // NX 语义
        );
        
        if (acquired) {
            holding_.store(true);
            // 启动后台续约线程，保证长任务不丢锁
            renew_thread_ = thread(&DistributedLock::renew_loop, this);
        }
        
        return acquired;
    }

    void unlock() {
        // ═══════════════════════════════════════════════════════
        // exchange(false) 原子地：读取旧值 + 设置新值
        // 返回 false 说明之前就是 false，已经释放过了（幂等）
        // ═══════════════════════════════════════════════════════
        if (!holding_.exchange(false)) {
            return;
        }
        
        // 必须等续约线程退出，否则可能：
        // 1. 续约线程还在跑，访问已析构的对象 → 崩溃
        // 2. 刚解锁又被续约 → 逻辑错误
        if (renew_thread_.joinable()) {
            renew_thread_.join();
        }
        
        // ═══════════════════════════════════════════════════════
        // Lua 脚本原子解锁：验证 + 删除
        // 必须验证！场景：
        //   1. A 加锁成功，锁过期时间 30s
        //   2. A 业务卡了 35s，锁已自动过期
        //   3. B 抢到锁，开始执行业务
        //   4. A 终于执行完，调用 DEL 删锁 → 删掉了 B 的锁！
        // 用 Lua 脚本保证"验证+删除"原子性
        // ═══════════════════════════════════════════════════════
        const string unlock_script = R"(
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            end
            return 0
        )";
        
        try {
            redis_.eval<long long>(unlock_script, {lock_key_}, {lock_val_});
        } catch (...) {
            // 解锁失败也没关系，锁会自动过期
            // 这是分布式锁的"最终安全性"保证
        }
    }
    
    bool is_holding() const {
        return holding_.load();
    }
};
```
```cpp
#include <iostream>
#include <sw/redis++/redis++.h>
#include "distributed_lock.hpp" // 包含上述DistributedLock类的头文件

std::string generate_uuid() {
    // 简单实现，实际应使用标准UUID库
    static std::atomic<int> counter{0};
    return "uuid-" + std::to_string(counter++);
}

int main() {
    // 连接Redis
    sw::redis::Redis redis("tcp://127.0.0.1:6379");
    
    // 创建分布式锁
    const std::string lockKey = "lock:order:12345";
    
    // 尝试获取锁
    {
        DistributedLock lock(redis, lockKey, 30); // 30秒过期时间
        
        if (lock.try_lock()) {
            std::cout << "获取锁成功，执行业务逻辑..." << std::endl;
            
            // 模拟耗时业务逻辑
            std::this_thread::sleep_for(std::chrono::seconds(15));
            
            std::cout << "业务逻辑执行完毕" << std::endl;
            // 锁会在作用域结束时自动释放
        } else {
            std::cout << "获取锁失败，资源被占用" << std::endl;
        }
    } // 离开作用域，锁自动释放
    
    return 0;
}

```

<a id="mechanism"></a>
# 附录2：Redis++ 重要机制
## 1.Redis++ 的心跳机制（非自己实现）
### （1）介绍
- **心跳机制**:
	- 是一种定期发送探测信号来确认通信双方是否存活的技术。
- **Redis++ 的心跳机制**：
	- **本质**：
		- 直接启用了 **Linux 内核的 TCP KeepAlive 机制**。
		- **Redis++ 没有自己实现心跳逻辑**，完全依赖操作系统内核的 TCP KeepAlive，这也是最高效的方式（零应用层开销）
	- {% post_link "TCP KeepAlive 心跳机制" 'TCP的心跳机制' %}


### （2）时间流程图
```plaintext
时间 ─────────────────────────────────────────────────────────────────────────▶

[最后数据交互]
      │
      │◀─────── 第1阶段：长等待 ───────▶│◀───── 第2阶段：短间隔探测 ─────▶ │
      │     tcp_keepalive_time          │     tcp_keepalive_intvl         │
      │         = 7200s (2小时)         │         = 75s × 9次              |
      │                                 │                                  │
      │         （空闲，不探测）          │  💓──75s──💓──75s──...──💓     │
      │                                 │  #1      #2          #9          │
      │                                 │                                  │
      └─────────────────────────────────┴──────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              最终判定逻辑                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  ✅ 任意探测收到 ACK → 重置计时器，回到第1阶段                                │
│  ❌ 连续9次无响应    → 判定连接死亡，关闭连接                                 │
└─────────────────────────────────────────────────────────────────────────────┘

总检测时间 = 7200s + 75s × 9 = 7875s ≈ 2小时11分钟
```

### （3）客户端 ↔ Redis 交互示意图
```plaintext
┌─────────────┐                                                         ┌─────────────┐
│  Redis++    │                                                         │   Redis     │
│  Client     │                                                         │  Server     │
└──────┬──────┘                                                         └──────┬──────┘
       │                                                                       │
       │ [业务数据交互] → 最后一次数据收发完成 → 连接进入空闲状态                  │
       │                                                                       │
       │────────────────────────── 等待 7200s ────────────────────────────────────│
       │                                                                       │
       │  💓 发送第1个 Keepalive 探测包                                       │
       │───────────────────────────────────────────────────────────────────────▶│
       │                                                                       │
       │◀───────────────────────────────────────────────────────────────────────│
       │       ✅ 收到 ACK → 重置计时器 → 回到空闲等待（最优路径）                │
       │                                                                       │
       │                                                                       │
       │  ❌ 未收到 ACK → 等待 75s                                              │
       │                                                                       │
       │  💓 发送第2个 Keepalive 探测包                                       │
       │───────────────────────────────────────────────────────────────────────▶│
       │                                                                       │
       │◀───────────────────────────────────────────────────────────────────────│
       │       ✅ 收到 ACK → 重置计时器 → 回到空闲等待                          │
       │                                                                       │
       │                                                                       │
       │  ... 重复探测逻辑，最多发送 9 次探测包 ...                              │
       │                                                                       │
       │  💓 发送第9个 Keepalive 探测包（最后一次）                              │
       │───────────────────────────────────────────────────────────────────────▶│
       │                                                                       │
       │  ❌ 仍未收到 ACK → 判定连接失效 → 关闭当前连接 → 触发应用层重连逻辑        │
       │                                                                       │
       ▼                                                                       ▼
```


<a id="install"></a>
# 附录3：Redis++ 安装教程

## 1.便捷安装 
```bash
# 先安装 hiredis（底层依赖）
sudo apt install libhiredis-dev

# 安装 redis-plus-plus
cd ~/thirdparty	# 切换到你自己的安装目录
git clone https://github.com/sewenew/redis-plus-plus.git
cd redis-plus-plus
mkdir build && cd build

cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DCMAKE_CXX_STANDARD=17 \
         -DHIREDIS_LIB=/usr/local/lib/libhiredis.so

make -j$(nproc)
sudo make install
```
**如果安装失败，可尝试下面的方法**

## 2.普通安装
**安装 hiredis**
```bash
# 安装依赖
sudo apt update && sudo apt install -y build-essential git

# 克隆并编译 hiredis
git clone https://github.com/redis/hiredis.git
cd hiredis
make && sudo make install
sudo ldconfig  # 更新动态库缓存
```

**安装Redis++**
```bash
# 克隆 Redis++ 源码
git clone https://github.com/sewenew/redis-plus-plus.git
cd redis-plus-plus

# 查看所有可用版本
git tag

# 切换到你想要的版本（以1.3.15为例）（可选）
# git checkout version
git checkout 1.3.15

# 创建构建目录
mkdir build && cd build

# 配置编译选项
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DREDIS_PLUS_PLUS_BUILD_TEST=OFF \
      -DREDIS_PLUS_PLUS_BUILD_STATIC=OFF \
      -DREDIS_PLUS_PLUS_BUILD_SHARED=ON \
      -DREDIS_PLUS_PLUS_CXX_STANDARD=17 \
      ..

# 编译（使用多核加速）
make -j$(nproc)

# 安装
sudo make install

# 更新动态链接库缓存
sudo ldconfig
```

## 3.验证安装是否成功

### （1）验证安装是否成功
1. 检查库文件是否存在
```bash
# 检查 redis-plus-plus 库
ls /usr/local/lib/libredis++.so
# 检查 hiredis 库
ls /usr/local/lib/libhiredis.so
```

2. 开启Redis服务
```bash
# Ubuntu/Debian
sudo systemctl start redis-server

# 设置开机自启
sudo systemctl enable redis-server
```

3. 编写简单测试代码（如 test_redis.cpp）:
```cpp
#include <sw/redis++/redis++.h>
#include <iostream>

int main() {
    sw::redis::ConnectionOptions opts;
    opts.host = "127.0.0.1";
    sw::redis::Redis redis(opts);
    
    redis.set("test", "hello");
    auto it = redis.get("test");
    
    if (it) {
        std::cout << *it << '\n';  // 注意 *it
    } else {
        std::cout << "(nil)" << '\n';
    }
    
    return 0;
}

```


4. 编译并运行测试代码
```bash
g++ test_redis.cpp -o test_redis -lredis++ -lhiredis -std=c++17

./test_redis
```

### （2）查看当前安装的版本
```bash
cat /usr/local/include/sw/redis++/version.h
```
**可以看到**：
```cpp
const int VERSION_MAJOR = 1;   // 主版本
const int VERSION_MINOR = 3;   // 次版本
const int VERSION_PATCH = 15;  // 补丁版本
// 当前版本：1.3.15 
```

### 4.Redis++版本升级（切换）
以我的安装目录“\~/thirdparty/redis-plus-plus”为例
```bash
cd ~/thirdparty/redis-plus-plus

# 清理旧的构建
rm -rf build

# 切换到最新稳定版
git checkout 1.3.15

# 编译安装
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DREDIS_PLUS_PLUS_CXX_STANDARD=17 \
      ..
make -j$(nproc)
sudo make install
sudo ldconfig
```

### 5.删除Redis++
如果安装失败，可通过下述方式清理Redis++文件
```bash
# ========== 1. 清理 Redis++ 库文件 ==========
# 删除动态库文件（包括软链接）
sudo rm -f /usr/local/lib/libredis++.so
sudo rm -f /usr/local/lib/libredis++.so.*
# 删除静态库文件（若有）
sudo rm -f /usr/local/lib/libredis++.a

# ========== 2. 清理 Redis++ CMake 配置文件 ==========
sudo rm -rf /usr/local/lib/cmake/redis++
# 删除 pkg-config 配置（若有）
sudo rm -f /usr/local/lib/pkgconfig/redis++.pc

# ========== 3. 清理 Redis++ 头文件 ==========
# 核心头文件目录（包含 StringView 定义）
sudo rm -rf /usr/local/include/sw/redis++
# 若存在 sw 空目录，可一并删除（可选）
sudo rmdir /usr/local/include/sw 2>/dev/null || true

# ========== 4. 刷新系统动态库缓存 ==========
sudo ldconfig

# ========== 5. 验证清理结果（可选） ==========
# 若以下命令均无输出，说明清理完成
find /usr/local/lib -name "libredis++*"
find /usr/local/include -name "redis++"

# ========== 6. 删除Redis++目录 ==============
sudo rm -rf redis-plus-plus/
```