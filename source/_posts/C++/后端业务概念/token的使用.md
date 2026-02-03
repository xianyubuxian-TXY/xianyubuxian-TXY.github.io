---
title: Token的使用
date: 2026-01-29 22:57:39
tags:
    - 后端业务相关
categories:
	- C++
	- 后端业务概念
	- Token的使用
---

# 一、为什么需要Token？
## http的无状态
### （1）什么是无状态？
- **无状态**意味着 HTTP 协议对于事务处理没有记忆能力。
- 每个请求都是**独立的**，服务器不会保留之前请求的任何信息。
```plaintext
请求1: GET /page1  →  服务器响应（然后忘记）
请求2: GET /page2  →  服务器响应（不知道请求1的存在）
```

### （2）为什么设计成无状态？
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129230810641.png)

### （3）无状态带来的问题
- 考虑一个常见场景：用户登录后，点击头像访问个人主页
	- **登录操作**：发送「HTTP 请求1」
	- **访问个人主页**：发送「HTTP 请求2」
- 由于 **HTTP 无状态**，「请求1」和「请求2」之间**相互独立**、**互不关联**
	- 服务器收到「请求2」时，**无法识别请求者身份**，因此无法返回对应的用户信息
- **直接在请求中携带用户名可行吗？**
	- 显然不行：服务器无法验证该用户名是否为本人，任何人都可以冒充

### （4）如何解决“无状态”的问题呢？
- 用户登录时，执行以下操作：
	- 服务器生成一个**唯一标识**，将 **<唯一标识, 用户ID> 的映射关系**存入数据库
	- 服务器将该「唯一标识」返回给客户端
	- 客户端将「唯一标识」保存在本地缓存中
- **后续操作时**（如访问个人主页）：
	- 客户端的每次 HTTP 请求都**携带该「唯一标识」**
	- 服务器收到请求后，解析出「唯一标识」，并查询数据库获取对应的「用户ID」
		- ✅ 查询成功 → 返回用户信息
		- ❌ 查询失败 → 返回错误响应
- **为什么这种方式有效？**
	- 该「唯一标识」由服务器生成并下发，只有合法登录的用户才能获得
	- 客户端凭借此标识向服务器证明身份，从而获取相应的用户数据
- **Token 正是这种「唯一标识」的一种实现方式**

# 二、Token详解
## 1.什么是Token？
- **Token**（令牌）是服务器生成的一串**加密字符串**，作为客户端身份的凭证
- 客户端持有 Token，就像持有一张「通行证」，用于**证明自己的身份**

## 2.Token的工作流程
```plaintext
┌──────────────┐                                    ┌──────────────┐
│    客户端     │                                    │    服务器     │
└──────┬───────┘                                    └──────┬───────┘
       │                                                   │
       │  ① 登录请求（用户名 + 密码）                        │
       │ ─────────────────────────────────────────────────→│
       │                                                   │ 验证账号密码
       │                                                   │ 生成 Token
       │  ② 返回 Token                                     │
       │ ←─────────────────────────────────────────────────│
       │                                                   │
       │  ③ 请求数据（携带 Token）                          │
       │ ─────────────────────────────────────────────────→│
       │                                                   │ 验证 Token
       │  ④ 返回请求的数据                                  │
       │ ←─────────────────────────────────────────────────│
```

## 3.Token的分类
### （1）普通 Token（传统方式）
- **生成方式**：服务器生成随机字符串作为 Token
- **存储方式**：服务器将 <Token, 用户信息> 存入数据库或 Redis
- **验证方式**：每次请求都需要查询数据库验证 Token
```plaintext
登录 → 服务器生成随机串「abc123」→ 存入数据库 → 返回给客户端
请求 → 携带「abc123」→ 服务器查库验证 → 返回数据
```
**缺点**：
- 每次请求都要查库，性能开销大
- 分布式环境下需要共享存储

