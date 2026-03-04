---
title: nginx的使用
date: 2026-02-25 18:24:41
tags:
	- 笔记
categories:
    - C++
    - C++后端开发库
    - nginx
    - nginx的使用
---

# 一、nginx简介
## 1.什么是nginx
- 高性能的 HTTP 服务器 和 反向代理服务器
- 支持 TCP/UDP 代理（stream 模块）
- 特点：高并发、低内存、热部署、高可用

## 2.核心功能
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260225182757597.png)

## 3.模型架构
```bash
                    ┌─────────────────┐
                    │   Master 进程    │ ← 管理配置、管理 Worker
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │  Worker 1   │   │  Worker 2   │   │  Worker N   │
    │  (处理请求)  │   │  (处理请求)  │   │  (处理请求)  │
    └─────────────┘   └─────────────┘   └─────────────┘
```

# 二、安装与启动
## 1.Ubuntu安装
```bash
# 基础安装
sudo apt update
sudo apt install nginx -y

# 安装 stream 模块（TCP/UDP 负载均衡必需）
sudo apt install libnginx-mod-stream -y

# 验证安装
nginx -v
```

## 2.服务管理
```bash
# systemctl 方式
sudo systemctl start nginx      # 启动
sudo systemctl stop nginx       # 停止
sudo systemctl restart nginx    # 重启
sudo systemctl reload nginx     # 热重载配置（推荐）
sudo systemctl status nginx     # 查看状态
sudo systemctl enable nginx     # 开机自启

# nginx 命令方式
sudo nginx                      # 启动
sudo nginx -s stop              # 快速停止
sudo nginx -s quit              # 优雅停止
sudo nginx -s reload            # 热重载
sudo nginx -t                   # 测试配置
```

# 三、配置
## 1.配置文件位置
```bash
/etc/nginx/nginx.conf           # 主配置文件
/etc/nginx/conf.d/*.conf        # 额外配置（自动加载）
/etc/nginx/sites-available/     # 站点配置（可用）
/etc/nginx/sites-enabled/       # 站点配置（已启用，软链接）
```

## 2.配置层级结构
```cpp
# 全局块 - 影响整体运行
user nginx;                          # 运行用户
worker_processes auto;               # Worker 进程数（auto=CPU核心数）
error_log /var/log/nginx/error.log;  # 错误日志
pid /run/nginx.pid;                  # PID 文件

# events 块 - 网络连接配置
events {
    worker_connections 1024;         # 单个 Worker 最大连接数
    use epoll;                       # 事件驱动模型（Linux 推荐）
    multi_accept on;                 # 同时接受多个连接
}

# http 块 - HTTP 服务配置
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';
    
    access_log /var/log/nginx/access.log main;
    
    sendfile on;
    keepalive_timeout 65;
    
    # 引入其他配置
    include /etc/nginx/conf.d/*.conf;
    
    # server 块 - 虚拟主机配置
    server {
        listen 80;
        server_name localhost;
        
        # location 块 - 路由匹配
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}

# stream 块 - TCP/UDP 代理配置
stream {
    upstream backend {
        server 127.0.0.1:6000;
        server 127.0.0.1:6001;
    }
    
    server {
        listen 8000;
        proxy_pass backend;
    }
}
```

## 3.TCP/UDP 负载均衡 (Stream)
```bash
客户端请求
    │
    ▼
┌─────────────────────────────────┐
│     Nginx Stream (Port 8000)    │
│  ┌───────────────────────────┐  │
│  │    least_conn 算法        │  │
│  │  选择连接数最少的后端      │  │
│  └───────────────────────────┘  │
└─────────────────┬───────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌───────┐   ┌───────┐   ┌───────┐
│ :6000 │   │ :6001 │   │ :6002 │
│ 连接:5│   │ 连接:3│   │ 连接:8│
└───────┘   └───────┘   └───────┘
                ▲
                │
           新连接分配到这里
          （当前连接数最少）

```

