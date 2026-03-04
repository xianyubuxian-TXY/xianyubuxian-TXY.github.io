---
title: zookeeper的使用
date: 2026-02-07 10:13:41
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- zookeeper的使用
---

# 一、zookeeper概述

## 1.Zookeeper的定义
- Zookeeper 是 Apache 基金会开源的**分布式协调服务**，核心定位是为分布式系统提供统一的**服务发现**、分布式锁、配置管理、集群选主等能力。
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                      Zookeeper 本质                              │
├─────────────────────────────────────────────────────────────────┤
│  • 一个分布式的、开放源码的分布式应用程序协调服务                    │
│  • 一个高性能的分布式数据一致性解决方案                             │
│  • 类似于一个 "分布式文件系统" + "通知机制"                        │
│  • 底层基于 ZAB 协议（Zookeeper Atomic Broadcast）保证一致性       │
└─────────────────────────────────────────────────────────────────┘
```

## 2.数据模型 — ZNode
- Zookeeper 的数据结构类似于 Unix 文件系统的**树形结构**
```cpp
/                                    # 根节点
├── /services                        # 服务注册根节点
│   ├── /auth-service               # 认证服务
│   │   ├── /node_0000000001        # 实例1: {"host":"192.168.1.10","port":50051}
│   │   ├── /node_0000000002        # 实例2: {"host":"192.168.1.11","port":50051}
│   │   └── /node_0000000003        # 实例3: {"host":"192.168.1.12","port":50051}
│   └── /user-service               # 用户服务
│       └── /node_0000000001        # 实例1: {"host":"192.168.1.20","port":50052}
├── /config                          # 配置中心
│   ├── /database                   # 数据库配置
│   └── /redis                      # Redis配置
└── /locks                           # 分布式锁
    └── /order-lock                 # 订单锁
```

## 3.ZNode 的四种类型
```cpp
// Zookeeper 节点类型常量
enum ZNodeType {
    PERSISTENT = 0,                    // 持久节点：创建后一直存在，除非主动删除
    PERSISTENT_SEQUENTIAL = 2,         // 持久顺序节点：持久 + 自动编号
    EPHEMERAL = 1,                     // 临时节点：会话结束自动删除 ⭐
    EPHEMERAL_SEQUENTIAL = 3           // 临时顺序节点：临时 + 自动编号 ⭐⭐（服务注册常用）
};
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260207102115806.png)


# 二、Zookeeper的核心功能

## 1.七大核心功能
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                    Zookeeper 核心功能                            │
├─────────────────────────────────────────────────────────────────┤
│  1. 服务注册与发现  ──  微服务架构的基础设施                        │
│  2. 配置管理       ──  集中化配置，动态更新                        │
│  3. 分布式锁       ──  跨进程/跨机器的互斥访问                     │
│  4. 集群管理       ──  节点存活监控，Master选举                    │
│  5. 命名服务       ──  全局唯一ID、路径标识                        │
│  6. 分布式队列     ──  FIFO队列、优先级队列                        │
│  7. 负载均衡       ──  服务实例的动态感知与选择                     │
└─────────────────────────────────────────────────────────────────┘
```
**注**：我们仅关注**服务注册与发现**功能

## 2.Watch 机制（核心中的核心）
- ZooKeeper 的 Watcher 机制本质上就是一个**事件驱动的“回调机制”**

### （1）核心定义
- Zookeeper 的 Watch 是一种**一次性、异步、轻量级**的**事件通知机制**：
	- 客户端可以为某个 ZNode（节点）注册 Watch 监听；
	- 当该 ZNode 发生指定变化（如创建、删除、数据修改、子节点变化）时，Zookeeper 服务端会主动向客户端推送事件通知；
	- 客户端收到通知后，可根据事件类型执行相应逻辑（如更新 gRPC 服务列表、刷新配置）
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Watch 工作原理                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Client A (订阅者)              ZooKeeper Server              Client B     │
│       │                              │                            │         │
│       │  1. getData("/config",      │                            │         │
│       │     watch=true)             │                            │         │
│       │ ──────────────────────────→ │                            │         │
│       │                              │                            │         │
│       │  2. 返回数据 + 注册Watch    │                            │         │
│       │ ←────────────────────────── │                            │         │
│       │                              │                            │         │
│       │     【Watch 已注册】         │                            │         │
│       │     等待事件触发...          │                            │         │
│       │                              │                            │         │
│       │                              │  3. setData("/config",    │         │
│       │                              │     "new_value")          │         │
│       │                              │ ←──────────────────────── │         │
│       │                              │                            │         │
│       │  4. Watch 事件通知           │                            │         │
│       │     (NodeDataChanged)       │                            │         │
│       │ ←────────────────────────── │                            │         │
│       │                              │                            │         │
│       │  5. 重新 getData + 注册Watch│                            │         │
│       │ ──────────────────────────→ │  ← 需要重新注册！          │         │
│       │                              │                            │         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### （2）Watch的核心特性
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260207103102243.png)

### （3）Watch 类型
#### <1>节点级 Watch
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                           两种 Watch 类型                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │      Data Watch (数据监听)      │  │    Child Watch (子节点监听)     │  │
│  ├─────────────────────────────────┤  ├─────────────────────────────────┤  │
│  │                                 │  │                                 │  │
│  │  注册方法:                      │  │  注册方法:                      │  │
│  │  • getData()                   │  │  • getChildren()               │  │
│  │  • exists()                    │  │                                 │  │
│  │                                 │  │                                 │  │
│  │  触发事件:                      │  │  触发事件:                      │  │
│  │  • NodeCreated    (节点创建)   │  │  • NodeChildrenChanged         │  │
│  │  • NodeDataChanged (数据变更)   │  │    (子节点增删)                 │  │
│  │  • NodeDeleted    (节点删除)   │  │                                 │  │
│  │                                 │  │                                 │  │
│  │  监听范围: 当前节点             │  │  监听范围: 直接子节点列表       │  │
│  │                                 │  │  (不监听子节点的数据变化)       │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### <2>全局会话 Watcher
- zookeeper_init 函数中的 全局会话 Watcher（watcher_fn fn）是一个**回调函数**，用于监控 ZooKeeper 客户端与服务器之间会话状态的变化。具体来说，它会在以下几种情况下被触发：
	- **会话连接成功/断开**
		- 当客户端成功连接到 ZooKeeper 服务端时，或者当连接断开、会话超时或者会话失效时，**这个回调函数就会被触发**。
	- **会话状态变化**
		- 会话状态会有多种状态变化，如：
			- ZOO_CONNECTED_STATE：客户端与 ZooKeeper 服务端成功建立连接并保持会话。
			- ZOO_CONNECTING_STATE：客户端正在尝试与 ZooKeeper 服务端建立连接。
			- ZOO_EXPIRED_SESSION_STATE：会话过期，客户端失去连接，需重新建立连接并重新注册 Watch。
			- ZOO_AUTH_FAILED_STATE：会话的认证失败，客户端无法连接。
```cpp
typedef void (*watcher_fn)(zhandle_t *zh,
                           int type,     // 事件类型
                           int state,    // 会话状态
                           const char *path, // 相关节点路径（会话事件通常为 NULL）
                           void *ctx);   // 你 init 传入的 context

// 示例
 // ==========================================================================
    // 1. 会话 Watcher（全局回调，处理连接状态变化）
    // ==========================================================================
    // 
    // 这个回调在以下情况被调用：
    // - 连接建立成功 → state = ZOO_CONNECTED_STATE
    // - 连接断开（网络问题）→ state = ZOO_CONNECTING_STATE（自动重连中）
    // - 会话过期（超过 timeout 未能重连）→ state = ZOO_EXPIRED_SESSION_STATE
    //
    // 参数说明：
    // - zh: 客户端句柄（可以在回调中继续调用 ZK API）
    // - type: 事件类型（这里固定是 ZOO_SESSION_EVENT = -1）
    // - state: 连接状态（ZOO_CONNECTED_STATE / ZOO_CONNECTING_STATE 等）
    // - path: 对于会话事件，此参数无意义
    // - ctx: 用户在 zookeeper_init 时传入的上下文指针（这里是 this）
    //
    // ★ 必须是 static 函数：C 回调不支持成员函数指针
    //
    static void SessionWatcher(zhandle_t* zh, int type, int state, 
                                const char* path, void* ctx) {
        // 只处理会话事件，忽略其他类型（如数据变更、子节点变更）
        // 因为这是全局 Watcher，可能收到所有类型的事件（如果用 zoo_get 传 watch=1）
        if (type == ZOO_SESSION_EVENT) {
            switch (state) {
                case ZOO_CONNECTED_STATE:
                    // 连接成功（首次连接或断线重连成功）
                    // 此时可以安全地进行 ZK 操作
                    printf("✓ 已连接\n");
                    break;
                    
                case ZOO_EXPIRED_SESSION_STATE:
                    // 会话过期 - 这是严重错误！
                    // 原因：客户端与服务器断开时间超过 session timeout
                    // 后果：
                    //   1. 所有临时节点（EPHEMERAL）已被服务器删除
                    //   2. 所有 Watch 已失效
                    //   3. 当前 zhandle_t 不可用
                    // 处理：必须关闭当前句柄，重新调用 zookeeper_init()
                    printf("✗ 会话过期，需重连\n");
                    // 可以通过 ctx 拿到 this 指针，调用重连逻辑
                    // auto* self = static_cast<ZkClientExample*>(ctx);
                    // self->Reconnect();
                    break;
                    
                case ZOO_CONNECTING_STATE:
                    // 正在连接/重连中
                    // ZK 客户端会自动尝试重连其他服务器节点
                    // 在此状态下调用 ZK API 会返回 ZCONNECTIONLOSS
                    printf("... 连接中\n");
                    break;
                    
                case ZOO_AUTH_FAILED_STATE:
                    // 认证失败（如果使用了 ACL 认证）
                    printf("✗ 认证失败\n");
                    break;
            }
        }
    }
```

### （4）事件类型
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260207103357954.png)


## 3.Session 会话机制
```cpp
// 会话状态
enum SessionState {
    CONNECTING = 1,        // 正在连接
    CONNECTED = 3,         // 已连接（正常工作状态）
    DISCONNECTED = 0,      // 连接断开（临时，可恢复）
    EXPIRED = -112         // 会话过期（需要重建）
};
```
```cpp
┌─────────────────────────────────────────────────────────────┐
│                    Session 生命周期                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client Start                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────┐                                            │
│  │ CONNECTING  │ ─────────────────────────────────────┐     │
│  └─────────────┘                                      │     │
│       │ 连接成功                                       │     │
│       ▼                                              │     │
│  ┌─────────────┐    网络闪断      ┌──────────────┐   │     │
│  │ CONNECTED   │ ───────────────► │ DISCONNECTED │   │     │
│  └─────────────┘ ◄─────────────── └──────────────┘   │     │
│       │           网络恢复                │          │     │
│       │                           超时未恢复 │          │     │
│       │                                  ▼          │     │
│       │                          ┌─────────────┐    │     │
│       │                          │  EXPIRED    │ ◄──┘     │
│       │                          └─────────────┘          │
│       ▼                                  │                │
│  临时节点保持                         临时节点删除          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

# 三、ZooKeeper C 客户端核心类型 C++ 伪代码
## 1.核心句柄与类型定义
```cpp
// ==================== 核心类型 ====================

// ZooKeeper 客户端句柄（不透明指针，所有操作都需要它）
typedef struct _zhandle zhandle_t;

// Watcher 回调函数原型
typedef void (*watcher_fn)(
    zhandle_t* zh,      // 客户端句柄
    int type,           // 事件类型 (ZOO_*_EVENT)
    int state,          // 连接状态 (ZOO_*_STATE)
    const char* path,   // 触发事件的节点路径
    void* watcherCtx    // 用户传入的上下文指针
);

// 字符串向量（用于存储子节点列表）
struct String_vector {
    int32_t count;      // 字符串数量
    char** data;        // 字符串数组
};

// 节点状态信息
struct Stat {
    int64_t czxid;          // 创建事务ID
    int64_t mzxid;          // 最后修改事务ID
    int64_t ctime;          // 创建时间戳
    int64_t mtime;          // 最后修改时间戳
    int32_t version;        // 数据版本号（每次修改+1）
    int32_t cversion;       // 子节点版本号
    int32_t aversion;       // ACL版本号
    int64_t ephemeralOwner; // 临时节点所属会话ID（0表示持久节点）
    int32_t dataLength;     // 数据长度
    int32_t numChildren;    // 子节点数量
    int64_t pzxid;          // 最后子节点修改事务ID
};
```

## 2.核心常量定义
```cpp
// ==================== 事件类型 (type 参数) ====================
#define ZOO_CREATED_EVENT       1   // 节点创建（watch exists 触发）
#define ZOO_DELETED_EVENT       2   // 节点删除
#define ZOO_CHANGED_EVENT       3   // 节点数据变更
#define ZOO_CHILD_EVENT         4   // 子节点列表变更
#define ZOO_SESSION_EVENT       -1  // 会话状态变更（连接/断开/过期）
#define ZOO_NOTWATCHING_EVENT   -2  // Watch 被移除

// ==================== 连接状态 (state 参数) ====================
#define ZOO_EXPIRED_SESSION_STATE   -112  // 会话过期（必须重连）
#define ZOO_AUTH_FAILED_STATE       -113  // 认证失败
#define ZOO_CONNECTING_STATE        1     // 正在连接
#define ZOO_ASSOCIATING_STATE       2     // 关联中
#define ZOO_CONNECTED_STATE         3     // 已连接 ✓
#define ZOO_READONLY_STATE          5     // 只读模式
#define ZOO_NOTCONNECTED_STATE      999   // 未连接

// ==================== 节点创建标志 ====================
#define ZOO_PERSISTENT      0         // 持久节点（永久存在，需手动删除）
#define ZOO_EPHEMERAL       1         // 临时节点（会话断开自动删除）
#define ZOO_SEQUENCE        2         // 顺序节点（自动追加序号）

// 组合使用：
// ZOO_PERSISTENT                     = 0  持久节点
// ZOO_EPHEMERAL                      = 1  临时节点
// ZOO_SEQUENCE                       = 2  持久顺序节点
// ZOO_EPHEMERAL | ZOO_SEQUENCE       = 3  临时顺序节点


// ==================== 错误码 ====================
#define ZOK                     0     // 成功
#define ZNONODE                 -101  // 节点不存在
#define ZNODEEXISTS             -110  // 节点已存在
#define ZNOTEMPTY               -111  // 节点有子节点，无法删除
#define ZBADVERSION             -103  // 版本号不匹配
#define ZCONNECTIONLOSS         -4    // 连接丢失
#define ZSESSIONEXPIRED         -112  // 会话过期
#define ZINVALIDSTATE           -9    // 无效状态
```