### （2）JWT（JSON Web Token）⭐ 主流方案
- **核心思想**：Token 本身就携带用户信息，服务器无需存储，只需验证签名
- **结构**：由三部分组成，用 . 连接
```plaintext
Header.Payload.Signature
  │       │        │
  │       │        └── 签名（防篡改）
  │       └── 载荷（用户信息）
  └── 头部（算法信息）
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129233733914.png)

## 4.为什么 Token 不能被伪造？
```plaintext
┌─────────────────────────────────────────────────────────┐
│  服务器持有一个「密钥」（Secret Key），仅服务器知道       │
└─────────────────────────────────────────────────────────┘
                           ↓
          Token = 用户信息 + 用密钥生成的签名
                           ↓
        客户端可以看到用户信息（Base64 可解码）
        但无法伪造签名（因为没有密钥）
                           ↓
          服务器收到 Token → 用密钥重新计算签名
                           ↓
                ┌─────────────────────┐
                │ 签名匹配？           │
                ├──────────┬──────────┤
                │ ✅ 匹配   │ ❌ 不匹配 │
                │ Token有效 │ Token被篡改│
                └──────────┴──────────┘
```

## 5.Token 的使用方式
```bash
# 方式一：放在请求头（推荐）
GET /api/user HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# 方式二：放在请求参数（不推荐，会被记录在日志中）
GET /api/user?token=eyJhbGciOiJIUzI1NiIs...
```

# 三、双Token机制详解
## 1.为什么需要双 Token？
### （1）单 Token 的困境
假设只使用一个Token，会面临一个**安全与体验的矛盾**：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260130000046323.png)
```bash
【单 Token 的两难选择】

方案A：Token 有效期 30 分钟
├── 用户刷个微博 → Token 过期 → 重新登录
├── 用户吃个饭回来 → Token 过期 → 重新登录
└── 体验极差 ❌

方案B：Token 有效期 7 天
├── Token 被盗 → 攻击者可使用 7 天
├── 无法主动让 Token 失效（无状态特性）
└── 安全风险高 ❌
```

### （2）双 Token 的解决方案
引入两种不同职责的 Token：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260130000103882.png)
```cpp
【核心思想】
├── Access Token：短命但频繁使用 → 即使泄露，危害时间短
└── Refresh Token：长命但很少使用 → 减少暴露机会
```

## 2.双 Token 的工作流程
### （1）完整流程图
```plaintext
┌──────────────┐                                         ┌──────────────┐
│    客户端     │                                         │    服务器     │
└──────┬───────┘                                         └──────┬───────┘
       │                                                        │
       │  ①【登录】用户名 + 密码                                 │
       │ ──────────────────────────────────────────────────────→│
       │                                                        │ 验证用户
       │                                                        │ 生成双 Token
       │  ② 返回 Access Token + Refresh Token                   │
       │ ←──────────────────────────────────────────────────────│
       │                                                        │
       │     ┌─────────────────────────────────────────────┐    │
       │     │ 客户端存储：                                  │    │
       │     │ • Access Token → 内存/sessionStorage        │    │
       │     │ • Refresh Token → HttpOnly Cookie/安全存储  │    │
       │     └─────────────────────────────────────────────┘    │
       │                                                        │
       │  ③【访问资源】携带 Access Token                         │
       │ ──────────────────────────────────────────────────────→│
       │                                                        │ 验证 Access Token
       │  ④ 返回请求的数据                                       │
       │ ←──────────────────────────────────────────────────────│
       │                                                        │
       │         ... 重复步骤 ③④，直到 Access Token 过期 ...      │
       │                                                        │
       │  ⑤【Access Token 过期】请求被拒绝（401）                 │
       │ ←──────────────────────────────────────────────────────│
       │                                                        │
       │  ⑥【刷新】携带 Refresh Token                            │
       │ ──────────────────────────────────────────────────────→│
       │                                                        │ 验证 Refresh Token
       │                                                        │ 生成新的双 Token
       │  ⑦ 返回新的 Access Token + Refresh Token               │
       │ ←──────────────────────────────────────────────────────│
       │                                                        │
       │         ... 继续使用新 Token 访问资源 ...                │
       │                                                        │

