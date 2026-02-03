---
title: uuid的使用
date: 2026-01-25 15:03:11
tags:
    - 后端业务相关
categories:
	- C++
	- 后端业务概念
	- uuid的使用
---

# 一、UUID简介
## 1.什么是UUID？
UUID（Universally Unique Identifier，**通用唯一识别码**）是一个 128 位（16 字节） 的标识符，用于在**分布式系统中**唯一标识信息，无需中央协调机构即**可保证“全局唯一性”**。

## 2.UUID格式
```bash
标准格式（36 字符）：xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
                     │        │    │    │    │
                     8位──────4位──4位──4位──12位
                              
示例：550e8400-e29b-41d4-a716-446655440000
                    │    │
                    │    N=变体标识（8/9/a/b 表示 RFC 4122）
                    M=版本号（这里是4，表示v4）

二进制格式：
	字节位置:  0  1  2  3   4  5   6  7   8  9  10 11 12 13 14 15
	          └────┬────┘ └──┬──┘ └──┬──┘ └──┬──┘ └──────┬──────┘
	          time_low    time_mid  time_hi  clk_seq    node
	                              +version  +variant
```
## 3.UUID版本对比
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260125151807866.png)
**v1 vs v4**
```cpp
选择 v1（时间戳）：
  ✓ 需要按时间排序
  ✓ 需要知道生成时间
  ✗ 会暴露 MAC 地址（隐私风险）

选择 v4（随机）：
  ✓ 最高随机性
  ✓ 无隐私泄露
  ✓ 最通用
  ✗ 无法排序
```

## 4.唯一性保证
```cpp
UUID v4 碰撞概率分析：

• 128 位中有 122 位随机（6 位用于版本和变体）
• 可能的组合数：2^122 ≈ 5.3 × 10^36
• 要达到 50% 碰撞概率，需生成约 2.71 × 10^18 个 UUID
• 假设每秒生成 10 亿个，需约 85 年才可能碰撞

结论：实际使用中可认为"永不重复"
```

# 二、UUID 使用场景
## 1.数据库主键
```bash
┌─────────────────────────────────────────────────────────┐
│ 传统自增 ID 的问题：                                      │
│ • 分布式环境下需要中央协调                                │
│ • 暴露业务信息（如用户量、注册顺序）                      │
│ • 数据迁移/合并时容易冲突                                 │
├─────────────────────────────────────────────────────────┤
│ UUID 作为主键的优势：                                     │
│ • 本地生成，无需协调                                      │
│ • 不暴露业务信息                                          │
│ • 天然支持分库分表                                        │
│ • 数据迁移无冲突                                          │
└─────────────────────────────────────────────────────────┘
```

## 2.分布式系统
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260125151925344.png)

## 3.文件/资源标识
```bash
应用示例：

• 上传文件重命名：user_avatar.jpg → 550e8400-e29b-41d4.jpg
  - 避免文件名冲突
  - 隐藏原始文件名

• 临时文件命名：避免并发冲突
• 缓存 Key 生成
• CDN 资源版本标识
```

## 4.安全相关
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260125152004200.png)

# 三、libuuid
## 1.libuuid概述
- libuuid 是一个 C 语言库，用于生成和解析 UUID（通用唯一识别码），是 **Linux 系统**中最常用的 UUID 实现。

**优缺点**：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260125153240377.png)

## 2.libuuid的核心伪代码
### （1）核心数据结构
```cpp
// uuid_t 定义（16 字节数组）
typedef unsigned char uuid_t[16];

// UUID 内部结构布局
struct uuid_layout {
    uint32_t time_low;          // 字节 0-3：时间戳低位
    uint16_t time_mid;          // 字节 4-5：时间戳中位
    uint16_t time_hi_version;   // 字节 6-7：时间戳高位 + 版本号
    uint8_t  clk_seq_hi_variant;// 字节 8：时钟序列高位 + 变体
    uint8_t  clk_seq_low;       // 字节 9：时钟序列低位
    uint8_t  node[6];           // 字节 10-15：节点标识（MAC地址）
};
```
- **uuid_t是一个数组**