### （1）配置
```cpp
stream {
    # 聊天服务器集群
    upstream chat_cluster {
        least_conn;  # 最少连接策略
        
        server 127.0.0.1:6000 weight=1 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:6001 weight=1 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:6002 weight=1 max_fails=3 fail_timeout=30s;
    }
    
    # 数据库代理
    upstream mysql_cluster {
        server 192.168.1.101:3306 weight=5;
        server 192.168.1.102:3306 weight=5;
        server 192.168.1.103:3306 backup;  # 备用服务器
    }
    
    # 聊天服务入口
    server {
        listen 8000;
        proxy_pass chat_cluster;
        proxy_connect_timeout 1s;
        proxy_timeout 600s;
        tcp_nodelay on;
    }
    
    # MySQL 代理入口
    server {
        listen 3307;
        proxy_pass mysql_cluster;
        proxy_connect_timeout 5s;
        proxy_timeout 300s;
    }
}
```

### （2）配置参数解析
```cpp
# =====================================================
# TCP/UDP 负载均衡模块 (Layer 4)
# 工作在传输层，转发原始 TCP/UDP 数据包
# 需要安装: sudo apt install libnginx-mod-stream
# =====================================================
stream {

    # ==================== 上游服务器集群定义 ====================
    
    # 聊天服务器集群
    upstream chat_cluster {
        # -------- 负载均衡策略 --------
        # least_conn: 最少连接数策略，新连接分配给当前连接数最少的服务器
        # 其他可选策略:
        #   (默认)轮询: 不写任何策略，按顺序轮流分配
        #   hash $remote_addr: IP哈希，同一IP始终访问同一后端
        #   random: 随机选择
        least_conn;
        
        # -------- 后端服务器列表 --------
        # 格式: server <地址>:<端口> [参数];
        #
        # weight=1        权重，默认1，数值越大分配请求越多
        # max_fails=3     最大失败次数，超过后标记为不可用
        # fail_timeout=30s 失败超时时间:
        #                   1) 达到 max_fails 后，暂停服务 30 秒
        #                   2) 30 秒内失败次数的统计窗口
        # backup          备用服务器，仅当所有主服务器不可用时启用
        # down            标记服务器永久不可用（维护时使用）
        
        server 127.0.0.1:6000 weight=1 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:6001 weight=1 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:6002 weight=1 max_fails=3 fail_timeout=30s;
    }
    
    # 数据库代理集群
    upstream mysql_cluster {
        # weight=5: 权重为5（相对值，用于多服务器间比例分配）
        server 192.168.1.101:3306 weight=5;
        server 192.168.1.102:3306 weight=5;
        
        # backup: 备用服务器
        # 正常情况不参与负载，仅当上面所有服务器都故障时才启用
        server 192.168.1.103:3306 backup;
    }

    # ==================== 监听服务配置 ====================
    
    # 聊天服务入口（对外暴露的端口）
    server {
        # listen: 监听端口
        # 可选参数:
        #   udp          - 监听 UDP（默认 TCP）
        #   ssl          - 启用 SSL/TLS
        #   backlog=N    - 连接队列大小
        #   so_keepalive - 启用 TCP keepalive
        listen 8000;
        
        # proxy_pass: 转发目标（上游集群名称）
        proxy_pass chat_cluster;
        
        # proxy_connect_timeout: 与后端建立连接的超时时间
        # 超过此时间连接失败，尝试下一个后端服务器
        proxy_connect_timeout 1s;
        
        # proxy_timeout: 连接空闲超时时间
        # 客户端和后端之间无数据传输超过此时间，连接将被关闭
        # 聊天服务设置较长（600s=10分钟），允许长连接
        proxy_timeout 600s;
        
        # tcp_nodelay: 禁用 Nagle 算法
        # on  - 立即发送小数据包，降低延迟（适合实时聊天）
        # off - 合并小数据包后发送，提高吞吐量
        tcp_nodelay on;
        
        # -------- 其他可选配置 --------
        # proxy_buffer_size 16k;     # 代理缓冲区大小
        # proxy_bind $remote_addr;   # 绑定客户端IP（透传真实IP）
        # proxy_protocol on;         # 启用 PROXY protocol
    }
    
    # MySQL 代理入口
    server {
        listen 3307;
        proxy_pass mysql_cluster;
        
        # 数据库连接超时可以稍长
        proxy_connect_timeout 5s;
        
        # 数据库查询超时（5分钟）
        # 需要根据最长 SQL 查询时间调整
        proxy_timeout 300s;
    }
}
```

