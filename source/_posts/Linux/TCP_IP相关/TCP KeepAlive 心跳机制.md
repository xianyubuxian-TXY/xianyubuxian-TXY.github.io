---
title: TCP KeepAlive 心跳机制
date: 2026-01-13 10:16:24
tags:
	- 笔记
	- Linux
categories:
	- Linux
	- TCP_IP相关
	- TCP KeepAlive 心跳机制
---

## 1.机制原理
TCP KeepAlive 是 **操作系统内核层面** 的连接保活机制，用于检测长时间空闲的 TCP 连接是否仍然有效。
```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                    TCP KeepAlive 工作原理                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   连接空闲                 开始探测                 判定结果      │
│      │                       │                       │          │
│      ▼                       ▼                       ▼          │
│  ┌───────┐    超时后    ┌───────┐   连续失败    ┌───────┐       │
│  │ IDLE  │───────────▶│ PROBE │─────────────▶│ DEAD  │       │
│  └───────┘             └───────┘              └───────┘       │
│      │                     │                                   │
│      │                     │ 收到 ACK                          │
│      │◀────────────────────┘                                   │
│   重置计时器                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

```

**时间线详解**：
```plaintext
时间轴 ════════════════════════════════════════════════════════════════▶

[最后一次数据交互]
        │
        │◀══════════ tcp_keepalive_time ══════════▶│
        │                 7200 秒                   │
        │              (连接空闲)                   │
        │                                          │
        │                                          ▼
        │                                    ┌──────────┐
        │                                    │ 发送探测 │
        │                                    │  包 #1   │
        │                                    └────┬─────┘
        │                                         │
        │                         ┌───────────────┴───────────────┐
        │                         ▼                               ▼
        │                    收到 ACK                         无响应
        │                        │                               │
        │                        ▼                               ▼
        │               重置计时器，回到                  等待 75 秒
        │               空闲等待状态                    (tcp_keepalive_intvl)
        │                                                        │
        │                                                        ▼
        │                                                  ┌──────────┐
        │                                                  │ 发送探测 │
        │                                                  │  包 #2   │
        │                                                  └────┬─────┘
        │                                                       │
        │                                          ... 重复最多 9 次 ...
        │                                                       │
        │                                                       ▼
        │                                              ┌─────────────────┐
        │                                              │ 连续 9 次无响应  │
        │                                              │ 判定连接死亡     │
        │                                              │ 内核关闭 socket  │
        │                                              └─────────────────┘

总检测时间 = 7200 + (75 × 9) = 7875 秒 ≈ 2小时11分钟
```



## 2.核心参数 与 相关配置
### （1）三个核心参数
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113104040724.png)

### （2）查看当前系统参数
```bash
# 查看当前系统参数
sysctl net.ipv4.tcp_keepalive_time
sysctl net.ipv4.tcp_keepalive_intvl
sysctl net.ipv4.tcp_keepalive_probes
```
{% post_link "sysctl 与 systemctl" 'sysctl介绍' %}

### （3）参数配置（系统级）
#### <1>临时修改（重启失效）
```bash
# 30秒空闲后开始探测
sudo sysctl -w net.ipv4.tcp_keepalive_time=30

# 每10秒探测一次
sudo sysctl -w net.ipv4.tcp_keepalive_intvl=10

# 最多探测3次
sudo sysctl -w net.ipv4.tcp_keepalive_probes=3

# 验证修改
sysctl net.ipv4.tcp_keepalive_time
sysctl net.ipv4.tcp_keepalive_intvl
sysctl net.ipv4.tcp_keepalive_probes
```

#### <2>永久修改
```bash
# 编辑配置文件
sudo vim /etc/sysctl.conf

# 添加以下内容
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 3

# 使配置生效
sudo sysctl -p

# 或者使用单行命令
echo "net.ipv4.tcp_keepalive_time = 30" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_keepalive_intvl = 10" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_keepalive_probes = 3" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### <3>生产环境推荐配置
```bash
# /etc/sysctl.conf

# ═══════════════════════════════════════════════
# TCP KeepAlive 配置（生产环境推荐）
# ═══════════════════════════════════════════════

# 内网稳定环境
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
# 总检测时间 = 60 + 10×6 = 120秒