```

### （2）时间线视角
```cpp
时间 ─────────────────────────────────────────────────────────────────────→

     登录                                    Access Token 过期
      ↓                                            ↓
      ├─────────── Access Token 有效期（30分钟）────┤
      │                                            │
      │  [请求1] [请求2] [请求3] ... [请求N]        │  [请求被拒绝 401]
      │     ↓       ↓       ↓           ↓          │         ↓
      │   成功    成功    成功        成功         │    用 Refresh Token 刷新
      │                                            │         ↓
      │                                            │    获得新 Access Token
      │                                            │         ↓
      │                                            ├─────────────────────→
      │                                            │   继续访问（新Token）
      │                                            │
      ├─────────────── Refresh Token 有效期（7天）─────────────────────────┤
      │                                                                   │
      │                                               Refresh Token 过期   │
      │                                                      ↓            │
      │                                               必须重新登录         │
```

## 3.Access Token vs Refresh Token 对比
### （1）存储策略
#### <1>Access Token 存储策略
- **不存储，自包含**
```cpp
┌─────────────────────────────────────────────────────────────┐
│                   Access Token - 无状态验证                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   存储位置：❌ 不存储（服务端）                                │
│                                                             │
│   原因：                                                     │
│   ├── JWT 是自包含的（Self-contained）                       │
│   ├── 签名验证 = 确认令牌未被篡改                             │
│   ├── 过期检查 = 检查 payload 中的 exp 字段                   │
│   └── 用户信息 = 直接从 payload 解析                          │
│                                                             │
│   验证流程：纯内存计算，无 IO 开销                             │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│   │ 解析JWT  │ -> │ 验证签名  │ -> │ 检查过期  │ -> 通过       │
│   └──────────┘    └──────────┘    └──────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### <2>Refresh Token 存储策略
```cpp
┌─────────────────────────────────────────────────────────────┐
│                  Refresh Token - 有状态管理                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   存储位置：✅ MySQL（user_sessions 表）                      │
│                                                             │
│   存储内容：Token 的 SHA256 哈希值（非原文）                   │
│                                                             │
│   原因：                                                     │
│   ├── 支持主动注销（登出时删除记录）                           │
│   ├── 支持"登出所有设备"功能                                  │
│   ├── 可限制同时在线设备数量                                  │
│   └── 存哈希防止数据库泄露时 Token 被盗用                      │
│                                                             │
│   验证流程：签名验证 + 数据库查询                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│   │ 验证签名  │ -> │ 计算哈希  │ -> │ 查询DB   │ -> 通过       │
│   └──────────┘    └──────────┘    └──────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```


### （2）对比表格
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260130092554246.png)

###（3）可选：强制失效机制（版本号）
```cpp
┌─────────────────────────────────────────────────────────────┐
│                      Token 版本号机制                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   users 表新增字段：token_version INT DEFAULT 1              │
│                                                             │
│   Access Token payload 携带版本号：                          │
│   {                                                         │
│     "uid": "12345",                                         │
│     "ver": 1,        <-- 版本号                              │
│     "exp": 1735689600                                       │
│   }                                                         │
│                                                             │
│   验证时：                                                   │
│   1. 验证签名                                                │
│   2. 从 Redis 获取当前 token_version                         │
│   3. 比对版本号是否一致                                       │
│                                                             │
│   强制失效时：                                                │
│   1. 数据库 token_version++                                  │
│   2. 更新 Redis 缓存                                         │
│   3. 删除所有 Refresh Token                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
```cpp
                        客户端请求
                            │
                            ▼
                    ┌───────────────┐
                    │   API 网关     │
                    └───────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
      携带 Access Token            携带 Refresh Token
              │                           │
              ▼                           ▼
      ┌───────────────┐           ┌───────────────┐
      │   验证签名     │           │   验证签名     │
      │   (内存计算)   │           │   (内存计算)   │
      └───────┬───────┘           └───────┬───────┘
              │                           │
              ▼                           ▼
      ┌───────────────┐           ┌───────────────┐
      │  [可选]检查    │           │  查询数据库    │
      │  Redis 版本号  │           │  验证有效性    │
      └───────┬───────┘           └───────┬───────┘
              │                           │
              ▼                           ▼
         访问业务接口               签发新 Token 对