## 3.核心 API 函数（重点！）
```cpp
// ==================== 连接管理 ====================
/*
- zookeeper_init 函数中的 全局会话 Watcher（watcher_fn fn）是一个回调函数，用于监控 ZooKeeper 客户端与服务器之间会话状态的变化。具体来说，它会在以下几种情况下被触发：
	1.会话连接成功/断开
		- 当客户端成功连接到 ZooKeeper 服务端时，或者当连接断开、会话超时或者会话失效时，这个回调函数就会被触发。

	2.会话状态变化
		- 会话状态会有多种状态变化，如：
			- ZOO_CONNECTED_STATE：客户端与 ZooKeeper 服务端成功建立连接并保持会话。
			- ZOO_CONNECTING_STATE：客户端正在尝试与 ZooKeeper 服务端建立连接。
			- ZOO_EXPIRED_SESSION_STATE：会话过期，客户端失去连接，需重新建立连接并重新注册 Watch。
			- ZOO_AUTH_FAILED_STATE：会话的认证失败，客户端无法连接。
*/


// 初始化连接（异步，返回后不一定已连接）
zhandle_t* zookeeper_init(
    const char* hosts,      // 服务器地址 "ip1:port1,ip2:port2"
    watcher_fn fn,          // 全局会话 Watcher：一个回调函数
    int recv_timeout,       // 会话超时时间(ms)
    const clientid_t* cid,  // 客户端ID（重连用，一般传0）
    void* context,          // 用户上下文
    int flags               // 标志位（一般传0）
);

// 关闭连接
int zookeeper_close(zhandle_t* zh);

// 获取当前状态
int zoo_state(zhandle_t* zh);  // 返回 ZOO_*_STATE

// ==================== 节点创建 ====================

// 同步创建节点
int zoo_create(
    zhandle_t* zh,
    const char* path,           	// 节点路径
    const char* value,          	// 节点数据
    int valuelen,               	// 数据长度
    const struct ACL_vector* acl, 	// 权限（用 &ZOO_OPEN_ACL_UNSAFE）
    int flags,                  	// 节点类型：ZOO_EPHEMERAL | ZOO_SEQUENCE
    char* path_buffer,          	// 输出：实际创建的路径（顺序节点用）
    int path_buffer_len         	// path_buffer 缓冲区大小
);

// ==================== 节点读取 ====================

// 获取数据（不带 Watch）
int zoo_get(
    zhandle_t* zh,
    const char* path,			// 节点路径
    int watch,                  // 0=不监听，1=使用全局Watcher
    char* buffer,               // 输出：数据缓冲区
    int* buffer_len,            // 输入输出：缓冲区大小/实际数据长度
    struct Stat* stat           // 输出：节点状态（可传NULL）
);

// 获取数据 + 注册自定义 Watcher（★推荐）
int zoo_wget(
    zhandle_t* zh,
    const char* path,
    watcher_fn watcher,         // 自定义回调
    void* watcherCtx,           // 回调上下文
    char* buffer,				// 输出：数据缓冲区
    int* buffer_len,			// 输入输出：缓冲区大小/实际数据长度
    struct Stat* stat 			// 输出：节点状态（可传NULL）
);

// 检查节点是否存在
int zoo_exists(
    zhandle_t* zh,
    const char* path,
    int watch,
    struct Stat* stat           // 输出：节点状态
);

// ==================== 子节点操作 ====================

// 获取子节点列表（不带 Watch）
int zoo_get_children(
    zhandle_t* zh,
    const char* path,
    int watch,
    struct String_vector* strings  // 输出：子节点名称列表
);

// 获取子节点 + 注册自定义 Watcher（★推荐）
int zoo_wget_children(
    zhandle_t* zh,
    const char* path,
    watcher_fn watcher,
    void* watcherCtx,
    struct String_vector* strings
);

// ★ 释放 String_vector 内存（必须调用！）
void deallocate_String_vector(struct String_vector* v);

// ==================== 节点修改/删除 ====================

// 设置节点数据
int zoo_set(
    zhandle_t* zh,
    const char* path,
    const char* buffer,
    int buflen,
    int version                 // 版本号（-1 表示忽略版本检查）
);

// 删除节点
int zoo_delete(
    zhandle_t* zh,
    const char* path,
    int version                 // -1 表示忽略版本
);
```

### （1）zookeeper_init 函数详解
- zookeeper_init 是**异步**的。它会**立即返回**一个 zhandle_t* 句柄，但此时**连接尚未建立完成**。
- 通常会使用“**轮询**”或“**条件变量**”来**等待连接的建立**
```cpp
// Watcher 回调函数原型
typedef void (*watcher_fn)(
    zhandle_t* zh,      // 客户端句柄
    int type,           // 事件类型 (ZOO_*_EVENT)
    int state,          // 连接状态 (ZOO_*_STATE)
    const char* path,   // 触发事件的节点路径
    void* watcherCtx    // 用户传入的上下文指针
);

// 初始化连接（异步，返回后不一定已连接）
zhandle_t* zookeeper_init(
    const char* hosts,      // 服务器地址 "ip1:port1,ip2:port2"
    watcher_fn fn,          // 全局会话 Watcher：一个回调函数
    int recv_timeout,       // 会话超时时间(ms)
    const clientid_t* cid,  // 客户端ID（重连用，一般传0）
    void* context,          // 用户上下文
    int flags               // 标志位（一般传0）
);
```

#### <1>工作流程
- 调用zookeeper_init后，立即返回zhandle_t* 句柄，在后台进行TCP连接，可能会经历ZOO_CONNECTED_STATE、ZOO_CONNECTING_STATE等状态变化
	- 每一个状态的变化，都会调用传入的**“watcher_fn fn”回调函数**
	- zookeeper的服务端会填充watcher_fn的type、state、path，而watcher_fn的watcherCtx则是zookeeper_init调用时传入的context
- 通过回调函数，我们可以知道连接的状态（连接、断开等），这里通常**配合“条件变量”使用**

##### 1）zookeeper_init + watcher 回调完整流程
```cpp
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                    zookeeper_init + watcher 回调完整流程                              │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│   用户代码（主线程）                 ZK 客户端库                   ZooKeeper 服务端    │
│       │                              │                              │                 │
│       │  ① zookeeper_init(           │                              │                 │
│       │       "127.0.0.1:2181",      │                              │                 │
│       │       global_watcher,   ─────┼──▶ 保存 watcher 函数指针     │                 │
│       │       30000,                 │                              │                 │
│       │       nullptr,               │                              │                 │
│       │       &my_context,      ─────┼──▶ 保存 context 指针         │                 │
│       │       0                      │                              │                 │
│       │     )                        │                              │                 │
│       │ ────────────────────────────▶│                              │                 │
│       │                              │                              │                 │
│       │  ② 立即返回 zhandle_t*        │  启动后台 I/O 线程           │                 │
│       │◀──────────────────────────── │      │                       │                 │
│       │                              │      │                       │                 │
│       │  ③ 主线程调用 wait_for 阻塞   │      │  ③ TCP 三次握手       │                 │
│       │  ┌─────────────────────────────────┐ │ ─────────────────────▶│                 │
│       │  │ bool success = cv.wait_for(    │ │                       │                 │
│       │  │     lock,                      │ │  ④ ZK 会话建立         │                 │
│       │  │     chrono::milliseconds(5000),│ │◀─────────────────────▶│                 │
│       │  │     [&] { return connected; }  │ │                       │                 │
│       │  │ );                             │ │  ⑤ 状态变为 CONNECTED  │                 │
│       │  │                                │ │      │                 │                 │
│       │  │ // 最多等待 5 秒               │ │      │                 │                 │
│       │  │ // 超时返回 false              │ │      │                 │                 │
│       │  └────────────┬───────────────────┘ │      │                 │                 │
│       │               │ (阻塞中，最多5秒)    │◀─────┘                 │                 │
│       │               │                     │                        │                 │
│       │               │  ⑥ 后台线程调用 watcher 回调                  │                 │
│       │               │  ┌──────────────────────────────────────────────┐             │
│       │               │  │ global_watcher(                              │             │
│       │               │  │     zh,                    // 句柄           │             │
│       │               │  │     ZOO_SESSION_EVENT,     // type           │             │
│       │               │  │     ZOO_CONNECTED_STATE,   // state          │             │
│       │               │  │     "",                    // path           │             │
│       │               │  │     &my_context            // ctx            │             │
│       │               │  │ ) {                                          │             │
│       │               │  │     lock_guard<mutex> lock(ctx->mtx);        │             │
│       │               │  │     ctx->connected = true;  // 设置条件      │             │
│       │               │  │     ctx->cv.notify_all();   // ⑦ 唤醒主线程  │             │
│       │               │  │ }                                            │             │
│       │               │  └──────────────────────────────────────────────┘             │
│       │               │                     │                        │                │
│       │  ⑧ wait_for 返回                    │                        │                │
│       │  ┌─────────────────────────────────┐│                        │                │
│       │  │ if (success) {                 ││                        │                │
│       │  │     // ✅ 连接成功              ││                        │                │
│       │  │ } else {                       ││                        │                │
│       │  │     // ❌ 超时，连接失败        ││                        │                │
│       │  │ }                              ││                        │                │
│       │  └─────────────────────────────────┘│                        │                │
│       │                                     │                        │                │
│       │  ⑨ 主线程继续执行                   │                        │                │
│       │  (成功则调用 zoo_create 等)         │                        │                │
│       │                                     │                        │                │
└──────────────────────────────────────────────────────────────────────────────────────┘

```

##### 2）实际封装示例
```cpp
class ZooKeeperClient {
public:
    /**
     * @brief 连接到 ZooKeeper
     * @param timeout_ms 连接超时时间（毫秒）
     * @return 连接成功返回 true
     */
    bool Connect(int timeout_ms = 10000);

private:
	    /**
     * @brief 全局 Watcher 回调（静态，供 C API 调用）
     * @param type 事件类型：ZOO_SESSION_EVENT（会话状态变更）、ZOO_CHILD_EVENT（子节点列表变更）等
     * @param state 会话状态：连接、断开、过期等（用于：ZOO_SESSION_EVENT）
     * @param path 子节点路径（用于：ZOO_CHILD_EVENT）
     * @param context 用户上下文，传入ZooKeeperClient*，方便在内部调用HandleXXX函数
     * @note 必须是static函数，因为是供 C API 调用，故不可以用普通成员函数
     */
    static void GlobalWatcher(zhandle_t* zh, int type, int state,
                              const char* path, void* context);

    /**
     * @brief 处理会话状态变化
     */
    void HandleSessionEvent(int state);

    // ZooKeeper 句柄
    zhandle_t* zh_ = nullptr;
    
    // 配置
    std::string hosts_;
    int session_timeout_ms_;
    
    // 连接状态（原子变量，线程安全）
    std::atomic<bool> connected_{false};
    
    // 连接等待
    std::mutex conn_mutex_;
    std::condition_variable conn_cv_;
}
```
```cpp
ZooKeeperClient::ZooKeeperClient(const std::string& hosts, int session_timeout_ms)
    : hosts_(hosts)
    , session_timeout_ms_(session_timeout_ms) {
    
    // 设置 ZK 日志级别（可选）
    zoo_set_debug_level(ZOO_LOG_LEVEL_WARN);
}

ZooKeeperClient::~ZooKeeperClient() {
    Close();
}

// ============================================================================
// 通用接口（gRPC 服务端 + gRPC 客户端 都使用）
// ============================================================================

bool ZooKeeperClient::Connect(int timeout_ms){
    std::unique_lock<std::mutex> lock(conn_mutex_);

    // 已连接
    if(zh_ && connected_.load()){
        return true;
    }

    // 关闭旧连接
    if(zh_){
        zookeeper_close(zh_);
        zh_=nullptr;
    }

    connected_=false;

    // 初始化 ZK 句柄
    zh_=zookeeper_init(hosts_.c_str(),GlobalWatcher,session_timeout_ms_,nullptr,this,0);

    if (!zh_) {
        LOG_ERROR("zookeeper_init failed, hosts={}", hosts_);
        return false;
    }

    // 等待连接建立（使用条件变量）——> 连接建立成功时，会在”回调函数“中将connected_设置为true
    bool success = conn_cv_.wait_for(lock,
                std::chrono::milliseconds(timeout_ms),
                [this]{return connected_.load();});

    if(!success){
        LOG_ERROR("ZooKeeper connection timeout, hosts={}", hosts_);
        zookeeper_close(zh_);
        zh_ = nullptr;
        return false; 
    }

    LOG_INFO("ZooKeeper client connected, hosts={}", hosts_);
    return true;
}

// 全局 Watcher 回调（静态，供 C API 调用）
void ZooKeeperClient::GlobalWatcher(zhandle_t* zh, int type, int state,
                    const char* path, void* context){
    
    ZooKeeperClient* client=static_cast<ZooKeeperClient*>(context);
    if(!client) return;

    
    if(type==ZOO_SESSION_EVENT){
        // 会话事件：连接、断开、过期等
        client->HandleSessionEvent(state);
    }else if(type==ZOO_CHILD_EVENT && path){
        // 子节点变化事件
        // client->HandleChildEvent(path);
    }
}

// 处理会话状态变化
void ZooKeeperClient::HandleSessionEvent(int state) {
    if (state == ZOO_CONNECTED_STATE) { 
        // 连接成功
        connected_ = true;
        LOG_INFO("ZooKeeper connected");
        conn_cv_.notify_all();  // 通知等待连接的线程
    } else if (state == ZOO_EXPIRED_SESSION_STATE) {
        // 会话过期
        connected_ = false;
        LOG_WARN("ZooKeeper session expired");
    } else if (state == ZOO_CONNECTING_STATE) {
        // 正在连接
        connected_ = false;
        LOG_INFO("ZooKeeper reconnecting...");
    } else if (state == ZOO_ASSOCIATING_STATE) {
        // 关联中
        LOG_DEBUG("ZooKeeper associating...");
    } else {
        LOG_WARN("ZooKeeper unknown state: {}", state);
    }
}

```

### （2）节点操作（gRPC服务端相关）
- gRPC服务端主要会用到zoo_create、zoo_exists、zoo_set、zoo_delete
	- **返回值**：zookeeper的错误码 
```cpp
// 同步创建节点
int zoo_create(
    zhandle_t* zh,
    const char* path,               // 节点路径
    const char* value,              // 节点数据
    int valuelen,                   // 数据长度
    const struct ACL_vector* acl,     // 权限（用 &ZOO_OPEN_ACL_UNSAFE）
    int flags,                      // 节点类型：ZOO_EPHEMERAL | ZOO_SEQUENCE
    char* path_buffer,              // 输出：实际创建的路径（顺序节点用）
    int path_buffer_len             // path_buffer 缓冲区大小
);

// 检查节点是否存在
int zoo_exists(
    zhandle_t* zh,
    const char* path,
    int watch,
    struct Stat* stat           // 输出：节点状态
);

// 设置节点数据
int zoo_set(
    zhandle_t* zh,
    const char* path,
    const char* buffer,
    int buflen,
    int version                 // 版本号（-1 表示忽略版本检查）
);

// 删除节点
int zoo_delete(
    zhandle_t* zh,
    const char* path,
    int version                 // -1 表示忽略版本
);
```

#### <1>两种核心节点（服务注册相关）
```cpp
// ==================== 节点创建标志 ====================
#define ZOO_PERSISTENT      0         // 持久节点（永久存在，需手动删除）
#define ZOO_EPHEMERAL       1         // 临时节点（会话断开自动删除）
```
- 父节点路径通常使用**持久节点**
	- 存储实例节点