# 跨机房/云环境（更激进）
# net.ipv4.tcp_keepalive_time = 30
# net.ipv4.tcp_keepalive_intvl = 5
# net.ipv4.tcp_keepalive_probes = 3
# 总检测时间 = 30 + 5×3 = 45秒
```

### （4）应用层配置（Socket 编程）
**封装为工具类**
```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <stdexcept>
#include <string>

class TcpKeepAlive {
public:
    struct Config {
        int idle_seconds = 30;      // 空闲多久后开始探测
        int interval_seconds = 10;  // 探测间隔
        int max_probes = 3;         // 最大探测次数
    };

    static void enable(int sockfd, const Config& cfg = {}) {
        set_option(sockfd, SOL_SOCKET, SO_KEEPALIVE, 1, "SO_KEEPALIVE");
        set_option(sockfd, IPPROTO_TCP, TCP_KEEPIDLE, cfg.idle_seconds, "TCP_KEEPIDLE");
        set_option(sockfd, IPPROTO_TCP, TCP_KEEPINTVL, cfg.interval_seconds, "TCP_KEEPINTVL");
        set_option(sockfd, IPPROTO_TCP, TCP_KEEPCNT, cfg.max_probes, "TCP_KEEPCNT");
    }

    static void disable(int sockfd) {
        set_option(sockfd, SOL_SOCKET, SO_KEEPALIVE, 0, "SO_KEEPALIVE");
    }

    static Config get_config(int sockfd) {
        Config cfg;
        cfg.idle_seconds = get_option(sockfd, IPPROTO_TCP, TCP_KEEPIDLE);
        cfg.interval_seconds = get_option(sockfd, IPPROTO_TCP, TCP_KEEPINTVL);
        cfg.max_probes = get_option(sockfd, IPPROTO_TCP, TCP_KEEPCNT);
        return cfg;
    }

    static bool is_enabled(int sockfd) {
        return get_option(sockfd, SOL_SOCKET, SO_KEEPALIVE) != 0;
    }

private:
    static void set_option(int fd, int level, int optname, int value, const char* name) {
        if (setsockopt(fd, level, optname, &value, sizeof(value)) < 0) {
            throw std::runtime_error(std::string("setsockopt ") + name + " failed");
        }
    }

    static int get_option(int fd, int level, int optname) {
        int value;
        socklen_t len = sizeof(value);
        if (getsockopt(fd, level, optname, &value, &len) < 0) {
            throw std::runtime_error("getsockopt failed");
        }
        return value;
    }
};
```

### （5）快速验证 KeepAlive 是否生效
```bash
# 查看某个连接的 KeepAlive 状态
ss -tno state established | grep <目标IP>

# 输出中 timer:(keepalive,...) 表示已启用
```

## 3.核心作用
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113102020940.png)

### （1）各作用解析
- “**Redis++的心跳机制**”就是直接使用“**Linux TCP的心跳机制**”，下面我们以Redis++为使用场景进行讲解

#### <1>检测"半死连接"（最核心）
**问题场景**：
```plaintext
Redis 服务器异常（断电/进程被杀/网线断开）
         ↓
客户端未收到 FIN/RST 包，误认为连接存活
         ↓
发送请求 → 永久阻塞/超时，业务异常
```
**心跳解决逻辑**：
```plaintext
定期发探测包 → 对端无响应 → 判定连接死亡 → 关闭连接 → 触发重连
```

#### <2>防止中间设备关闭空闲连接
- 中间设备（如防火墙、负载均衡器）的核心资源是**「连接表项」**
	- 每维护一个 TCP 连接，都需要占用内存存储连接四元组（源 IP、源端口、目的 IP、目的端口）、连接状态（ESTABLISHED/CLOSED 等）、超时计时器等信息。
- 但设备的内存、连接表容量是**有限的**（比如一台普通防火墙可能仅支持数万到数十万并发连接）
	- 如果长期保留大量空闲连接，会耗尽连接表资源，导致无法接收新的正常连接，直接影响网络服务可用性。
- 对中间设备而言，「空闲连接」意味着长时间没有数据交互，属于 “低价值” 连接，保留这类连接的收益极低；

**问题场景**：
```plaintext
客户端 ←──→ [防火墙/NAT/云负载均衡] ←──→ Redis 服务器
                    │
                    │ 空闲超阈值 → 直接断开（不通知两端）
                    ↓