```

## 4.为什么双Token更安全？
### （1）攻击面分析
```plaintext
┌─────────────────────────────────────────────────────────────────────────┐
│                           攻击场景分析                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  【场景1】Access Token 被盗                                              │
│  ├── 攻击者能做什么？访问用户资源                                         │
│  ├── 危害持续多久？最多 30 分钟（Token 过期后失效）                        │
│  ├── 用户需要做什么？等待过期即可，或修改密码强制失效                       │
│  └── 结论：危害有限且可控 ✅                                              │
│                                                                         │
│  【场景2】Refresh Token 被盗                                             │
│  ├── 攻击者能做什么？获取新的 Access Token                                │
│  ├── 如何防范？                                                         │
│  │   ├── Refresh Token 仅在刷新接口使用，暴露机会少                       │
│  │   ├── 使用 HttpOnly Cookie 存储，防止 XSS 窃取                        │
│  │   ├── 刷新时检测异常（IP变化、设备变化）                               │
│  │   └── 实现 Token Rotation（每次刷新都生成新的 Refresh Token）          │
│  └── 用户发现异常后：修改密码 → 所有 Token 失效                           │
│                                                                         │
│  【场景3】同时被盗                                                       │
│  ├── 概率较低（存储位置不同）                                             │
│  └── 解决方案：修改密码 + 强制下线所有设备                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### （2）Token Rotation（令牌轮换）
```plaintext
【Token Rotation 机制】

每次使用 Refresh Token 刷新时：
├── 生成新的 Access Token ✅
├── 生成新的 Refresh Token ✅（关键！）
└── 将旧的 Refresh Token 加入黑名单

好处：
├── 即使 Refresh Token 被盗，攻击者使用后，原用户的 Token 也会失效
├── 原用户再次刷新时会失败 → 发现异常 → 重新登录 → 攻击者 Token 失效
└── 形成"抢占"机制，增加攻击难度
```
```plaintext
时间线示意：

正常用户持有：RT-1 (Refresh Token)
攻击者盗取：RT-1

      攻击者先使用 RT-1 刷新
              ↓
      服务器返回新 Token：AT-2, RT-2
      服务器将 RT-1 加入黑名单
              ↓
      正常用户使用 RT-1 刷新
              ↓
      服务器：RT-1 在黑名单中 → 拒绝 → 要求重新登录
              ↓
      用户发现异常，修改密码
              ↓
      攻击者的 RT-2 也失效
```

## 5.Token 刷新的时机与策略
### （1）被动刷新（常见）
```cpp
【策略】Access Token 过期后再刷新

客户端                                     服务器
   │                                          │
   │  请求资源（携带过期的 Access Token）       │
   │ ────────────────────────────────────────→│
   │                                          │
   │  返回 401 Unauthorized                    │
   │ ←────────────────────────────────────────│
   │                                          │
   │  检测到 401，自动用 Refresh Token 刷新    │
   │ ────────────────────────────────────────→│
   │                                          │
   │  返回新 Token                             │
   │ ←────────────────────────────────────────│
   │                                          │
   │  用新 Token 重新发起原请求                 │
   │ ────────────────────────────────────────→│
```

### （2）主动刷新（更优）
```cpp
【策略】Access Token 即将过期时提前刷新

                Access Token 有效期 30 分钟
├────────────────────────────────────────────────┤
│                                                │
│  正常使用区域            │   提前刷新区域       │
│      (25分钟)           │     (最后5分钟)      │
│                         │                      │
│  ────────────────────── │ ─────────────────── │
│                         ↑                      │
│                    检测到即将过期               │
│                    后台静默刷新                 │
│                    用户无感知                   │
```


# 四、JWT 在 C++ 后端的使用
## 1.JWT的介绍
### （1）什么是JWT？
- JWT（JSON Web Token）是一种开放标准（RFC 7519），用于在各方之间安全传输信息
- JWT 是一个自包含的令牌，**本身携带用户信息**，服务器无需存储会话状态