- 实例节点通常使用**临时节点**
	- 存储服务的“ip:port”
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                      服务注册节点结构                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  /services                          ← 持久节点（ZOO_PERSISTENT = 0）         │
│      │                                                                       │
│      └── /user-service              ← 持久节点（ZOO_PERSISTENT = 0）         │
│              │                                                               │
│              ├── instance_001       ← 临时节点（ZOO_EPHEMERAL = 1）          │
│              ├── instance_002       ← 临时节点（ZOO_EPHEMERAL = 1）          │
│              └── instance_003       ← 临时节点（ZOO_EPHEMERAL = 1）          │
│                                                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  为什么这样设计？                                                             │
│                                                                              │
│  • 父节点（持久）：目录结构，不应随服务实例变化而消失                          │
│  • 实例节点（临时）：服务宕机 → 会话断开 → 节点自动删除 → 自动下线             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### <2>实际封装示例
```cpp
class ZooKeeperClient {
public:
	......

    // ============================================================================
    // gRPC 服务端使用的接口（服务注册）
    // ============================================================================
    
    /**
     * @brief 创建单个节点（父节点必须存在）
     * @param path 节点路径
     * @param data 节点数据
     * @param ephemeral 是否为临时节点（会话断开自动删除）
     * @return 创建成功或节点已存在返回 true
     * 
     * @note 服务端用于：注册服务实例（创建临时节点）
     */
    bool CreateNode(const std::string& path, const std::string& data, 
                    bool ephemeral = false);
    
    /**
     * @brief 递归创建整个路径（自动创建所有父节点）
     * @param path 服务根路径
     * @return 创建成功返回 true
     * 
     * @note 服务端用于：创建服务根路径 /services/user-service ——> 永久性节点
     * @note 服务注册的典型用法：
        void RegisterService(const std::string& service_name, 
                            const std::string& host, int port) {
            
            // 1. 先用 CreatePath 确保服务路径存在（持久节点）
            std::string service_path = "/services/" + service_name;
            zk_client->CreatePath(service_path);  // 递归创建 /services/user-service
            
            // 2. 再用 CreateNode 创建实例节点（临时节点）
            std::string instance_path = service_path + "/" + host + ":" + std::to_string(port);
            std::string data = R"({"host":")" + host + R"(","port":)" + std::to_string(port) + "}";
            zk_client->CreateNode(instance_path, data, true);  // ephemeral=true
        }

        ┌─────────────────────────────────────────────────────────────────────────────┐
        │  /services/user-service/192.168.1.100:50051                                 │
        │  ├────────┬────────────┬─────────────────────┤                              │
        │  │  持久  │    持久    │        临时          │                              │
        │  └────────┴────────────┴─────────────────────┘                              │
        │                                                                              │
        │  • /services          → 持久：服务根目录，永远存在                            │
        │  • /user-service      → 持久：服务名，永远存在                                │
        │  • /192.168.1.100:50051 → 临时：实例节点，服务下线自动删除                     │
        │                                                                              │
        │  好处：服务全部下线后，/services/user-service 还在，                           │
        │       客户端可以继续 Watch，等待新实例上线                                     │
        └─────────────────────────────────────────────────────────────────────────────┘
     */
    bool CreateParentPath(const std::string& path);
    
    /**
     * @brief 删除节点
     * @param path 节点路径
     * @return 删除成功或节点不存在返回 true
     * 
     * @note 服务端用于：主动注销服务实例
     */
    bool DeleteNode(const std::string& path);
    
    /**
     * @brief 检查节点是否存在
     * 
     * @note 服务端用于：检查路径是否已创建
     */
    bool Exists(const std::string& path);
    
    /**
     * @brief 设置节点数据
     * @param path 节点路径
     * @param data 新数据
     * @return 设置成功返回 true
     * 
     * @note 服务端用于：更新服务实例的元信息（如权重、状态）
     */
    bool SetData(const std::string& path, const std::string& data);

    ......
}
```
```cpp
// ============================================================================
// gRPC 服务端使用的接口（服务注册）
// ============================================================================

// 创建节点
bool ZooKeeperClient::CreateNode(const std::string& path, 
                                const std::string& data,
                                bool ephemeral){
    if(!IsConnected()){
        LOG_ERROR("ZK not connected, cannot create node: {}", path);
        return false;
    }

    // 节点类型（临时节点/顺序节点/临时顺序节点）
    int flags=ephemeral ? ZOO_EPHEMERAL:0;

    // 创建节点
    int rc=zoo_create(zh_,path.c_str(),data.c_str(),data.size(),
                        &ZOO_OPEN_ACL_UNSAFE,flags,nullptr,0);
    
    if(rc==ZOK){    // 节点创建成功
        LOG_DEBUG("ZK node created: {} (ephemeral={})", path, ephemeral);
        return true;
    }else if(rc==ZNODEEXISTS){
        // 节点已存在，对于持久节点视为成功
        // 对于临时节点，可能是上次会话残留，需要特殊处理
        if(ephemeral){
            LOG_WARN("ZK ephemeral node already exists: {}, attempting recovery...", path);
            
            // 尝试删除旧节点后重新创建
            // ⚠️ 问题：如果是别人创建的临时节点，不应该删除
            if(DeleteNode(path)){
                rc=zoo_create(zh_,path.c_str(),data.c_str(),data.size(),
                              &ZOO_OPEN_ACL_UNSAFE,flags,nullptr,0);
                if(rc==ZOK){
                    LOG_INFO("ZK ephemeral node recreated: {}", path);
                    return true;
                }
            }
            
            // 如果删除重建失败，尝试直接更新数据
            if(SetData(path,data)){
                LOG_INFO("ZK ephemeral node data updated: {}", path);
                return true;
            }
            
            LOG_ERROR("Failed to recover ephemeral node: {}", path);
            return false;
        }
        return true;
    }else{
        LOG_ERROR("ZK create failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

bool ZooKeeperClient::CreateParentPath(const std::string& path){
    if(!IsConnected()){
        return false;
    }

    // 使用stringstream 分割路径
    std::string current;
    std::istringstream ss(path);
    std::string token;

    /*
    输入：/services/user
    循环过程大概是：
        - token=services → current=/services → 不存在就创建
        - token=user → current=/services/user → 不存在就创建
    最终路径保证存在。
    */
    while(std::getline(ss,token,'/')){  
        if(token.empty()) continue;

        current += "/" + token;

        if(!Exists(current)){   // 节点不存在
            // 创建持久节点(节点值为空)
            if(!CreateNode(current,"",false)){
                return false;
            }
        }
    }

    return true;
}

// 删除节点
bool ZooKeeperClient::DeleteNode(const std::string& path){
    if(!IsConnected()){
        return false;
    }

    int rc = zoo_delete(zh_,path.c_str(),-1);    // -1 表示忽略版本

    if (rc == ZOK) {
        LOG_DEBUG("ZK node deleted: {}", path);
        return true;
    } else if (rc == ZNONODE) {
        // 节点不存在，视为删除成功
        return true;
    } else {
        LOG_ERROR("ZK delete failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

// 判断节点是否存在
bool ZooKeeperClient::Exists(const std::string& path) {
    if(!IsConnected()){
        return false;
    }

    struct Stat stat;
    int rc=zoo_exists(zh_,path.c_str(),0,&stat);
    return rc == ZOK;
}

// 设置节点数据
bool ZooKeeperClient::SetData(const std::string& path, const std::string& data) {
    if(!IsConnected()){
        return false;
    }

    int rc = zoo_set(zh_, path.c_str(), data.c_str(), data.size(), -1);

    if (rc == ZOK) {
        LOG_DEBUG("ZK set data: path={}, size={}", path, data.size());
        return true;
    } else {
        LOG_ERROR("ZK set data failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}
```


### （3）节点操作（gRPC客户端相关）
- 主要涉及的API为zoo_get、zoo_wget、zoo_get_children、zoo_wget_children
- zoo_wget 和 zoo_wget_children 的核心作用就是**获取数据 + 注册 Watcher**，它们是**带 Watcher 版本的 API**。
	- **w 前缀的函数核心就是多了"注册 Watcher"这个功能**
	- **zoo_wget**：监听指定节点自身
	- **zoo_wget_children**：监听指定节点的子节点 ——> **服务发现的核心**
```cpp
// 获取数据（不带 Watch）
int zoo_get(
    zhandle_t* zh,
    const char* path,            // 节点路径
    int watch,                  // 0=不监听，1=使用全局Watcher
    char* buffer,               // 输出：数据缓冲区
    int* buffer_len,            // 输入输出：缓冲区大小/实际数据长度
    struct Stat* stat           // 输出：节点状态（可传NULL）
);

// 获取数据 + 注册自定义 Watcher
int zoo_wget(
    zhandle_t* zh,
    const char* path,
    watcher_fn watcher,         // 自定义回调
    void* watcherCtx,           // 回调上下文
    char* buffer,                // 输出：数据缓冲区
    int* buffer_len,            // 输入输出：缓冲区大小/实际数据长度
    struct Stat* stat             // 输出：节点状态（可传NULL）
);

// 获取子节点列表（不带 Watch）
int zoo_get_children(
    zhandle_t* zh,
    const char* path,
    int watch,
    struct String_vector* strings  // 输出：子节点名称列表
);

// 获取子节点 + 注册自定义 Watcher（★推荐）
int zoo_wget_children(
    zhandle_t* zh,
    const char* path,
    watcher_fn watcher,
    void* watcherCtx,
    struct String_vector* strings
);

```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260207222120354.png)

##### <1>不带 Watcher vs 带 Watcher
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                      不带 Watcher vs 带 Watcher                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  获取节点数据                                                        │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  zoo_get()          → 只获取数据                                    │    │
│  │  zoo_wget()         → 获取数据 + 注册 Watcher（监听数据变化）        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  获取子节点列表                                                      │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │  zoo_get_children() → 只获取子节点列表                               │    │
│  │  zoo_wget_children()→ 获取子节点列表 + 注册 Watcher（监听子节点增删） │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260207221934554.png)

#### <2>zoo_wget_children 是服务发现的核心
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                      服务注册与发现 API 使用                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  服务端（Provider）—— 服务注册                                        │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │                                                                     │    │
│  │  • zoo_exists()    → 检查父路径是否存在                              │    │
│  │  • zoo_create()    → 创建服务节点（临时节点）                         │    │
│  │  • zoo_delete()    → 注销服务                                       │    │
│  │  • zoo_set()       → 更新服务信息                                    │    │
│  │                                                                     │    │
│  │  ❌ 不需要 Watcher                                                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  客户端（Consumer）—— 服务发现                                        │    │
│  │  ─────────────────────────────────────────────────────────────────  │    │
│  │                                                                     │    │
│  │  • zoo_wget_children() → 获取实例列表 + 监听实例增删  ⭐ 核心        │    │
│  │  • zoo_get()           → 获取单个实例的地址信息                      │    │
│  │                                                                     │    │
│  │  ✅ 需要 Watcher（监听服务实例上线/下线）                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                      客户端服务发现流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   客户端                              ZooKeeper                              │
│      │                                   │                                   │
│      │  ① zoo_wget_children("/services/user", watcher, ...)                 │
│      │ ─────────────────────────────────▶│                                   │
│      │                                   │                                   │
│      │  ② 返回子节点列表 + 注册 Watcher   │                                   │
│      │  ["instance_001", "instance_002"] │                                   │
│      │◀───────────────────────────────── │                                   │
│      │                                   │                                   │
│      │  ③ zoo_get("/services/user/instance_001")                            │
│      │ ─────────────────────────────────▶│                                   │
│      │  返回 "192.168.1.10:50051"        │                                   │
│      │◀───────────────────────────────── │                                   │
│      │                                   │                                   │
│      │  ④ zoo_get("/services/user/instance_002")                            │
│      │ ─────────────────────────────────▶│                                   │
│      │  返回 "192.168.1.11:50051"        │                                   │
│      │◀───────────────────────────────── │                                   │
│      │                                   │                                   │
│      │         ... 运行中 ...             │                                   │
│      │                                   │                                   │
│      │  ⑤ instance_003 上线              │◀── 新服务注册                     │
│      │                                   │                                   │
│      │  ⑥ Watcher 触发 ZOO_CHILD_EVENT   │                                   │
│      │◀───────────────────────────────── │                                   │
│      │                                   │                                   │
│      │  ⑦ 重新 zoo_wget_children (重新注册 Watcher)                          │
│      │ ─────────────────────────────────▶│                                   │
│      │                                   │                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### <3>实际封装示例
```cpp
class ZooKeeperClient {
public:
    // 子节点变化回调类型
    using WatchCallback = std::function<void(const std::string& path)>;

  	......

  	    // ============================================================================
    // gRPC 客户端使用的接口（服务发现）
    // ============================================================================
    
    /**
     * @brief 获取节点数据
     * @param path 节点路径
     * @return 节点数据，失败返回空字符串
     * 
     * @note 客户端用于：读取服务实例信息（host、port、metadata）
     */
    std::string GetData(const std::string& path);
    
    /**
     * @brief 获取子节点列表
     * @param path 父节点路径
     * @return 子节点名称列表（不含路径前缀）
     * 
     * @note 客户端用于：获取所有服务实例节点名
     */
    std::vector<std::string> GetChildren(const std::string& path);
    
    /**
     * @brief 监听子节点变化
     * @param path 监听路径
     * @param callback 变化回调
     * 
     * @note ZK 的 watch 是一次性的，本方法会自动重新注册
     * @note 回调在 ZK 事件线程中执行，避免长时间阻塞
     * @note 客户端用于：监听服务实例上下线
     */
    void WatchChildren(const std::string& path, WatchCallback callback);
    
    /**
     * @brief 取消监听
     * @param path 监听路径
     * 
     * @note 客户端用于：停止监听某个服务
     */
    void UnwatchChildren(const std::string& path);

private:
	    /**
     * @brief 全局 Watcher 回调（静态，供 C API 调用）
     * @param type 事件类型：ZOO_SESSION_EVENT（会话状态变更）、ZOO_CHILD_EVENT（子节点列表变更）等
     * @param state 会话状态：连接、断开、过期等（用于：ZOO_SESSION_EVENT）
     * @param path 子节点路径（用于：ZOO_CHILD_EVENT）
     * @param context 用户上下文，传入ZooKeeperClient*，方便在内部调用HandleXXX函数
     * @note 必须是static函数，因为是供 C API 调用，故不可以用普通成员函数
     */
    static void GlobalWatcher(zhandle_t* zh, int type, int state,
                              const char* path, void* context);
    
	/**
     * @brief 处理子节点变化事件
     */
    void HandleChildEvent(const std::string& path);
    
    /**
     * @brief 重新注册指定路径的 watch
     */
    void ResetWatch(const std::string& path);

    // ZooKeeper 句柄
    zhandle_t* zh_ = nullptr;

    // Watch 回调管理（因为watch只能使用一次，所以会多次注册watch，故需要存储“子节点路径——>回调处理函数”）
    std::mutex watch_mutex_;
    std::map<std::string, WatchCallback> watches_;  // 回调map：子节点路径——>回调处理函数
};
```
```cpp
// ============================================================================
// gRPC 客户端使用的接口（服务发现）
// ============================================================================

// 获取节点数据
std::string ZooKeeperClient::GetData(const std::string& path){
    if(!IsConnected()){
        return "";
    }

    // 使用 vector 在堆上分配内存，避免栈溢出风险
    constexpr int kMaxDataSize = 65536;  // 64KB
    std::vector<char> buffer(kMaxDataSize);
    int buffer_len = static_cast<int>(buffer.size());
    struct Stat stat;

    // 获取指定实例节点的数据，如"ip:port"
    int rc = zoo_get(zh_, path.c_str(), 0, buffer.data(), &buffer_len, &stat);

    if (rc == ZOK && buffer_len > 0) {
        return std::string(buffer.data(), buffer_len);
    } else if (rc != ZOK) {
        LOG_WARN("ZK get data failed: path={}, error={}", path, zerror(rc));
    }

    return "";
}

// 获取指定目录下所有子节点
std::vector<std::string> ZooKeeperClient::GetChildren(const std::string& path) {
    std::vector<std::string> result;
    
    if (!IsConnected()) {
        return result;
    }
    
    struct String_vector children;
    int rc = zoo_get_children(zh_, path.c_str(), 0, &children);
    
    if (rc == ZOK) {
        result.reserve(children.count);
        for (int i = 0; i < children.count; ++i) {
            result.emplace_back(children.data[i]);
        }
        // ⚠️ 必须调用！否则内存泄漏
		deallocate_String_vector(&children);
    } else {
        LOG_WARN("ZK get children failed: path={}, error={}", path, zerror(rc));
    }
    
    return result;
}

// 监听子节点
void ZooKeeperClient::WatchChildren(const std::string& path, WatchCallback callback){
    if(!IsConnected()){
        LOG_ERROR("ZK not connected, cannot watch: {}", path);
        return;
    }

    // 保存回调
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        /* 存储客户端传入的回调函数
           - 每次触发watch后，都需要重新设置，也就需要再次使用“客户端传入的回调函数”
           - 故：需要存储 path ——> callback 的map映射，方便重复使用 
        */
        watches_[path] = std::move(callback);
    }

    // 设置 watch 并获取当前子节点
    struct String_vector children;
    int rc=zoo_wget_children(zh_, path.c_str(), GlobalWatcher, this, &children);

    if (rc == ZOK) {
        deallocate_String_vector(&children);
        LOG_DEBUG("ZK watch set: {}", path);
    } else {
        LOG_ERROR("ZK watch failed: path={}, error={}", path, zerror(rc));
    }
}

// 取消监听子节点
void ZooKeeperClient::UnwatchChildren(const std::string& path) {
    std::lock_guard<std::mutex> lock(watch_mutex_);
    /* watch是一次性的，触发后需要重新注册，即从watches_中通过path取出callback进行注册
        - 只要将对应的callback从watches_中移除，之后path对应的节点也就无法再注册watch
          也就达到了“取消监听子节点”的目的*/
    watches_.erase(path);
    LOG_DEBUG("ZK watch removed: {}", path);
}


// ============================================================================
// 内部实现（静态 Watcher 回调 + 事件处理）
// ============================================================================

// 全局 Watcher 回调（静态，供 C API 调用）
void ZooKeeperClient::GlobalWatcher(zhandle_t* zh, int type, int state,
                    const char* path, void* context){
    
    ZooKeeperClient* client=static_cast<ZooKeeperClient*>(context);
    if(!client) return;

    
    if(type==ZOO_SESSION_EVENT){
        // // 会话事件：连接、断开、过期等
        // client->HandleSessionEvent(state);
    }else if(type==ZOO_CHILD_EVENT && path){
        // 子节点变化事件
        client->HandleChildEvent(path);
    }
}

// 处理子节点列表变更
void ZooKeeperClient::HandleChildEvent(const std::string& path){
    WatchCallback callback;
    
    // 获取回调（加锁）
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        auto it=watches_.find(path);
        if (it != watches_.end()) {
            callback = it->second;
        }
    }

    // 触发回调（不持锁，避免死锁）
    if(callback){
        callback(path);
    }

    // 重新注册 watch（ZK 的 watch 是一次性的）
    ResetWatch(path);
}

// 重新注册指定路径的 watch
void ZooKeeperClient::ResetWatch(const std::string& path){
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        // 检查是否需要再监听
        if(watches_.find(path) == watches_.end()){
            return;
        }
    }

    struct String_vector children;

    int rc=zoo_wget_children(zh_, path.c_str(), GlobalWatcher, this, &children);

    if(rc==ZOK){
        deallocate_String_vector(&children);
    }else{
        LOG_WARN("Failed to reset watch for {}: {}", path, zerror(rc));
    }
}
```