# 四、Nginx配置详解与调优
## 1.Nginx 架构概述
- Nginx 采用的是**多 Reactor 多进程模型**
```bash
┌─────────────────────────────────────────────────────────────────────────┐
│                    Nginx 的 Reactor 架构                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────┐                                                   │
│   │  Master Process │  ◄── 不处理网络 IO，只负责管理                      │
│   │                 │      • 读取配置                                    │
│   │                 │      • 管理 Worker                                 │
│   │                 │      • 热重载 / 平滑升级                            │
│   └────────┬────────┘                                                   │
│            │ fork                                                       │
│            ▼                                                            │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                    Worker Processes                          │      │
│   │                                                              │      │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │      │
│   │  │   Worker 1   │  │   Worker 2   │  │   Worker N   │       │      │
│   │  │              │  │              │  │              │       │      │
│   │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │       │      │
│   │  │ │  Reactor │ │  │ │  Reactor │ │  │ │  Reactor │ │       │      │
│   │  │ │  (epoll) │ │  │ │  (epoll) │ │  │ │  (epoll) │ │       │      │
│   │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │       │      │
│   │  │              │  │              │  │              │       │      │
│   │  │ 事件循环:     │  │ 事件循环:     │  │ 事件循环:     │       │      │
│   │  │ • 监听事件    │  │ • 监听事件    │  │ • 监听事件    │       │      │
│   │  │ • 接受连接    │  │ • 接受连接    │  │ • 接受连接    │       │      │
│   │  │ • 读写数据    │  │ • 读写数据    │  │ • 读写数据    │       │      │
│   │  │ • 处理请求    │  │ • 处理请求    │  │ • 处理请求    │       │      │
│   │  └──────────────┘  └──────────────┘  └──────────────┘       │      │
│   │                                                              │      │
│   │  每个 Worker = 一个独立的 Reactor（单线程事件循环）             │      │
│   └──────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

```

## 2.核心配置文件结构
```bash
# /etc/nginx/nginx.conf 完整结构

# ==================== 全局块 ====================
user www-data;                      # 运行用户
worker_processes auto;              # Worker 进程数
pid /run/nginx.pid;                 # PID 文件位置
error_log /var/log/nginx/error.log; # 错误日志
worker_rlimit_nofile 100000;        # Worker 文件描述符限制

# ==================== events 块 ====================
events {
    worker_connections 65535;       # 单 Worker 最大连接数
    use epoll;                      # 事件驱动模型
    multi_accept on;                # 一次接受多个连接
    accept_mutex off;               # 关闭互斥锁（高并发建议关闭）
}

# ==================== http 块（Web 服务）====================
http {
    # ... HTTP 相关配置
}

# ==================== stream 块（TCP/UDP 代理）====================
stream {
    # ... TCP/UDP 四层代理配置
}
```

## 3.全局配置参数详解
### （1）Worker 相关
```bash
# worker_processes - Worker 进程数
worker_processes auto;          # 自动检测 CPU 核数（推荐）
# worker_processes 8;           # 手动指定

# 说明：
# • auto = CPU 核心数
# • 每个 Worker 独立处理连接，互不影响
# • 过多会增加上下文切换开销
```
```bash
# worker_rlimit_nofile - Worker 可打开的最大文件描述符
worker_rlimit_nofile 100000;

# 计算公式：
# worker_rlimit_nofile >= worker_connections × 2
# 
# 为什么要 ×2？
# • 每个客户端连接 = 1 个 fd
# • 每个上游连接（代理后端）= 1 个 fd
# • 还需要预留给日志、共享内存等
```
```bash
# worker_cpu_affinity - CPU 亲和性绑定
worker_cpu_affinity auto;       # 自动绑定（推荐）
# worker_cpu_affinity 0001 0010 0100 1000;  # 手动绑定 4 核

# 作用：
# • 减少 CPU 缓存失效
# • 减少跨核调度开销
# • 提高缓存命中率
```