## （2）uuid_generate_random（v4 随机生成）
```cpp
// 核心伪代码：生成随机 UUID (v4)
void uuid_generate_random(uuid_t out) {
    // 1. 用随机数填充全部 16 字节
    //    从 /dev/urandom 或 getrandom() 读取
    int fd = open("/dev/urandom", O_RDONLY);
    read(fd, out, 16);
    close(fd);
    
    // 2. 设置版本号 = 4（第 7 字节的高 4 位）
    //    原始: xxxxxxxx
    //    结果: 0100xxxx（版本 4）
    out[6] = (out[6] & 0x0F) | 0x40;
    
    // 3. 设置变体标识（第 9 字节的高 2 位 = 10）
    //    原始: xxxxxxxx
    //    结果: 10xxxxxx（RFC 4122 变体）
    out[8] = (out[8] & 0x3F) | 0x80;
}
```

## （3）uuid_generate_time（v1 时间戳生成
```cpp
// 核心伪代码：生成时间戳 UUID (v1)
void uuid_generate_time(uuid_t out) {
    // 1. 获取当前时间戳（100纳秒精度，从1582年10月15日起）
    uint64_t timestamp = get_uuid_timestamp();
    
    // 2. 填充时间字段
    out[0] = (timestamp >>  0) & 0xFF;  // time_low
    out[1] = (timestamp >>  8) & 0xFF;
    out[2] = (timestamp >> 16) & 0xFF;
    out[3] = (timestamp >> 24) & 0xFF;
    out[4] = (timestamp >> 32) & 0xFF;  // time_mid
    out[5] = (timestamp >> 40) & 0xFF;
    out[6] = (timestamp >> 48) & 0x0F;  // time_hi（低4位）
    
    // 3. 设置版本号 = 1
    out[6] |= 0x10;  // 0001xxxx
    
    // 4. 获取或生成时钟序列（防止时间回拨导致重复）
    uint16_t clock_seq = get_clock_sequence();
    out[8] = (clock_seq >> 8) & 0x3F;
    out[8] |= 0x80;  // 设置变体
    out[9] = clock_seq & 0xFF;
    
    // 5. 填充节点标识（MAC 地址）
    uint8_t mac[6];
    get_mac_address(mac);
    memcpy(out + 10, mac, 6);
}
```

## （4）uuid_generate（自动选择生成）
```cpp
// 核心伪代码：自动选择最佳方式生成 UUID
void uuid_generate(uuid_t out) {
    // 尝试生成 v1（时间戳），失败则回退到 v4（随机）
    
    // 1. 首先尝试获取 MAC 地址
    uint8_t mac[6];
    bool has_mac = get_mac_address(mac);  // 从网卡获取
    
    if (has_mac) {
        // ========== 生成 UUID v1（时间戳）==========
        
        // 2. 获取当前时间戳
        //    UUID 时间戳：从 1582-10-15 00:00:00 起的 100 纳秒间隔数
        uint64_t timestamp = get_uuid_timestamp();
        
        // 3. 填充时间字段（小端序）
        out[0] = (timestamp >>  0) & 0xFF;  // time_low (字节 0-3)
        out[1] = (timestamp >>  8) & 0xFF;
        out[2] = (timestamp >> 16) & 0xFF;
        out[3] = (timestamp >> 24) & 0xFF;
        out[4] = (timestamp >> 32) & 0xFF;  // time_mid (字节 4-5)
        out[5] = (timestamp >> 40) & 0xFF;
        out[6] = (timestamp >> 48) & 0x0F;  // time_hi (字节 6, 低4位)
        
        // 4. 设置版本号 = 1（第 7 字节的高 4 位）
        //    原始: 0000xxxx
        //    结果: 0001xxxx（版本 1）
        out[6] |= 0x10;
        
        // 5. 获取/生成时钟序列（防止时间回拨导致重复）
        static uint16_t clock_seq = random() & 0x3FFF;  // 14 位随机数
        static uint64_t last_time = 0;
        
        if (timestamp <= last_time) {
            clock_seq = (clock_seq + 1) & 0x3FFF;  // 时间回拨，递增序列
        }
        last_time = timestamp;
        
        // 6. 填充时钟序列（字节 8-9）
        out[8] = (clock_seq >> 8) & 0x3F;  // clk_seq_hi (高6位)
        out[9] = clock_seq & 0xFF;          // clk_seq_low
        
        // 7. 设置变体标识（第 9 字节的高 2 位 = 10）
        //    原始: 00xxxxxx
        //    结果: 10xxxxxx（RFC 4122 变体）
        out[8] |= 0x80;
        
        // 8. 填充节点标识 = MAC 地址（字节 10-15）
        memcpy(out + 10, mac, 6);
        
    } else {
        // ========== 回退：生成 UUID v4（随机）==========
        uuid_generate_random(out);
    }
}
```
```cpp
                    uuid_generate()
                          │
                          ▼
               ┌──────────────────────┐
               │  尝试获取 MAC 地址    │
               └──────────┬───────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
           成功 ✓                   失败 ✗
              │                       │
              ▼                       ▼
    ┌─────────────────┐     ┌─────────────────┐
    │  生成 UUID v1   │     │  生成 UUID v4   │
    │  (时间戳+MAC)   │     │  (纯随机)       │
    └─────────────────┘     └─────────────────┘
              │                       │
              └───────────┬───────────┘
                          ▼
                     返回 uuid_t
```