# 四、zookeeper的“服务注册与发现”功能
## 0.概述
### （1）为什么需要“服务注册与发现”功能？
- 在分布式服务中，不同的服务可能分别运行在不同的服务器（ip:port）上，或同一服务器的不同端口上
- **需要解决的问题**：gRPC客户端如何知道想要调用的**RPC服务**运行在哪个服务器上呢?
	- gRPC客户端需要知道**RPC服务**所在的**ip：port**，进而连接对应的gRPC服务器，才能调用对应服务下的**RPC方法**
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                          分布式服务的核心问题                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   传统方式（硬编码地址）：                                                     │
│                                                                              │
│   ┌──────────────┐                      ┌──────────────┐                    │
│   │  gRPC 客户端  │ ───────────────────▶ │  gRPC 服务端  │                    │
│   │              │   192.168.1.100:50051 │  user-service │                    │
│   └──────────────┘   （写死在代码里）      └──────────────┘                    │
│                                                                              │
│   ❌ 问题：                                                                   │
│      • 服务地址变了怎么办？→ 改代码、重新部署                                   │
│      • 服务有多个实例怎么办？→ 客户端要维护地址列表                              │
│      • 某个实例挂了怎么办？→ 客户端不知道，继续调用失败的地址                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### （2）解决方法：“服务注册与发现”功能
- **使用一个中间服务器**（以zookeeper为例）
	- **gRPC服务端**：通过zookeeper客户端，连接zookeeper服务端后，通过创建节点（**服务名**为“节点名”，**节点数据**为服务所在的“ip：port”）来**注册服务**
	- **gRPC客户端**：通过zookeeper客户端，连接zookeeper服务端后，以**服务名**作为“节点名”，获取对应节点下的数据，即服务所在的**ip：port** ——> 连接该gRPC服务端

#### <1>分布式微服务架构
```cpp
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              分布式微服务架构                                          │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│  ┌──────────────────┐                                       ┌──────────────────┐     │
│  │  auth-service    │                                       │  user-service    │     │
│  │  (认证服务)       │                                       │  (用户服务)       │     │
│  │                  │                                       │                  │     │
│  │  • Login         │                                       │  • GetUser       │     │
│  │  • Logout        │                                       │  • CreateUser    │     │
│  │  • VerifyToken   │                                       │  • UpdateUser    │     │
│  │                  │                                       │                  │     │
│  │  192.168.1.50    │                                       │  192.168.1.100   │     │
│  │  :50050          │                                       │  :50051          │     │
│  └────────┬─────────┘                                       └────────┬─────────┘     │
│           │                                                          │               │
│           │ ① 注册服务                                     ① 注册服务 │               │
│           │ CreateNode(临时节点)                           CreateNode │               │
│           │                                                          │               │
│           ▼                                                          ▼               │
│  ┌────────────────────────────────────────────────────────────────────────────┐     │
│  │                                                                            │     │
│  │                           ZooKeeper (注册中心)                              │     │
│  │                           127.0.0.1:2181                                   │     │
│  │                                                                            │     │
│  │   节点结构:                                                                 │     │
│  │   /services                                                                │     │
│  │   ├── /auth-service                                                        │     │
│  │   │   └── /192.168.1.50:50050  ← 临时节点，数据: {"host":..,"port":..}     │     │
│  │   │                                                                        │     │
│  │   └── /user-service                                                        │     │
│  │       ├── /192.168.1.100:50051 ← 临时节点 (实例A)                          │     │
│  │       └── /192.168.1.101:50051 ← 临时节点 (实例B)                          │     │
│  │                                                                            │     │
│  └────────────────────────────────────────────────────────────────────────────┘     │
│                                       ▲                                              │
│                                       │ ② 发现服务                                   │
│                                       │ GetChildren() + GetData()                   │
│                                       │ WatchChildren() 监听变化                     │
│                                       │                                              │
│           ┌───────────────────────────┴───────────────────────────────┐             │
│           │                    API Gateway / gRPC 客户端               │             │
│           │                                                           │             │
│           │  ┌─────────────────────────────────────────────────────┐  │             │
│           │  │                   ZK 客户端                          │  │             │
│           │  │  zk_client->Connect("127.0.0.1:2181")               │  │             │
│           │  │                                                     │  │             │
│           │  │  // 查询 auth-service                               │  │             │
│           │  │  GetChildren("/services/auth-service")              │  │             │
│           │  │    → ["192.168.1.50:50050"]                         │  │             │
│           │  │                                                     │  │             │
│           │  │  // 查询 user-service                               │  │             │
│           │  │  GetChildren("/services/user-service")              │  │             │
│           │  │    → ["192.168.1.100:50051", "192.168.1.101:50051"] │  │             │
│           │  └─────────────────────────────────────────────────────┘  │             │
│           │                          │                                │             │
│           │                          ▼                                │             │
│           │  ┌─────────────────────────────────────────────────────┐  │             │
│           │  │                  本地缓存 + 负载均衡                   │  │             │
│           │  │                                                     │  │             │
│           │  │  cache["auth-service"] → [192.168.1.50:50050]       │  │             │
│           │  │  cache["user-service"] → [192.168.1.100:50051,      │  │             │
│           │  │                           192.168.1.101:50051]      │  │             │
│           │  │                                                     │  │             │
│           │  │  GetOneInstance("user-service")                     │  │             │
│           │  │    → 轮询/随机 选择一个实例                           │  │             │
│           │  └─────────────────────────────────────────────────────┘  │             │
│           └───────────────┬───────────────────────────┬───────────────┘             │
│                           │                           │                             │
│                           │ ③ gRPC调用                 │ ③ gRPC调用                  │
│                           │ Login()/VerifyToken()     │ GetUser()                   │
│                           ▼                           ▼                             │
│  ┌──────────────────┐                                       ┌──────────────────┐    │
│  │  auth-service    │                                       │  user-service    │    │
│  │  192.168.1.50    │                                       │  192.168.1.100   │    │
│  │  :50050          │                                       │  :50051          │    │
│  └──────────────────┘                                       └──────────────────┘    │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

#### <2>流程时序图
```cpp
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                完整调用流程                                           │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│   gRPC 服务端                    ZooKeeper                    gRPC 客户端             │
│       │                            │                              │                  │
│       │  ① 注册服务                 │                              │                  │
│       │  CreateNode(临时节点)       │                              │                  │
│       │ ──────────────────────────▶│                              │                  │
│       │                            │                              │                  │
│       │                            │  ② 发现服务                   │                  │
│       │                            │  GetChildren + GetData       │                  │
│       │                            │◀──────────────────────────── │                  │
│       │                            │                              │                  │
│       │                            │  返回实例列表                  │                  │
│       │                            │ ────────────────────────────▶│                  │
│       │                            │                              │                  │
│       │                            │  WatchChildren(监听变化)      │                  │
│       │                            │◀──────────────────────────── │                  │
│       │                            │                              │                  │
│       │  ③ gRPC 调用（直连，不经过 ZK）                             │                  │
│       │◀───────────────────────────────────────────────────────── │                  │
│       │                            │                              │                  │
│       │  返回响应                   │                              │                  │
│       │ ─────────────────────────────────────────────────────────▶│                  │
│       │                            │                              │                  │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘

箭头说明：
  • ① 服务端 ──▶ ZK：服务端主动注册
  • ② 客户端 ──▶ ZK：客户端主动查询（箭头从客户端指向ZK）
  • ③ 客户端 ──▶ 服务端：gRPC直连调用
```

### （3）Watch机制
- **Watch 的特点**:
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260207210113591.png)
- **Watch 事件类型**：
```cpp
// ZooKeeper Watch 事件类型
ZOO_CREATED_EVENT     // 节点创建
ZOO_DELETED_EVENT     // 节点删除
ZOO_CHANGED_EVENT     // 节点数据变化
ZOO_CHILD_EVENT       // 子节点列表变化 ← 服务发现主要用这个
ZOO_SESSION_EVENT     // 会话状态变化
```

#### <1>为什么需要 Watch？
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                          没有 Watch 的问题                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   场景：user-service 的实例B（192.168.1.101:50051）宕机了                     │
│                                                                              │
│   ┌──────────────┐                      ┌──────────────┐                    │
│   │  gRPC 客户端  │                      │  ZooKeeper   │                    │
│   │              │                      │              │                    │
│   │  本地缓存:    │                      │  实例B的临时  │                    │
│   │  [实例A,实例B]│  ← 缓存已过时！       │  节点已删除   │                    │
│   └──────────────┘                      └──────────────┘                    │
│          │                                                                   │
│          │ 调用实例B                                                         │
│          ▼                                                                   │
│   ┌──────────────┐                                                          │
│   │   ❌ 失败     │  ← 实例B已经不存在了                                      │
│   └──────────────┘                                                          │
│                                                                              │
│   ❌ 问题：客户端不知道实例变化，继续使用过时的缓存                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### <2>Watch 机制原理
```cpp
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              Watch 机制工作原理                                        │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│  ┌──────────────────┐                                       ┌──────────────────┐     │
│  │  user-service    │                                       │  user-service    │     │
│  │  实例A (正常)     │                                       │  实例B (宕机)     │     │
│  │  192.168.1.100   │                                       │  192.168.1.101   │     │
│  └────────┬─────────┘                                       └────────┬─────────┘     │
│           │                                                          │               │
│           │ 保持心跳                                         会话断开 │               │
│           │                                                  (宕机)   │               │
│           ▼                                                          ▼               │
│  ┌────────────────────────────────────────────────────────────────────────────┐     │
│  │                           ZooKeeper                                        │     │
│  │                                                                            │     │
│  │   /services/user-service                                                   │     │
│  │   ├── /192.168.1.100:50051  ← 正常                                        │     │
│  │   └── /192.168.1.101:50051  ← ❌ 临时节点自动删除（会话超时）               │     │
│  │                                                                            │     │
│  │   ┌─────────────────────────────────────────────────────────────────┐     │     │
│  │   │  Watch 触发：检测到 /services/user-service 子节点变化            │     │     │
│  │   │  事件类型：ZOO_CHILD_EVENT                                       │     │     │
│  │   └─────────────────────────────────────────────────────────────────┘     │     │
│  │                                                                            │     │
│  └────────────────────────────────────────────────────────────────────────────┘     │
│                                       │                                              │
│                                       │ ③ 主动推送通知                               │
│                                       │ "子节点发生变化"                              │
│                                       ▼                                              │
│           ┌───────────────────────────────────────────────────────────┐             │
│           │                      gRPC 客户端                           │             │
│           │                                                           │             │
│           │  ┌─────────────────────────────────────────────────────┐  │             │
│           │  │              Watch 回调被触发                        │  │             │
│           │  │                                                     │  │             │
│           │  │  void OnChildrenChanged(path) {                     │  │             │
│           │  │      // 1. 重新获取子节点列表                         │  │             │
│           │  │      auto children = GetChildren(path);             │  │             │
│           │  │                                                     │  │             │
│           │  │      // 2. 更新本地缓存                               │  │             │
│           │  │      cache["user-service"] = children;              │  │             │
│           │  │                                                     │  │             │
│           │  │      // 3. 重新设置 Watch（一次性）                    │  │             │
│           │  │      WatchChildren(path, callback);                 │  │             │
│           │  │  }                                                  │  │             │
│           │  └─────────────────────────────────────────────────────┘  │             │
│           │                          │                                │             │
│           │                          ▼                                │             │
│           │  ┌─────────────────────────────────────────────────────┐  │             │
│           │  │              缓存自动更新                             │  │             │
│           │  │                                                     │  │             │
│           │  │  更新前: [192.168.1.100:50051, 192.168.1.101:50051] │  │             │
│           │  │  更新后: [192.168.1.100:50051]  ← 实例B已移除        │  │             │
│           │  └─────────────────────────────────────────────────────┘  │             │
│           │                                                           │             │
│           └───────────────────────────────────────────────────────────┘             │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