客户端下次请求 → 连接失效 → 报错
```
- 常见设备空闲超时阈值:
	- ![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113102652571.png)

**心跳解决逻辑**：
```plaintext
按小于超时阈值的频率发探测包（如每30秒）→ 中间设备判定连接活跃 → 不被断开
```

#### <3>及时回收连接池资源
**问题场景**：
```plaintext
连接池含无效连接 → 高并发时拿到死连接 → 请求失败→重试 → 性能下降
```

**心跳解决逻辑**：
```plaintext
检测到死连接 → 从池移除 → 创建新连接补充 → 连接池全为有效连接
```

### （2）完整交互流程图（简化版）
```plaintext
┌─────────────┐                              ┌─────────────┐
│  Redis++    │                              │   Redis     │
│  Client     │                              │  Server     │
└──────┬──────┘                              └──────┬──────┘
       │                                            │
       │ ══════ 正常业务交互 ══════                  │
       │ ─ ─ ─ 连接空闲（如30秒） ─ ─ ─              │
       │  💓 发送KeepAlive探测包                     │
       │───────────────────────────────────────────▶│
       │◀───────────────────────────────────────────│
       │         ✅ ACK响应（服务器正常）             │
       │ ─ ─ ─ 连接继续保持，重置空闲计时 ─ ─ ─       │
       │                                            │
       │ ═════════ 异常场景：服务器崩溃 ═════════     │
       │  💓 发送KeepAlive探测包                     │
       │───────────────────────────────────────────▶│ (无响应)
       │  💓 重试探测 × 9次                          │
       │───────────────────────────────────────────▶│ (仍无响应)
       │  ❌ 判定连接死亡                            │
       │  → 关闭socket → 从连接池移除 → 自动重连      │
       ▼                                            ▼
```

### （3）“开启心跳机制” vs “不开心跳机制”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113103013458.png)

## 4.注意事项与建议
### （1）系统级 vs 应用级配置优先级
```plaintext
┌───────────────────────────────────────────────────────┐
│              配置优先级（从高到低）                     │
├───────────────────────────────────────────────────────┤
│  1. setsockopt() 应用级配置     ← 最高优先级          │
│  2. /etc/sysctl.conf 永久配置                         │
│  3. 内核默认值                  ← 最低优先级          │
└───────────────────────────────────────────────────────┘
```
**注意!!**：应用级配置仅影响当前 socket，不影响其他连接

### （2）生产环境配置建议
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113111046526.png)

### （3）常见问题
```cpp
// ❌ 错误：只开启 KeepAlive，不设置参数
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &flag, sizeof(flag));
// 结果：使用系统默认值（7200秒），2小时后才开始检测，生产环境太慢！

// ✅ 正确：开启后必须设置参数
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &flag, sizeof(flag));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &idle, sizeof(idle));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &interval, sizeof(interval));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &count, sizeof(count));

```

## 5.补充
### （1）探测包
#### <1>KeepAlive 探测包的结构特征
**KeepAlive 探测包**是一个**特殊的 TCP 包**：
- **序列号** = 上一次发送的最后字节序号 - 1（故意"倒退"）
- **数据长度** = 0 字节
- **ACK 标志** = 1
- **目的**：触发对端回复一个 ACK，以此确认连接存活


#### <2>抓包命令
```bash
# 使用 tcpdump 抓取
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-ack != 0' -nn

# Wireshark 过滤器
tcp.analysis.keep_alive
```

### （2）KeepAlive 的局限性
**局限性**：
- **检测延迟高**：即使优化参数，最快也要几十秒才能发现死连接
- **无法检测应用层故障**：对端进程死锁但 TCP 栈正常时，KeepAlive 仍会成功
- **跨 NAT 场景**：某些 NAT 设备会"代答" KeepAlive，导致检测失效

**解决方案**：
- 关键业务可增加**应用层心跳**（如 Redis 的 PING 命令）
- 结合 `socket_timeout` 设置读写超时

### （3）“TCP KeepAlive” vs “应用层心跳”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113112324791.png)