## （5）uuid_unparse_lower（转字符串）
```cpp
// 核心伪代码：UUID 转字符串
void uuid_unparse_lower(const uuid_t uu, char* out) {
    static const char hex_lower[] = "0123456789abcdef";
    
    int pos = 0;
    for (int i = 0; i < 16; i++) {
        // 每字节转为 2 个十六进制字符
        out[pos++] = hex_lower[(uu[i] >> 4) & 0x0F];
        out[pos++] = hex_lower[uu[i] & 0x0F];
        
        // 在特定位置插入连字符 (4-2-2-2-6 格式)
        if (i == 3 || i == 5 || i == 7 || i == 9) {
            out[pos++] = '-';
        }
    }
    out[pos] = '\0';  // 共 36 字符 + 结尾符
}
```

## （6）uuid_parse（解析字符串）
```cpp
// 核心伪代码：字符串转 UUID
int uuid_parse(const char* in, uuid_t out) {
    int idx = 0;
    
    for (int i = 0; i < 36; i++) {
        char c = in[i];
        
        // 跳过连字符
        if (c == '-') {
            if (i != 8 && i != 13 && i != 18 && i != 23) {
                return -1;  // 连字符位置错误
            }
            continue;
        }
        
        // 十六进制字符转数值
        uint8_t val;
        if (c >= '0' && c <= '9')      val = c - '0';
        else if (c >= 'a' && c <= 'f') val = c - 'a' + 10;
        else if (c >= 'A' && c <= 'F') val = c - 'A' + 10;
        else return -1;  // 无效字符
        
        // 每两个字符组成一
        if (idx % 2 == 0) {
            out[idx / 2] = val << 4;
        } else {
            out[idx / 2] |= val;
        }
        idx++;
    }
    
    return 0;  // 成功
}

```