#### <3>Watch 时序图
```cpp
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              Watch 机制时序图                                         │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│   gRPC 客户端                    ZooKeeper                   user-service 实例B       │
│       │                            │                              │                  │
│       │  ① 设置 Watch              │                              │                  │
│       │  WatchChildren(path, cb)   │                              │                  │
│       │ ──────────────────────────▶│                              │                  │
│       │                            │                              │                  │
│       │  返回当前子节点列表         │                              │                  │
│       │◀────────────────────────── │                              │                  │
│       │                            │                              │                  │
│       │                            │              ② 实例B宕机      │                  │
│       │                            │              会话断开         │                  │
│       │                            │◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │                  │
│       │                            │                              │                  │
│       │                            │  ③ 会话超时                   │                  │
│       │                            │  删除临时节点                  │                  │
│       │                            │  /192.168.1.101:50051        │                  │
│       │                            │                              │                  │
│       │  ④ Watch 回调触发          │                              │                  │
│       │  (ZOO_CHILD_EVENT)        │                              │                  │
│       │◀────────────────────────── │                              │                  │
│       │                            │                              │                  │
│       │  ⑤ 重新获取子节点          │                              │                  │
│       │  GetChildren(path)        │                              │                  │
│       │ ──────────────────────────▶│                              │                  │
│       │                            │                              │                  │
│       │  返回: [192.168.1.100:50051]                              │                  │
│       │◀────────────────────────── │                              │                  │
│       │                            │                              │                  │
│       │  ⑥ 更新本地缓存            │                              │                  │
│       │  ⑦ 重新设置 Watch          │                              │                  │
│       │ ──────────────────────────▶│                              │                  │
│       │                            │                              │                  │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

## 2.zookeeper客户端综合封装
- 无论是gRPC的服务端还是客户端，都需要使用zookeeper的客户端API进行“服务的注册”或“服务的获取”

### （1）zk_client.h
```cpp
// src/discovery/zk_client.h
#pragma once

#include <zookeeper/zookeeper.h>
#include <string>
#include <vector>
#include <map>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <memory>

namespace user_service {

/**
 * @brief ZooKeeper 客户端封装
 * 
 * 提供 ZooKeeper 基本操作的 C++ 封装，包括：
 * - 节点的 CRUD 操作
 * - 子节点监听（Watch）
 * - 自动重连处理
 * 
 * 使用示例：
 * @code
 * ZooKeeperClient zk("127.0.0.1:2181", 15000);
 * if (zk.Connect()) {
 *     zk.CreateNode("/test", "data", false);
 * }
 * @endcode
 */
class ZooKeeperClient {
public:
    // 子节点变化回调类型
    using WatchCallback = std::function<void(const std::string& path)>;
    
    /**
     * @brief 构造函数
     * @param hosts ZK 服务器地址，格式: "ip1:port1,ip2:port2"
     * @param session_timeout_ms 会话超时时间（毫秒），推荐 10000-30000
     */
    ZooKeeperClient(const std::string& hosts, int session_timeout_ms = 30000);
    
    /**
     * @brief 析构函数，自动关闭连接
     */
    ~ZooKeeperClient();
    
    // 禁止拷贝
    ZooKeeperClient(const ZooKeeperClient&) = delete;
    ZooKeeperClient& operator=(const ZooKeeperClient&) = delete;
    
    // ============================================================================
    // 通用接口（gRPC 服务端 + gRPC 客户端 都使用）
    // ============================================================================
    
    /**
     * @brief 连接到 ZooKeeper
     * @param timeout_ms 连接超时时间（毫秒）
     * @return 连接成功返回 true
     */
    bool Connect(int timeout_ms = 10000);
    
    /**
     * @brief 关闭连接
     */
    void Close();
    
    /**
     * @brief 是否已连接
     */
    bool IsConnected() const;
    
    // ============================================================================
    // gRPC 服务端使用的接口（服务注册）
    // ============================================================================
    
    /**
     * @brief 创建单个节点（父节点必须存在）
     * @param path 节点路径
     * @param data 节点数据
     * @param ephemeral 是否为临时节点（会话断开自动删除）
     * @return 创建成功或节点已存在返回 true
     * 
     * @note 服务端用于：注册服务实例（创建临时节点）
     */
    bool CreateNode(const std::string& path, const std::string& data, 
                    bool ephemeral = false);
    
    /**
     * @brief 递归创建整个路径（自动创建所有父节点）
     * @param path 服务根路径
     * @return 创建成功返回 true
     * 
     * @note 服务端用于：创建服务根路径 /services/user-service ——> 永久性节点
     * @note 服务注册的典型用法：
        void RegisterService(const std::string& service_name, 
                            const std::string& host, int port) {
            
            // 1. 先用 CreatePath 确保服务路径存在（持久节点）
            std::string service_path = "/services/" + service_name;
            zk_client->CreatePath(service_path);  // 递归创建 /services/user-service
            
            // 2. 再用 CreateNode 创建实例节点（临时节点）
            std::string instance_path = service_path + "/" + host + ":" + std::to_string(port);
            std::string data = R"({"host":")" + host + R"(","port":)" + std::to_string(port) + "}";
            zk_client->CreateNode(instance_path, data, true);  // ephemeral=true
        }

        ┌─────────────────────────────────────────────────────────────────────────────┐
        │  /services/user-service/192.168.1.100:50051                                 │
        │  ├────────┬────────────┬─────────────────────┤                              │
        │  │  持久  │    持久    │        临时          │                              │
        │  └────────┴────────────┴─────────────────────┘                              │
        │                                                                              │
        │  • /services          → 持久：服务根目录，永远存在                            │
        │  • /user-service      → 持久：服务名，永远存在                                │
        │  • /192.168.1.100:50051 → 临时：实例节点，服务下线自动删除                     │
        │                                                                              │
        │  好处：服务全部下线后，/services/user-service 还在，                           │
        │       客户端可以继续 Watch，等待新实例上线                                     │
        └─────────────────────────────────────────────────────────────────────────────┘
     */
    bool CreateParentPath(const std::string& path);
    
    /**
     * @brief 删除节点
     * @param path 节点路径
     * @return 删除成功或节点不存在返回 true
     * 
     * @note 服务端用于：主动注销服务实例
     */
    bool DeleteNode(const std::string& path);
    
    /**
     * @brief 检查节点是否存在
     * 
     * @note 服务端用于：检查路径是否已创建
     */
    bool Exists(const std::string& path);
    
    /**
     * @brief 设置节点数据
     * @param path 节点路径
     * @param data 新数据
     * @return 设置成功返回 true
     * 
     * @note 服务端用于：更新服务实例的元信息（如权重、状态）
     */
    bool SetData(const std::string& path, const std::string& data);
    
    // ============================================================================
    // gRPC 客户端使用的接口（服务发现）
    // ============================================================================
    
    /**
     * @brief 获取节点数据
     * @param path 节点路径
     * @return 节点数据，失败返回空字符串
     * 
     * @note 客户端用于：读取服务实例信息（host、port、metadata）
     */
    std::string GetData(const std::string& path);
    
    /**
     * @brief 获取子节点列表
     * @param path 父节点路径
     * @return 子节点名称列表（不含路径前缀）
     * 
     * @note 客户端用于：获取所有服务实例节点名
     */
    std::vector<std::string> GetChildren(const std::string& path);
    
    /**
     * @brief 监听子节点变化
     * @param path 监听路径
     * @param callback 变化回调
     * 
     * @note ZK 的 watch 是一次性的，本方法会自动重新注册
     * @note 回调在 ZK 事件线程中执行，避免长时间阻塞
     * @note 客户端用于：监听服务实例上下线
     */
    void WatchChildren(const std::string& path, WatchCallback callback);
    
    /**
     * @brief 取消监听
     * @param path 监听路径
     * 
     * @note 客户端用于：停止监听某个服务
     */
    void UnwatchChildren(const std::string& path);
    
private:
    /**
     * @brief 全局 Watcher 回调（静态，供 C API 调用）
     * @param type 事件类型：ZOO_SESSION_EVENT（会话状态变更）、ZOO_CHILD_EVENT（子节点列表变更）等
     * @param state 会话状态：连接、断开、过期等（用于：ZOO_SESSION_EVENT）
     * @param path 子节点路径（用于：ZOO_CHILD_EVENT）
     * @param context 用户上下文，传入ZooKeeperClient*，方便在内部调用HandleXXX函数
     * @note 必须是static函数，因为是供 C API 调用，故不可以用普通成员函数
     */
    static void GlobalWatcher(zhandle_t* zh, int type, int state,
                              const char* path, void* context);
    
    /**
     * @brief 处理会话状态变化
     */
    void HandleSessionEvent(int state);
    
    /**
     * @brief 处理子节点变化事件
     */
    void HandleChildEvent(const std::string& path);
    
    /**
     * @brief 重新注册指定路径的 watch
     */
    void ResetWatch(const std::string& path);

    // ZooKeeper 句柄
    zhandle_t* zh_ = nullptr;
    
    // 配置
    std::string hosts_;
    int session_timeout_ms_;
    
    // 连接状态（原子变量，线程安全）
    std::atomic<bool> connected_{false};
    
    // 连接等待
    std::mutex conn_mutex_;
    std::condition_variable conn_cv_;
    
    // Watch 回调管理（因为watch只能使用一次，所以会多次注册watch，故需要存储“子节点路径——>回调处理函数”）
    std::mutex watch_mutex_;
    std::map<std::string, WatchCallback> watches_;  // 回调map：子节点路径——>回调处理函数
};

} // namespace user_service
```

### （2）zk_client.cpp
```cpp
// src/discovery/zk_client.cpp
#include "zk_client.h"
#include "common/logger.h"
#include <sstream>
#include <cstring>
#include <zookeeper/zookeeper.h>