### （2）Worker 进程数设置原则
```bash
┌─────────────────────────────────────────────────────────────┐
│                  Worker 进程数选择                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   场景                          推荐值                       │
│   ─────────────────────────────────────────────────────     │
│   纯静态文件服务                 = CPU 核心数                 │
│   反向代理（无计算）             = CPU 核心数                 │
│   有 CPU 密集操作（压缩/加密）    = CPU 核心数                 │
│   有大量磁盘 IO                  = CPU 核心数 × 1.5~2        │
│                                                             │
│   通用建议：使用 auto，让 Nginx 自动决定                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4.Events 块详解
```bash
events {
    # ========== 连接数配置 ==========
    worker_connections 65535;
    # • 单个 Worker 的最大并发连接数
    # • 总并发 = worker_processes × worker_connections
    # • 示例：8 × 65535 = 524,280 理论最大并发
    
    # ========== 事件模型 ==========
    use epoll;
    # • Linux 首选 epoll（高效）
    # • FreeBSD 使用 kqueue
    # • 不写则自动选择最优
    
    # ========== 连接接受策略 ==========
    multi_accept on;
    # • on:  一次接受所有等待的连接（高并发推荐）
    # • off: 一次只接受一个连接
    
    # ========== 惊群问题 ==========
    accept_mutex off;
    # • on:  Worker 轮流接受连接（低并发时均衡）
    # • off: 所有 Worker 同时接受（高并发推荐）
    # 
    # Linux 3.9+ 内核有 SO_REUSEPORT，建议关闭 accept_mutex
}
```
**Events 参数关系图**
```bash
                     新连接到达
                         │
                         ▼
              ┌─────────────────────┐
              │   accept_mutex=off   │
              │   所有 Worker 竞争    │
              └──────────┬──────────┘
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
      ┌─────────┐   ┌─────────┐   ┌─────────┐
      │Worker 1 │   │Worker 2 │   │Worker 3 │
      │         │   │         │   │         │
      │ 65535   │   │ 65535   │   │ 65535   │
      │连接上限  │   │连接上限  │   │连接上限  │
      └─────────┘   └─────────┘   └─────────┘
           │             │             │
           └─────────────┼─────────────┘
                         ▼
              ┌─────────────────────┐
              │   multi_accept=on    │
              │   一次接受多个连接    │
              └─────────────────────┘
```

## 5.Stream 块详解（TCP 四层代理）
- 为什么叫四层代理？
	- 工作在 OSI **第 4 层（传输层）**，只看 TCP/IP，不解析应用协议

### （1）基础配置
```bash
stream {
    # ========== 日志格式 ==========
    log_format tcp_log '$remote_addr [$time_local] '
                       '$protocol $status $bytes_sent $bytes_received '
                       '$session_time "$upstream_addr"';
    
    access_log /var/log/nginx/tcp_access.log tcp_log;
    
    # ========== upstream 后端组 ==========
    upstream chat_backend {
        # 负载均衡策略
        least_conn;                     # 最少连接数（长连接推荐）
        # hash $remote_addr consistent; # IP 哈希（会话保持）
        # round_robin;                  # 轮询（默认）
        
        # 后端服务器
        server 127.0.0.1:6000 weight=1 max_fails=3 fail_timeout=30s max_conns=20000;
        server 127.0.0.1:6001 weight=1 max_fails=3 fail_timeout=30s max_conns=20000;
        server 127.0.0.1:6002 weight=1 max_fails=3 fail_timeout=30s max_conns=20000;
    }
    
    # ========== server 监听配置 ==========
    server {
        listen 8000;                    # 监听端口
        # listen 8000 reuseport;        # 启用 SO_REUSEPORT（高并发推荐）
        
        proxy_pass chat_backend;        # 代理到后端组
        
        # 超时配置
        proxy_connect_timeout 10s;      # 连接后端超时
        proxy_timeout 300s;             # 数据传输超时（长连接设大些）
        
        # 性能优化
        tcp_nodelay on;                 # 禁用 Nagle 算法
        proxy_buffer_size 16k;          # 代理缓冲区大小
    }
}
```

### （2）Upstream 参数详解
```bash
upstream chat_backend {
    # ========== 负载均衡策略 ==========
    
    # 1. 轮询（默认）
    # 按顺序分配请求
    
    # 2. 权重轮询
    # server 127.0.0.1:6000 weight=3;  # 权重高，分配更多请求
    # server 127.0.0.1:6001 weight=1;
    
    # 3. 最少连接数（长连接推荐）
    least_conn;
    
    # 4. IP 哈希（会话保持）
    # hash $remote_addr consistent;
    
    # ========== 服务器参数 ==========
    server 127.0.0.1:6000
        weight=1              # 权重（默认 1）
        max_fails=3           # 最大失败次数
        fail_timeout=30s      # 失败后暂停时间 & 失败计数重置时间
        max_conns=20000       # 单服务器最大连接数
        backup               # 备份服务器（主服务器全挂时启用）
        down;                # 标记为下线（不参与负载均衡）
}
```