### （2）JWT的结构
JWT 由三部分组成，用 . 连接：
```plaintext
xxxxx.yyyyy.zzzzz
  │      │     │
  │      │     └── Signature（签名）
  │      └── Payload（载荷）
  └── Header（头部）
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129233733914.png)

### （3） JWT 的工作原理
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                         JWT 签名验证原理                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【生成 Token】                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Header (JSON)  +  Payload (JSON)  +  Secret Key        │  │
│   │       ↓                ↓                  ↓              │  │
│   │   Base64Url        Base64Url          HMAC-SHA256       │  │
│   │       ↓                ↓                  ↓              │  │
│   │   "eyJhbG..."  +  "eyJ1c2Vy..."  +  签名运算             │  │
│   │                                          ↓              │  │
│   │                                    "SflKxwRJ..."        │  │
│   │                                                         │  │
│   │   最终 Token = Header.Payload.Signature                 │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   【验证 Token】                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  收到 Token → 分离出 Header.Payload 和 Signature         │  │
│   │       ↓                                                 │  │
│   │  用同样的 Secret Key 对 Header.Payload 重新计算签名       │  │
│   │       ↓                                                 │  │
│   │  比较「计算出的签名」与「Token 中的 Signature」           │  │
│   │       ↓                                                 │  │
│   │  相同 → ✅ 有效     不同 → ❌ 被篡改                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 2.JWT 库的核心伪代码
### （1）Base64Url 编码/解码
```cpp
/**
 * Base64Url 编码（与标准 Base64 的区别）
 * - 将 '+' 替换为 '-'
 * - 将 '/' 替换为 '_'
 * - 去掉末尾的 '=' 填充
 */

// 标准 Base64 字符表
const char BASE64_CHARS[] = 
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

// Base64Url 字符表
const char BASE64URL_CHARS[] = 
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_";

std::string base64url_encode(const std::string& input) {
    std::string output;
    int val = 0, valb = -6;
    
    for (unsigned char c : input) {
        val = (val << 8) + c;
        valb += 8;
        while (valb >= 0) {
            output.push_back(BASE64URL_CHARS[(val >> valb) & 0x3F]);
            valb -= 6;
        }
    }
    
    if (valb > -6) {
        output.push_back(BASE64URL_CHARS[((val << 8) >> (valb + 8)) & 0x3F]);
    }
    
    // 注意：Base64Url 不添加 '=' 填充
    return output;
}

std::string base64url_decode(const std::string& input) {
    // 构建反向查找表
    std::vector<int> lookup(256, -1);
    for (int i = 0; i < 64; i++) {
        lookup[BASE64URL_CHARS[i]] = i;
    }
    
    std::string output;
    int val = 0, valb = -8;
    
    for (char c : input) {
        if (lookup[c] == -1) break;
        val = (val << 6) + lookup[c];
        valb += 6;
        if (valb >= 0) {
            output.push_back(char((val >> valb) & 0xFF));
            valb -= 8;
        }
    }
    
    return output;
}
```

### （2）HMAC-SHA256 签名
```cpp
/**
 * HMAC-SHA256 签名算法伪代码
 * 实际使用 OpenSSL 库实现
 */

#include <openssl/hmac.h>
#include <openssl/sha.h>

std::string hmac_sha256(const std::string& key, const std::string& data) {
    unsigned char hash[SHA256_DIGEST_LENGTH];
    unsigned int hash_len;
    
    HMAC(
        EVP_sha256(),                           // 使用 SHA256 算法
        key.c_str(), key.length(),              // 密钥
        (unsigned char*)data.c_str(), data.length(),  // 待签名数据
        hash, &hash_len                         // 输出
    );
    
    // 将二进制哈希转换为字符串
    return std::string((char*)hash, hash_len);
}
```

### （3）JWT 生成核心逻辑
```cpp
/**
 * JWT 生成伪代码
 */