namespace user_service{

// ============================================================================
// 构造与析构
// ============================================================================

ZooKeeperClient::ZooKeeperClient(const std::string& hosts, int session_timeout_ms)
    : hosts_(hosts)
    , session_timeout_ms_(session_timeout_ms) {
    
    // 设置 ZK 日志级别（可选）
    zoo_set_debug_level(ZOO_LOG_LEVEL_WARN);
}

ZooKeeperClient::~ZooKeeperClient() {
    Close();
}

// ============================================================================
// 通用接口（gRPC 服务端 + gRPC 客户端 都使用）
// ============================================================================

bool ZooKeeperClient::Connect(int timeout_ms){
    std::unique_lock<std::mutex> lock(conn_mutex_);

    // 已连接
    if(zh_ && connected_.load()){
        return true;
    }

    // 关闭旧连接
    if(zh_){
        zookeeper_close(zh_);
        zh_=nullptr;
    }

    connected_=false;

    // 初始化 ZK 句柄
    zh_=zookeeper_init(hosts_.c_str(),GlobalWatcher,session_timeout_ms_,nullptr,this,0);

    if (!zh_) {
        LOG_ERROR("zookeeper_init failed, hosts={}", hosts_);
        return false;
    }

    // 等待连接建立（使用条件变量）——> 连接建立成功时，会在”回调函数“中将connected_设置为true
    bool success = conn_cv_.wait_for(lock,
                std::chrono::milliseconds(timeout_ms),
                [this]{return connected_.load();});

    if(!success){
        LOG_ERROR("ZooKeeper connection timeout, hosts={}", hosts_);
        zookeeper_close(zh_);
        zh_ = nullptr;
        return false; 
    }

    LOG_INFO("ZooKeeper client connected, hosts={}", hosts_);
    return true;
}

void ZooKeeperClient::Close(){
    if(zh_){
        zookeeper_close(zh_);
        zh_=nullptr;
        connected_=false;
        LOG_INFO("ZooKeeper connection closed");
    }

    // 清理watches
    std::lock_guard<std::mutex> lock(watch_mutex_);
    watches_.clear();
}

bool ZooKeeperClient::IsConnected() const {
    return zh_ && connected_.load() && zoo_state(zh_) == ZOO_CONNECTED_STATE;
}

// ============================================================================
// gRPC 服务端使用的接口（服务注册）
// ============================================================================

// 创建节点
bool ZooKeeperClient::CreateNode(const std::string& path, 
                                const std::string& data,
                                bool ephemeral){
    if(!IsConnected()){
        LOG_ERROR("ZK not connected, cannot create node: {}", path);
        return false;
    }

    // 节点类型（临时节点/顺序节点/临时顺序节点）
    int flags=ephemeral ? ZOO_EPHEMERAL:0;

    // 创建节点
    int rc=zoo_create(zh_,path.c_str(),data.c_str(),data.size(),
                        &ZOO_OPEN_ACL_UNSAFE,flags,nullptr,0);
    
    if(rc==ZOK){    // 节点创建成功
        LOG_DEBUG("ZK node created: {} (ephemeral={})", path, ephemeral);
        return true;
    }else if(rc==ZNODEEXISTS){
        // 节点已存在，对于持久节点视为成功
        // 对于临时节点，可能是上次会话残留，需要特殊处理
        if(ephemeral){
            LOG_WARN("ZK ephemeral node already exists: {}, attempting recovery...", path);
            
            // 尝试删除旧节点后重新创建
            if(DeleteNode(path)){
                rc=zoo_create(zh_,path.c_str(),data.c_str(),data.size(),
                              &ZOO_OPEN_ACL_UNSAFE,flags,nullptr,0);
                if(rc==ZOK){
                    LOG_INFO("ZK ephemeral node recreated: {}", path);
                    return true;
                }
            }
            
            // 如果删除重建失败，尝试直接更新数据
            if(SetData(path,data)){
                LOG_INFO("ZK ephemeral node data updated: {}", path);
                return true;
            }
            
            LOG_ERROR("Failed to recover ephemeral node: {}", path);
            return false;
        }
        return true;
    }else{
        LOG_ERROR("ZK create failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

bool ZooKeeperClient::CreateParentPath(const std::string& path){
    if(!IsConnected()){
        return false;
    }

    // 使用stringstream 分割路径
    std::string current;
    std::istringstream ss(path);
    std::string token;

    /*
    输入：/services/user
    循环过程大概是：
        - token=services → current=/services → 不存在就创建
        - token=user → current=/services/user → 不存在就创建
    最终路径保证存在。
    */
    while(std::getline(ss,token,'/')){  
        if(token.empty()) continue;

        current += "/" + token;

        if(!Exists(current)){   // 节点不存在
            // 创建持久节点(节点值为空)
            if(!CreateNode(current,"",false)){
                return false;
            }
        }
    }

    return true;
}

// 删除节点
bool ZooKeeperClient::DeleteNode(const std::string& path){
    if(!IsConnected()){
        return false;
    }

    int rc = zoo_delete(zh_,path.c_str(),-1);    // -1 表示忽略版本

    if (rc == ZOK) {
        LOG_DEBUG("ZK node deleted: {}", path);
        return true;
    } else if (rc == ZNONODE) {
        // 节点不存在，视为删除成功
        return true;
    } else {
        LOG_ERROR("ZK delete failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

// 判断节点是否存在
bool ZooKeeperClient::Exists(const std::string& path) {
    if(!IsConnected()){
        return false;
    }

    struct Stat stat;
    int rc=zoo_exists(zh_,path.c_str(),0,&stat);
    return rc == ZOK;
}

// 设置节点数据
bool ZooKeeperClient::SetData(const std::string& path, const std::string& data) {
    if(!IsConnected()){
        return false;
    }

    int rc = zoo_set(zh_, path.c_str(), data.c_str(), data.size(), -1);

    if (rc == ZOK) {
        LOG_DEBUG("ZK set data: path={}, size={}", path, data.size());
        return true;
    } else {
        LOG_ERROR("ZK set data failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

// ============================================================================
// gRPC 客户端使用的接口（服务发现）
// ============================================================================

// 获取节点数据
std::string ZooKeeperClient::GetData(const std::string& path){
    if(!IsConnected()){
        return "";
    }

    // 使用 vector 在堆上分配内存，避免栈溢出风险
    constexpr int kMaxDataSize = 65536;  // 64KB
    std::vector<char> buffer(kMaxDataSize);
    int buffer_len = static_cast<int>(buffer.size());
    struct Stat stat;

    // 获取指定实例节点的数据，如"ip:port"
    int rc = zoo_get(zh_, path.c_str(), 0, buffer.data(), &buffer_len, &stat);

    if (rc == ZOK && buffer_len > 0) {
        return std::string(buffer.data(), buffer_len);
    } else if (rc != ZOK) {
        LOG_WARN("ZK get data failed: path={}, error={}", path, zerror(rc));
    }

    return "";
}

// 获取指定目录下所有子节点
std::vector<std::string> ZooKeeperClient::GetChildren(const std::string& path) {
    std::vector<std::string> result;
    
    if (!IsConnected()) {
        return result;
    }
    
    struct String_vector children;
    int rc = zoo_get_children(zh_, path.c_str(), 0, &children);
    
    if (rc == ZOK) {
        result.reserve(children.count);
        for (int i = 0; i < children.count; ++i) {
            result.emplace_back(children.data[i]);
        }
        deallocate_String_vector(&children);
    } else {
        LOG_WARN("ZK get children failed: path={}, error={}", path, zerror(rc));
    }
    
    return result;
}

// 监听子节点
void ZooKeeperClient::WatchChildren(const std::string& path, WatchCallback callback){
    if(!IsConnected()){
        LOG_ERROR("ZK not connected, cannot watch: {}", path);
        return;
    }

    // 保存回调
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        /* 存储客户端传入的回调函数
           - 每次触发watch后，都需要重新设置，也就需要再次使用“客户端传入的回调函数”
           - 故：需要存储 path ——> callback 的map映射，方便重复使用 
        */
        watches_[path] = std::move(callback);
    }

    // 设置 watch 并获取当前子节点
    struct String_vector children;
    int rc=zoo_wget_children(zh_, path.c_str(), GlobalWatcher, this, &children);

    if (rc == ZOK) {
        deallocate_String_vector(&children);
        LOG_DEBUG("ZK watch set: {}", path);
    } else {
        LOG_ERROR("ZK watch failed: path={}, error={}", path, zerror(rc));
    }
}

// 取消监听子节点
void ZooKeeperClient::UnwatchChildren(const std::string& path) {
    std::lock_guard<std::mutex> lock(watch_mutex_);
    /* watch是一次性的，触发后需要重新注册，即从watches_中通过path取出callback进行注册
        - 只要将对应的callback从watches_中移除，之后path对应的节点也就无法再注册watch
          也就达到了“取消监听子节点”的目的*/
    watches_.erase(path);
    LOG_DEBUG("ZK watch removed: {}", path);
}

// ============================================================================
// 内部实现（静态 Watcher 回调 + 事件处理）
// ============================================================================

// 全局 Watcher 回调（静态，供 C API 调用）
void ZooKeeperClient::GlobalWatcher(zhandle_t* zh, int type, int state,
                    const char* path, void* context){
    
    ZooKeeperClient* client=static_cast<ZooKeeperClient*>(context);
    if(!client) return;

    
    if(type==ZOO_SESSION_EVENT){
        // 会话事件：连接、断开、过期等
        client->HandleSessionEvent(state);
    }else if(type==ZOO_CHILD_EVENT && path){
        // 子节点变化事件
        client->HandleChildEvent(path);
    }
}

// 处理会话状态变化
void ZooKeeperClient::HandleSessionEvent(int state) {
    if (state == ZOO_CONNECTED_STATE) { 
        // 连接成功
        connected_ = true;
        LOG_INFO("ZooKeeper connected");
        conn_cv_.notify_all();  // 通知等待连接的线程
    } else if (state == ZOO_EXPIRED_SESSION_STATE) {
        // 会话过期
        connected_ = false;
        LOG_WARN("ZooKeeper session expired");
    } else if (state == ZOO_CONNECTING_STATE) {
        // 正在连接
        connected_ = false;
        LOG_INFO("ZooKeeper reconnecting...");
    } else if (state == ZOO_ASSOCIATING_STATE) {
        // 关联中
        LOG_DEBUG("ZooKeeper associating...");
    } else {
        LOG_WARN("ZooKeeper unknown state: {}", state);
    }
}

// 处理子节点列表变更
void ZooKeeperClient::HandleChildEvent(const std::string& path){
    WatchCallback callback;
    
    // 获取回调（加锁）
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        auto it=watches_.find(path);
        if (it != watches_.end()) {
            callback = it->second;
        }
    }

    // 触发回调（不持锁，避免死锁）
    if(callback){
        callback(path);
    }

    // 重新注册 watch（ZK 的 watch 是一次性的）
    ResetWatch(path);
}

// 重新注册指定路径的 watch
void ZooKeeperClient::ResetWatch(const std::string& path){
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        // 检查是否需要再监听
        if(watches_.find(path) == watches_.end()){
            return;
        }
    }

    struct String_vector children;

    int rc=zoo_wget_children(zh_, path.c_str(), GlobalWatcher, this, &children);

    if(rc==ZOK){
        deallocate_String_vector(&children);
    }else{
        LOG_WARN("Failed to reset watch for {}: {}", path, zerror(rc));
    }
}

}   // namespace user_service
```

# 五、zookeeper客户端封装
## 1.zk_client.h
```cpp
// src/discovery/zk_client.h
#pragma once

#include <zookeeper/zookeeper.h>
#include <string>
#include <vector>
#include <map>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <memory>

namespace user_service {

/**
 * @brief ZooKeeper 客户端封装
 * 
 * 提供 ZooKeeper 基本操作的 C++ 封装，包括：
 * - 节点的 CRUD 操作
 * - 子节点监听（Watch）
 * - 自动重连处理
 * 
 * 使用示例：
 * @code
 * ZooKeeperClient zk("127.0.0.1:2181", 15000);
 * if (zk.Connect()) {
 *     zk.CreateNode("/test", "data", false);
 * }
 * @endcode
 */
class ZooKeeperClient {
public:
    // 子节点变化回调类型
    using WatchCallback = std::function<void(const std::string& path)>;
    
    /**
     * @brief 构造函数
     * @param hosts ZK 服务器地址，格式: "ip1:port1,ip2:port2"
     * @param session_timeout_ms 会话超时时间（毫秒），推荐 10000-30000
     */
    ZooKeeperClient(const std::string& hosts, int session_timeout_ms = 30000);
    
    /**
     * @brief 析构函数，自动关闭连接
     */
    ~ZooKeeperClient();
    
    // 禁止拷贝
    ZooKeeperClient(const ZooKeeperClient&) = delete;
    ZooKeeperClient& operator=(const ZooKeeperClient&) = delete;
    
    // ============================================================================
    // 通用接口（gRPC 服务端 + gRPC 客户端 都使用）
    // ============================================================================
    
    /**
     * @brief 连接到 ZooKeeper
     * @param timeout_ms 连接超时时间（毫秒）
     * @return 连接成功返回 true
     */
    bool Connect(int timeout_ms = 10000);
    
    /**
     * @brief 关闭连接
     */
    void Close();
    
    /**
     * @brief 是否已连接
     */
    bool IsConnected() const;
    
    // ============================================================================
    // gRPC 服务端使用的接口（服务注册）
    // ============================================================================
    
    /**
     * @brief 创建单个节点（f服务路径<父节点>必须存在）
     * @param path 节点路径
     * @param data 节点数据
     * @param ephemeral 是否为临时节点（会话断开自动删除）
     * @return 创建成功或节点已存在返回 true
     * 
     * @note 服务端用于：注册服务实例（创建临时节点）
     */
    bool CreateNode(const std::string& path, const std::string& data, 
                    bool ephemeral = false);
    
    /**
     * @brief 递归创建整个路径（自动创建所有父节点）
     * @param path 服务根路径
     * @return 创建成功返回 true
     * 
     * @note 服务端用于：创建服务根路径 /services/user-service ——> 永久性节点
     * @note 服务注册的典型用法：
        void RegisterService(const std::string& service_name, 
                            const std::string& host, int port) {
            
            // 1. 先用 CreatePath 确保服务路径存在（持久节点）
            std::string service_path = "/services/" + service_name;
            zk_client->CreatePath(service_path);  // 递归创建 /services/user-service
            
            // 2. 再用 CreateNode 创建实例节点（临时节点）
            std::string instance_path = service_path + "/" + host + ":" + std::to_string(port);
            std::string data = R"({"host":")" + host + R"(","port":)" + std::to_string(port) + "}";
            zk_client->CreateNode(instance_path, data, true);  // ephemeral=true
        }

        ┌─────────────────────────────────────────────────────────────────────────────┐
        │  /services/user-service/192.168.1.100:50051                                 │
        │  ├────────┬────────────┬─────────────────────┤                              │
        │  │  持久  │    持久    │        临时          │                              │
        │  └────────┴────────────┴─────────────────────┘                              │
        │                                                                              │
        │  • /services          → 持久：服务根目录，永远存在                            │
        │  • /user-service      → 持久：服务名，永远存在                                │
        │  • /192.168.1.100:50051 → 临时：实例节点，服务下线自动删除                     │
        │                                                                              │
        │  好处：服务全部下线后，/services/user-service 还在，                           │
        │       客户端可以继续 Watch，等待新实例上线                                     │
        └─────────────────────────────────────────────────────────────────────────────┘
     */
    bool CreateServicePath(const std::string& path);
    
    /**
     * @brief 删除节点
     * @param path 节点路径
     * @return 删除成功或节点不存在返回 true
     * 
     * @note 服务端用于：主动注销服务实例
     */
    bool DeleteNode(const std::string& path);
    
    /**
     * @brief 检查节点是否存在
     * 
     * @note 服务端用于：检查路径是否已创建
     */
    bool Exists(const std::string& path);
    
    /**
     * @brief 设置节点数据，用于更新已有节点数据
     * @param path 节点路径
     * @param data 新数据
     * @return 设置成功返回 true
     * 
     * @note 服务端用于：更新服务实例的元信息（如权重、状态）
     */
    bool SetData(const std::string& path, const std::string& data);
    
    // ============================================================================
    // gRPC 客户端使用的接口（服务发现）
    // ============================================================================
    
    /**
     * @brief 获取节点数据
     * @param path 节点路径
     * @return 节点数据，失败返回空字符串
     * 
     * @note 客户端用于：读取服务实例信息（host、port、metadata）
     */
    std::string GetData(const std::string& path);
    
    /**
     * @brief 获取子节点列表
     * @param path 父节点路径
     * @return 子节点名称列表（不含路径前缀）
     * 
     * @note 客户端用于：获取所有服务实例节点名
     */
    std::vector<std::string> GetChildren(const std::string& path);
    
    /**
     * @brief 监听子节点变化
     * @param path 监听路径
     * @param callback 变化回调
     * 
     * @note ZK 的 watch 是一次性的，本方法会自动重新注册
     * @note 回调在 ZK 事件线程中执行，避免长时间阻塞
     * @note 客户端用于：监听服务实例上下线
     */
    void WatchChildren(const std::string& path, WatchCallback callback);
    
    /**
     * @brief 取消监听
     * @param path 监听路径
     * 
     * @note 客户端用于：停止监听某个服务
     */
    void UnwatchChildren(const std::string& path);
    
private:
    /**
     * @brief 全局 Watcher 回调（静态，供 C API 调用）
     * @param type 事件类型：ZOO_SESSION_EVENT（会话状态变更）、ZOO_CHILD_EVENT（子节点列表变更）等
     * @param state 会话状态：连接、断开、过期等（用于：ZOO_SESSION_EVENT）
     * @param path 子节点路径（用于：ZOO_CHILD_EVENT）
     * @param context 用户上下文，传入ZooKeeperClient*，方便在内部调用HandleXXX函数
     * @note 必须是static函数，因为是供 C API 调用，故不可以用普通成员函数
     */
    static void GlobalWatcher(zhandle_t* zh, int type, int state,
                              const char* path, void* context);
    
    /**
     * @brief 处理会话状态变化
     */
    void HandleSessionEvent(int state);
    
    /**
     * @brief 处理子节点变化事件
     */
    void HandleChildEvent(const std::string& path);
    
    /**
     * @brief 重新注册指定路径的 watch
     */
    void ResetWatch(const std::string& path);

    // ZooKeeper 句柄
    zhandle_t* zh_ = nullptr;
    
    // 配置
    std::string hosts_;
    int session_timeout_ms_;
    
    // 连接状态（原子变量，线程安全）
    std::atomic<bool> connected_{false};
    
    // 连接等待
    std::mutex conn_mutex_;
    std::condition_variable conn_cv_;
    
    // Watch 回调管理（因为watch只能使用一次，所以会多次注册watch，故需要存储“子节点路径——>回调处理函数”）
    std::mutex watch_mutex_;
    std::map<std::string, WatchCallback> watches_;  // 回调map：子节点路径——>回调处理函数（客户端设置）
};

} // namespace user_service
```

## 2.zk_client.cpp
```cpp
// src/discovery/zk_client.cpp
#include "zk_client.h"
#include "common/logger.h"
#include <sstream>
#include <cstring>
#include <zookeeper/zookeeper.h>

namespace user_service{

// ============================================================================
// 构造与析构
// ============================================================================

ZooKeeperClient::ZooKeeperClient(const std::string& hosts, int session_timeout_ms)
    : hosts_(hosts)
    , session_timeout_ms_(session_timeout_ms) {
    
    // 设置 ZK 日志级别（可选）
    zoo_set_debug_level(ZOO_LOG_LEVEL_WARN);
}

ZooKeeperClient::~ZooKeeperClient() {
    Close();
}

// ============================================================================
// 通用接口（gRPC 服务端 + gRPC 客户端 都使用）
// ============================================================================

bool ZooKeeperClient::Connect(int timeout_ms){
    std::unique_lock<std::mutex> lock(conn_mutex_);

    // 已连接
    if(zh_ && connected_.load()){
        return true;
    }

    // 关闭旧连接
    if(zh_){
        zookeeper_close(zh_);
        zh_=nullptr;
    }

    connected_=false;

    // 初始化 ZK 句柄
    zh_=zookeeper_init(hosts_.c_str(),GlobalWatcher,session_timeout_ms_,nullptr,this,0);

    if (!zh_) {
        LOG_ERROR("zookeeper_init failed, hosts={}", hosts_);
        return false;
    }

    // 等待连接建立（使用条件变量）——> 连接建立成功时，会在”回调函数“中将connected_设置为true
    bool success = conn_cv_.wait_for(lock,
                std::chrono::milliseconds(timeout_ms),
                [this]{return connected_.load();});

    if(!success){
        LOG_ERROR("ZooKeeper connection timeout, hosts={}", hosts_);
        zookeeper_close(zh_);
        zh_ = nullptr;
        return false; 
    }

    LOG_INFO("ZooKeeper client connected, hosts={}", hosts_);
    return true;
}

void ZooKeeperClient::Close(){
    if(zh_){
        zookeeper_close(zh_);
        zh_=nullptr;
        connected_=false;
        LOG_INFO("ZooKeeper connection closed");
    }

    // 清理watches
    std::lock_guard<std::mutex> lock(watch_mutex_);
    watches_.clear();
}

bool ZooKeeperClient::IsConnected() const {
    return zh_ && connected_.load() && zoo_state(zh_) == ZOO_CONNECTED_STATE;
}

// ============================================================================
// gRPC 服务端使用的接口（服务注册）
// ============================================================================

// 创建节点
bool ZooKeeperClient::CreateNode(const std::string& path, 
                                const std::string& data,
                                bool ephemeral){
    if(!IsConnected()){
        LOG_ERROR("ZK not connected, cannot create node: {}", path);
        return false;
    }

    // 节点类型（临时节点/顺序节点/临时顺序节点）
    int flags=ephemeral ? ZOO_EPHEMERAL:0;

    // 创建节点
    int rc=zoo_create(zh_,path.c_str(),data.c_str(),data.size(),
                        &ZOO_OPEN_ACL_UNSAFE,flags,nullptr,0);
    
    if(rc==ZOK){    // 节点创建成功
        LOG_DEBUG("ZK node created: {} (ephemeral={})", path, ephemeral);
        return true;
    }else if(rc==ZNODEEXISTS){
        // 节点已存在，对于持久节点视为成功
        // 对于临时节点，可能是上次会话残留，需要特殊处理
        if(ephemeral){
            LOG_WARN("ZK ephemeral node already exists: {}, attempting recovery...", path);
            
            // 尝试删除旧节点后重新创建
            if(DeleteNode(path)){
                rc=zoo_create(zh_,path.c_str(),data.c_str(),data.size(),
                              &ZOO_OPEN_ACL_UNSAFE,flags,nullptr,0);
                if(rc==ZOK){
                    LOG_INFO("ZK ephemeral node recreated: {}", path);
                    return true;
                }
            }
            
            // 如果删除重建失败，尝试直接更新数据
            if(SetData(path,data)){
                LOG_INFO("ZK ephemeral node data updated: {}", path);
                return true;
            }
            
            LOG_ERROR("Failed to recover ephemeral node: {}", path);
            return false;
        }
        return true;
    }else{
        LOG_ERROR("ZK create failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

bool ZooKeeperClient::CreateServicePath(const std::string& path){
    if(!IsConnected()){
        return false;
    }

    // 使用stringstream 分割路径
    std::string current;
    std::istringstream ss(path);
    std::string token;

    /*
    输入：/services/user
    循环过程大概是：
        - token=services → current=/services → 不存在就创建
        - token=user → current=/services/user → 不存在就创建
    最终路径保证存在。
    */
    while(std::getline(ss,token,'/')){  
        if(token.empty()) continue;

        current += "/" + token;

        if(!Exists(current)){   // 节点不存在
            // 创建持久节点(节点值为空)
            if(!CreateNode(current,"",false)){
                return false;
            }
        }
    }

    return true;
}

// 删除节点
bool ZooKeeperClient::DeleteNode(const std::string& path){
    if(!IsConnected()){
        return false;
    }

    int rc = zoo_delete(zh_,path.c_str(),-1);    // -1 表示忽略版本

    if (rc == ZOK) {
        LOG_DEBUG("ZK node deleted: {}", path);
        return true;
    } else if (rc == ZNONODE) {
        // 节点不存在，视为删除成功
        return true;
    } else {
        LOG_ERROR("ZK delete failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

// 判断节点是否存在
bool ZooKeeperClient::Exists(const std::string& path) {
    if(!IsConnected()){
        return false;
    }

    struct Stat stat;
    int rc=zoo_exists(zh_,path.c_str(),0,&stat);
    return rc == ZOK;
}

// 设置节点数据
bool ZooKeeperClient::SetData(const std::string& path, const std::string& data) {
    if(!IsConnected()){
        return false;
    }

    int rc = zoo_set(zh_, path.c_str(), data.c_str(), data.size(), -1);

    if (rc == ZOK) {
        LOG_DEBUG("ZK set data: path={}, size={}", path, data.size());
        return true;
    } else {
        LOG_ERROR("ZK set data failed: path={}, error={}", path, zerror(rc));
        return false;
    }
}

// ============================================================================
// gRPC 客户端使用的接口（服务发现）
// ============================================================================

// 获取节点数据
std::string ZooKeeperClient::GetData(const std::string& path){
    if(!IsConnected()){
        return "";
    }

    // 使用 vector 在堆上分配内存，避免栈溢出风险
    constexpr int kMaxDataSize = 65536;  // 64KB
    std::vector<char> buffer(kMaxDataSize);
    int buffer_len = static_cast<int>(buffer.size());
    struct Stat stat;

    // 获取指定实例节点的数据，如"ip:port"
    int rc = zoo_get(zh_, path.c_str(), 0, buffer.data(), &buffer_len, &stat);

    if (rc == ZOK && buffer_len > 0) {
        return std::string(buffer.data(), buffer_len);
    } else if (rc != ZOK) {
        LOG_WARN("ZK get data failed: path={}, error={}", path, zerror(rc));
    }

    return "";
}

// 获取指定目录下所有子节点
std::vector<std::string> ZooKeeperClient::GetChildren(const std::string& path) {
    std::vector<std::string> result;
    
    if (!IsConnected()) {
        return result;
    }
    
    struct String_vector children;
    int rc = zoo_get_children(zh_, path.c_str(), 0, &children);
    
    if (rc == ZOK) {
        result.reserve(children.count);
        for (int i = 0; i < children.count; ++i) {
            result.emplace_back(children.data[i]);
        }
        deallocate_String_vector(&children);
    } else {
        LOG_WARN("ZK get children failed: path={}, error={}", path, zerror(rc));
    }
    
    return result;
}

// 监听子节点
void ZooKeeperClient::WatchChildren(const std::string& path, WatchCallback callback){
    if(!IsConnected()){
        LOG_ERROR("ZK not connected, cannot watch: {}", path);
        return;
    }

    // 保存回调
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        /* 存储客户端传入的回调函数
           - 每次触发watch后，都需要重新设置，也就需要再次使用“客户端传入的回调函数”
           - 故：需要存储 path ——> callback 的map映射，方便重复使用 
        */
        watches_[path] = std::move(callback);
    }

    // 设置 watch 并获取当前子节点
    struct String_vector children;
    int rc=zoo_wget_children(zh_, path.c_str(), GlobalWatcher, this, &children);

    if (rc == ZOK) {
        // ★ 释放 String_vector 内存（必须调用！）
        deallocate_String_vector(&children);
        LOG_DEBUG("ZK watch set: {}", path);
    } else {
        LOG_ERROR("ZK watch failed: path={}, error={}", path, zerror(rc));
    }
}

// 取消监听子节点
void ZooKeeperClient::UnwatchChildren(const std::string& path) {
    std::lock_guard<std::mutex> lock(watch_mutex_);
    /* watch是一次性的，触发后需要重新注册，即从watches_中通过path取出callback进行注册
        - 只要将对应的callback从watches_中移除，之后path对应的节点也就无法再注册watch
          也就达到了“取消监听子节点”的目的*/
    watches_.erase(path);
    LOG_DEBUG("ZK watch removed: {}", path);
}

// ============================================================================
// 内部实现（静态 Watcher 回调 + 事件处理）
// ============================================================================

// 全局 Watcher 回调（静态，供 C API 调用）
void ZooKeeperClient::GlobalWatcher(zhandle_t* zh, int type, int state,
                    const char* path, void* context){
    
    ZooKeeperClient* client=static_cast<ZooKeeperClient*>(context);
    if(!client) return;

    
    if(type==ZOO_SESSION_EVENT){
        // 会话事件：连接、断开、过期等
        client->HandleSessionEvent(state);
    }else if(type==ZOO_CHILD_EVENT && path){
        // 子节点变化事件
        client->HandleChildEvent(path);
    }
}

// 处理会话状态变化
void ZooKeeperClient::HandleSessionEvent(int state) {
    if (state == ZOO_CONNECTED_STATE) { 
        // 连接成功
        connected_ = true;
        LOG_INFO("ZooKeeper connected");
        conn_cv_.notify_all();  // 通知等待连接的线程
    } else if (state == ZOO_EXPIRED_SESSION_STATE) {
        // 会话过期
        connected_ = false;
        LOG_WARN("ZooKeeper session expired");
    } else if (state == ZOO_CONNECTING_STATE) {
        // 正在连接
        connected_ = false;
        LOG_INFO("ZooKeeper reconnecting...");
    } else if (state == ZOO_ASSOCIATING_STATE) {
        // 关联中
        LOG_DEBUG("ZooKeeper associating...");
    } else {
        LOG_WARN("ZooKeeper unknown state: {}", state);
    }
}

// 处理子节点列表变更
void ZooKeeperClient::HandleChildEvent(const std::string& path){
    WatchCallback callback;
    
    // 获取回调（加锁）
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        auto it=watches_.find(path);
        if (it != watches_.end()) {
            callback = it->second;
        }
    }

    // 触发回调（不持锁，避免死锁）
    if(callback){
        callback(path);
    }

    // 重新注册 watch（ZK 的 watch 是一次性的）
    ResetWatch(path);
}

// 重新注册指定路径的 watch
void ZooKeeperClient::ResetWatch(const std::string& path){
    {
        std::lock_guard<std::mutex> lock(watch_mutex_);
        // 检查是否需要再监听
        if(watches_.find(path) == watches_.end()){
            return;
        }
    }

    struct String_vector children;

    int rc=zoo_wget_children(zh_, path.c_str(), GlobalWatcher, this, &children);

    if(rc==ZOK){
        deallocate_String_vector(&children);
    }else{
        LOG_WARN("Failed to reset watch for {}: {}", path, zerror(rc));
    }
}

}   // namespace user_service
```


# 六、服务注册（服务端封装）
- 要用的“五、zookeeper客户端封装”

## 0.概述
- **一个 Registry 对应一个实例**
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                      一个进程 = 一个服务实例                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   进程 1 (192.168.1.10:50051)          进程 2 (192.168.1.11:50051)           │
│   ┌─────────────────────────┐          ┌─────────────────────────┐          │
│   │  gRPC Server            │          │  gRPC Server            │          │
│   │  ServiceRegistry ───────┼──┐       │  ServiceRegistry ───────┼──┐       │
│   │  (只注册自己)            │  │       │  (只注册自己)            │  │       │
│   └─────────────────────────┘  │       └─────────────────────────┘  │       │
│                                │                                    │       │
│                                ▼                                    ▼       │
│                         ┌─────────────────────────────────────────────┐     │
│                         │              ZooKeeper                      │     │
│                         │  /services/user-service/                    │     │
│                         │      ├── 192.168.1.10:50051  (进程1创建)    │     │
│                         │      └── 192.168.1.11:50051  (进程2创建)    │     │
│                         └─────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260211163840746.png)

## 1.service_instance.h
- 服务示例信息
```cpp
// src/discovery/service_instance.h
#pragma once

#include <string>
#include <map>
#include "thirdparty/json/json.hpp"

namespace user_service {

/**
 * @brief 服务实例信息
 * 
 * @note services/user-service/192.168.1.10:50051
 *       - serivces：root_path
 *       - services/user-service：服务路径（永久节点） 
 *           - user-service：服务名
 *       - services/user-service/192.168.1.10:50051：实例节点（临时节点）
 *           - 192.168.1.10:50051：实例节点数据（存储用json格式）
 */
struct ServiceInstance {
    std::string service_name;   // 服务名，如 "user-service"
    std::string instance_id;    // 实例ID，如 "192.168.1.10:50051"
    std::string host;           // 主机地址
    int port;                   // 端口号
    int weight = 100;           // 权重（负载均衡用）
    std::map<std::string, std::string> metadata;  // 元数据
    
    /**
     * @brief 获取完整地址
     */
    std::string GetAddress() const {
        return host + ":" + std::to_string(port);
    }
    
    /**
     * @brief 序列化为 JSON
     */
    std::string ToJson() const {
        nlohmann::json j;
        j["service_name"] = service_name;
        j["instance_id"] = instance_id;
        j["host"] = host;
        j["port"] = port;
        j["weight"] = weight;
        j["metadata"] = metadata;
        return j.dump();
    }
    
    /**
     * @brief 从 JSON 反序列化
     */
    static ServiceInstance FromJson(const std::string& json_str) {
        ServiceInstance instance;
        try {
            auto j = nlohmann::json::parse(json_str);
            instance.service_name = j.value("service_name", "");
            instance.instance_id = j.value("instance_id", "");
            instance.host = j.value("host", "");
            instance.port = j.value("port", 0);
            instance.weight = j.value("weight", 100);
            if (j.contains("metadata")) {
                instance.metadata = j["metadata"].get<std::map<std::string, std::string>>();
            }
        } catch (...) {
            // 解析失败，返回空实例
        }
        return instance;
    }
    
    bool IsValid() const {
        return !host.empty() && port > 0;
    }
};

} // namespace user_service
```

## 2.service_registry.h
```cpp
#pragma once

#include <memory>
#include <thread>
#include <atomic>
#include "zk_client.h"
#include "service_instance.h"

namespace user_service {

/**
 * @brief 服务注册器（gRPC 服务端使用）——> 一个进程 = 一个服务 = 一个服务注册器
 * 
 * 功能：
 * - 将服务实例注册到 ZooKeeper
 * - 自动心跳保活（通过临时节点实现）
 * - 优雅下线
 * 
 * 使用示例：
 * @code
 * auto registry = std::make_shared<ServiceRegistry>(zk_client);
 * ServiceInstance instance;
 * instance.service_name = "user-service";
 * instance.host = "192.168.1.10";
 * instance.port = 50051;
 * registry->Register(instance);
 * // ... 服务运行 ...
 * registry->Unregister();
 * @endcode
 */
class ServiceRegistry {
public:
    /**
     * @brief 构造函数
     * @param zk_client ZooKeeper 客户端（需要已连接）
     * @param root_path 服务根路径，默认 "/services"
     */
    explicit ServiceRegistry(std::shared_ptr<ZooKeeperClient> zk_client,
                            const std::string& root_path = "/services");
    
    ~ServiceRegistry();
    
    // 禁止拷贝
    ServiceRegistry(const ServiceRegistry&) = delete;
    ServiceRegistry& operator=(const ServiceRegistry&) = delete;
    
    /**
     * @brief 注册服务实例
     * @param instance 服务实例信息
     * @return 注册成功返回 true
     * 
     * @note 会自动创建父路径
     * @note 创建临时节点，服务下线时自动删除
     */
    bool Register(const ServiceInstance& instance);
    
    /**
     * @brief 注销服务实例
     * @return 注销成功返回 true
     */
    bool Unregister();
    
    /**
     * @brief 更新服务实例信息
     * @param instance 新的实例信息
     * @return 更新成功返回 true
     */
    bool Update(const ServiceInstance& instance);
    
    /**
     * @brief 是否已注册
     */
    bool IsRegistered() const { return registered_.load(); }
    
private:
    /**
     * @brief 构建实例节点路径
     * 格式：/services/{service_name}/{instance_id}
     */
    std::string BuildInstancePath(const ServiceInstance& instance) const;
    
    /**
     * @brief 构建服务路径
     * 格式：/services/{service_name}
     */
    std::string BuildServicePath(const std::string& service_name) const;
    
    std::shared_ptr<ZooKeeperClient> zk_client_;    // ZooKeeper 客户端（需要已连接）
    std::string root_path_;                         // 服务根路径，默认 "/services"
    
    // 当前注册的实例信息（微服务架构中：一个进程 = 一个服务实例）
    ServiceInstance current_instance_;
    std::string current_path_;
    std::atomic<bool> registered_{false};
    
    mutable std::mutex mutex_;
};

} // namespace user_service
```

## 3.service_registry.cpp
```cpp
#include "service_registry.h"
#include "common/logger.h"

namespace user_service{


ServiceRegistry::ServiceRegistry(std::shared_ptr<ZooKeeperClient> zk_client,
    const std::string& root_path)
    : zk_client_(std::move(zk_client))
    , root_path_(root_path) {
}


ServiceRegistry::~ServiceRegistry() {
    Unregister();
}

bool ServiceRegistry::Register(const ServiceInstance& instance){
    std::lock_guard<std::mutex> lock(mutex_);

    // 检查zookeeper客户端句柄的有效性
    if(!zk_client_ || !zk_client_->IsConnected()){
        LOG_ERROR("ZK client not connected, cannot register service");
        return false;
    }

    // 检查 服务示例信息 的有效性
    if(!instance.IsValid()){
        LOG_ERROR("Invalid service instance: host={}, port={}", 
            instance.host, instance.port);
            return false;
    }

    // 1.确保服务路径存在（持久节点）
    std::string service_path=BuildServicePath(instance.service_name);
    if(!zk_client_->CreateServicePath(service_path)){
        LOG_ERROR("Failed to create service path: {}", service_path);
        return false;
    }

    // 2.创建实例节点（临时节点）
    std::string instance_path = BuildInstancePath(instance);
    std::string data=instance.ToJson();

    if(!zk_client_->CreateNode(instance_path,data,true)){
        LOG_ERROR("Failed to create instance node: {}", instance_path);
        return false;
    }

    // 3.保存状态（方便之后进行修改、取消注册）
    current_instance_ =instance;
    current_path_=instance_path;
    registered_=true;

    LOG_INFO("Service registered: {} at {}", 
        instance.service_name, instance.GetAddress());

    return true;
}

bool ServiceRegistry::Unregister(){
    std::lock_guard<std::mutex> lock(mutex_);

    // 检查是否注册
    if(!registered_.load()){
        return true;  // 未注册，视为成功
    }

    // 句柄失效，则服务实例也就失效了（因为是临时节点）
    if(!zk_client_){
        registered_ = false;
        return true;
    }

    // 删除实例节点
    bool success=zk_client_->DeleteNode(current_path_);

    if (success) {
        LOG_INFO("Service unregistered: {} at {}", 
                 current_instance_.service_name, 
                 current_instance_.GetAddress());
    } else {
        LOG_WARN("Failed to unregister service, node may already be deleted");
    }

    // 更新状态
    registered_=false;
    current_path_.clear();

    return true;
}



bool ServiceRegistry::Update(const ServiceInstance& instance){
    std::lock_guard<std::mutex> lock(mutex_);
    
    // 检查是否注册
    if(!registered_.load()){
        LOG_ERROR("Service not registered, cannot update");
        return false;
    }

    // 检查zookeeper客户端句柄的有效性
    if (!zk_client_ || !zk_client_->IsConnected()) {
        LOG_ERROR("ZK client not connected, cannot update");
        return false;
    }

    // 序列化节点数据
    std::string data = instance.ToJson();

    if (!zk_client_->SetData(current_path_, data)) {
        LOG_ERROR("Failed to update service instance: {}", current_path_);
        return false;
    }

    current_instance_ = instance;
    LOG_DEBUG("Service instance updated: {}", current_path_);
    
    return true;
}

std::string ServiceRegistry::BuildInstancePath(const ServiceInstance& instance) const {
    // 格式：/services/user-service/192.168.1.10:50051
    return root_path_ + "/" + instance.service_name + "/" + instance.GetAddress();
}

std::string ServiceRegistry::BuildServicePath(const std::string& service_name) const {
    // 格式：/services/user-service
    return root_path_ + "/" + service_name;
}

}
```


# 七、服务发现
- gRPC的客户端通过“Zookeeper客户端”向“Zookeeper服务端”获取gRPC服务端注册的“服务列表”

## 1.service_discory.h
```cpp
#pragma once

#include "zk_client.h"
#include "service_instance.h"
#include <memory>
#include <vector>
#include <map>
#include <shared_mutex>
#include <functional>
#include <random>

namespace user_service {

/**
 * @brief 服务发现器（gRPC 客户端使用）
 * 
 * 功能：
 * - 从 ZooKeeper 获取服务实例列表
 * - 订阅服务变化（实例上线/下线）
 * - 本地缓存 + 自动更新
 * - 负载均衡选择实例
 * 
 * 使用示例：
 * @code
 * auto discovery = std::make_shared<ServiceDiscovery>(zk_client);
 * 
 * // 订阅服务变化
 * discovery->Subscribe("user-service", [](const std::string& service) {
 *     LOG_INFO("Service {} instances changed", service);
 * });
 * 
 * // 获取一个实例（负载均衡）
 * auto instance = discovery->SelectInstance("user-service");
 * if (instance) {
 *     auto channel = grpc::CreateChannel(instance->GetAddress(), ...);
 * }
 * @endcode
 */
class ServiceDiscovery {
public:
    // 服务变化回调
    using ServiceChangeCallback = std::function<void(const std::string& service_name)>;
    
    /**
     * @brief 构造函数
     * @param zk_client ZooKeeper 客户端（需要已连接）
     * @param root_path 服务根路径，默认 "/services"
     */
    explicit ServiceDiscovery(std::shared_ptr<ZooKeeperClient> zk_client,
                             const std::string& root_path = "/services");
    
    ~ServiceDiscovery();
    
    // 禁止拷贝
    ServiceDiscovery(const ServiceDiscovery&) = delete;
    ServiceDiscovery& operator=(const ServiceDiscovery&) = delete;
    
    /**
     * @brief 订阅服务变化（设置watch）
     * @param service_name 服务名称
     * @param callback 变化回调（可选）
     * 
     * @note 会立即拉取一次实例列表
     * @note 之后实例变化时自动更新本地缓存并触发回调
     */
    void Subscribe(const std::string& service_name, 
                   ServiceChangeCallback callback = nullptr);
    
    /**
     * @brief 取消订阅（取消watch）
     * @param service_name 服务名称
     */
    void Unsubscribe(const std::string& service_name);
    
    /**
     * @brief 获取服务的所有实例（业务一个服务在多个ip，或多个port上运行）
     * @param service_name 服务名称
     * @return 实例列表（可能为空）
     */
    std::vector<ServiceInstance> GetInstances(const std::string& service_name);
    
    /**
     * @brief 选择一个实例（随机负载均衡，自己实现）
     * @param service_name 服务名称
     * @return 实例指针，无可用实例返回 nullptr
     */
    std::shared_ptr<ServiceInstance> SelectInstance(const std::string& service_name);
    
    /**
     * @brief 选择一个实例（加权随机，自己实现）
     * @param service_name 服务名称
     * @return 实例指针，无可用实例返回 nullptr
     */
    std::shared_ptr<ServiceInstance> SelectInstanceWeighted(const std::string& service_name);
    
private:
    /**
     * @brief 刷新指定服务的实例列表
     */
    void RefreshInstances(const std::string& service_name);
    
    /**
     * @brief 处理子节点变化（Watch 回调）
     */
    void OnChildrenChanged(const std::string& path);
    
    /**
     * @brief 从路径提取服务名
     * /services/user-service -> user-service
     */
    std::string ExtractServiceName(const std::string& path) const;
    
    /**
     * @brief 构建服务路径
     */
    std::string BuildServicePath(const std::string& service_name) const;
    

    std::shared_ptr<ZooKeeperClient> zk_client_;    // zookeeper客户端句柄
    std::string root_path_;                         // 服务根路径，默认 "/services"
    
    // 服务实例缓存：service_name -> instances
    mutable std::shared_mutex cache_mutex_;
    std::map<std::string, std::vector<ServiceInstance>> instance_cache_;
    
    // 服务变化回调：service_name -> callback
    mutable std::mutex callback_mutex_;
    std::map<std::string, ServiceChangeCallback> callbacks_;
    
    // 随机数生成器（负载均衡用）
    mutable std::mt19937 rng_{std::random_device{}()};
};

} // namespace user_service
```

## 2.service_discovery.cpp
```cpp
#include "service_discovery.h"
#include "common/logger.h"

namespace user_service{

ServiceDiscovery::ServiceDiscovery(std::shared_ptr<ZooKeeperClient> zk_client,
    const std::string& root_path)
    : zk_client_(std::move(zk_client))
    , root_path_(root_path) {
}

ServiceDiscovery::~ServiceDiscovery() {
    // 取消所有订阅
    std::vector<std::string> services;
    {
        std::shared_lock<std::shared_mutex> lock(cache_mutex_);
        for (const auto& [name, _] : instance_cache_) {
            services.push_back(name);
        }
    }
    
    for (const auto& service : services) {
        Unsubscribe(service);
    }
}

void ServiceDiscovery::Subscribe(const std::string& service_name,
                                 ServiceChangeCallback callback) {
    
    // 检查zookeeper客户端句柄的有效性
    if (!zk_client_ || !zk_client_->IsConnected()) {
        LOG_ERROR("ZK client not connected, cannot subscribe: {}", service_name);
        return;
    }
    
    // 保存回调
    if (callback) {
        std::lock_guard<std::mutex> lock(callback_mutex_);
        callbacks_[service_name] = std::move(callback);
    }
    
    // 立即刷新一次实例列表
    RefreshInstances(service_name);
    
    // 设置 Watch
    std::string service_path = BuildServicePath(service_name);
    zk_client_->WatchChildren(service_path, 
        // 子节点改变是的“回调函数”
        [this](const std::string& path) {
            OnChildrenChanged(path);
        });
    
    LOG_INFO("Subscribed to service: {}", service_name);
}

void ServiceDiscovery::Unsubscribe(const std::string& service_name) {
    
    // 取消 Watch
    std::string service_path = BuildServicePath(service_name);
    if (zk_client_) {
        zk_client_->UnwatchChildren(service_path);
    }
    
    // 清理回调
    {
        std::lock_guard<std::mutex> lock(callback_mutex_);
        callbacks_.erase(service_name);
    }
    
    // 清理缓存
    {
        std::unique_lock<std::shared_mutex> lock(cache_mutex_);
        instance_cache_.erase(service_name);
    }
    
    LOG_INFO("Unsubscribed from service: {}", service_name);
}

// 获取服务的所有实例
std::vector<ServiceInstance> ServiceDiscovery::GetInstances(const std::string& service_name) {
    std::shared_lock<std::shared_mutex> lock(cache_mutex_);
    
    auto it = instance_cache_.find(service_name);
    if (it != instance_cache_.end()) {
        return it->second;
    }
    
    return {};
}

// 选择一个实例（随机负载均衡，自己实现）
std::shared_ptr<ServiceInstance> ServiceDiscovery::SelectInstance(const std::string& service_name) {
    // 获取实例列表
    auto instances = GetInstances(service_name);
    
    if (instances.empty()) {
        LOG_WARN("No available instance for service: {}", service_name);
        return nullptr;
    }
    
    // 随机选择一个实例
    std::uniform_int_distribution<size_t> dist(0, instances.size() - 1);
    size_t index = dist(rng_);
    
    return std::make_shared<ServiceInstance>(instances[index]);
}

// 选择一个实例（加权随机，自己实现）
std::shared_ptr<ServiceInstance> ServiceDiscovery::SelectInstanceWeighted(
    const std::string& service_name) {
    
    // 获取实例列表
    auto instances = GetInstances(service_name);
    
    if (instances.empty()) {
        LOG_WARN("No available instance for service: {}", service_name);
        return nullptr;
    }
    
    // 计算总权重
    int total_weight = 0;
    for (const auto& inst : instances) {
        total_weight += inst.weight;
    }
    
    if (total_weight <= 0) {
        // 权重都为0，退化为随机选择
        return SelectInstance(service_name);
    }
    
    // 加权随机
    std::uniform_int_distribution<int> dist(1, total_weight);
    int random_weight = dist(rng_);
    
    int current_weight = 0;
    for (const auto& inst : instances) {
        current_weight += inst.weight;
        if (random_weight <= current_weight) {
            return std::make_shared<ServiceInstance>(inst);
        }
    }
    
    // 理论上不会到这里
    return std::make_shared<ServiceInstance>(instances.back());
}

// 刷新指定服务的实例列表
void ServiceDiscovery::RefreshInstances(const std::string& service_name) {
    if (!zk_client_ || !zk_client_->IsConnected()) {
        return;
    }
    
    // 构造 服务路径名
    std::string service_path = BuildServicePath(service_name);
    
    // 获取子节点列表（实例ID列表，ip:port）
    auto children = zk_client_->GetChildren(service_path);
    
    std::vector<ServiceInstance> instances;
    instances.reserve(children.size());
    
    // 获取每个实例的详细信息
    for (const auto& child : children) {
        // 通过实例节点路径，获取实例节点数据（json存储）
        std::string instance_path = service_path + "/" + child;
        std::string data = zk_client_->GetData(instance_path);
        
        if (!data.empty()) {
            // 反序列化出“实例节点数据”
            ServiceInstance instance = ServiceInstance::FromJson(data);
            if (instance.IsValid()) {
                instances.push_back(std::move(instance));
            } else {
                LOG_WARN("Invalid instance data at: {}", instance_path);
            }
        }
    }
    
    // 更新缓存
    {
        std::unique_lock<std::shared_mutex> lock(cache_mutex_);
        instance_cache_[service_name] = std::move(instances);
    }
    
    LOG_DEBUG("Refreshed service {}: {} instances", 
              service_name, instance_cache_[service_name].size());
}

void ServiceDiscovery::OnChildrenChanged(const std::string& path) {
    
    // 获取服务名
    std::string service_name = ExtractServiceName(path);
    
    if (service_name.empty()) {
        LOG_WARN("Cannot extract service name from path: {}", path);
        return;
    }
    
    LOG_INFO("Service {} instances changed, refreshing...", service_name);
    
    // 刷新实例列表
    RefreshInstances(service_name);
    
    // 触发回调
    ServiceChangeCallback callback;
    {
        std::lock_guard<std::mutex> lock(callback_mutex_);
        auto it = callbacks_.find(service_name);
        if (it != callbacks_.end()) {
            callback = it->second;
        }
    }
    
    if (callback) {
        callback(service_name);
    }
}

std::string ServiceDiscovery::ExtractServiceName(const std::string& path) const {
    // /services/user-service -> user-service
    if (path.length() <= root_path_.length() + 1) {
        return "";
    }
    
    return path.substr(root_path_.length() + 1);  // +1 跳过 '/'
}

std::string ServiceDiscovery::BuildServicePath(const std::string& service_name) const {
    return root_path_ + "/" + service_name;
}

}
```

## 3.客户端负载均衡
- 一个服务可能在多个服务器上运行，故可以在客户端进行负载均衡
```cpp
// 从多个服务实例中选择一个
std::shared_ptr<ServiceInstance> SelectInstance(const std::string& service_name);

// 使用场景
auto instance = discovery->SelectInstance("user-service");
// instance 可能是：
//   - "10.0.0.1:8001"  
//   - "10.0.0.2:8001"  ← 随机选中
//   - "10.0.0.3:8001"

auto channel = grpc::CreateChannel(instance->GetAddress(), ...);
// gRPC 客户端直接连接选中的服务实例
```
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                      方式一：客户端负载均衡（你的代码）                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                           ZooKeeper                                         │
│                    ┌─────────────────────┐                                  │
│                    │  /services/user     │                                  │
│                    │   ├── node-001      │                                  │
│                    │   ├── node-002      │                                  │
│                    │   └── node-003      │                                  │
│                    └─────────────────────┘                                  │
│                              │                                              │
│                              │ 获取服务列表                                  │
│                              ▼                                              │
│   ┌─────────────────────────────────────────────┐                           │
│   │              gRPC Client                    │                           │
│   │  ┌─────────────────────────────────────┐   │                           │
│   │  │     ServiceDiscovery                │   │                           │
│   │  │  ┌───────────────────────────────┐  │   │                           │
│   │  │  │ 服务列表缓存:                  │  │   │                           │
│   │  │  │  - 10.0.0.1:8001              │  │   │                           │
│   │  │  │  - 10.0.0.2:8001              │  │   │                           │
│   │  │  │  - 10.0.0.3:8001              │  │   │                           │
│   │  │  └───────────────────────────────┘  │   │                           │
│   │  │                                     │   │                           │
│   │  │  SelectInstance() ──► 随机选一个    │   │                           │
│   │  └─────────────────────────────────────┘   │                           │
│   └─────────────────────────────────────────────┘                           │
│                     │         │         │                                   │
│                     │ 直接连接 │         │                                   │
│                     ▼         ▼         ▼                                   │
│               ┌─────────┐ ┌─────────┐ ┌─────────┐                           │
│               │ Server1 │ │ Server2 │ │ Server3 │                           │
│               │ :8001   │ │ :8001   │ │ :8001   │                           │
│               └─────────┘ └─────────┘ └─────────┘                           │
│                                                                             │
│   ★ 特点：客户端直接连接服务端，没有中间代理                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```