## 3.libuuid的封装与使用
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260125153550290.png)
### （1）C++封装类
```cpp
#ifndef UUID_HELPER_H
#define UUID_HELPER_H

#include <uuid/uuid.h>
#include <string>
#include <cstring>

namespace user_service {

/**
 * UUID 工具类 - 针对用户管理系统
 * 
 * 对应 proto 中的使用场景：
 * - User.id         → UserId()
 * - Token           → Token()
 * - 通用场景        → Generate()
 */
class UUIDHelper {
public:
    // ==================== 基础生成 ====================
    
    /// 生成标准格式 UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    static std::string Generate() {
        uuid_t uuid;
        char str[37];
        uuid_generate_random(uuid);
        uuid_unparse_lower(uuid, str);
        return std::string(str);
    }
    
    /// 生成紧凑格式（无连字符）: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    static std::string GenerateCompact() {
        uuid_t uuid;
        char str[37];
        uuid_generate_random(uuid);
        uuid_unparse_lower(uuid, str);
        
        // 移除连字符
        std::string result;
        result.reserve(32);
        for (int i = 0; i < 36; ++i) {
            if (str[i] != '-') {
                result += str[i];
            }
        }
        return result;
    }
    
    // ==================== 业务专用（对应 proto 字段）====================
    
    /// 用户 ID（对应 User.id）
    /// 格式: usr_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    static std::string UserId() {
        return "usr_" + Generate();
    }
    
    /// 认证 Token（对应 AuthenticateResponse.token）
    /// 格式: 紧凑型，便于传输
    static std::string Token() {
        return GenerateCompact();
    }
    
    /// Session ID（如需会话管理）
    /// 格式: sess_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    static std::string SessionId() {
        return "sess_" + GenerateCompact();
    }
    
    // ==================== 工具方法 ====================
    
    /// 验证 UUID 格式是否正确
    static bool IsValid(const std::string& str) {
        std::string uuid_part = ExtractUUID(str);
        
        // 标准格式 36 字符
        if (uuid_part.length() == 36) {
            uuid_t tmp;
            return uuid_parse(uuid_part.c_str(), tmp) == 0;
        }
        
        // 紧凑格式 32 字符
        if (uuid_part.length() == 32) {
            for (char c : uuid_part) {
                if (!std::isxdigit(static_cast<unsigned char>(c))) {
                    return false;
                }
            }
            return true;
        }
        
        return false;
    }
    
    /// 从带前缀的 ID 中提取纯 UUID 部分
    /// "usr_xxxx-xxxx" → "xxxx-xxxx"
    static std::string ExtractUUID(const std::string& prefixed_id) {
        size_t pos = prefixed_id.find('_');
        if (pos != std::string::npos && pos + 1 < prefixed_id.length()) {
            return prefixed_id.substr(pos + 1);
        }
        return prefixed_id;
    }
    
    /// 判断 ID 类型
    enum class IDType { USER, SESSION, TOKEN, UNKNOWN };
    
    static IDType GetIDType(const std::string& id) {
        if (id.compare(0, 4, "usr_") == 0)  return IDType::USER;
        if (id.compare(0, 5, "sess_") == 0) return IDType::SESSION;
        if (id.length() == 32 && IsValid(id)) return IDType::TOKEN;
        return IDType::UNKNOWN;
    }
};

} // namespace user_service

#endif // UUID_HELPER_H
```

## （2）使用示例
```cpp
// user_service_impl.cpp
#include "uuid_helper.h"
#include "user_service.grpc.pb.h"
#include <iostream>

using namespace user_service;

// 模拟 CreateUser 实现
CreateUserResponse CreateUserImpl(const CreateUserRequest& request) {
    CreateUserResponse response;
    User* user = response.mutable_user();
    
    // 使用封装的 UUID 生成用户 ID
    user->set_id(UUIDHelper::UserId());
    user->set_username(request.username());
    user->set_email(request.email());
    user->set_mobile(request.mobile());
    user->set_display_name(request.display_name());
    // password_hash = hash(request.password()) ...
    
    return response;
}

// 模拟 Authenticate 实现
AuthenticateResponse AuthenticateImpl(const AuthenticateRequest& request) {
    AuthenticateResponse response;
    
    // 验证用户名密码后，生成 Token
    response.set_token(UUIDHelper::Token());
    response.set_expires_in(3600);  // 1小时有效期
    
    return response;
}

int main() {
    std::cout << "========== UUID 使用示例 ==========\n" << std::endl;
    
    // 1. 用户 ID
    std::string user_id = UUIDHelper::UserId();
    std::cout << "User.id:    " << user_id << std::endl;
    
    // 2. Token
    std::string token = UUIDHelper::Token();
    std::cout << "Token:      " << token << std::endl;
    
    // 3. Session ID
    std::string session = UUIDHelper::SessionId();
    std::cout << "Session:    " << session << std::endl;
    
    // 4. 通用 UUID
    std::string uuid = UUIDHelper::Generate();
    std::cout << "通用 UUID:  " << uuid << std::endl;
    
    // 5. 验证
    std::cout << "\n========== 验证测试 ==========\n" << std::endl;
    std::cout << user_id << " → " 
              << (UUIDHelper::IsValid(user_id) ? "✓ 有效" : "✗ 无效") << std::endl;
    std::cout << "invalid-id → " 
              << (UUIDHelper::IsValid("invalid-id") ? "✓ 有效" : "✗ 无效") << std::endl;
    
    // 6. 提取纯 UUID
    std::cout << "\n========== 提取 UUID ==========\n" << std::endl;
    std::cout << "原始: " << user_id << std::endl;
    std::cout << "提取: " << UUIDHelper::ExtractUUID(user_id) << std::endl;
    
    return 0;
}

```