std::string createJWT(
    const std::string& secret_key,
    int64_t user_id,
    const std::string& username,
    const std::string& role,
    int64_t expire_timestamp
) {
    // ========== 1. 构建 Header ==========
    // JSON 格式
    std::string header_json = R"({"alg":"HS256","typ":"JWT"})";
    // Base64Url 编码
    std::string header_encoded = base64url_encode(header_json);
    
    // ========== 2. 构建 Payload ==========
    // JSON 格式（实际使用 JSON 库构建）
    std::string payload_json = 
        "{\"userId\":" + std::to_string(user_id) + 
        ",\"username\":\"" + username + "\"" +
        ",\"role\":\"" + role + "\"" +
        ",\"exp\":" + std::to_string(expire_timestamp) + 
        ",\"iat\":" + std::to_string(current_timestamp()) + "}";
    // Base64Url 编码
    std::string payload_encoded = base64url_encode(payload_json);
    
    // ========== 3. 生成 Signature ==========
    // 待签名内容 = Header.Payload
    std::string sign_input = header_encoded + "." + payload_encoded;
    // HMAC-SHA256 签名
    std::string signature_raw = hmac_sha256(secret_key, sign_input);
    // Base64Url 编码签名
    std::string signature_encoded = base64url_encode(signature_raw);
    
    // ========== 4. 拼接最终 Token ==========
    return header_encoded + "." + payload_encoded + "." + signature_encoded;
}
```

### （4）JWT 验证核心逻辑
```cpp
/**
 * JWT 验证伪代码
 */

struct JWTPayload {
    int64_t user_id;
    std::string username;
    std::string role;
    int64_t exp;
    int64_t iat;
    bool valid = false;
    std::string error;
};

JWTPayload verifyJWT(const std::string& token, const std::string& secret_key) {
    JWTPayload result;
    
    // ========== 1. 分割 Token ==========
    std::vector<std::string> parts = split(token, '.');
    if (parts.size() != 3) {
        result.error = "Token 格式错误";
        return result;
    }
    
    std::string header_encoded = parts[0];
    std::string payload_encoded = parts[1];
    std::string signature_received = parts[2];
    
    // ========== 2. 重新计算签名 ==========
    std::string sign_input = header_encoded + "." + payload_encoded;
    std::string signature_calculated = base64url_encode(
        hmac_sha256(secret_key, sign_input)
    );
    
    // ========== 3. 比较签名 ==========
    if (signature_calculated != signature_received) {
        result.error = "签名验证失败（Token 可能被篡改）";
        return result;
    }
    
    // ========== 4. 解码 Payload ==========
    std::string payload_json = base64url_decode(payload_encoded);
    // 使用 JSON 库解析（这里伪代码简化）
    auto json = parse_json(payload_json);
    
    result.user_id = json["userId"];
    result.username = json["username"];
    result.role = json["role"];
    result.exp = json["exp"];
    result.iat = json["iat"];
    
    // ========== 5. 检查过期时间 ==========
    if (result.exp < current_timestamp()) {
        result.error = "Token 已过期";
        return result;
    }
    
    result.valid = true;
    return result;
}
```
### （5）核心流程图示
```plaintext
┌─────────────────────────────────────────────────────────────────────┐
│                        JWT 生成流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Header JSON ──────→ Base64Url ──────→ eyJhbGciOiJIUzI1NiJ9        │
│       │                                        │                   │
│       │                                        ├────────┐          │
│       │                                        │        │          │
│  Payload JSON ─────→ Base64Url ──────→ eyJ1c2VySWQiOjEyM30        │
│       │                                        │        │          │
│       │                                        ├────────┤          │
│       │                                        │        ↓          │
│       │                                   Header.Payload           │
│       │                                        │                   │
│       │              Secret Key ───────────────┤                   │
│       │                                        ↓                   │
│       │                                  HMAC-SHA256               │
│       │                                        ↓                   │
│       │                                   Base64Url                │
│       │                                        ↓                   │
│       │                                   Signature                │
│       │                                        │                   │
│       └────────────────────────────────────────┴───────────────┐   │
│                                                                ↓   │
│              最终 Token = Header.Payload.Signature                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```