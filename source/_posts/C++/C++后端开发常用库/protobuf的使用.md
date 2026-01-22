---
title: protobuf的使用
date: 2026-01-19 22:04:28
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- protobuf的使用
---

# 一、概述
- Protobuf 是 Google 开发的一种轻量级、高效的结构化数据序列化协议，比 JSON、XML 更紧凑、更快，广泛用于网络通信和数据存储。

## 1.核心优势
- **高效性**：
	- **二进制格式**，序列化后体积小，解析速度快（比 JSON 快 3-5 倍）
- **跨语言**：
	- 支持 C++、Java、Python、Go、JavaScrip 等几乎所有主流语言

- **向后兼容**：
	- 字段可以新增/废弃，旧版本程序能兼容新版本数据

- **强类型**：
	- 通过 .proto 文件定义数据结构，编译后生成强类型代码，减少类型错误

## 2.核心术语
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119221513229.png)

## 3.Proto文件
正常来说，应该先讲Proto文件，但感觉这部分很枯燥，就放在了[附录：Proto文件](#Proto)中，感兴趣的可以先去看一下。


# 二、数据类型
Protobuf 的类型分为两大类：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119222351058.png)

## 1.标量类型
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119222225945.png)

### （1）int32 vs sint32（负数问题）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119222446609.png)

### （2）fixed32 vs int32（大数值）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119222513321.png)

### （3）string vs bytes
```cpp
// string: 会做 UTF-8 校验（某些语言）
// bytes: 纯二进制，无校验

// 场景区分
message File {
    string filename = 1;    // 文件名是文本
    bytes content = 2;      // 文件内容是二进制
}
```

### （4）选型决策树
```cpp
需要存整数？
├─ 纯正数 → uint32 / uint64
├─ 可能有负数？
│   ├─ 是 → sint32 / sint64 ✅
│   └─ 否 → int32 / int64
└─ 值经常很大（>2^28）？
    └─ 是 → fixed32 / fixed64

需要存字符串？
├─ UTF-8 文本 → string
└─ 二进制数据 → bytes

需要存小数？
├─ 高精度（金额/坐标）→ double
└─ 一般精度 → float
```

## 2.复合类型 & 字段规则
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119223758714.png)

<a id="message"></a>
### （1）message（消息）
#### <1>Proto 定义
```cpp
message Address {
    string city = 1;
}

message User {
    int32 id = 1;
    string name = 2;
    Address address = 3;
}
```

#### <2>C++ 生成类型伪代码
```cpp
class Address {
private:
    // ========== 存储字段 ==========
    std::string city_;
    
public:
    // ========== Getter ==========
    const std::string& city() const;
    
    // ========== Setter ==========
    void set_city(const std::string& value);
    void set_city(std::string&& value);
    void set_city(const char* value);
    
    // ========== Clear ==========
    void clear_city();
};

class User {
private:
    // ========== 存储字段 ==========
    int32_t id_;
    std::string name_;
    Address* address_;              // 指针，延迟分配
    uint32_t _has_bits_[1];         // 位图，跟踪 message 字段是否设置
    
public:
    // ========== Getter ==========
    int32_t id() const;
    const std::string& name() const;
    const Address& address() const;
    
    // ========== Setter ==========
    void set_id(int32_t value);
    void set_name(const std::string& value);
    
    // ========== Mutable（获取可修改指针）==========
    std::string* mutable_name();
    Address* mutable_address();        // 不存在则自动创建
    
    // ========== Has（存在性检查）==========
    bool has_address() const;          // 检查 _has_bits_
    
    // ========== Clear（重置为默认值） ==========
    void clear_id();
    void clear_name();
    void clear_address();
    void Clear();
};
```
**注意**：嵌套 message 都是指针存储

#### <3>使用示例
```cpp
#include "user.pb.h"

// 创建并设置
User user;
user.set_id(1001);
user.set_name("Alice");
user.mutable_address()->set_city("Beijing");

// 读取
std::cout << user.id() << std::endl;              // 1001
std::cout << user.name() << std::endl;            // Alice
std::cout << user.address().city() << std::endl;  // Beijing

// 检查嵌套消息
if (user.has_address()) {
    std::cout << "地址已设置" << std::endl;
}

// 序列化 & 反序列化
std::string data = user.SerializeAsString();
User user2;
user2.ParseFromString(data);
```

### （2）enum（枚举）
#### <1>Proto 定义
```cpp
enum Status {
    STATUS_UNKNOWN = 0;
    STATUS_ACTIVE = 1;
    STATUS_DELETED = 2;
}

message Order {
    Status status = 1;
}
```

#### <2>C++ 生成类型伪代码
```cpp
// ========== 枚举定义 ==========
enum Status : int {
    STATUS_UNKNOWN = 0,
    STATUS_ACTIVE = 1,
    STATUS_DELETED = 2
};

// ========== 枚举辅助函数 ==========
// 1.把枚举值转换为可读的字符串名称
const std::string& Status_Name(Status value);
// 2.把字符串解析为枚举值（反向转换）
bool Status_Parse(const std::string& name, Status* value);
// 3.检查整数值是否是合法的枚举值
bool Status_IsValid(int value);

class Order {
private:
    // ========== 存储字段 ==========
    int status_;                       // 枚举存储为 int
    
public:
    // ========== Getter ==========
    Status status() const;
    
    // ========== Setter ==========
    void set_status(Status value);
    
    // ========== Clear ==========
    void clear_status();
};
```

#### <3>使用示例
```cpp
#include "order.pb.h"

// 设置枚举
Order order;
order.set_status(Status::STATUS_ACTIVE);

// 读取并判断
if (order.status() == Status::STATUS_ACTIVE) {
    std::cout << "订单激活中" << std::endl;
}

// 枚举 → 字符串
std::cout << Status_Name(order.status()) << std::endl;  // STATUS_ACTIVE

// 字符串 → 枚举
Status s;
if (Status_Parse("STATUS_DELETED", &s)) {
    order.set_status(s);
}

// 验证值是否有效
if (Status_IsValid(1)) {
    std::cout << "1 是有效的 Status 值" << std::endl;
}
```

### <3>map<K, V>（映射）
#### <1>Proto 定义
```cpp
message Address {
    string city = 1;
}

message Student {
    map<string, int32> scores = 1;
    map<int32, Address> addresses = 2;
}
```

#### <2>C++ 生成类型伪代码
```cpp
class Address {
private:
    // ========== 存储字段 ==========
    std::string city_;
    
public:
    // ========== Getter ==========
    const std::string& city() const;
    
    // ========== Setter ==========
    void set_city(const std::string& value);
    void set_city(std::string&& value);
    void set_city(const char* value);
    
    // ========== Clear ==========
    void clear_city();
};

class Student {
private:
    // ========== 存储字段 ==========
    google::protobuf::Map<std::string, int32_t> scores_;
    google::protobuf::Map<int32_t, Address> addresses_;
    
public:
    // ========== Getter（只读引用）==========
    const google::protobuf::Map<std::string, int32_t>& scores() const;
    const google::protobuf::Map<int32_t, Address>& addresses() const;
    
    // ========== Mutable（可修改指针）==========
    google::protobuf::Map<std::string, int32_t>* mutable_scores();
    google::protobuf::Map<int32_t, Address>* mutable_addresses();
    
    // ========== Clear ==========
    void clear_scores();
    void clear_addresses();
};

// ========== Map 容器类 ==========
template<typename K, typename V>
class Map {
private:
    // ========== 内部存储 ==========
    std::unordered_map<K, V> elements_;  // 类似 unordered_map 实现
    Arena* arena_;                        // 可选的内存池
    
public:
    // ----- Access（访问）-----
    V& operator[](const K& key);
    const V& at(const K& key) const;
    
    // ----- Lookup（查找）-----
    iterator find(const K& key);
    const_iterator find(const K& key) const;
    bool contains(const K& key) const;
    size_t count(const K& key) const;
    
    // ----- Modify（修改）-----
    std::pair<iterator, bool> insert(const value_type& value);
    size_t erase(const K& key);
    void clear();
    
    // ----- Capacity（容量）-----
    size_t size() const;
    bool empty() const;
    
    // ----- Iterate（遍历）-----
    iterator begin();
    iterator end();
};
```

#### <3>使用示例
```cpp
#include "student.pb.h"

Student stu;

// 插入标量 map
(*stu.mutable_scores())["math"] = 95;
(*stu.mutable_scores())["english"] = 88;

// 插入消息 map
(*stu.mutable_addresses())[1].set_city("Beijing");
(*stu.mutable_addresses())[2].set_city("Shanghai");

// 读取
std::cout << stu.scores().at("math") << std::endl;       // 95
std::cout << stu.addresses().at(1).city() << std::endl;  // Beijing

// 遍历
for (const auto& [subject, score] : stu.scores()) {
    std::cout << subject << ": " << score << std::endl;
}

// 查找
auto it = stu.scores().find("math");
if (it != stu.scores().end()) {
    std::cout << "找到: " << it->second << std::endl;
}

// 删除
stu.mutable_scores()->erase("english");

// 大小 & 清空
std::cout << stu.scores().size() << std::endl;  // 1
stu.clear_scores();
```

### （4）oneof（互斥字段）
#### <1>Proto 定义
```cpp
message Contact {
    string id = 1;
    oneof method {
        string email = 2;
        string phone = 3;
        string wechat = 4;
    }
}
```

#### <2>C++ 生成类型伪代码
```cpp
class Contact {
private:
    // ========== 存储字段 ==========
    std::string id_;
    
    // oneof 使用 union 存储（节省内存）
    union MethodUnion {
        std::string* email_;
        std::string* phone_;
        std::string* wechat_;
    } method_;
    
    uint32_t _oneof_case_[1];          // 记录当前设置的是哪个字段
    
public:
    // ========== Oneof Case 枚举 ==========
    enum MethodCase {
        kEmail = 2,
        kPhone = 3,
        kWechat = 4,
        METHOD_NOT_SET = 0
    };
    
    // ========== Case 判断 ==========
    MethodCase method_case() const;
    
    // ========== Getter ==========
    const std::string& id() const;
    const std::string& email() const;
    const std::string& phone() const;
    const std::string& wechat() const;
    
    // ========== Setter（设置一个会清除其他）==========
    void set_id(const std::string& value);
    void set_email(const std::string& value);
    void set_phone(const std::string& value);
    void set_wechat(const std::string& value);
    
    // ========== Mutable ==========
    std::string* mutable_id();
    std::string* mutable_email();
    std::string* mutable_phone();
    std::string* mutable_wechat();
    
    // ========== Has ==========
    bool has_email() const;
    bool has_phone() const;
    bool has_wechat() const;
    
    // ========== Clear ==========
    void clear_id();
    void clear_method();
};
```

#### <3>使用示例
```cpp
#include "contact.pb.h"

Contact c;
c.set_id("user_001");

// 设置 email
c.set_email("alice@example.com");
std::cout << c.email() << std::endl;  // alice@example.com

// 设置 phone → 自动清除 email！
c.set_phone("13800138000");
std::cout << c.has_email() << std::endl;  // false
std::cout << c.phone() << std::endl;      // 13800138000

// 判断当前设置的是哪个
switch (c.method_case()) {
    case Contact::kEmail:
        std::cout << "Email: " << c.email() << std::endl;
        break;
    case Contact::kPhone:
        std::cout << "Phone: " << c.phone() << std::endl;
        break;
    case Contact::kWechat:
        std::cout << "WeChat: " << c.wechat() << std::endl;
        break;
    case Contact::METHOD_NOT_SET:
        std::cout << "未设置联系方式" << std::endl;
        break;
}

// 清除整个 oneof
c.clear_method();
std::cout << (c.method_case() == Contact::METHOD_NOT_SET) << std::endl;  // true
```

### （5）Any（动态类型）
#### <1>Proto 定义
```cpp
import "google/protobuf/any.proto";

message Response {
    google.protobuf.Any data = 1;
}
```

#### <2>C++ 生成类型伪代码
```cpp
namespace google::protobuf {

class Any {
private:
    // ========== 存储字段 ==========
    std::string type_url_;    // 类型标识，如 "type.googleapis.com/User"
    std::string value_;       // 序列化后的二进制数据
    
public:
    // ========== Pack（打包）==========
    template<typename T>
    void PackFrom(const T& message);
    
    template<typename T>
    void PackFrom(const T& message, const std::string& type_url_prefix);
    
    // ========== Unpack（解包）==========
    template<typename T>
    bool UnpackTo(T* message) const;
    
    // ========== Type Check（类型检查）==========
    template<typename T>
    bool Is() const;
    
    // ========== Getter ==========
    const std::string& type_url() const;
    const std::string& value() const;
    
    // ========== Setter ==========
    void set_type_url(const std::string& value);
    void set_value(const std::string& value);
    
    // ========== Clear ==========
    void Clear();
};

}  // namespace google::protobuf

class Response {
private:
    // ========== 存储字段 ==========
    google::protobuf::Any* data_;     // 指针，延迟分配
    uint32_t _has_bits_[1];
    
public:
    // ========== Getter ==========
    const google::protobuf::Any& data() const;
    
    // ========== Mutable ==========
    google::protobuf::Any* mutable_data();
    
    // ========== Has ==========
    bool has_data() const;
    
    // ========== Clear ==========
    void clear_data();
};
```

#### <3>使用示例
```cpp
#include <google/protobuf/any.pb.h>
#include "response.pb.h"
#include "user.pb.h"

// 打包 User 到 Any
User user;
user.set_id(1);
user.set_name("Alice");

Response resp;
resp.mutable_data()->PackFrom(user);

// 获取类型 URL
std::cout << resp.data().type_url() << std::endl;
// 输出: type.googleapis.com/User

// 类型检查 & 解包
if (resp.data().Is<User>()) {
    User u;
    if (resp.data().UnpackTo(&u)) {
        std::cout << u.name() << std::endl;  // Alice
    }
}

// 也可以打包其他类型
Order order;
order.set_status(Status::STATUS_ACTIVE);
resp.mutable_data()->PackFrom(order);

// 现在是 Order 类型
std::cout << resp.data().Is<User>() << std::endl;   // false
std::cout << resp.data().Is<Order>() << std::endl;  // true
```

#### <4>Protobuf Any 的"动态类型"解析
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119230729435.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119230756739.png)

### （6）Timestamp（时间点）
#### <1>Proto 定义
```cpp
import "google/protobuf/timestamp.proto";

message Event {
    google.protobuf.Timestamp created_at = 1;
}
```

#### <2>C++ 生成类型伪代码
```cpp
namespace google::protobuf {

class Timestamp {
private:
    // ========== 存储字段 ==========
    int64_t seconds_;    // Unix 时间戳（秒）
    int32_t nanos_;      // 纳秒部分 [0, 999999999]
    
public:
    // ========== Getter ==========
    int64_t seconds() const;
    int32_t nanos() const;
    
    // ========== Setter ==========
    void set_seconds(int64_t value);
    void set_nanos(int32_t value);
    
    // ========== Clear ==========
    void Clear();
};

namespace util {
class TimeUtil {
public:
    // ========== Create（创建）==========
    static Timestamp GetCurrentTime();
    static Timestamp SecondsToTimestamp(int64_t seconds);
    static Timestamp MillisecondsToTimestamp(int64_t millis);
    
    // ========== Convert（转换）==========
    static int64_t TimestampToSeconds(const Timestamp& ts);
    static int64_t TimestampToMilliseconds(const Timestamp& ts);
    static int64_t TimestampToMicroseconds(const Timestamp& ts);
    static int64_t TimestampToNanoseconds(const Timestamp& ts);
    
    // ========== String（字符串）==========
    static std::string ToString(const Timestamp& ts);
    static bool FromString(const std::string& str, Timestamp* ts);
};
}  // namespace util

}  // namespace google::protobuf

class Event {
private:
    // ========== 存储字段 ==========
    google::protobuf::Timestamp* created_at_;
    uint32_t _has_bits_[1];
    
public:
    // ========== Getter ==========
    const google::protobuf::Timestamp& created_at() const;
    
    // ========== Mutable ==========
    google::protobuf::Timestamp* mutable_created_at();
    
    // ========== Has ==========
    bool has_created_at() const;
    
    // ========== Clear ==========
    void clear_created_at();
};
```

#### <3>使用示例
```cpp
#include <google/protobuf/timestamp.pb.h>
#include <google/protobuf/util/time_util.h>
#include "event.pb.h"

using google::protobuf::util::TimeUtil;

Event event;

// 方式1: 设置当前时间
*event.mutable_created_at() = TimeUtil::GetCurrentTime();

// 方式2: 手动设置秒和纳秒
event.mutable_created_at()->set_seconds(1705312200);
event.mutable_created_at()->set_nanos(500000000);  // 0.5秒

// 方式3: 从毫秒创建
*event.mutable_created_at() = TimeUtil::MillisecondsToTimestamp(1705312200500);

// 读取
std::cout << event.created_at().seconds() << std::endl;  // 1705312200
std::cout << event.created_at().nanos() << std::endl;    // 500000000

// 转为毫秒
int64_t ms = TimeUtil::TimestampToMilliseconds(event.created_at());
std::cout << ms << std::endl;  // 1705312200500

// 转为 RFC 3339 字符串
std::string time_str = TimeUtil::ToString(event.created_at());
std::cout << time_str << std::endl;  // 2024-01-15T10:30:00.500Z

// 从字符串解析
google::protobuf::Timestamp ts;
TimeUtil::FromString("2024-01-15T10:30:00Z", &ts);

```

### （7）Duration（时间段）
#### <1>Proto 定义
```cpp
import "google/protobuf/duration.proto";

message Task {
    google.protobuf.Duration timeout = 1;
}
```

#### <2>C++ 生成类型伪代码
```cpp
namespace google::protobuf {

class Duration {
private:
    // ========== 存储字段 ==========
    int64_t seconds_;    // 秒（可为负）
    int32_t nanos_;      // 纳秒 [-999999999, 999999999]，与 seconds 同号
    
public:
    // ========== Getter ==========
    int64_t seconds() const;
    int32_t nanos() const;
    
    // ========== Setter ==========
    void set_seconds(int64_t value);
    void set_nanos(int32_t value);
    
    // ========== Clear ==========
    void Clear();
};

namespace util {
class TimeUtil {
public:
    // ========== Create（创建）==========
    static Duration SecondsToDuration(int64_t seconds);
    static Duration MillisecondsToDuration(int64_t millis);
    static Duration MicrosecondsToDuration(int64_t micros);
    static Duration NanosecondsToDuration(int64_t nanos);
    
    // ========== Convert（转换）==========
    static int64_t DurationToSeconds(const Duration& d);
    static int64_t DurationToMilliseconds(const Duration& d);
    static int64_t DurationToMicroseconds(const Duration& d);
    static int64_t DurationToNanoseconds(const Duration& d);
    
    // ========== String（字符串）==========
    static std::string ToString(const Duration& d);
    static bool FromString(const std::string& str, Duration* d);
};
}  // namespace util

}  // namespace google::protobuf

class Task {
private:
    // ========== 存储字段 ==========
    google::protobuf::Duration* timeout_;
    uint32_t _has_bits_[1];
    
public:
    // ========== Getter ==========
    const google::protobuf::Duration& timeout() const;
    
    // ========== Mutable ==========
    google::protobuf::Duration* mutable_timeout();
    
    // ========== Has ==========
    bool has_timeout() const;
    
    // ========== Clear ==========
    void clear_timeout();
};
```

#### <3>使用示例
```cpp
#include <google/protobuf/duration.pb.h>
#include <google/protobuf/util/time_util.h>
#include "task.pb.h"

using google::protobuf::util::TimeUtil;

Task task;

// 方式1: 从秒创建
*task.mutable_timeout() = TimeUtil::SecondsToDuration(30);

// 方式2: 从毫秒创建（5.5秒）
*task.mutable_timeout() = TimeUtil::MillisecondsToDuration(5500);

// 方式3: 手动设置
task.mutable_timeout()->set_seconds(10);
task.mutable_timeout()->set_nanos(500000000);  // 10.5秒

// 读取
std::cout << task.timeout().seconds() << std::endl;  // 10
std::cout << task.timeout().nanos() << std::endl;    // 500000000

// 转为毫秒
int64_t ms = TimeUtil::DurationToMilliseconds(task.timeout());
std::cout << ms << std::endl;  // 10500

// 转为字符串
std::string dur_str = TimeUtil::ToString(task.timeout());
std::cout << dur_str << std::endl;  // 10.500s

// 负数时间段（表示过去）
*task.mutable_timeout() = TimeUtil::SecondsToDuration(-60);
std::cout << task.timeout().seconds() << std::endl;  // -60

```

### （8）optional T（可选字段）
#### <1>Proto 定义
```cpp
message Profile {
    optional string nickname = 1;
    optional int32 age = 2;
}
```

#### <2>C++ 生成类型伪代码
```cpp
class Profile {
private:
    // ========== 存储字段 ==========
    std::string nickname_;
    int32_t age_;
    
    // optional 字段使用位图跟踪是否设置
    uint32_t _has_bits_[1];           // 位图
    // bit 0 → has_nickname
    // bit 1 → has_age
    
public:
    // ========== Getter ==========
    const std::string& nickname() const;
    int32_t age() const;
    
    // ========== Setter ==========
    void set_nickname(const std::string& value);
    void set_nickname(std::string&& value);
    void set_nickname(const char* value);
    void set_age(int32_t value);
    
    // ========== Mutable ==========
    std::string* mutable_nickname();
    
    // ========== Has（✅ optional 核心特性）==========
    bool has_nickname() const;    // 检查 _has_bits_[0] & 0x01
    bool has_age() const;         // 检查 _has_bits_[0] & 0x02
    
    // ========== Clear ==========
    void clear_nickname();        // 清除值 + 清除 has 位
    void clear_age();
};
```

#### <3>使用示例
```cpp
#include "profile.pb.h"

Profile p;

// ===== 核心区别：区分 "未设置" vs "设置为默认值" =====

// 初始状态：未设置
std::cout << p.has_nickname() << std::endl;  // false
std::cout << p.has_age() << std::endl;       // false
std::cout << p.nickname() << std::endl;      // ""（默认值）
std::cout << p.age() << std::endl;           // 0（默认值）

// 设置值
p.set_nickname("Ali");
p.set_age(25);
std::cout << p.has_nickname() << std::endl;  // true
std::cout << p.has_age() << std::endl;       // true

// ===== 关键：设置为空/零 ≠ 未设置 =====
p.set_nickname("");
p.set_age(0);
std::cout << p.has_nickname() << std::endl;  // true!（已设置，只是值为空）
std::cout << p.has_age() << std::endl;       // true!（已设置，只是值为0）

// 清除后才是"未设置"
p.clear_nickname();
p.clear_age();
std::cout << p.has_nickname() << std::endl;  // false
std::cout << p.has_age() << std::endl;       // false

// ===== 实际应用：可选参数处理 =====
void updateProfile(const Profile& p) {
    if (p.has_nickname()) {
        // 用户明确提供了昵称（即使是空字符串）
        db.update("nickname", p.nickname());
    }
    // 未设置则不更新
    
    if (p.has_age()) {
        db.update("age", p.age());
    }
}
```

### （9）repeated T（数组）
#### <1>Proto 定义
```cpp
message Cart {
    repeated string tags = 1;
    repeated int32 counts = 2;
    repeated Item items = 3;
}

message Item {
    string name = 1;
    int32 price = 2;
}
```

#### <2>C++ 生成类型伪代码
```cpp
class Cart {
private:
    // ========== 存储字段 ==========
    google::protobuf::RepeatedPtrField<std::string> tags_;   // 字符串数组
    google::protobuf::RepeatedField<int32_t> counts_;        // 标量数组
    google::protobuf::RepeatedPtrField<Item> items_;         // 消息数组
    
public:
    // ==================== repeated string ====================
    
    // ----- Size（大小）-----
    int tags_size() const;
    
    // ----- Getter（只读访问）-----
    const std::string& tags(int index) const;
    const google::protobuf::RepeatedPtrField<std::string>& tags() const;
    
    // ----- Setter（修改指定位置）-----
    void set_tags(int index, const std::string& value);
    
    // ----- Add（追加）-----
    void add_tags(const std::string& value);
    std::string* add_tags();
    
    // ----- Mutable（可修改访问）-----
    std::string* mutable_tags(int index);
    google::protobuf::RepeatedPtrField<std::string>* mutable_tags();
    
    // ----- Clear（清空）-----
    void clear_tags();
    
    // ==================== repeated int32 ====================
    
    // ----- Size -----
    int counts_size() const;
    
    // ----- Getter -----
    int32_t counts(int index) const;
    const google::protobuf::RepeatedField<int32_t>& counts() const;
    
    // ----- Setter -----
    void set_counts(int index, int32_t value);
    
    // ----- Add -----
    void add_counts(int32_t value);
    
    // ----- Mutable -----
    google::protobuf::RepeatedField<int32_t>* mutable_counts();
    
    // ----- Clear -----
    void clear_counts();
    
    // ==================== repeated message ====================
    
    // ----- Size -----
    int items_size() const;
    
    // ----- Getter -----
    const Item& items(int index) const;
    const google::protobuf::RepeatedPtrField<Item>& items() const;
    
    // ----- Add（返回新元素指针）-----
    Item* add_items();
    
    // ----- Mutable -----
    Item* mutable_items(int index);
    google::protobuf::RepeatedPtrField<Item>* mutable_items();
    
    // ----- Clear -----
    void clear_items();
};

// ==================== RepeatedField（标量数组）====================
template<typename T>
class RepeatedField {
private:
    // ========== 内部存储 ==========
    T* elements_;           // 动态数组
    int current_size_;      // 当前元素数
    int total_size_;        // 已分配容量
    Arena* arena_;          // 可选内存池
    
public:
    // ----- Size -----
    int size() const;
    bool empty() const;
    
    // ----- Access -----
    T Get(int index) const;
    T operator[](int index) const;
    T at(int index) const;
    
    // ----- Modify -----
    void Set(int index, T value);
    void Add(T value);
    T* Add();
    
    // ----- Remove -----
    void RemoveLast();
    void ExtractSubrange(int start, int num, T* elements);
    void Truncate(int new_size);
    void Clear();
    
    // ----- Iterate -----
    iterator begin();
    iterator end();
    
    // ----- Raw Data -----
    T* mutable_data();
    const T* data() const;
};

// ==================== RepeatedPtrField（消息/字符串数组）====================
template<typename T>
class RepeatedPtrField {
private:
    // ========== 内部存储 ==========
    T** elements_;          // 指针数组
    int current_size_;
    int total_size_;
    Arena* arena_;
    
public:
    // ----- Size -----
    int size() const;
    bool empty() const;
    
    // ----- Access -----
    const T& Get(int index) const;
    const T& operator[](int index) const;
    
    // ----- Mutable Access -----
    T* Mutable(int index);
    
    // ----- Add -----
    T* Add();
    void Add(T&& value);
    
    // ----- Remove -----
    void RemoveLast();
    void DeleteSubrange(int start, int num);
    void Clear();
    
    // ----- Swap -----
    void SwapElements(int index1, int index2);
    
    // ----- Iterate -----
    iterator begin();
    iterator end();
};
```

#### <3>使用示例
```cpp
#include "cart.pb.h"

Cart cart;

// ==================== repeated string ====================
// 添加
cart.add_tags("electronics");
cart.add_tags("sale");
cart.add_tags("hot");

// 读取
std::cout << cart.tags_size() << std::endl;  // 3
std::cout << cart.tags(0) << std::endl;      // electronics

// 遍历
for (const auto& tag : cart.tags()) {
    std::cout << tag << std::endl;
}

// 修改
cart.set_tags(0, "digital");

// 清空
cart.clear_tags();

// ==================== repeated int32 ====================
cart.add_counts(10);
cart.add_counts(20);
cart.add_counts(30);

std::cout << cart.counts(1) << std::endl;  // 20
cart.set_counts(1, 25);

// 遍历
for (int i = 0; i < cart.counts_size(); i++) {
    std::cout << cart.counts(i) << std::endl;
}

// ==================== repeated message ====================
// 添加并设置（返回指针）
Item* item1 = cart.add_items();
item1->set_name("iPhone");
item1->set_price(999);

Item* item2 = cart.add_items();
item2->set_name("AirPods");
item2->set_price(199);

// 读取
std::cout << cart.items_size() << std::endl;        // 2
std::cout << cart.items(0).name() << std::endl;     // iPhone
std::cout << cart.items(0).price() << std::endl;    // 999

// 遍历
for (const auto& item : cart.items()) {
    std::cout << item.name() << ": $" << item.price() << std::endl;
}

// 修改已存在的元素
cart.mutable_items(0)->set_price(899);

// 删除元素
cart.mutable_items()->DeleteSubrange(0, 1);  // 删除索引0，删1个
std::cout << cart.items_size() << std::endl;  // 1

// 交换元素位置
cart.add_items()->set_name("iPad");
cart.mutable_items()->SwapElements(0, 1);

// 删除最后一个
cart.mutable_items()->RemoveLast();
```

## 3.通用 Message 基类方法
### （1）C++ 生成伪代码
```cpp
class Message {
protected:
    mutable int _cached_size_;  // 缓存序列化后的字节大小，避免重复计算
    Arena* _arena_;             // 内存池指针，用于高效内存管理
    
public:
    // ==================== 序列化（对象 → 二进制）====================
    
    bool SerializeToString(std::string* output) const;  // 序列化到 string，成功返回 true
    std::string SerializeAsString() const;              // 序列化并返回 string（失败返回空）
    bool SerializeToArray(void* data, int size) const;  // 序列化到预分配的 buffer
    size_t ByteSizeLong() const;                        // 计算序列化后的字节数
    
    // ==================== 反序列化（二进制 → 对象）====================
    
    bool ParseFromString(const std::string& data);      // 从 string 解析，成功返回 true
    bool ParseFromArray(const void* data, int size);    // 从 buffer 解析
    
    // ==================== 复制 ====================
    
    void CopyFrom(const Message& from);   // 完全覆盖：先 Clear()，再复制
    void MergeFrom(const Message& from);  // 合并：只覆盖 from 中已设置的字段
    
    // ==================== 清空 ====================
    
    void Clear();  // 重置所有字段为默认值
    
    // ==================== 调试 ====================
    
    std::string DebugString() const;       // 格式化输出（多行，带缩进）
    std::string ShortDebugString() const;  // 紧凑输出（单行）
};
```

### （2）序列化/反序列化
```cpp
User user;
user.set_id(42);
user.set_name("Alice");

// 序列化
std::string data = user.SerializeAsString();

// 反序列化
User user2;
user2.ParseFromString(data);

// 调试输出
std::cout << user.DebugString();
// 输出:
// id: 42
// name: "Alice"

std::cout << user.ShortDebugString();
// 输出: id: 42 name: "Alice"
```

## 补充：Well-Known Types
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120090503033.png)

### （1）Wrapper Types（包装类型）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120091029916.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120091111812.png)

#### <1>Proto 定义
```cpp
import "google/protobuf/wrappers.proto";

message User {
    google.protobuf.Int32Value age = 1;        // 可判断是否设置
    google.protobuf.StringValue nickname = 2;
    google.protobuf.BoolValue is_vip = 3;
}
```

#### <2>C++ 生成类型伪代码
```cpp
namespace google::protobuf {

// 以 Int32Value 为例，其他类似
class Int32Value {
private:
    int32_t value_;
    
public:
    // ========== Getter ==========
    int32_t value() const;
    
    // ========== Setter ==========
    void set_value(int32_t value);
    
    // ========== Clear ==========
    void Clear();
};

}  // namespace google::protobuf

class User {
private:
    google::protobuf::Int32Value* age_;           // 指针存储
    google::protobuf::StringValue* nickname_;
    google::protobuf::BoolValue* is_vip_;
    uint32_t _has_bits_[1];                       // 位图跟踪
    
public:
    // ========== Has（核心特性）==========
    bool has_age() const;
    bool has_nickname() const;
    bool has_is_vip() const;
    
    // ========== Getter ==========
    const google::protobuf::Int32Value& age() const;
    
    // ========== Mutable ==========
    google::protobuf::Int32Value* mutable_age();
    
    // ========== Clear ==========
    void clear_age();
};
```

#### <3>使用示例
```cpp
#include <google/protobuf/wrappers.pb.h>
#include "user.pb.h"

User user;

// ===== 核心用法：区分"未设置" vs "设置为0/空" =====

// 初始状态：未设置
std::cout << user.has_age() << std::endl;  // false

// 设置值
user.mutable_age()->set_value(25);
std::cout << user.has_age() << std::endl;      // true
std::cout << user.age().value() << std::endl;  // 25

// ===== 关键：设置为 0 也能检测到 =====
user.mutable_age()->set_value(0);
std::cout << user.has_age() << std::endl;      // true! ✅
std::cout << user.age().value() << std::endl;  // 0

// 清除后才是"未设置"
user.clear_age();
std::cout << user.has_age() << std::endl;  // false

// ===== 实际应用：API 更新接口 =====
void updateUser(const User& req) {
    if (req.has_age()) {
        // 用户明确传了 age（即使是 0）
        db.update("age", req.age().value());
    }
    // 未设置则不更新该字段
    
    if (req.has_nickname()) {
        db.update("nickname", req.nickname().value());
    }
}
```

#### <4>Wrapper vs optional 对比
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120091236245.png)


### （2）Struct / Value / ListValue（动态 JSON 结构）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120091314284.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120091430492.png)

#### <1>Proto 定义
```cpp
import "google/protobuf/struct.proto";

message Config {
    google.protobuf.Struct settings = 1;   // 任意 JSON 对象
    google.protobuf.Value metadata = 2;    // 任意 JSON 值
    google.protobuf.ListValue items = 3;   // JSON 数组
}
```

#### <2>C++ 生成类型伪代码
```cpp
namespace google::protobuf {

// ==================== Value（任意 JSON 值）====================
class Value {
public:
    enum KindCase {
        kNullValue = 1,
        kNumberValue = 2,
        kStringValue = 3,
        kBoolValue = 4,
        kStructValue = 5,
        kListValue = 6,
        KIND_NOT_SET = 0
    };
    
private:
    // oneof 存储
    union {
        NullValue null_value_;
        double number_value_;
        std::string* string_value_;
        bool bool_value_;
        Struct* struct_value_;
        ListValue* list_value_;
    };
    KindCase kind_case_;
    
public:
    // ========== Kind 判断 ==========
    KindCase kind_case() const;
    
    // ========== Getter ==========
    NullValue null_value() const;
    double number_value() const;
    const std::string& string_value() const;
    bool bool_value() const;
    const Struct& struct_value() const;
    const ListValue& list_value() const;
    
    // ========== Setter ==========
    void set_null_value(NullValue value);
    void set_number_value(double value);
    void set_string_value(const std::string& value);
    void set_bool_value(bool value);
    
    // ========== Mutable ==========
    Struct* mutable_struct_value();
    ListValue* mutable_list_value();
};

// ==================== Struct（JSON 对象）====================
class Struct {
private:
    google::protobuf::Map<std::string, Value> fields_;
    
public:
    // ========== 字段访问 ==========
    const Map<std::string, Value>& fields() const;
    Map<std::string, Value>* mutable_fields();
};

// ==================== ListValue（JSON 数组）====================
class ListValue {
private:
    google::protobuf::RepeatedPtrField<Value> values_;
    
public:
    // ========== 元素访问 ==========
    int values_size() const;
    const Value& values(int index) const;
    Value* add_values();
    Value* mutable_values(int index);
    const RepeatedPtrField<Value>& values() const;
    RepeatedPtrField<Value>* mutable_values();
};

}  // namespace google::protobuf
```

#### <3>使用示例
```cpp
#include <google/protobuf/struct.pb.h>
#include "config.pb.h"

using google::protobuf::Struct;
using google::protobuf::Value;
using google::protobuf::ListValue;

Config config;

// ==================== 构建 Struct（JSON 对象）====================
// 目标: { "name": "Alice", "age": 25, "active": true }

Struct* settings = config.mutable_settings();

// 设置 string
(*settings->mutable_fields())["name"].set_string_value("Alice");

// 设置 number
(*settings->mutable_fields())["age"].set_number_value(25);

// 设置 bool
(*settings->mutable_fields())["active"].set_bool_value(true);

// 设置 null
(*settings->mutable_fields())["deleted_at"].set_null_value(
    google::protobuf::NULL_VALUE);

// ==================== 嵌套对象 ====================
// 目标: { "address": { "city": "Beijing", "zip": "100000" } }

Struct* address = (*settings->mutable_fields())["address"].mutable_struct_value();
(*address->mutable_fields())["city"].set_string_value("Beijing");
(*address->mutable_fields())["zip"].set_string_value("100000");

// ==================== 数组 ====================
// 目标: { "tags": ["vip", "new", 123] }

ListValue* tags = (*settings->mutable_fields())["tags"].mutable_list_value();
tags->add_values()->set_string_value("vip");
tags->add_values()->set_string_value("new");
tags->add_values()->set_number_value(123);

// ==================== 读取 ====================

// 读取 string
if (settings->fields().contains("name")) {
    const Value& v = settings->fields().at("name");
    if (v.kind_case() == Value::kStringValue) {
        std::cout << v.string_value() << std::endl;  // Alice
    }
}

// 读取 number
double age = settings->fields().at("age").number_value();
std::cout << age << std::endl;  // 25

// 遍历所有字段
for (const auto& [key, value] : settings->fields()) {
    std::cout << key << ": ";
    switch (value.kind_case()) {
        case Value::kStringValue:
            std::cout << value.string_value();
            break;
        case Value::kNumberValue:
            std::cout << value.number_value();
            break;
        case Value::kBoolValue:
            std::cout << (value.bool_value() ? "true" : "false");
            break;
        case Value::kNullValue:
            std::cout << "null";
            break;
        case Value::kStructValue:
            std::cout << "{...}";
            break;
        case Value::kListValue:
            std::cout << "[...]";
            break;
        default:
            std::cout << "unknown";
    }
    std::cout << std::endl;
}

// 读取数组
const ListValue& tag_list = settings->fields().at("tags").list_value();
for (int i = 0; i < tag_list.values_size(); i++) {
    std::cout << "tag[" << i << "]: ";
    const Value& v = tag_list.values(i);
    if (v.kind_case() == Value::kStringValue) {
        std::cout << v.string_value();
    } else if (v.kind_case() == Value::kNumberValue) {
        std::cout << v.number_value();
    }
    std::cout << std::endl;
}
```

#### <4>Struct 与 JSON 互转
```cpp
#include <google/protobuf/util/json_util.h>

using google::protobuf::util::JsonStringToMessage;
using google::protobuf::util::MessageToJsonString;

// JSON → Struct
std::string json = R"({
    "name": "Alice",
    "age": 25,
    "tags": ["vip", "new"]
})";

Struct s;
auto status = JsonStringToMessage(json, &s);
if (status.ok()) {
    std::cout << "解析成功" << std::endl;
}

// Struct → JSON
std::string output;
MessageToJsonString(s, &output);
std::cout << output << std::endl;
// {"name":"Alice","age":25,"tags":["vip","new"]}
```

### （3）Empty（空消息）
#### <1>Proto 定义
```cpp
import "google/protobuf/empty.proto";

service HealthService {
    rpc Ping(google.protobuf.Empty) returns (google.protobuf.Empty);
    rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
}

message DeleteUserRequest {
    int32 user_id = 1;
}
```

#### <2>C++ 生成类型伪代码
```cpp
namespace google::protobuf {

class Empty {
public:
    // 几乎没有方法，就是一个空消息
    void Clear();
    bool IsInitialized() const;
    size_t ByteSizeLong() const;  // 始终返回 0
};

}  // namespace google::protobuf
```

#### <3>使用示例
```cpp
#include <google/protobuf/empty.pb.h>
#include "health_service.grpc.pb.h"

using google::protobuf::Empty;

// ==================== gRPC 客户端调用 ====================

// Ping：无参数，无返回
Empty req, resp;
grpc::ClientContext context;
grpc::Status status = stub->Ping(&context, req, &resp);

if (status.ok()) {
    std::cout << "Ping 成功" << std::endl;
}

// DeleteUser：有参数，无返回
DeleteUserRequest del_req;
del_req.set_user_id(12345);

Empty del_resp;
status = stub->DeleteUser(&context, del_req, &del_resp);

// ==================== gRPC 服务端实现 ====================

class HealthServiceImpl final : public HealthService::Service {
    grpc::Status Ping(grpc::ServerContext* context,
                      const Empty* request,
                      Empty* response) override {
        // 什么都不用做，直接返回 OK
        return grpc::Status::OK;
    }
    
    grpc::Status DeleteUser(grpc::ServerContext* context,
                            const DeleteUserRequest* request,
                            Empty* response) override {
        int32_t user_id = request->user_id();
        db.deleteUser(user_id);
        // 不需要填充 response
        return grpc::Status::OK;
    }
};
```

### （4）FieldMask（字段掩码）
#### <1>Proto 定义
```cpp
import "google/protobuf/field_mask.proto";

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    string phone = 4;
}

message UpdateUserRequest {
    User user = 1;
    google.protobuf.FieldMask update_mask = 2;
}
```

#### <2>C++ 生成类型伪代码
```cpp
namespace google::protobuf {

class FieldMask {
private:
    google::protobuf::RepeatedPtrField<std::string> paths_;
    
public:
    // ----- Size -----
    int paths_size() const;
    
    // ----- Getter -----
    const std::string& paths(int index) const;
    const RepeatedPtrField<std::string>& paths() const;
    
    // ----- Add -----
    void add_paths(const std::string& value);
    std::string* add_paths();
    
    // ----- Mutable -----
    std::string* mutable_paths(int index);
    RepeatedPtrField<std::string>* mutable_paths();
    
    // ----- Clear -----
    void clear_paths();
    void Clear();
};

}  // namespace google::protobuf
```

#### <3>使用示例
```cpp
#include <google/protobuf/field_mask.pb.h>
#include <google/protobuf/util/field_mask_util.h>
#include "user.pb.h"

using google::protobuf::FieldMask;
using google::protobuf::util::FieldMaskUtil;

// ==================== 客户端：构建部分更新请求 ====================

UpdateUserRequest req;

// 只更新 name 和 email
req.mutable_user()->set_id(12345);
req.mutable_user()->set_name("Alice");
req.mutable_user()->set_email("alice@example.com");

// 指定要更新的字段
req.mutable_update_mask()->add_paths("name");
req.mutable_update_mask()->add_paths("email");

// ==================== 服务端：根据 FieldMask 处理 ====================

void handleUpdateUser(const UpdateUserRequest& req) {
    const User& user = req.user();
    const FieldMask& mask = req.update_mask();
    
    // 方式1：手动遍历
    for (const auto& path : mask.paths()) {
        if (path == "name") {
            db.update(user.id(), "name", user.name());
        } else if (path == "email") {
            db.update(user.id(), "email", user.email());
        } else if (path == "phone") {
            db.update(user.id(), "phone", user.phone());
        }
    }
    
    // 方式2：使用 FieldMaskUtil 合并
    User existing_user = db.getUser(user.id());
    FieldMaskUtil::MergeMessageTo(user, mask, &existing_user);
    db.saveUser(existing_user);
}

// ==================== FieldMaskUtil 工具函数 ====================

User src, dst;
src.set_name("Alice");
src.set_email("alice@example.com");
src.set_phone("13800138000");

dst.set_id(1);
dst.set_name("Bob");
dst.set_email("bob@example.com");

FieldMask mask;
mask.add_paths("name");
mask.add_paths("email");

// 只合并 mask 指定的字段
FieldMaskUtil::MergeMessageTo(src, mask, &dst);

std::cout << dst.id() << std::endl;     // 1（未被覆盖）
std::cout << dst.name() << std::endl;   // Alice（被覆盖）
std::cout << dst.email() << std::endl;  // alice@example.com（被覆盖）
std::cout << dst.phone() << std::endl;  // ""（未被覆盖，dst 原本没有）

// ==================== 其他工具函数 ====================

// 检查 path 是否在 mask 中
bool has_name = FieldMaskUtil::IsPathInFieldMask("name", mask);  // true
bool has_phone = FieldMaskUtil::IsPathInFieldMask("phone", mask); // false

// 转为字符串
std::string mask_str = FieldMaskUtil::ToString(mask);  // "name,email"

// 从字符串解析
FieldMask mask2;
FieldMaskUtil::FromString("name,phone", &mask2);
```

# 三、RPC（远程过程调用）
## 1.RPC的功能
- **远程过程调用（Remote Procedure Call）** 的核心目标是：
	- 让使用者在**一台服务器上调用另一台服务器上的函数**时，感觉**就像调用本地函数**一样简单。
- 举个例子，假设你在服务器 A 上写代码，想获取服务器 B 上的用户信息：
```cpp
// 调用者视角：看起来就像调用本地函数
UserResponse resp = userService.GetUser(request);
// 实际上，这个请求通过网络发送到了服务器 B，服务器 B 执行后把结果返回
```
从代码层面看，你完全 **“感知不到”中间经历了网络传输、序列化、反序列化等复杂过程**，这就是 RPC 的魅力所在。

<a id="RPC（2）"></a>
## 2.RPC底层原理（重点！！）
### （1）RPC核心步骤（概述）
- **“双方定义一模一样的API”** + **“服务端注册方法”** + **“序列化”（序列化客户端发送的请求）** + **“网络通信”** + **“反序列化”（反序列化客户端发送的请求）**+ **“路由”（获取具体的方法进行执行）**

### （2）RPC过程详解
RPC 是如何实现"像调用本地函数一样"的效果呢？我们以客户端、服务端为例，服务端不同服务可能有同名的方法，所以我们以 **（服务名,方法名）二元组** 来定位一个“具体方法”
- **第一步：在客户端与服务端上定义好“一模一样”的API**
	- 客户端和服务端必须事先约定好接口的定义——**服务名**是什么、**方法名**叫什么、**参数**是什么类型、**返回值**是什么类型 ——> 只有双方的"契约"一致，才能正确地互相理解。
- **第二步：服务端“注册方法”**
	- 如通过 **std::map<服务名:方法名,具体方法>** 来进行映射存储	——>这样就可以通过（服务名，方法名）来定位“具体方法”（**具体方法=map[服务名:方法名]**）
- **第三步：客户端调用时，将"服务名+方法名 + 参数"进行“序列化”，再通过“网络发送”到目标服务端**
	- 当客户端执行User服务的GetUser(request)方法 时，RPC 框架会在背后做这些事情：
		- 1.把服务名（如 "User"）、方法名（如 "GetUser"）、参数（如 request 对象）**序列化**成二进制数据
		- 2.通过网络把这些数据发送到服务端
- **第四步：到达服务端后，进行“路由”后得到“具体方法”，进而执行对应方法**
	- 服务端收到数据后：
		- 1.**反序列化**得到“服务名”、“方法名”、“参数”
		- 2.**具体方法 = map[服务名:方法名]**，再用反序列化出的参数调用它
		- 3.得到返回值后，再将返回值**序列化**，通过网络传回给客户端
		- **路由**：就是指找到对应的“具体方法”进行执行
- **第五步：客户端收到返回值，继续执行**
	- 客户端收到返回的二进制数据后，反序列化得到返回值，然后就像普通函数调用一样继续往下执行。

**关键点总结**：
- RPC 的核心就是**封装了中间网络通信的全部细节**（序列化、网络传输、反序列化、路由），让开发者只需要关注"调用什么函数、传什么参数、拿什么返回值"，从而达到"像调用本地 API 一样"的使用体验。
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                           RPC 调用完整流程                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   调用端服务器 (A)                              目标服务器 (B)            │
│                                                                         │
│   ┌──────────────┐                            ┌──────────────┐          │
│   │  业务代码     │                            │  相同的 API   │          │
│   │              │                            │              │          │
│   │ GetUser(req) │                            │ GetUser(req) │          │
│   └──────┬───────┘                            └──────▲───────┘          │
│          │ ①调用                                     │ ④执行            │
│          ▼                                           │                  │
│   ┌──────────────┐                            ┌──────┴───────┐          │
│   │  RPC 框架     │                            │  RPC 框架     │          │
│   │              │                            │              │          │
│   │ ②序列化       │     ═══════════════════    │ ③反序列化     │          │
│   │ 方法名+参数   │────►   网络传输请求   ────►│ 方法名+参数   │          │
│   │              │                            │              │          │
│   │ ⑥反序列化    │◄────   网络传输响应   ◄────│ ⑤序列化      │          │
│   │ 返回值       │     ═══════════════════    │ 返回值       │          │
│   └──────┬───────┘                            └──────────────┘          │
│          │ ⑦返回                                                        │
│          ▼                                                              │
│   ┌──────────────┐                                                      │
│   │  业务代码     │                                                      │
│   │ 拿到返回值    │                                                      │
│   │ 继续执行...   │                                                      │
│   └──────────────┘                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3.RPC 的性能与价值
- **性能方面**：
	- 由于 RPC 调用中间**增加了序列化、网络传输、反序列化等步骤**，其**性能必然“低于”直接调用本地函数**。
	- 一次本地函数调用可能只需要几纳秒，而一次 RPC 调用即使在同一局域网内，也至少需要几百微秒到几毫秒。
- **那为什么还要用 RPC？**
	- 因为在现代软件系统中，**单台服务器的能力是有限的**：
		- **资源有限**：一台机器的 CPU、内存、磁盘都是有上限的
		- **负载能力有限**：单机能承受的并发请求量有限
		- **容错能力差**：单机故障会导致整个服务不可用
		- **扩展困难**：业务增长时，单机很难快速扩容
	- 要解决这些问题，必须采用**分布式架构**——把服务拆分到多台机器上运行。而**机器之间的通信，就需要 RPC 来完成**。
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                     为什么需要 RPC？                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   单机困境                              分布式解决方案            │
│   ──────────                            ──────────────          │
│                                                                 │
│   资源有限                    ──────►   多机水平扩展             │
│   (CPU/内存/磁盘)                       (加机器就能加资源)        │
│                                                                 │
│   负载能力差                  ──────►   负载均衡                 │
│   (单机扛不住高并发)                    (请求分散到多台机器)      │
│                                                                 │
│   单点故障                    ──────►   容错冗余                 │
│   (一台挂了全挂)                        (一台挂了其他顶上)        │
│                                                                 │
│   扩展困难                    ──────►   弹性伸缩                 │
│   (业务增长难应对)                      (按需增减机器)           │
│                                                                 │
│                                                                 │
│            ┌─────────────────────────────────────┐              │
│            │  RPC 是分布式系统的"神经网络"        │              │
│            │  让多台机器能像一台机器一样协同工作   │              │
│            └─────────────────────────────────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


# 四、Proto Service（纯 Protobuf 层面）
- 前面提到，RPC 要求调用端和被调用端有"**一模一样的 API**"。那这个 API 谁来定义？手写容易出错、跨语言还得写多份，非常麻烦 ——> **Proto Service 就是来解决这个问题的**
- 通过protoc编译后，Proto中的Service就会**自动生成** C++ 对应的抽象类和 Stub（客户端桩），也就不用我们自己去定义“**一模一样的API**”
- 除了自动生成代码，Proto Service 还**自带了 Protobuf 的序列化、反序列化能力** ——> 消息的序列化和反序列化都由生成的代码自动处理，你不需要关心数据在网络上是怎么传输的。
```cpp
┌───────────────────────────────────────────────────────────────────┐
│                    Proto Service 的价值                           │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│   传统方式（手写）                  Proto Service 方式             │
│                                                                   │
│   ┌─────────────────┐              ┌─────────────────┐            │
│   │ 服务端手写 API   │              │  .proto 文件     │            │
│   │ (C++ 版本)      │              │  定义一次        │            │
│   └─────────────────┘              └────────┬────────┘            │
│            +                                │                     │
│   ┌─────────────────┐                       │ protoc 编译         │
│   │ 客户端手写 API   │                       ▼                     │
│   │ (C++ 版本)      │              ┌────────┴────────┐            │
│   └─────────────────┘              │                 │            │
│            +                       ▼                 ▼            │
│   ┌─────────────────┐      ┌─────────────┐   ┌─────────────┐     │
│   │ 其他语言再写一遍  │      │ 服务端基类   │   │ 客户端 Stub │     │
│   │ (Java/Go/...)   │      │ (自动生成)   │   │ (自动生成)  │     │
│   └─────────────────┘      └─────────────┘   └─────────────┘     │
│                                                                   │
│   ❌ 繁琐、易出错、难维护    ✅ 一次定义、多端生成、“序列化、反序列化”   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

## 1.概述
**注意**：本节聚焦于 **Protobuf 原生 Service 机制**，与 gRPC 框架无关。Protobuf 的 Service 只是一个「接口描述层」，**需要自己实现 RPC 通信层**。
- 在Protobuf中，message描述 **数据结构**，service描述 **RPC接口**
- 在 .proto 文件里 **“定义服务的方法名、参数、返回值”** ，通过 protoc 编译后，会**自动生成** C++ 对应的 **抽象基类（服务端实现）** 和 **Stub 类（客户端桩）**
- 需要注意的是：原生的Proto Service **本身不包含网络传输逻辑**，它只是**定义"接口长什么样"**。
	- 要实现真正的远程调用，通常需要搭配 **RPC 框架**（如 **gRPC**，这是目前最主流的选择）来处理底层的网络通信。（或者**自己实现网络层**）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120103057891.png)

<a id="proto_service"></a>
## 2.Service的使用
### （1）基本语法
**以EchoService为例**：客户端发送消息给服务端，服务端原封不动
```cpp
syntax = "proto3";
package myservice;

// 用于 C++ 代码生成优化
option cc_generic_services = true;  // 【关键】必须开启才会生成 Service 代码

// 定义请求消息
message EchoRequest {
  string message = 1;	//客户端消息

}

// 定义响应消息
message EchoResponse {
  string reply = 1;		//服务端回复
}

// 定义服务
service EchoService {
  // RPC 方法定义
  rpc Echo(EchoRequest) returns (EchoResponse);
  rpc Ping(EchoRequest) returns (EchoResponse);

  // 补充：支持流式 RPC（Protobuf 原生支持，无需额外扩展）
  // rpc ListUsers(GetUserRequest) returns (stream GetUserResponse);  // 服务端流式
  // rpc BatchUpdateUser(stream GetUserRequest) returns (GetUserResponse);  // 客户端流式
  // rpc BidirectionalTalk(stream GetUserRequest) returns (stream GetUserResponse);  // 双向流式
}
```
**⚠️ 重要**：cc_generic_services = true 是使用 Protobuf 原生 Service 的前提！

### （2）编译命令
1. **基本编译**
```bash
# 只生成消息类和 Service 类
protoc --cpp_out=./generated echo.proto
```
2. **生成文件**
```bash
├── echo.pb.h      # 头文件（消息类 + Service 类）
└── echo.pb.cc     # 实现文件
```
3. **编译参数详解**
```bash
protoc \
  --cpp_out=OUT_DIR \          # 指定输出目录
  --proto_path=IMPORT_PATH \   # 指定 .proto 文件搜索路径（可多次使用）
  echo.proto                    # 输入文件

# 示例：多路径
protoc \
  --cpp_out=./generated \
  --proto_path=./protos \
  --proto_path=./third_party \
  echo.proto
```

### （3）生成的 C++ 伪代码
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120234637423.png)
```cpp
// =====================================================
// echo.pb.h - 生成的头文件
// =====================================================

#include <google/protobuf/service.h>
#include <google/protobuf/descriptor.h>
#include <google/protobuf/stubs/callback.h>

namespace myservice {

// =====================================================
// 消息类（略，与普通 message 相同）
// =====================================================
class EchoRequest : public ::google::protobuf::Message { /* ... */ };
class EchoResponse : public ::google::protobuf::Message { /* ... */ };


// =====================================================
// Service 抽象基类 - 服务端需要继承实现
// =====================================================
class EchoService : public ::google::protobuf::Service {
protected:
    // 禁止直接构造，必须继承
    EchoService() = default;
    
public:
    virtual ~EchoService() = default;
    // 禁止拷贝
    EchoService(const EchoService&) = delete;
    EchoService& operator=(const EchoService&) = delete;

    // 类型别名（方便使用）
    typedef EchoService_Stub Stub;
    


    // ===================== 【核心】纯虚方法 - 需要子类实现的业务逻辑 =====================
    // 由 rpc Echo(EchoRequest) returns (EchoResponse) 生成
    virtual void Echo(
        ::google::protobuf::RpcController* controller,	//RPC 控制器（用于设置/获取错误信息、取消等）
        const EchoRequest* request,						//请求消息类（由框架反序列化后传入）
        EchoResponse* response,							//响应消息类（子类填充后由框架序列化）
        ::google::protobuf::Closure* done 				//完成回调（异步场景使用，调用 done->Run() 表示完成）
    ) = 0;
    
    // 由 rpc Ping(EchoRequest) returns (EchoResponse) 生成
    virtual void Ping(
        ::google::protobuf::RpcController* controller,
        const EchoRequest* request,
        EchoResponse* response,
        ::google::protobuf::Closure* done
    ) = 0;
    
    
    // 【核心】通用调用入口 - 根据方法描述符分发到具体方法
    // 框架层调用此方法，由它路由到 Echo() 或 Ping()
    void CallMethod(
        const ::google::protobuf::MethodDescriptor* method, // 方法描述符（包含服务名、方法名、索引等元信息）
        ::google::protobuf::RpcController* controller,	    // 控制器（用于取消、超时、错误处理）	
        const ::google::protobuf::Message* request,			// 请求消息（已序列化或待序列化）
        ::google::protobuf::Message* response,			    // 响应消息（调用完成后由框架填充）
        ::google::protobuf::Closure* done					//完成回调 :
    ){														// - 异步调用：传回调对象，CallMethod 立即返回
															// - 同步调用：传 nullptr，CallMethod 阻塞直到完成
	    // 1. 断言：确保传入的方法确实属于本服务
	    GOOGLE_DCHECK_EQ(method->service(), EchoService::descriptor());
	    
	    // 2. 根据方法索引进行路由分发
	    switch (method->index()) {
	        case 0:  // 索引 0 对应 Echo 方法（proto 文件中定义的第一个 rpc）
	            Echo(
	                controller,
	                // 向下转型：从通用 Message* 转为具体的 EchoRequest*
	                ::google::protobuf::down_cast<const EchoRequest*>(request),
	                // 向下转型：从通用 Message* 转为具体的 EchoResponse*
	                ::google::protobuf::down_cast<EchoResponse*>(response),
	                done
	            );
	            break;
	        case 1:  // 索引 1 对应 Ping 方法（proto 文件中定义的第二个 rpc）
	            Ping(
	                controller,
	                ::google::protobuf::down_cast<const EchoRequest*>(request),
	                ::google::protobuf::down_cast<EchoResponse*>(response),
	                done
	            );
	            break;
	        default:
	            // 非法索引，致命错误
	            GOOGLE_LOG(FATAL) << "Bad method index: " << method->index() 
	                              << ". This method doesn't belong to this service.";
	            break;
	    }
	} 		
    



    // --------------------------------------------------
    // 静态方法：获取服务描述符
    // --------------------------------------------------
    static const ::google::protobuf::ServiceDescriptor* descriptor();

    // 获取服务描述符（实例方法）
    const ::google::protobuf::ServiceDescriptor* GetDescriptor() override;

    // 获取请求/响应消息的原型（用于创建消息实例）
    const ::google::protobuf::Message& GetRequestPrototype(
        const ::google::protobuf::MethodDescriptor* method
    ) const override;
    
    const ::google::protobuf::Message& GetResponsePrototype(
        const ::google::protobuf::MethodDescriptor* method
    ) const override;
};


// =====================================================
// Stub 类 - 客户端桩，用于发起 RPC 调用
// =====================================================
class EchoService_Stub : public EchoService {
public:
    
	EchoService_Stub::EchoService_Stub(::google::protobuf::RpcChannel* channel)
	    : channel_(channel), owns_channel_(false) {
	}

	EchoService_Stub::EchoService_Stub(
	    ::google::protobuf::RpcChannel* channel,
	    ::google::protobuf::Service::ChannelOwnership ownership)
	    : channel_(channel),
	      owns_channel_(ownership == ::google::protobuf::Service::STUB_OWNS_CHANNEL) {
	}

	EchoService_Stub::~EchoService_Stub() {
	    if (owns_channel_) {
	        delete channel_;
	    }
	}
    
    // 获取 Channel
    inline ::google::protobuf::RpcChannel* channel() { return channel_; }
    

    // Echo 方法： rpc Echo(EchoRequest) returns (EchoResponse)
	void EchoService_Stub::Echo(
	    ::google::protobuf::RpcController* controller,
	    const EchoRequest* request,						// 请求：EchoRequest
	    EchoResponse* response,							// 响应：EchoResponse
	    ::google::protobuf::Closure* done
	) {
	    // 1. 获取方法描述符
	    const ::google::protobuf::MethodDescriptor* method =
	        EchoService::descriptor()->method(0);	// 获取索引为 0 的方法（即 Echo）
	    
	    // 2. 直接委托给 channel 的 CallMethod
	    //    所有序列化、网络传输、反序列化都由 channel 负责
	    channel_->CallMethod(
	        method,      // 方法描述符（携带服务名、方法名等元信息）
	        controller,  // 控制器（透传）
	        request,     // 请求消息（透传）
	        response,    // 响应消息（透传）
	        done         // 完成回调（透传）
	    );
	}
    
    // Ping 方法： rpc Ping(EchoRequest) returns (EchoResponse);
    void Ping(
        ::google::protobuf::RpcController* controller,
        const EchoRequest* request,						// 请求：EchoRequest
        EchoResponse* response,							// 响应：EchoResponse
        ::google::protobuf::Closure* done
    ) {
	    // 1. 获取方法描述符（索引 1 = Ping）
	    const ::google::protobuf::MethodDescriptor* method =
	        EchoService::descriptor()->method(1);
	    
	    // 2. 委托给 channel
	    channel_->CallMethod(
	        method,
	        controller,
	        request,
	        response,
	        done
	    );
	}
    
private:
    ::google::protobuf::RpcChannel* channel_;      // RPC 通道
    bool owns_channel_;                             // 是否拥有所有权
};

// RPC 通道抽象基类 ——> 客户端通过它发送请求到服务端（网络传输层抽象）
class RpcChannel {
public:
    virtual ~RpcChannel() = default;
    
    // 【核心】发起 RPC 调用
    virtual void CallMethod(
    	// 方法描述符（包含服务名、方法名、索引等元信息）
        const ::google::protobuf::MethodDescriptor* method,		
        // 控制器（用于取消、超时、错误处理）
        ::google::protobuf::RpcController* controller,		
        // 请求消息（已序列化或待序列化）
        const ::google::protobuf::Message* request,		
        // 响应消息（调用完成后由框架填充）
        ::google::protobuf::Message* response,			
        /*完成回调
        	- 同步调用：传 nullptr，CallMethod 阻塞直到完成	
        	- 异步调用：传回调对象，CallMethod 立即返回
        */
        ::google::protobuf::Closure* done					
    ) = 0;									
}; 	

} // namespace myservice
```
```cpp
#include <google/protobuf/service.h>
#include <google/protobuf/descriptor.h>
#include <google/protobuf/stubs/callback.h>

namespace google {
namespace protobuf {

// RPC 控制器抽象基类 ——> 用于控制单次 RPC 调用的行为（取消、超时、错误处理等）
class RpcController {
public:
    virtual ~RpcController() = default;

    // ============================== 客户端侧方法 ==============================
    
    // 重置控制器状态（复用控制器时调用）
    virtual void Reset() = 0;
    
    // 检查调用是否失败
    virtual bool Failed() const = 0;
    
    // 获取错误信息（Failed() 为 true 时有效）
    virtual std::string ErrorText() const = 0;
    
    // 发起取消请求（通知服务端停止处理）
    // 注意：服务端可能忽略，调用后 done 回调仍会被执行
    virtual void StartCancel() = 0;

   
    // ============================== 服务端侧方法 ==============================
    
    // 设置失败状态和错误信息
    virtual void SetFailed(const std::string& reason) = 0;
    
    // 检查客户端是否已请求取消
    virtual bool IsCanceled() const = 0;
    
    // 注册取消回调（当客户端取消时被调用）
    // @param callback: 取消时执行的回调，传 nullptr 取消注册
    virtual void NotifyOnCancel(Closure* callback) = 0;
};


// RPC 回调函数抽象基类 ——> 用于“异步操作”完成时的通知机制
class Closure {
public:
    virtual ~Closure() = default;
    
    // 执行回调（调用后对象可能被删除，取决于实现）
    virtual void Run() = 0;
};

// ============================== Closure 工厂函数 ==============================

// 创建一次性 Closure（Run() 后自动 delete this）
// @param function: 无参函数指针
inline Closure* NewCallback(void (*function)()) {
    // 内部实现：返回一个调用 function() 后自删除的 Closure
    return /* 实现细节 */;
}

// 创建一次性 Closure（带对象方法）
// @param object: 对象指针
// @param method: 成员函数指针
template <typename Class>
inline Closure* NewCallback(Class* object, void (Class::*method)()) {
    // 内部实现：返回一个调用 object->method() 后自删除的 Closure
    return /* 实现细节 */;
}

// 创建一次性 Closure（带参数）
template <typename Class, typename Arg1>
inline Closure* NewCallback(Class* object, void (Class::*method)(Arg1), Arg1 arg1) {
    return /* 实现细节 */;
}

// 创建永久 Closure（Run() 后不自动删除，可重复使用）
inline Closure* NewPermanentCallback(void (*function)()) {
    return /* 实现细节 */;
}

template <typename Class>
inline Closure* NewPermanentCallback(Class* object, void (Class::*method)()) {
    return /* 实现细节 */;
}		

} // namespace protobuf
} // namespace google
```

### （4）Stub类 与 RpcChannel类 解析
#### <1>使用伪代码
[自定义RpcChannel示例](#channel)
```cpp
	// ==================== 客户端使用 ====================
    // 1. 创建 Channel（注入网络实现）
    MyRpcChannel channel("127.0.0.1:8080");
    
    // 2. 创建 Stub（绑定 Channel）
    UserService_Stub stub(&channel);
    
    // 3. 创建 Controller
    MyRpcController controller;
    
    // 4. 准备请求和响应参数
    GetUserRequest request;
    request.set_user_id(12345);
    
    GetUserResponse response;
    
    // 5. 调用！
    stub.GetUser(&controller, &request, &response, nullptr);
    
    // 6. 检查结果
    if (controller.Failed()) {
        std::cerr << "RPC failed: " << controller.ErrorText() << std::endl;
    } else {
        std::cout << "User name: " << response.name() << std::endl;
    }
    
    return 0;
```
#### <2>Stub类 与 RpcChannel类 的关系
- 在上一节"**生成的C++伪代码**"中可以看到，每一个定义在 proto 中的服务（如：service UserService）都会生成对应的 XXX_Stub 类，这正是"**服务在客户端的句柄**"，客户端用它来发起 RPC 调用
	- 如：**stub.GetUser(&controller, &request, &response, nullptr);**
- Stub类：
	- 作为服务在客户端的句柄，内部包含该服务的**所有 RPC 方法调用接口**
	- 持有一个 RpcChannel* 数据成员，在构造时注入
		- 如： **UserService_Stub stub(&channel);**
	- 内部的 RPC 方法实现非常简单：**封装并调用 channel_->CallMethod(...)**
- RpcChannel类：
	- 理解它需要“[反射机制](#reflect)”的基础
	- **核心作用**：网络连接、序列化、反序列化
	- **工作流程**：
		- 1.通过 Stub 传入的 MethodDescriptor* 提取"服务名、方法名"
		- 2.将"服务名、方法名、request"序列化后通过网络发送到服务端
		- 3.接收响应并反序列化为 response
	- 这正好对应"RPC 底层原理"中所说：客户端通过 **（服务名, 方法）二元名组** 定位具体方法
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                      客户端 RPC 调用完整流程                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   用户代码                                                                   │
│   ┌─────────────────────────────────────────┐                               │
│   │ stub.GetUser(&ctrl, &req, &resp, done)  │                               │
│   └─────────────────┬───────────────────────┘                               │
│                     │                                                       │
│                     ▼                                                       │
│   Stub 类（Proto 生成）                                                      │
│   ┌─────────────────────────────────────────┐                               │
│   │ 1. 获取 MethodDescriptor（方法元信息）   │                               │
│   │ 2. 调用 channel_->CallMethod(...)       │                               │
│   └─────────────────┬───────────────────────┘                               │
│                     │                                                       │
│                     ▼                                                       │
│   RpcChannel（用户实现）                                                     │
│   ┌─────────────────────────────────────────┐                               │
│   │ 1. 从 MethodDescriptor 提取服务名、方法名│                               │
│   │ 2. 序列化 request                       │                               │
│   │ 3. 构建协议包 [服务名][方法名][数据]      │                               │
│   │ 4. 通过 socket 发送到服务端              │                               │
│   │ 5. 接收响应数据                          │                               │
│   │ 6. 反序列化到 response                  │                               │
│   └─────────────────────────────────────────┘                               │
│                     │                                                       │
│                     ▼                                                       │
│   ┌─────────────────────────────────────────┐                               │
│   │ 用户拿到 response，继续业务逻辑          │                               │
│   └─────────────────────────────────────────┘                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
```cpp
┌──────────────────────────────────────────────────────────────────────────┐
│                          工厂类比                                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   你（用户）                                                              │
│     │                                                                    │
│     │ 带着原材料（request）                                               │
│     │ 指定产品交付处（response）                                          │
│     ▼                                                                    │
│   ┌────────────────┐                                                     │
│   │   门  户        │  ◄─── Stub 类                                      │
│   │   (Stub)       │       • 接收原材料                                  │
│   │                │       • 登记加工需求（获取 MethodDescriptor）         │
│   └───────┬────────┘       • 转交给内部员工                              │
│           │                                                              │
│           ▼                                                              │
│   ┌────────────────┐                                                     │
│   │  内部员工       │  ◄─── RpcChannel                                   │
│   │  (RpcChannel)  │       • 打包原材料（序列化）                         │
│   │                │       • 运送到加工车间（网络传输）                    │
│   │                │       • 带回成品（接收响应）                         │
│   │                │       • 拆包（反序列化）                             │
│   └───────┬────────┘                                                     │
│           │                                                              │
│           │  ════════════════════════════════                            │
│           │        网络（运输通道）                                        │
│           │  ════════════════════════════════                            │
│           ▼                                                              │
│   ┌────────────────┐                                                     │
│   │  加工车间       │  ◄─── 服务端                                        │
│   │  (Server)      │       • 根据订单（服务名+方法名）找到机器             │
│   │                │       • 加工原材料                                   │
│   │                │       • 产出成品                                     │
│   └────────────────┘                                                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### （5）RpcController类 与 Closure类
#### <1> RpcController 类
**核心功能**：
	- 控制单次 RPC 调用的行为（取消、超时、错误处理等）
**设计原因**：
	- CallMethod 方法的返回值是 void，调用结果（成功/失败/错误信息）无法通过返回值告知用户
	- 因此通过"**出参**"的形式将 RpcController 传入，**内部信息填充后，用户即可获取调用状态**
```cpp
// 使用示例
MyRpcController controller;
stub.GetUser(&controller, &request, &response, nullptr);

// 通过 controller 获取调用状态
if (controller.Failed()) {
    std::cerr << "Error: " << controller.ErrorText() << std::endl;
}
```

#### <2> Closure 类
[Closure 常用构造方法](#Closure)
**核心功能**：决定是"同步操作"还是"异步操作"
- **同步**：即阻塞，用户在通过Stub调用函数时，会一直阻塞，直到函数处理结束后将响应填入“reponse出参”，用户再处理reponse
- **异步**：即非阻塞，用户事先将reponse的处理逻辑封装到Clouser中（回调函数），并在通过Stub调用函数时一同传入，然后函数就直接返回，用户可以直接去做其它业务，因为响应到来时，直接交给Clouser中封装好的处理逻辑去处理即可
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122100053975.png)
```cpp
// ========== 统一的 response 处理函数 ==========
void HandleResponse(GetUserResponse* response) {
    std::cout << "User name: " << response->name() << std::endl;
    std::cout << "User age: " << response->age() << std::endl;
    // ... 其他业务逻辑
}

// ========== 同步调用 ==========
stub.GetUser(&controller, &request, &response, nullptr);  // 阻塞等待
HandleResponse(&response);  // 响应就绪后，手动调用处理函数

// ========== 异步调用 ==========
auto* done = NewCallback(&HandleResponse, &response);     // 将处理函数封装为 Closure
stub.GetUser(&controller, &request, &response, done);     // 立即返回
// 继续做其他事情...
// 响应到来时，框架自动执行 HandleResponse(&response)
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122100154823.png)

### （6）服务端的使用
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122103313708.png)
- 从“生成的C++伪代码”中我们可以看到，proto中定义的service，会生成一个同名的Service类（如：service EchoService ——>class EchoService）
- Service类中的方法（如Echo、Ping）都是纯虚函数，需要服务端自己继承并实现（毕竟要实现的是自己的业务嘛）
- 服务端会对每一个服务进行注册，如：**std::map(服务名，Service)**
	- 为什么不注册方法？因为Protobuf的“**反射机制**”在获取去Service后，可以通过“方法名”获取“**方法描述符MethodDescriptor**” ——> 结合起来，正好对应（服务名，方法名）二元组定位具体方法
- **CallMethod方法**：
	- 会传入“方法描述符MethodDescriptor”，在内部根据“**方法索引**”找到具体方法进行执行

**用户实现服务**
```cpp
// ========== 用户继承并实现业务逻辑 ==========
class EchoServiceImpl : public EchoService::Service {
public:
    Status Echo(ServerContext* ctx, const EchoRequest* req, EchoResponse* resp) override {
        resp->set_message("Echo: " + req->message());  // 业务逻辑
        return Status::OK;
    }
    
    Status Ping(ServerContext* ctx, const PingRequest* req, PingResponse* resp) override {
        resp->set_pong(true);
        return Status::OK;
    }
};
```
**服务注册（服务端框架）**
```cpp
// ========== gRPC 服务端内部注册表 ==========
class ServiceRegistry {
    // 只注册服务，不注册方法！
    std::map<std::string, google::protobuf::Service*> services_;
    //        服务名（如 "EchoService"）    服务实例
    
public:
    void RegisterService(google::protobuf::Service* service) {
        // 通过反射获取服务名
        const std::string& name = service->GetDescriptor()->name();
        services_[name] = service;
    }
    
    google::protobuf::Service* FindService(const std::string& name) {
        return services_[name];
    }
}
```
**请求分发流程**
```cpp
// ========== 收到 RPC 请求时的处理 ==========
void HandleRpcRequest(
    const std::string& service_name,   // 如 "EchoService"
    const std::string& method_name,    // 如 "Echo"
    const std::string& request_data    // 序列化的请求
) {
    // Step 1: 根据服务名找到服务实例
    auto* service = registry_.FindService(service_name);
    
    // Step 2: 通过反射，根据方法名获取方法描述符
    const ServiceDescriptor* service_desc = service->GetDescriptor();
    const MethodDescriptor* method_desc = service_desc->FindMethodByName(method_name);
    //                                    ↑ 【反射机制】方法名 → 方法描述符
    
    // Step 3: 通过反射创建请求/响应对象
    Message* request = service->GetRequestPrototype(method_desc).New();
    Message* response = service->GetResponsePrototype(method_desc).New();
    request->ParseFromString(request_data);
    
    // Step 4: 调用 CallMethod，内部根据方法索引分发到具体实现
    service->CallMethod(method_desc, controller, request, response, done);
    //                  ↑ 传入方法描述符，CallMethod 内部用 method->index() 分发
}
```
**流程图**
```cpp
客户端请求: { service: "EchoService", method: "Echo", data: ... }
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 1: 服务名 → 服务实例                                    │
│          registry_["EchoService"] → EchoServiceImpl*         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2: 方法名 → 方法描述符（反射）                           │
│          service_desc->FindMethodByName("Echo")              │
│          → MethodDescriptor { index=0, name="Echo", ... }    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3: CallMethod 根据 method->index() 分发                │
│          switch(0) → Echo(ctx, req, resp)                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    EchoServiceImpl::Echo() 执行
```


<a id="channel"></a>
### 附录1：自定义RpcChannel示例
```cpp
//自定义 RpcChannel
class MyRpcChannel : public ::google::protobuf::RpcChannel {
public:
   MyRpcChannel(const std::string& addr) : server_addr_(addr) {
       socket_fd_ = connectToServer(addr);
   }
   
   ~MyRpcChannel() {
       close(socket_fd_);
   }
   
   void CallMethod(
       const ::google::protobuf::MethodDescriptor* method,
       ::google::protobuf::RpcController* controller,
       const ::google::protobuf::Message* request,
       ::google::protobuf::Message* response,
       ::google::protobuf::Closure* done) override 
   {
       // ========== 1. 序列化请求 ==========
       std::string request_data = request->SerializeAsString();
       
       // ========== 2. 构建协议包 ==========
       // 协议：[服务名长度][服务名][方法名长度][方法名][数据长度][数据]
       std::string service_name = method->service()->full_name();  // "package.UserService"
       std::string method_name = method->name();                    // "GetUser"
       
       uint32_t service_len = service_name.size();
       uint32_t method_len = method_name.size();
       uint32_t data_len = request_data.size();
       
       // ========== 3. 发送请求 ==========
       // 服务名
       send(socket_fd_, &service_len, sizeof(service_len), 0);
       send(socket_fd_, service_name.data(), service_len, 0);
       // 方法名
       send(socket_fd_, &method_len, sizeof(method_len), 0);
       send(socket_fd_, method_name.data(), method_len, 0);
       // 数据
       send(socket_fd_, &data_len, sizeof(data_len), 0);
       send(socket_fd_, request_data.data(), data_len, 0);
       
       // ========== 4. 接收响应 ==========
       uint32_t resp_len;
       recv(socket_fd_, &resp_len, sizeof(resp_len), 0);
       
       std::vector<char> resp_data(resp_len);
       recv(socket_fd_, resp_data.data(), resp_len, 0);
       
       // ========== 5. 反序列化响应 ==========
       if (!response->ParseFromArray(resp_data.data(), resp_len)) {
           controller->SetFailed("Failed to parse response");
       }
       
       // ========== 6. 回调 ==========
       if (done != nullptr) {
           done->Run();
       }
   }
};

private:
	// ==================== 辅助函数：建立连接 ====================
	int connectToServer(const std::string& addr) {
	   // 解析地址 "127.0.0.1:8080"
	   size_t pos = addr.find(':');
	   std::string ip = addr.substr(0, pos);
	   int port = std::stoi(addr.substr(pos + 1));
	   
	   // 创建 socket
	   int fd = socket(AF_INET, SOCK_STREAM, 0);
	   if (fd < 0) {
	       throw std::runtime_error("Failed to create socket");
	   }
	   
	   // 设置服务器地址
	   struct sockaddr_in server_addr;
	   memset(&server_addr, 0, sizeof(server_addr));
	   server_addr.sin_family = AF_INET;
	   server_addr.sin_port = htons(port);
	   inet_pton(AF_INET, ip.c_str(), &server_addr.sin_addr);
	   
	   // 连接
	   if (connect(fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
	       close(fd);
	       throw std::runtime_error("Failed to connect to server");
	   }
	   
	   return fd;
	}

private:
   std::string server_addr_;
   int socket_fd_;
```

**协议格式图示**：
```cpp
┌────────────────────────────────────────────────────────────────────────┐
│                          自定义 RPC 协议包格式                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   字节偏移:  0        4        4+S      8+S      8+S+M    12+S+M       │
│            ┌────────┬────────┬────────┬────────┬────────┬────────────┐ │
│            │服务名   │ 服务名  │方法名   │ 方法名  │数据     │   数据     │ │
│            │长度(4B)│ (S字节) │长度(4B)│ (M字节) │长度(4B)│  (D字节)   │ │
│            └────────┴────────┴────────┴────────┴────────┴────────────┘ │
│                                                                        │
│   示例：                                                                │
│   ┌────┬─────────────────────┬────┬─────────┬────┬──────────────────┐  │
│   │ 19 │ "package.UserService"│  7 │"GetUser"│ 12 │ <protobuf数据>   │  │
│   └────┴─────────────────────┴────┴─────────┴────┴──────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

<a id="Closure"></a>
### 附录2：Closure 常用构造方法
Google Protobuf 提供了 NewCallback 系列**工厂函数**来创建 Closure：
```cpp
// ========== 1. 普通函数 ==========
void HandleResponse(GetUserResponse* response) { /* ... */ }

auto* done = NewCallback(&HandleResponse, &response);

// ========== 2. 成员函数（需要对象指针）==========
class MyHandler {
public:
    void OnDone(GetUserResponse* response) { /* ... */ }
};

MyHandler handler;
auto* done = NewCallback(&handler, &MyHandler::OnDone, &response);

// ========== 3. 无参回调 ==========
void OnComplete() { /* ... */ }

auto* done = NewCallback(&OnComplete);

// ========== 4. 多参数回调 ==========
void ProcessResult(int code, GetUserResponse* response) { /* ... */ }

auto* done = NewCallback(&ProcessResult, 200, &response);

// ========== 5. 永久回调（可重复执行，不会自动删除）==========
auto* done = NewPermanentCallback(&HandleResponse, &response);
// 注意：需要手动 delete
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122100447091.png)


## 3.Service在RPC中的作用
[RPC底层原理](#RPC（2）)
通过proto的service生成的C++伪代码，已经满足了RPC的如下条件
- 1.双方定义一模一样的API
	- 通过上述C++伪代码可以看到，服务端（EchoService）与客户端（EchoService_Stub）拥有相同的“核心API”
		- 如：Echo、Ping、CallMethod（其中客户端通过RpcChannel间接拥有CallMethod）
- 2.“序列化”、“反序列化”能力
	- proto中的message自带“序列化”、“反序列化”能力
- **欠缺**：
	- **服务端注册方法**：
		- 这需要服务端借助 “**反射机制**” 获取 “服务名、方法名”
	- **网络通信**：
		- 这需要自己实现（或使用已有框架）
	- **路由**：
		- 这也需要服务端借助 “**反射机制**” 获取 “服务名、方法名”

**PRC在proto中的调用链路**
```cpp
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            完整 RPC 调用链路                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  客户端                                              服务端                      │
│  ──────                                              ──────                      │
│                                                                                 │
│  ① 业务代码调用                                                                  │
│     stub->Echo(controller, &req, &resp, done)                                   │
│         │                                                                       │
│         ▼                                                                       │
│  ② Stub::Echo() 转发给 Channel                                                  │
│     channel_->CallMethod(descriptor()->method(0), ...)                          │
│         │                                                                       │
│         ▼                                                                       │
│  ③ RpcChannel::CallMethod() 序列化 + 发送                                       │
│     • 序列化 request → bytes                                                    │
│     • 构造包头：service="EchoService", method="Echo", index=0                   │
│     • 发送到网络                                                                │
│         │                                                                       │
│         │ ═══════════════════ 网络传输 ═══════════════════                      │
│         ▼                                                                       │
│  ④ 服务端框架接收                                              ┌──────────────┐ │
│     • 解析包头，找到 ServiceDescriptor                          │  RPC Server  │ │
│     • 根据 method_index 获取 MethodDescriptor                  │              │ │
│     • 创建 request/response 实例（通过反射）                     │  接收数据    │ │
│     • 反序列化 request                                          │  解析路由    │ │
│         │                                                      └──────┬───────┘ │
│         ▼                                                             │         │
│  ⑤ 调用 Service::CallMethod()                                         │         │
│     service->CallMethod(method, controller, request, response, done)  │         │
│         │                                                             │         │
│         ▼                                                             │         │
│  ⑥ CallMethod 路由到具体方法                                                     │
│     switch (method->index())                                                    │
│         case 0: Echo(...)  ←── 你实现的业务逻辑                                  │
│         │                                                                       │
│         ▼                                                                       │
│  ⑦ 业务处理完成，response 已填充                                                 │
│     done->Run()  // 触发响应发送                                                │
│         │                                                                       │
│         │ ═══════════════════ 网络传输 ═══════════════════                      │
│         ▼                                                                       │
│  ⑧ 客户端接收响应                                                                │
│     • 反序列化 response                                                         │
│     • 调用 done->Run()（异步）或直接返回（同步）                                  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

<a id="reflect"></a>
## 4.Protobuf的反射机制
[RPC底层原理](#RPC（2）)
- 什么是“反射机制”？说实话，我自己现在也不是很明白，但我们只需要知道以下几点即可
	- 通过反射机制，我们可以获取 **ServiceDescriptor、MethodDescriptor**，进而可以获取“服务名”、“方法名”、“请求消息类型描述符”、“响应消息类型描述符”等
	- 获得“服务名”、“方法名”等，客户端就可以封装到“请求中”，二服务端就可以实现“服务的注册”、“路由”等功能。

### （1）Protobuf的反射层次结构
```cpp
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        Protobuf 反射层次结构                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  DescriptorPool（描述符池）                                                       │
│       │   • 全局唯一的描述符管理器                                                 │
│       │   • 存储所有已加载的 .proto 文件描述信息                                    │
│       │   • 提供按名称查找描述符的能力                                             │
│       │   • generated_pool() 获取编译期生成的描述符池                              │
│       │                                                                         │
│       ▼                                                                         │
│  FileDescriptor（文件描述符）                                                     │
│       │   • 对应一个 .proto 文件                                                  │
│       │   • 包含文件名、包名、依赖关系等元信息                                       │
│       │   • name()        → "echo.proto"                                        │
│       │   • package()     → "myservice"                                         │
│       │   • dependency()  → 导入的其他 .proto 文件                                │
│       │                                                                         │
│       │                                                                         │
│       ├──▶ MessageDescriptor[]（消息描述符数组）                                  │
│       │         │   • 对应一个 message 定义                                       │
│       │         │   • name()         → "EchoRequest"                            │
│       │         │   • full_name()    → "myservice.EchoRequest"                  │
│       │         │   • field_count()  → 字段数量                                  │
│       │         │   • nested_type()  → 嵌套的 message                            │
│       │         │                                                               │
│       │         ├──▶ FieldDescriptor[]（字段描述符数组）                          │
│       │         │         • 对应 message 中的每个字段                             │
│       │         │         • name()         → "message"                          │
│       │         │         • number()       → 1（字段编号）                        │
│       │         │         • type()         → TYPE_STRING / TYPE_INT32 / ...     │
│       │         │         • is_repeated()  → 是否为 repeated                     │
│       │         │         • is_optional()  → 是否为 optional                     │
│       │         │         • default_value_string() → 默认值                      │
│       │         │         • message_type() → 如果是嵌套消息，返回其描述符          │
│       │         │                                                               │
│       │         └──▶ EnumDescriptor[]（枚举描述符数组）                           │
│       │                   • 对应 enum 定义                                       │
│       │                   • name()         → "Status"                           │
│       │                   • value_count()  → 枚举值数量                          │
│       │                   └──▶ EnumValueDescriptor[]                            │
│       │                             • name()   → "OK"                           │
│       │                             • number() → 0                              │
│       │                                                                         │
│       │                                                                         │
│       └──▶ ServiceDescriptor[]（服务描述符数组）                                  │
│                 │   • 对应一个 service 定义                                       │
│                 │   • name()         → "EchoService"                            │
│                 │   • full_name()    → "myservice.EchoService"                  │
│                 │   • method_count() → RPC 方法数量                              │
│                 │                                                               │
│                 └──▶ MethodDescriptor[]（方法描述符数组）                         │
│                           • 对应 service 中的每个 rpc 方法                        │
│                           • name()          → "Echo"                            │
│                           • full_name()     → "myservice.EchoService.Echo"      │
│                           • index()         → 方法在 service 中的索引             │
│                           • input_type()    → 请求消息的 MessageDescriptor       │
│                           • output_type()   → 响应消息的 MessageDescriptor       │
│                           • client_streaming() → 客户端是否流式                   │
│                           • server_streaming() → 服务端是否流式                   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### （2）ServiceDescriptor（服务描述符）
```cpp
class ServiceDescriptor {
public:

    const std::string& name() const;		    		// 服务名称（如 "EchoService"）

    const std::string& full_name() const;	    		// 完整名称（如 "myservice.EchoService"）

    int index() const;						    		// 在文件中的索引
    

    const FileDescriptor* file() const;		    		// 所属文件
    

    int method_count() const;				    		// 方法数量
    
	// 按索引获取方法描述符：O(log n)
    const MethodDescriptor* method(int index) const {	
        // 边界检查
        if (index < 0 || index >= static_cast<int>(methods_.size())) {
            return nullptr;
        }
        return methods_[index];
    }	    
    

    // 按名称查找方法描述符O()
    const MethodDescriptor* FindMethodByName(const std::string& name) const {
        auto it = methods_by_name_.find(name);
        if (it == methods_by_name_.end()) {
            return nullptr;
        }
        return it->second;
    }
    
    const ServiceOptions& options() const;			    // 选项
    
    std::string DebugString() const;				    // 调试
private:
    std::string name_;                                    // 服务名
    std::vector<const MethodDescriptor*> methods_;        // 方法列表（按定义顺序）
    std::map<std::string, const MethodDescriptor*> methods_by_name_;  // 名称索引
};

```

### （3）MethodDescriptor （方法描述符）
```cpp
class MethodDescriptor {
public:
    const std::string& name() const;		    // 方法名称（如 "Echo"）
    

    const std::string& full_name() const;	    // 完整名称（如 "myservice.EchoService.Echo"）
    

    int index() const;						    // 方法在服务中的索引
    
    const ServiceDescriptor* service() const;   // 所属服务描述符
    
    const FileDescriptor* file() const;		    // 所属文件描述符
    
    const Descriptor* input_type() const;	    // 请求消息类型描述符
    
    const Descriptor* output_type() const;	    // 响应消息类型描述符
    

    // 流式标记（Proto3 支持）
    bool client_streaming() const;			    // 客户端是否为流式
    

    bool server_streaming() const;			    // 服务端是否为流式
    
    // 选项
    const MethodOptions& options() const;
    
    // 调试
    std::string DebugString() const;
};
```


### （4）DescriptorPool（描述符池）
```cpp
class DescriptorPool {
public:
    // --------------------------------------------------
    // 单例访问
    // --------------------------------------------------
    
    // 获取编译期生成的全局描述符池（包含所有 .pb.cc 注册的类型）
    static const DescriptorPool* generated_pool();
    
    // --------------------------------------------------
    // 构造/析构
    // --------------------------------------------------
    
    // 创建空的描述符池
    DescriptorPool();
    
    // 创建带回退池的描述符池（找不到时从 fallback_pool 查找）
    explicit DescriptorPool(const DescriptorPool* fallback_pool);
    
    ~DescriptorPool();
    
    // 查找方法（如 "myservice.EchoService.Echo"）
    const MethodDescriptor* FindMethodByName(const std::string& name) const;

    // 查找服务（如 "myservice.EchoService"）
    const ServiceDescriptor* FindServiceByName(const std::string& name) const;
    
    // 查找文件描述符（如 "echo.proto"）
    const FileDescriptor* FindFileByName(const std::string& name) const;
    
    // 查找消息类型（如 "myservice.EchoRequest"）
    const Descriptor* FindMessageTypeByName(const std::string& name) const;
    
    // 查找字段（全局扩展字段）
    const FieldDescriptor* FindFieldByName(const std::string& name) const;
    
    // 查找扩展字段（按编号）
    const FieldDescriptor* FindExtensionByNumber(
        const Descriptor* extendee, 
        int number
    ) const;
    
    // 查找枚举类型（如 "myservice.Status"）
    const EnumDescriptor* FindEnumTypeByName(const std::string& name) const;
    
    // 查找枚举值（如 "myservice.Status.OK"）
    const EnumValueDescriptor* FindEnumValueByName(const std::string& name) const;
    

    
    // --------------------------------------------------
    // 动态构建（运行时加载 .proto）
    // --------------------------------------------------
    
    // 从 FileDescriptorProto 构建（proto 的自描述格式）
    const FileDescriptor* BuildFile(const FileDescriptorProto& proto);
    
    // 带错误收集器的构建
    const FileDescriptor* BuildFileCollectingErrors(
        const FileDescriptorProto& proto,
        ErrorCollector* error_collector
    );
    
private:
    // 禁止拷贝
    DescriptorPool(const DescriptorPool&) = delete;
    DescriptorPool& operator=(const DescriptorPool&) = delete;
};
```

### （5）FileDescriptor（文件描述符）
```cpp
class FileDescriptor {
public:

    const std::string& name() const;	    // 文件名（如 "echo.proto"）
    const std::string& package() const;	    // 包名（如 "myservice"）
    const DescriptorPool* pool() const;     // 所属的描述符池
    
    enum Syntax {						    // 语法版本
        SYNTAX_UNKNOWN = 0,
        SYNTAX_PROTO2 = 2,
        SYNTAX_PROTO3 = 3,
    };

    Syntax syntax() const;				    // 获取语法版本
    

    int dependency_count() const;		    // 直接依赖的文件数量
    

    // ========================================= 文件描述符相关 =========================================
    const FileDescriptor* dependency(int index) const;		    // 获取第 i 个依赖文件

    int public_dependency_count() const;					    // public import 的数量

    const FileDescriptor* public_dependency(int index) const;   // 获取第 i 个 public 依赖

    int weak_dependency_count() const;						    // weak import 的数量

    const FileDescriptor* weak_dependency(int index) const;	    // 获取第 i 个 weak 依赖
    
    // ========================================= message相关 =========================================
    int message_type_count() const;							    // 顶层 message 数量
    const Descriptor* message_type(int index) const;		    // 获取第 i 个 message 描述符
    const Descriptor* FindMessageTypeByName(const std::string& name) const;    // 按名称查找 message（只在当前文件查找）
    
    // ========================================= enum相关 =========================================
    int enum_type_count() const;	    // 顶层 enum 数量
    const EnumDescriptor* enum_type(int index) const;	    // 获取第 i 个 enum 描述符
    const EnumDescriptor* FindEnumTypeByName(const std::string& name) const;    // 按名称查找 enum
    const EnumValueDescriptor* FindEnumValueByName(const std::string& name) const;    // 按名称查找 enum 值
    
    
    // ========================================= service相关 =========================================
    int service_count() const;	    // service 数量
    const ServiceDescriptor* service(int index) const;    // 获取第 i 个 service 描述符
    const ServiceDescriptor* FindServiceByName(const std::string& name) const;    // 按名称查找 service
    

	// ========================================= 扩展相关 =========================================    

    int extension_count() const;    // 顶层扩展字段数量
    const FieldDescriptor* extension(int index) const;    // 获取第 i 个扩展字段
    const FieldDescriptor* FindExtensionByName(const std::string& name) const;    // 按名称查找扩展
    const FieldDescriptor* FindExtensionByLowercaseName(const std::string& name) const;    // 按被扩展类型和编号查找
    
    // 选项
    const FileOptions& options() const;
    
    // ========================================= 调试 =========================================
    
    // 输出 .proto 格式的字符串表示
    std::string DebugString() const;
    // 输出带源码位置信息的字符串
    std::string DebugStringWithOptions(const DebugStringOptions& options) const;
};
```

### （6）MessageDescriptor（消息描述符）
```cpp
class Descriptor {
public:    
    const std::string& name() const;			    	// 消息名称（如 "EchoRequest"）
    
    const std::string& full_name() const;		    	// 完整名称（如 "myservice.EchoRequest"）    

    int index() const;							    	// 在父容器中的索引
    
    const FileDescriptor* file() const;			   	 	// 所属文件
    
    const Descriptor* containing_type() const;	    	// 父消息（如果是嵌套定义） 
    
    // ===================================== 字段 =====================================
    // 字段数量（不含扩展）
    int field_count() const;							
    // 获取第 i 个字段
    const FieldDescriptor* field(int index) const;
    // 按名称查找字段
    const FieldDescriptor* FindFieldByName(const std::string& name) const;
    // 按字段编号查找
    const FieldDescriptor* FindFieldByNumber(int number) const;
    // 按小写名称查找（用于 JSON 映射）
    const FieldDescriptor* FindFieldByLowercaseName(const std::string& lowercase_name) const;
    // 按驼峰名称查找
    const FieldDescriptor* FindFieldByCamelcaseName(const std::string& camelcase_name) const;
 
    // ===================================== oneof（联合字段） =====================================
    int oneof_decl_count() const;    // oneof 数量
    const OneofDescriptor* oneof_decl(int index) const;    // 获取第 i 个 oneof
    const OneofDescriptor* FindOneofByName(const std::string& name) const;    // 按名称查找 oneof
    // 真实 oneof 数量（不含 synthetic oneof，即 optional 的内部实现）
    int real_oneof_decl_count() const;
    
    // ===================================== 嵌套类型 =====================================
    // 嵌套 message 数量
    int nested_type_count() const;
    // 获取第 i 个嵌套 message
    const Descriptor* nested_type(int index) const;
    // 按名称查找嵌套 message
    const Descriptor* FindNestedTypeByName(const std::string& name) const;
    

    // ===================================== 嵌套枚举 =====================================
    // 嵌套 enum 数量
    int enum_type_count() const;
    // 获取第 i 个嵌套 enum
    const EnumDescriptor* enum_type(int index) const;
    // 按名称查找嵌套 enum
    const EnumDescriptor* FindEnumTypeByName(const std::string& name) const;
    // 按值名称查找 enum value
    const EnumValueDescriptor* FindEnumValueByName(const std::string& name) const;
    
    // ===================================== 扩展范围=====================================
    struct ExtensionRange {
        int start;  // 起始编号（含）
        int end;    // 结束编号（不含）
        const ExtensionRangeOptions* options;
    };
    // 扩展范围数量
    int extension_range_count() const;
    // 获取第 i 个扩展范围
    const ExtensionRange* extension_range(int index) const;
    // 检查编号是否在扩展范围内
    bool IsExtensionNumber(int number) const;
    

    // ===================================== 扩展字段=====================================
    int extension_count() const;
    const FieldDescriptor* extension(int index) const;
    const FieldDescriptor* FindExtensionByName(const std::string& name) const;
    

    // ===================================== 保留字段（reserved）=====================================
    // 保留的编号范围数量
    int reserved_range_count() const;
    
    struct ReservedRange {
        int start;  // 起始编号（含）
        int end;    // 结束编号（含）
    };
    const ReservedRange* reserved_range(int index) const;
    // 保留的名称数量
    int reserved_name_count() const;
    // 获取第 i 个保留名称
    const std::string& reserved_name(int index) const;

    // ===================================== 特性检查 =====================================   
    // 是否是 map entry（map<K,V> 内部生成的消息）
    bool map_key() const;
    bool map_value() const;
    // 获取 map 的 key/value 字段描述符
    const FieldDescriptor* map_key() const;
    const FieldDescriptor* map_value() const;
    

    // 选项
    const MessageOptions& options() const;
    

    // 调试
    std::string DebugString() const;
};

```

### （7）FieldDescriptor（字段描述符）
```cpp
class FieldDescriptor {
public:
    // --------------------------------------------------
    // 字段类型枚举
    // --------------------------------------------------
    
    enum Type {
        TYPE_DOUBLE         = 1,
        TYPE_FLOAT          = 2,
        TYPE_INT64          = 3,
        TYPE_UINT64         = 4,
        TYPE_INT32          = 5,
        TYPE_FIXED64        = 6,
        TYPE_FIXED32        = 7,
        TYPE_BOOL           = 8,
        TYPE_STRING         = 9,
        TYPE_GROUP          = 10,  // 已废弃
        TYPE_MESSAGE        = 11,  // 嵌套消息
        TYPE_BYTES          = 12,
        TYPE_UINT32         = 13,
        TYPE_ENUM           = 14,
        TYPE_SFIXED32       = 15,
        TYPE_SFIXED64       = 16,
        TYPE_SINT32         = 17,
        TYPE_SINT64         = 18,
        MAX_TYPE            = 18,
    };
    
    // C++ 类型枚举（用于代码生成）
    enum CppType {
        CPPTYPE_INT32       = 1,
        CPPTYPE_INT64       = 2,
        CPPTYPE_UINT32      = 3,
        CPPTYPE_UINT64      = 4,
        CPPTYPE_DOUBLE      = 5,
        CPPTYPE_FLOAT       = 6,
        CPPTYPE_BOOL        = 7,
        CPPTYPE_ENUM        = 8,
        CPPTYPE_STRING      = 9,
        CPPTYPE_MESSAGE     = 10,
        MAX_CPPTYPE         = 10,
    };
    
    // 标签（proto2）
    enum Label {
        LABEL_OPTIONAL      = 1,
        LABEL_REQUIRED      = 2,  // proto2 only
        LABEL_REPEATED      = 3,
    };
    
    // --------------------------------------------------
    // 基本信息
    // --------------------------------------------------
    
    // 字段名称（如 "message"）
    const std::string& name() const;
    
    // 完整名称（如 "myservice.EchoRequest.message"）
    const std::string& full_name() const;
    
    // 字段编号
    int number() const;
    
    // 在消息中的索引
    int index() const;
    
    // --------------------------------------------------
    // 类型信息
    // --------------------------------------------------
    
    // Proto 类型
    Type type() const;
    
    // 类型名称字符串（如 "string", "int32"）
    const char* type_name() const;
    
    // C++ 类型
    CppType cpp_type() const;
    
    // C++ 类型名称字符串
    static const char* CppTypeName(CppType cpp_type);
    
    // 标签
    Label label() const;
    
    // --------------------------------------------------
    // 类型判断
    // --------------------------------------------------
    
    // 是否为 repeated
    bool is_repeated() const;
    
    // 是否为 optional（proto3 显式 optional 或 proto2 optional）
    bool is_optional() const;
    
    // 是否为 required（proto2）
    bool is_required() const;
    
    // 是否为 map 类型
    bool is_map() const;
    
    // 是否为 packed 编码
    bool is_packed() const;
    
    // 是否为扩展字段
    bool is_extension() const;
    
    // --------------------------------------------------
    // 所属关系
    // --------------------------------------------------
    
    // 所属文件
    const FileDescriptor* file() const;
    
    // 所属消息（普通字段）
    const Descriptor* containing_type() const;
    
    // 被扩展的消息（扩展字段）
    const Descriptor* extension_scope() const;
    
    // 所属 oneof（如果有）
    const OneofDescriptor* containing_oneof() const;
    
    // 真实 oneof 索引（-1 表示不属于 oneof）
    int index_in_oneof() const;
    
    // --------------------------------------------------
    // 嵌套类型信息
    // --------------------------------------------------
    
    // 如果是 TYPE_MESSAGE 或 TYPE_GROUP，返回消息描述符
    const Descriptor* message_type() const;
    
    // 如果是 TYPE_ENUM，返回枚举描述符
    const EnumDescriptor* enum_type() const;
    
    // --------------------------------------------------
    // 默认值
    // --------------------------------------------------
    
    // 是否有显式设置默认值
    bool has_default_value() const;
    
    // 各类型的默认值获取
    int32_t default_value_int32() const;
    int64_t default_value_int64() const;
    uint32_t default_value_uint32() const;
    uint64_t default_value_uint64() const;
    float default_value_float() const;
    double default_value_double() const;
    bool default_value_bool() const;
    const std::string& default_value_string() const;
    const EnumValueDescriptor* default_value_enum() const;
    
    // --------------------------------------------------
    // JSON 相关
    // --------------------------------------------------
    
    // JSON 字段名（默认用驼峰命名）
    const std::string& json_name() const;
    
    // 小写名称（下划线）
    const std::string& lowercase_name() const;
    
    // 驼峰名称
    const std::string& camelcase_name() const;
    
    // --------------------------------------------------
    // 选项
    // --------------------------------------------------
    
    const FieldOptions& options() const;
    
    // --------------------------------------------------
    // 调试
    // --------------------------------------------------
    
    std::string DebugString() const;
};
```

# 五、gRPC 的使用
## 1.gRPC 编译命令
### （1）基本编译命令
```bash
# 生成 C++ 代码（同时生成 protobuf 和 grpc 代码）
protoc --cpp_out=. --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` your_service.proto
```
生成文件：
- your_service.pb.h / your_service.pb.cc — Protobuf 消息类
- your_service.grpc.pb.h / your_service.grpc.pb.cc — gRPC 服务类（Stub 等）

### （2）指定输出目录
```cpp
protoc -I=./proto \
       --cpp_out=./generated \
       --grpc_out=./generated \
       --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` \
       ./proto/your_service.proto
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122101152562.png)


## 2.四种 RPC 模式
Proto 定义了 4 种标准 RPC 模式，核心区别是「**请求/响应是否为流式**」，每种模式对应 C++ 编译后不同的函数签名和调用方式
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260120111458667.png)
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                        四种 RPC 模式                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Unary:            Client ──Req──▶ Server ──Resp──▶ Client     │
│                                                                 │
│  Server Stream:    Client ──Req──▶ Server ══Resp══▶            │
│                                           ══Resp══▶            │
│                                           ══Resp══▶ Client     │
│                                                                 │
│  Client Stream:    Client ══Req══▶                              │
│                           ══Req══▶                              │
│                           ══Req══▶ Server ──Resp──▶ Client     │
│                                                                 │
│  Bidirectional:    Client ══Req══▶ Server                       │
│                           ◀══Resp══                             │
│                           ══Req══▶                              │
│                           ◀══Resp══  (实时双向)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

图例: ── 单条消息   ══ 流式消息
```

## 3.使用示例（综合）
### （1）Proto定义
```cpp
syntax = "proto3";

package demo;

// 请求和响应消息定义
message Request {
    string data = 1;
}

message Response {
    string result = 1;
}

// 定义四种RPC模式的服务
service DemoService {
    // 1. Unary（一元RPC）：单请求-单响应
    rpc UnaryCall(Request) returns (Response);
    
    // 2. Server Streaming（服务端流）：单请求-多响应流
    rpc ServerStreamingCall(Request) returns (stream Response);
    
    // 3. Client Streaming（客户端流）：多请求流-单响应
    rpc ClientStreamingCall(stream Request) returns (Response);
    
    // 4. Bidirectional Streaming（双向流）：多请求流-多响应流
    rpc BidirectionalStreamingCall(stream Request) returns (stream Response);
}
```

### （2）生成的C++伪代码
运行 protoc --grpc_out=. --cpp_out=. demo.proto 后生成

#### <1>服务端基类（demo.grpc.pb.h）
```cpp
namespace demo {
class ChatService final {
public:
    static constexpr char const* service_full_name() {
        return "im.ChatService";
    }
    
    // ========== 同步 Service 基类 ==========
    class Service : public grpc::Service {
    public:
        Service();
        virtual ~Service();
        
        virtual grpc::Status Chat(
            grpc::ServerContext* context,
            grpc::ServerReaderWriter<ChatMessage, ChatMessage>* stream
        );
    };
    
    // ========== 【关键】异步 Service 基类 ==========
    class AsyncService : public grpc::Service {
    public:
        AsyncService();
        ~AsyncService();
        
        // 请求处理一个 Chat 调用
        // 当有新的 Chat 请求到达时，通过 cq 通知
        void RequestChat(
            grpc::ServerContext* context,
            grpc::ServerAsyncReaderWriter<ChatMessage, ChatMessage>* stream,
            //                            ↑ 写出          ↑ 读入
            grpc::CompletionQueue* new_call_cq,  // 用于新调用通知
            grpc::ServerCompletionQueue* notification_cq,  // 用于完成通知
            void* tag  // 完成时的标识
        );
    };
    
    // ========== Stub 类（完整版） ==========
    class Stub final : public StubInterface {
    public:
        explicit Stub(const std::shared_ptr<grpc::ChannelInterface>& channel);
        
        // ---------- 同步接口 ----------
        std::unique_ptr<grpc::ClientReaderWriter<ChatMessage, ChatMessage>> 
        Chat(grpc::ClientContext* context);
        
        // ---------- 异步接口 ----------
        std::unique_ptr<grpc::ClientAsyncReaderWriter<ChatMessage, ChatMessage>> 
        AsyncChat(
            grpc::ClientContext* context,
            grpc::CompletionQueue* cq,
            void* tag
        );
        
        // PrepareAsync 版本（延迟启动）
        std::unique_ptr<grpc::ClientAsyncReaderWriter<ChatMessage, ChatMessage>> 
        PrepareAsyncChat(
            grpc::ClientContext* context,
            grpc::CompletionQueue* cq
        );
        
    private:
        std::shared_ptr<grpc::ChannelInterface> channel_;
        const grpc::internal::RpcMethod rpcmethod_Chat_;
    };
    
    static std::unique_ptr<Stub> NewStub(
        const std::shared_ptr<grpc::Channel>& channel,
        const grpc::StubOptions& options = grpc::StubOptions()
    );
};
} // namespace demo
```
#### <2>核心流操作类（gRPC 库提供）
```cpp
namespace grpc {

// ========== 服务端使用 ==========

// 服务端写入流（Server Streaming 用）
template <class W>
class ServerWriter {
public:
    bool Write(const W& msg);                    // 发送一个响应
    bool Write(const W& msg, WriteOptions opts); // 带选项写入
};

// 服务端读取流（Client Streaming 用）
template <class R>
class ServerReader {
public:
    bool Read(R* msg);  // 读取一个请求，返回 false 表示流结束
};

// 服务端双向流（Bidirectional 用）
template <class W, class R>
class ServerReaderWriter {
public:
    bool Read(R* msg);         // 读取客户端请求
    bool Write(const W& msg);  // 发送响应给客户端
};


// ========== 客户端使用 ==========

// 客户端读取流（Server Streaming 用）
template <class R>
class ClientReader {
public:
    bool Read(R* msg);        // 读取服务端响应
    grpc::Status Finish();    // 结束并获取状态
};

// 客户端写入流（Client Streaming 用）
template <class W>
class ClientWriter {
public:
    bool Write(const W& msg);  // 发送请求
    bool WritesDone();         // 标记发送完成
    grpc::Status Finish();     // 等待服务端响应并获取状态
};

// 客户端双向流（Bidirectional 用）
template <class W, class R>
class ClientReaderWriter {
public:
    bool Write(const W& msg);  // 发送请求
    bool Read(R* msg);         // 读取响应
    bool WritesDone();         // 标记发送完成
    grpc::Status Finish();     // 获取最终状态
};

} // namespace grpc
```

### （3）gRPC 新增的关键类型（C++ 伪代码）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122105626627.png)

#### <1>grpc::Status
- 在原生Proto的Service中，是通过Context返回“执行状态”（成功/失败/错误码/错误信息），grpc框架直接将其抽取出来实现了grpc::Status专门存储“执行状态”。

##### 1)内部伪代码
```cpp
namespace grpc {

class Status {
public:
    // 构造
    Status();  // 默认 OK
    Status(StatusCode code, const std::string& error_message);
    Status(StatusCode code, const std::string& error_message, const std::string& error_details);
    
    // 状态判断
    bool ok() const;                      // 是否成功
    StatusCode error_code() const;        // 错误码
    std::string error_message() const;    // 错误信息
    std::string error_details() const;    // 详细信息
    
    // 预定义状态
    static const Status& OK;
    static const Status& CANCELLED;
};

// 状态码枚举
enum StatusCode {
    OK = 0,
    CANCELLED = 1,
    UNKNOWN = 2,
    INVALID_ARGUMENT = 3,
    DEADLINE_EXCEEDED = 4,
    NOT_FOUND = 5,
    ALREADY_EXISTS = 6,
    PERMISSION_DENIED = 7,
    RESOURCE_EXHAUSTED = 8,
    FAILED_PRECONDITION = 9,
    ABORTED = 10,
    OUT_OF_RANGE = 11,
    UNIMPLEMENTED = 12,
    INTERNAL = 13,
    UNAVAILABLE = 14,
    DATA_LOSS = 15,
    UNAUTHENTICATED = 16,
};

} // namespace grpc
```

##### 2）使用示例
```cpp

// 作用：告诉调用者"这次 RPC 成功了吗？失败原因是什么？"

grpc::Status status = stub->Echo(&context, request, &response);

if (status.ok()) {
    // 成功，使用 response
} else {
    // 失败
    std::cout << "错误码: " << status.error_code() << std::endl;
    std::cout << "错误信息: " << status.error_message() << std::endl;
}
```

#### <2>grpc::ClientContext — 客户端调用上下文
- 在原生Proto的Service中，是通过Context在设置“超时、取消”，这部分功能现在也被抽取出来封装在grpc::ClinetContext中，方便客户端使用
- 相比与原生Proto的Context，**新增了“元数据、认证”等功能**
	- **元数据**：元数据是附加在 RPC 调用上的**键值对信息**，类似于 HTTP Headers，用于传递额外的键值对信息
	- **认证功能**：用于身份验证，如 Token、证书等

##### 1）内部伪代码
```cpp
namespace grpc {

class ClientContext {
public:
    ClientContext();
    ~ClientContext();
    
    // ========== 元数据 (Metadata) ==========
    // 发送给服务端的元数据（调用前设置）
    void AddMetadata(const std::string& key, const std::string& value);
    
    // 获取服务端返回的初始元数据
    const std::multimap<grpc::string_ref, grpc::string_ref>& GetServerInitialMetadata() const;
    
    // 获取服务端返回的尾部元数据
    const std::multimap<grpc::string_ref, grpc::string_ref>& GetServerTrailingMetadata() const;
    
    // ========== 超时控制 ==========
    void set_deadline(const std::chrono::system_clock::time_point& deadline);
    void set_deadline(const gpr_timespec& deadline);
    std::chrono::system_clock::time_point deadline() const;
    
    // ========== 取消调用 ==========
    void TryCancel();               // 尝试取消本次 RPC
    
    // ========== 认证 ==========
    void set_credentials(const std::shared_ptr<CallCredentials>& creds);
    std::shared_ptr<const AuthContext> auth_context() const;
    
    // ========== 压缩 ==========
    void set_compression_algorithm(grpc_compression_algorithm algorithm);
    
    // ========== 其他 ==========
    std::string peer() const;       // 获取对端地址
    void set_wait_for_ready(bool wait);  // 等待 channel ready
    
    // ⚠️ 不可复制，每次调用需要新建
    ClientContext(const ClientContext&) = delete;
    ClientContext& operator=(const ClientContext&) = delete;
};

} // namespace grpc
```

##### 2）使用示例
```cpp
// 作用：配置本次 RPC 调用的超时、元数据、认证等参数

// ⚠️ 每次 RPC 调用都必须新建 ClientContext，不可复用！
grpc::ClientContext context;

// ========== 设置超时 ==========
// 5秒后超时
context.set_deadline(std::chrono::system_clock::now() + std::chrono::seconds(5));

// ========== 添加元数据（类似 HTTP Header） ==========
context.AddMetadata("x-request-id", "uuid-12345");      // 请求追踪
context.AddMetadata("authorization", "Bearer my-token"); // 自定义认证

// ========== 设置认证凭证 ==========
auto call_creds = grpc::AccessTokenCredentials("jwt-token");
context.set_credentials(call_creds);

// ========== 发起调用 ==========
grpc::Status status = stub->Echo(&context, request, &response);

// ========== 获取服务端返回的元数据 ==========
if (status.ok()) {
    auto& metadata = context.GetServerTrailingMetadata();
    auto it = metadata.find("x-server-time");
    if (it != metadata.end()) {
        std::cout << "服务端时间: " << it->second << std::endl;
    }
}

// ========== 取消调用（通常在另一个线程） ==========
// context.TryCancel();
```

#### <3>grpc::ServerContext — 服务端处理上下文
- 与 ClientContext 对应，服务端用于获取客户端信息和设置响应元数据
- 可以检测客户端是否取消了请求、获取客户端设置的超时时间
- 可以获取客户端的认证信息和地址

##### 1)内部伪代码
```cpp
namespace grpc {

class ServerContext {
public:
    ServerContext();
    ~ServerContext();
    
    // ========== 元数据 (Metadata) ==========
    // 获取客户端发来的元数据
    const std::multimap<grpc::string_ref, grpc::string_ref>& client_metadata() const;
    
    // 发送初始元数据给客户端（在第一个响应之前）
    void AddInitialMetadata(const std::string& key, const std::string& value);
    
    // 发送尾部元数据给客户端（在 RPC 结束时）
    void AddTrailingMetadata(const std::string& key, const std::string& value);
    
    // ========== 取消/超时检测 ==========
    bool IsCancelled() const;       // 客户端是否取消了请求
    std::chrono::system_clock::time_point deadline() const;  // 客户端设置的 deadline
    
    // ========== 认证信息 ==========
    std::shared_ptr<const AuthContext> auth_context() const;
    std::string peer() const;       // 客户端地址（如 "ipv4:127.0.0.1:12345"）
    
    // ========== 压缩 ==========
    void set_compression_level(grpc_compression_level level);
    void set_compression_algorithm(grpc_compression_algorithm algorithm);
    
    // ========== 异步通知 ==========
    // 当 RPC 被取消时触发回调
    void AsyncNotifyWhenDone(void* tag);
};

} // namespace grpc
```

##### 2）使用示例
```cpp
// 作用：在服务端处理方法中，获取客户端信息、设置响应元数据

grpc::Status MyServiceImpl::Echo(
    grpc::ServerContext* context,    // 框架自动传入，无需手动创建
    const EchoRequest* request,
    EchoResponse* response) 
{
    // ========== 获取客户端元数据 ==========
    auto& metadata = context->client_metadata();
    auto it = metadata.find("x-request-id");
    if (it != metadata.end()) {
        std::string request_id(it->second.data(), it->second.size());
        std::cout << "请求ID: " << request_id << std::endl;
    }

    // ========== 获取客户端地址 ==========
    std::cout << "客户端地址: " << context->peer() << std::endl;  // 如 "ipv4:192.168.1.100:54321"

    // ========== 检测是否超时/取消 ==========
    if (context->IsCancelled()) {
        return grpc::Status(grpc::CANCELLED, "客户端已取消请求");
    }

    // ========== 长时间处理时定期检查 ==========
    for (int i = 0; i < 100; i++) {
        if (context->IsCancelled()) {
            return grpc::Status(grpc::CANCELLED, "处理中被取消");
        }
        // 处理逻辑...
    }

    // ========== 设置响应元数据 ==========
    context->AddInitialMetadata("x-server-version", "1.0.0");   // 初始元数据
    context->AddTrailingMetadata("x-process-time-ms", "42");    // 尾部元数据

    response->set_message(request->message());
    return grpc::Status::OK;
}
```


#### <4>grpc::Channel
- 原生 Proto 中需要自己实现 RpcChannel（纯虚类），gRPC 直接提供了**内置 HTTP/2 实现**
- Channel 代表客户端到服务端的逻辑连接，**可被多个 Stub 共享**
- 内置**连接池、重连、负载均衡**等功能

##### 1）内部伪代码
```cpp
namespace grpc {

class Channel : public ChannelInterface {
public:
    // 获取连接状态
    grpc_connectivity_state GetState(bool try_to_connect);
    
    // 等待状态变化
    bool WaitForStateChange(grpc_connectivity_state last_observed,
                            std::chrono::system_clock::time_point deadline);
    
    // 等待连接就绪
    bool WaitForConnected(std::chrono::system_clock::time_point deadline);
};

// 连接状态枚举
enum grpc_connectivity_state {
    GRPC_CHANNEL_IDLE,              // 空闲
    GRPC_CHANNEL_CONNECTING,        // 连接中
    GRPC_CHANNEL_READY,             // 就绪
    GRPC_CHANNEL_TRANSIENT_FAILURE, // 暂时失败（会重试）
    GRPC_CHANNEL_SHUTDOWN,          // 已关闭
};

// ========== Channel 创建函数 ==========
std::shared_ptr<Channel> CreateChannel(
    const std::string& target,                      // 地址，如 "localhost:50051"
    const std::shared_ptr<ChannelCredentials>& creds
);

// 创建凭证
std::shared_ptr<ChannelCredentials> InsecureChannelCredentials();  // 不加密
std::shared_ptr<ChannelCredentials> SslCredentials(const SslCredentialsOptions& options);

} // namespace grpc
```

##### 2）使用示例
```cpp
// 作用：建立到服务端的连接，创建 Stub 时使用

// ========== 创建不加密的 Channel（开发测试用） ==========
auto channel = grpc::CreateChannel(
    "localhost:50051", 
    grpc::InsecureChannelCredentials()
);

// ========== 创建 TLS 加密的 Channel（生产环境） ==========
grpc::SslCredentialsOptions ssl_opts;
ssl_opts.pem_root_certs = ReadFile("ca.crt");  // CA 证书
auto secure_channel = grpc::CreateChannel(
    "server.example.com:443",
    grpc::SslCredentials(ssl_opts)
);

// ========== 使用 Channel 创建 Stub ==========
auto stub = MyService::NewStub(channel);

// ========== 检查连接状态 ==========
auto state = channel->GetState(true);  // true = 尝试连接
switch (state) {
    case GRPC_CHANNEL_IDLE:       std::cout << "空闲" << std::endl; break;
    case GRPC_CHANNEL_CONNECTING: std::cout << "连接中" << std::endl; break;
    case GRPC_CHANNEL_READY:      std::cout << "已就绪" << std::endl; break;
    case GRPC_CHANNEL_TRANSIENT_FAILURE: std::cout << "暂时失败" << std::endl; break;
    case GRPC_CHANNEL_SHUTDOWN:   std::cout << "已关闭" << std::endl; break;
}

// ========== 等待连接就绪（带超时） ==========
bool connected = channel->WaitForConnected(
    std::chrono::system_clock::now() + std::chrono::seconds(10)
);
if (!connected) {
    std::cerr << "连接超时！" << std::endl;
}

// ========== 多个 Stub 共享同一个 Channel（推荐） ==========
auto stub1 = MyService::NewStub(channel);
auto stub2 = MyService::NewStub(channel);  // 复用连接
```

#### <5>grpc::Server & grpc::ServerBuilder
- 原生 Proto 没有提供服务端框架，需要自己处理网络监听、线程管理等
- gRPC 通过 ServerBuilder（建造者模式） 简化服务器配置和启动、**服务注册**等
- Server 提供阻塞等待和优雅关闭功能

##### 1）内部伪代码
```cpp
namespace grpc {

// ========== ServerBuilder - 构建服务器 ==========
class ServerBuilder {
public:
    ServerBuilder();
    
    // 添加监听地址
    ServerBuilder& AddListeningPort(
        const std::string& addr,                    // 如 "0.0.0.0:50051"
        std::shared_ptr<ServerCredentials> creds,
        int* selected_port = nullptr                // 输出实际端口
    );
    
    // 注册服务实现
    ServerBuilder& RegisterService(Service* service);
    
    // 设置线程池
    ServerBuilder& SetSyncServerOption(SyncServerOption option, int value);
    
    // 设置最大消息大小
    ServerBuilder& SetMaxReceiveMessageSize(int max_size);
    ServerBuilder& SetMaxSendMessageSize(int max_size);
    
    // 构建并启动
    std::unique_ptr<Server> BuildAndStart();
};

// ========== Server - 服务器实例 ==========
class Server {
public:
    // 阻塞等待服务关闭
    void Wait();
    
    // 关闭服务
    void Shutdown();
    void Shutdown(const std::chrono::system_clock::time_point& deadline);
};

// 服务端凭证
std::shared_ptr<ServerCredentials> InsecureServerCredentials();
std::shared_ptr<ServerCredentials> SslServerCredentials(const SslServerCredentialsOptions& options);

} // namespace grpc
```
##### 2）使用示例
```cpp
// 作用：配置、启动、管理 gRPC 服务器

// ========== 1. 创建服务实现 ==========
class MyServiceImpl final : public MyService::Service {
    grpc::Status Echo(grpc::ServerContext* context,
                      const EchoRequest* request,
                      EchoResponse* response) override {
        response->set_message("Echo: " + request->message());
        return grpc::Status::OK;
    }
};

// ========== 2. 使用 ServerBuilder 构建服务器 ==========
void RunServer() {
    std::string server_address("0.0.0.0:50051");
    MyServiceImpl service;

    grpc::ServerBuilder builder;
    
    // 添加监听端口（不加密）
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    
    // 添加监听端口（TLS 加密）
    // grpc::SslServerCredentialsOptions ssl_opts;
    // ssl_opts.pem_key_cert_pairs.push_back({ReadFile("server.key"), ReadFile("server.crt")});
    // builder.AddListeningPort("0.0.0.0:443", grpc::SslServerCredentials(ssl_opts));
    
    // 注册服务
    builder.RegisterService(&service);
    
    // 设置参数
    builder.SetMaxReceiveMessageSize(10 * 1024 * 1024);  // 10MB
    builder.SetMaxSendMessageSize(10 * 1024 * 1024);
    
    // 构建并启动
    std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
    std::cout << "服务器启动: " << server_address << std::endl;

    // ========== 3. 阻塞等待 ==========
    server->Wait();  // 阻塞直到 Shutdown 被调用
}

// ========== 4. 优雅关闭（通常在信号处理中） ==========
std::unique_ptr<grpc::Server> g_server;

void SignalHandler(int signal) {
    if (g_server) {
        std::cout << "正在关闭服务器..." << std::endl;
        // 优雅关闭：等待现有请求完成
        g_server->Shutdown(std::chrono::system_clock::now() + std::chrono::seconds(5));
    }
}
```

#### <6>流操作类
- 原生 Proto 不支持流式传输，gRPC 新增了**四种流模式**的支持
- 服务端使用 ServerWriter/Reader/ReaderWriter
- 客户端使用 ClientWriter/Reader/ReaderWriter

##### 1）内部伪代码
```cpp
namespace grpc {

// ========== 服务端流类 ==========
template<class W>
class ServerWriter {                    // Server Streaming 用
    bool Write(const W& msg);
    bool Write(const W& msg, WriteOptions options);
};

template<class R>
class ServerReader {                    // Client Streaming 用
    bool Read(R* msg);
};

template<class W, class R>
class ServerReaderWriter {              // Bidirectional 用
    bool Read(R* msg);
    bool Write(const W& msg);
};

// ========== 客户端流类 ==========
template<class R>
class ClientReader {                    // Server Streaming 用
    bool Read(R* msg);
    Status Finish();
};

template<class W>
class ClientWriter {                    // Client Streaming 用
    bool Write(const W& msg);
    bool WritesDone();                  // 标记写入完成
    Status Finish();
};

template<class W, class R>
class ClientReaderWriter {              // Bidirectional 用
    bool Read(R* msg);
    bool Write(const W& msg);
    bool WritesDone();
    Status Finish();
};

// ========== 写入选项 ==========
class WriteOptions {
public:
    WriteOptions& set_buffer_hint();    // 提示可缓冲
    WriteOptions& set_last_message();   // 标记最后一条
    WriteOptions& clear_buffer_hint();
};

} // namespace grpc
```

##### 2）使用示例
**Server Streaming（服务端流）**
```cpp
// 场景：客户端发一个请求，服务端返回多个响应（如分页数据、实时推送）

// ===== 服务端实现 =====
grpc::Status ListItems(grpc::ServerContext* context,
                       const ListRequest* request,
                       grpc::ServerWriter<Item>* writer) {
    for (int i = 0; i < 100; i++) {
        if (context->IsCancelled()) break;  // 检查取消
        
        Item item;
        item.set_id(i);
        item.set_name("Item " + std::to_string(i));
        
        if (!writer->Write(item)) break;  // 写入失败则退出
    }
    return grpc::Status::OK;
}

// ===== 客户端调用 =====
grpc::ClientContext context;
ListRequest request;
auto reader = stub->ListItems(&context, request);

Item item;
while (reader->Read(&item)) {  // 循环读取
    std::cout << "收到: " << item.name() << std::endl;
}

grpc::Status status = reader->Finish();  // 获取最终状态
```

**Client Streaming（客户端流）**
```cpp
// 场景：客户端发多个请求，服务端返回一个响应（如上传文件、批量提交）

// ===== 服务端实现 =====
grpc::Status UploadFile(grpc::ServerContext* context,
                        grpc::ServerReader<FileChunk>* reader,
                        UploadResult* result) {
    FileChunk chunk;
    int total_bytes = 0;
    
    while (reader->Read(&chunk)) {  // 循环读取客户端发来的数据块
        total_bytes += chunk.data().size();
        // 处理数据块...
    }
    
    result->set_total_bytes(total_bytes);
    result->set_success(true);
    return grpc::Status::OK;
}

// ===== 客户端调用 =====
grpc::ClientContext context;
UploadResult result;
auto writer = stub->UploadFile(&context, &result);

for (int i = 0; i < 10; i++) {
    FileChunk chunk;
    chunk.set_data("chunk data " + std::to_string(i));
    
    if (!writer->Write(chunk)) break;  // 写入失败则退出
}

writer->WritesDone();  // ⚠️ 必须调用！告诉服务端发送完毕
grpc::Status status = writer->Finish();  // 等待服务端响应

std::cout << "上传完成，共 " << result.total_bytes() << " 字节" << std::endl;
```

**Bidirectional Streaming（双向流）**
```cpp
// 场景：客户端和服务端同时发送多个消息（如聊天、实时协作）

// ===== 服务端实现 =====
grpc::Status Chat(grpc::ServerContext* context,
                  grpc::ServerReaderWriter<ChatMessage, ChatMessage>* stream) {
    ChatMessage msg;
    while (stream->Read(&msg)) {  // 读取客户端消息
        std::cout << "收到: " << msg.content() << std::endl;
        
        // 回复
        ChatMessage reply;
        reply.set_content("服务端回复: " + msg.content());
        stream->Write(reply);
    }
    return grpc::Status::OK;
}

// ===== 客户端调用 =====
grpc::ClientContext context;
auto stream = stub->Chat(&context);

// 启动读取线程
std::thread reader_thread([&stream]() {
    ChatMessage msg;
    while (stream->Read(&msg)) {
        std::cout << "收到: " << msg.content() << std::endl;
    }
});

// 主线程发送
for (int i = 0; i < 5; i++) {
    ChatMessage msg;
    msg.set_content("消息 " + std::to_string(i));
    stream->Write(msg);
}

stream->WritesDone();  // 发送完毕
reader_thread.join();  // 等待读取完成

grpc::Status status = stream->Finish();
```

## 4.Unary（一元）
### （1）功能设计
```cpp
场景：用户登录验证

┌─────────────┐         LoginRequest          ┌─────────────┐
│   客户端     │ ──────────────────────────→   │   服务端     │
│             │      (username, password)     │             │
│             │                               │             │
│             │ ←──────────────────────────   │             │
└─────────────┘        LoginResponse          └─────────────┘
                  (success, token, message)

特点：一问一答，最简单的 RPC 模式
```
### （2）Proto定义
```cpp
// login.proto
syntax = "proto3";

package auth;

// 登录请求
message LoginRequest {
    string username = 1;
    string password = 2;
}

// 登录响应
message LoginResponse {
    bool success = 1;
    string token = 2;
    string message = 3;
}

// 登录服务
service AuthService {
    // Unary：无 stream 关键字
    rpc Login(LoginRequest) returns (LoginResponse);
}
```

### （3）生成的 C++ 关键伪代码
```cpp
// ==================== auth.grpc.pb.h ====================
namespace auth {

class AuthService final {
public:
    static constexpr char const* service_full_name() {
        return "auth.AuthService";
    }
    
    // ========== 同步 Service 基类 ==========
    class Service : public grpc::Service {
    public:
        Service();
        virtual ~Service();
        
        virtual grpc::Status Login(
            grpc::ServerContext* context,
            const LoginRequest* request,
            LoginResponse* response
        );
    };
    
    // ========== 异步 Service 基类 ==========
    class AsyncService : public grpc::Service {
    public:
        AsyncService();
        ~AsyncService();
        
        // 请求处理一个 Login 调用
        void RequestLogin(
            grpc::ServerContext* context,
            LoginRequest* request,                        // 请求会被填充到这里
            grpc::ServerAsyncResponseWriter<LoginResponse>* writer,  // 异步响应写入器
            grpc::CompletionQueue* new_call_cq,
            grpc::ServerCompletionQueue* notification_cq,
            void* tag
        );
    };
    
    // ========== Stub 类（完整版） ==========
    class Stub final : public StubInterface {
    public:
        explicit Stub(const std::shared_ptr<grpc::ChannelInterface>& channel);
        
        // ---------- 同步接口 ----------
        grpc::Status Login(
            grpc::ClientContext* context,
            const LoginRequest& request,
            LoginResponse* response
        );
        
        // ---------- 异步接口 ----------
        std::unique_ptr<grpc::ClientAsyncResponseReader<LoginResponse>> 
        AsyncLogin(
            grpc::ClientContext* context,
            const LoginRequest& request,
            grpc::CompletionQueue* cq
        );
        
        // PrepareAsync 版本（延迟启动，需手动调用 StartCall）
        std::unique_ptr<grpc::ClientAsyncResponseReader<LoginResponse>> 
        PrepareAsyncLogin(
            grpc::ClientContext* context,
            const LoginRequest& request,
            grpc::CompletionQueue* cq
        );
        
    private:
        std::shared_ptr<grpc::ChannelInterface> channel_;
        const grpc::internal::RpcMethod rpcmethod_Login_;
    };
    
    static std::unique_ptr<Stub> NewStub(
        const std::shared_ptr<grpc::Channel>& channel,
        const grpc::StubOptions& options = grpc::StubOptions()
    );
};

} // namespace auth
```
```cpp
// ==================== auth.grpc.pb.cc（实现伪代码） ====================
namespace auth {

// Stub 同步调用实现
grpc::Status AuthService::Stub::Login(
    grpc::ClientContext* context,
    const LoginRequest& request,
    LoginResponse* response) 
{
    // 1. 序列化请求
    grpc::ByteBuffer request_buf;
    SerializationTraits<LoginRequest>::Serialize(request, &request_buf);
    
    // 2. 通过 Channel 发送请求并等待响应
    grpc::ByteBuffer response_buf;
    grpc::Status status = channel_->SyncUnaryCall(
        "/auth.AuthService/Login",  // 方法路径
        context,
        request_buf,
        &response_buf
    );
    
    // 3. 反序列化响应
    if (status.ok()) {
        SerializationTraits<LoginResponse>::Deserialize(response_buf, response);
    }
    
    return status;
}

// Service 默认实现（返回未实现）
grpc::Status AuthService::Service::Login(
    grpc::ServerContext* context,
    const LoginRequest* request,
    LoginResponse* response) 
{
    return grpc::Status(grpc::UNIMPLEMENTED, "Method not implemented");
}

} // namespace auth
```

### （4）使用示例
**服务端实现**
```cpp
// ==================== server.cpp ====================
#include <grpcpp/grpcpp.h>
#include "auth.grpc.pb.h"

// 继承生成的 Service 基类
class AuthServiceImpl final : public auth::AuthService::Service {
public:
    grpc::Status Login(
        grpc::ServerContext* context,
        const auth::LoginRequest* request,
        auth::LoginResponse* response) override 
    {
        // 获取客户端信息
        std::cout << "客户端地址: " << context->peer() << std::endl;
        
        // 检查是否被取消
        if (context->IsCancelled()) {
            return grpc::Status(grpc::CANCELLED, "请求被取消");
        }
        
        // 业务逻辑：验证用户
        if (request->username() == "admin" && request->password() == "123456") {
            response->set_success(true);
            response->set_token("jwt-token-xxxxx");
            response->set_message("登录成功");
        } else {
            response->set_success(false);
            response->set_message("用户名或密码错误");
        }
        
        return grpc::Status::OK;
    }
};

int main() {
    std::string server_address("0.0.0.0:50051");
    AuthServiceImpl service;
    
    grpc::ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);
    
    std::unique_ptr<grpc::Server> server = builder.BuildAndStart();
    std::cout << "服务器启动: " << server_address << std::endl;
    
    server->Wait();
    return 0;
}
```

**客户端调用**
```cpp
// ==================== client.cpp ====================
#include <grpcpp/grpcpp.h>
#include "auth.grpc.pb.h"

int main() {
    // 1. 创建 Channel
    auto channel = grpc::CreateChannel(
        "localhost:50051",
        grpc::InsecureChannelCredentials()
    );
    
    // 2. 创建 Stub
    auto stub = auth::AuthService::NewStub(channel);
    
    // 3. 准备请求
    auth::LoginRequest request;
    request.set_username("admin");
    request.set_password("123456");
    
    // 4. 创建 Context（每次调用必须新建！）
    grpc::ClientContext context;
    context.set_deadline(
        std::chrono::system_clock::now() + std::chrono::seconds(5)
    );
    context.AddMetadata("client-version", "1.0.0");
    
    // 5. 调用 RPC
    auth::LoginResponse response;
    grpc::Status status = stub->Login(&context, request, &response);
    
    // 6. 处理结果
    if (status.ok()) {
        if (response.success()) {
            std::cout << "✓ 登录成功！" << std::endl;
            std::cout << "  Token: " << response.token() << std::endl;
        } else {
            std::cout << "✗ 登录失败: " << response.message() << std::endl;
        }
    } else {
        std::cout << "✗ RPC 错误: " << status.error_message() << std::endl;
        std::cout << "  错误码: " << status.error_code() << std::endl;
    }
    
    return 0;
}
```

### （5）数据流图
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                        Unary 调用流程                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   客户端                                              服务端             │
│                                                                         │
│   LoginRequest req;                                                     │
│   req.set_username("admin");                                           │
│   req.set_password("123456");                                          │
│                                                                         │
│   stub->Login(&ctx, req, &resp);                                       │
│         │                                                               │
│         │  ┌──────────────────┐                                        │
│         └─→│  HTTP/2 POST     │                                        │
│            │  /auth.AuthService/Login                                   │
│            │  Body: [序列化的 LoginRequest]                             │
│            └────────────────────────────────────────→ 收到请求          │
│                                                      │                  │
│                                                      ▼                  │
│                                              Login(ctx, req, resp)     │
│                                              验证用户名密码              │
│                                              填充 resp                  │
│                                              return Status::OK         │
│                                                      │                  │
│            ┌────────────────────────────────────────←┘                  │
│            │  HTTP/2 Response                                          │
│            │  Body: [序列化的 LoginResponse]                            │
│            │  Trailers: grpc-status=0                                   │
│         ←──┘                                                            │
│                                                                         │
│   status.ok() == true                                                   │
│   resp.success() == true                                               │
│   resp.token() == "jwt-token-xxxxx"                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### （6）Unary模式总结
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122153138053.png)

## 5.Server Streaming（服务端流）
### （1）功能设计
```cpp
场景：查询订单列表（分批返回）

┌─────────────┐       OrderQuery              ┌─────────────┐
│   客户端     │ ──────────────────────────→   │   服务端     │
│             │      (user_id, status)        │             │
│             │                               │             │
│             │ ←─────── Order 1 ─────────    │             │
│             │ ←─────── Order 2 ─────────    │             │
│             │ ←─────── Order 3 ─────────    │             │
│             │ ←─────── ...     ─────────    │             │
└─────────────┘                               └─────────────┘

特点：一个请求，多个响应（服务端推送）
```

### （2）Proto定义
```cpp
// order.proto
syntax = "proto3";

package shop;

message OrderQuery {
    string user_id = 1;
    string status = 2;  // "pending", "completed", "all"
}

message Order {
    string order_id = 1;
    string product_name = 2;
    double price = 3;
    string status = 4;
}

service OrderService {
    // Server Streaming：返回值加 stream
    rpc ListOrders(OrderQuery) returns (stream Order);
}
```

### （3）生成的 C++ 关键伪代码
```cpp
// ==================== order.grpc.pb.h ====================
namespace shop {

class OrderService final {
public:
    static constexpr char const* service_full_name() {
        return "shop.OrderService";
    }
    
    // ========== 同步 Service 基类 ==========
    class Service : public grpc::Service {
    public:
        Service();
        virtual ~Service();
        
        virtual grpc::Status ListOrders(
            grpc::ServerContext* context,
            const OrderQuery* request,
            grpc::ServerWriter<Order>* writer
        );
    };
    
    // ========== 异步 Service 基类 ==========
    class AsyncService : public grpc::Service {
    public:
        AsyncService();
        ~AsyncService();
        
        // 请求处理一个 ListOrders 调用
        void RequestListOrders(
            grpc::ServerContext* context,
            OrderQuery* request,                    // 请求会被填充到这里
            grpc::ServerAsyncWriter<Order>* writer, // 异步写入器
            grpc::CompletionQueue* new_call_cq,
            grpc::ServerCompletionQueue* notification_cq,
            void* tag
        );
    };
    
    // ========== Stub 类（完整版） ==========
    class Stub final : public StubInterface {
    public:
        explicit Stub(const std::shared_ptr<grpc::ChannelInterface>& channel);
        
        // ---------- 同步接口 ----------
        std::unique_ptr<grpc::ClientReader<Order>> ListOrders(
            grpc::ClientContext* context,
            const OrderQuery& request
        );
        
        // ---------- 异步接口 ----------
        std::unique_ptr<grpc::ClientAsyncReader<Order>> AsyncListOrders(
            grpc::ClientContext* context,
            const OrderQuery& request,
            grpc::CompletionQueue* cq,
            void* tag
        );
        
        // PrepareAsync 版本（延迟启动，需手动调用 StartCall）
        std::unique_ptr<grpc::ClientAsyncReader<Order>> PrepareAsyncListOrders(
            grpc::ClientContext* context,
            const OrderQuery& request,
            grpc::CompletionQueue* cq
        );
        
    private:
        std::shared_ptr<grpc::ChannelInterface> channel_;
        const grpc::internal::RpcMethod rpcmethod_ListOrders_;
    };
    
    static std::unique_ptr<Stub> NewStub(
        const std::shared_ptr<grpc::Channel>& channel,
        const grpc::StubOptions& options = grpc::StubOptions()
    );
};

} // namespace shop
```
```cpp
// ==================== order.grpc.pb.cc（实现伪代码） ====================
namespace shop {

// Stub 实现
std::unique_ptr<grpc::ClientReader<Order>> OrderService::Stub::ListOrders(
    grpc::ClientContext* context,
    const OrderQuery& request) 
{
    // 1. 创建调用
    auto call = channel_->CreateCall("/shop.OrderService/ListOrders", context);
    
    // 2. 发送请求（只发一次）
    grpc::ByteBuffer request_buf;
    SerializationTraits<OrderQuery>::Serialize(request, &request_buf);
    call.Write(request_buf);
    call.WritesDone();  // 客户端发送完成
    
    // 3. 返回 Reader，让调用者循环读取
    return std::make_unique<grpc::ClientReader<Order>>(std::move(call), context);
}

// Service 默认实现
grpc::Status OrderService::Service::ListOrders(
    grpc::ServerContext* context,
    const OrderQuery* request,
    grpc::ServerWriter<Order>* writer) 
{
    return grpc::Status(grpc::UNIMPLEMENTED, "Method not implemented");
}

} // namespace shop
```

### （4）使用示例
**服务端实现**
```cpp
// ==================== server.cpp ====================
#include <grpcpp/grpcpp.h>
#include "order.grpc.pb.h"

class OrderServiceImpl final : public shop::OrderService::Service {
public:
    grpc::Status ListOrders(
        grpc::ServerContext* context,
        const shop::OrderQuery* request,
        grpc::ServerWriter<shop::Order>* writer) override 
    {
        std::cout << "查询用户订单: " << request->user_id() << std::endl;
        
        // 模拟从数据库查询订单
        std::vector<std::pair<std::string, double>> orders = {
            {"iPhone 15", 7999.0},
            {"MacBook Pro", 14999.0},
            {"AirPods Pro", 1899.0},
        };
        
        for (size_t i = 0; i < orders.size(); i++) {
            // 检查客户端是否取消
            if (context->IsCancelled()) {
                return grpc::Status(grpc::CANCELLED, "客户端取消请求");
            }
            
            // 构造响应
            shop::Order order;
            order.set_order_id("ORD-" + std::to_string(i + 1));
            order.set_product_name(orders[i].first);
            order.set_price(orders[i].second);
            order.set_status("completed");
            
            // 写入流
            if (!writer->Write(order)) {
                // 写入失败（可能客户端断开）
                return grpc::Status(grpc::UNKNOWN, "写入失败");
            }
            
            // 模拟延迟
            std::this_thread::sleep_for(std::chrono::milliseconds(500));
        }
        
        return grpc::Status::OK;
    }
};

int main() {
    std::string server_address("0.0.0.0:50051");
    OrderServiceImpl service;
    
    grpc::ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);
    
    auto server = builder.BuildAndStart();
    std::cout << "服务器启动: " << server_address << std::endl;
    server->Wait();
    return 0;
}
```
**客户端调用**
```cpp
// ==================== client.cpp ====================
#include <grpcpp/grpcpp.h>
#include "order.grpc.pb.h"

int main() {
    auto channel = grpc::CreateChannel("localhost:50051", 
                                       grpc::InsecureChannelCredentials());
    auto stub = shop::OrderService::NewStub(channel);
    
    // 准备请求
    shop::OrderQuery query;
    query.set_user_id("user-123");
    query.set_status("all");
    
    // 创建 Context
    grpc::ClientContext context;
    context.set_deadline(
        std::chrono::system_clock::now() + std::chrono::seconds(30)
    );
    
    // 调用 RPC，获取 Reader
    auto reader = stub->ListOrders(&context, query);
    
    // 循环读取响应
    shop::Order order;
    int count = 0;
    while (reader->Read(&order)) {
        std::cout << "订单 " << ++count << ": " << std::endl;
        std::cout << "  ID: " << order.order_id() << std::endl;
        std::cout << "  商品: " << order.product_name() << std::endl;
        std::cout << "  价格: ¥" << order.price() << std::endl;
        std::cout << std::endl;
    }
    
    // 获取最终状态
    grpc::Status status = reader->Finish();
    
    if (status.ok()) {
        std::cout << "✓ 共收到 " << count << " 个订单" << std::endl;
    } else {
        std::cout << "✗ 错误: " << status.error_message() << std::endl;
    }
    
    return 0;
}
```

### （5）数据流图
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                    Server Streaming 调用流程                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   客户端                                              服务端             │
│                                                                         │
│   auto reader = stub->ListOrders(&ctx, query);                         │
│         │                                                               │
│         │  ┌──────────────────┐                                        │
│         └─→│  POST /shop.OrderService/ListOrders                       │
│            │  Body: [OrderQuery]                                       │
│            └────────────────────────────────────────→ 收到请求          │
│                                                      │                  │
│                                                      ▼                  │
│                                             for each order:            │
│                                               writer->Write(order)     │
│            ┌────────────────────────────────────────←┐                  │
│            │  DATA frame: [Order 1]                  │                  │
│   reader->Read(&order) ←─┘                           │                  │
│   处理 Order 1                                       │                  │
│                                                      │                  │
│            ┌────────────────────────────────────────←┤                  │
│            │  DATA frame: [Order 2]                  │                  │
│   reader->Read(&order) ←─┘                           │                  │
│   处理 Order 2                                       │                  │
│                                                      │                  │
│            ┌────────────────────────────────────────←┤                  │
│            │  DATA frame: [Order 3]                  │                  │
│   reader->Read(&order) ←─┘                           │                  │
│   处理 Order 3                                       │                  │
│                                                      │                  │
│                                             return Status::OK          │
│            ┌────────────────────────────────────────←┘                  │
│            │  HEADERS: grpc-status=0 (Trailers)                        │
│   reader->Read() 返回 false ←─┘                                        │
│   status = reader->Finish()                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### （6）Server Streaming 模式总结
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122153119547.png)

## 6.Client Streaming（客户端流）
### （1）功能设计
```cpp
场景：上传文件（分块传输）

┌─────────────┐       FileChunk 1             ┌─────────────┐
│   客户端     │ ──────────────────────────→   │   服务端     │
│             │       FileChunk 2             │             │
│             │ ──────────────────────────→   │             │
│             │       FileChunk 3             │             │
│             │ ──────────────────────────→   │             │
│             │       ...                     │             │
│             │                               │             │
│             │ ←──────────────────────────   │             │
└─────────────┘      UploadResult             └─────────────┘

特点：多个请求，一个响应（批量提交）
```

### （2）Proto 定义
```cpp
// file.proto
syntax = "proto3";

package storage;

message FileChunk {
    string filename = 1;
    bytes data = 2;
    int32 chunk_index = 3;
    bool is_last = 4;
}

message UploadResult {
    bool success = 1;
    string file_id = 2;
    int64 total_bytes = 3;
    string message = 4;
}

service FileService {
    // Client Streaming：请求加 stream
    rpc UploadFile(stream FileChunk) returns (UploadResult);
}
```

### （3）生成的 C++ 关键伪代码
```cpp
// ==================== file.grpc.pb.h ====================
namespace storage {

class FileService final {
public:
    static constexpr char const* service_full_name() {
        return "storage.FileService";
    }
    
    // ========== 同步 Service 基类 ==========
    class Service : public grpc::Service {
    public:
        Service();
        virtual ~Service();
        
        virtual grpc::Status UploadFile(
            grpc::ServerContext* context,
            grpc::ServerReader<FileChunk>* reader,
            UploadResult* response
        );
    };
    
    // ========== 异步 Service 基类 ==========
    class AsyncService : public grpc::Service {
    public:
        AsyncService();
        ~AsyncService();
        
        // 请求处理一个 UploadFile 调用
        void RequestUploadFile(
            grpc::ServerContext* context,
            grpc::ServerAsyncReader<UploadResult, FileChunk>* reader,
            //                      ↑ 响应类型      ↑ 请求类型
            grpc::CompletionQueue* new_call_cq,
            grpc::ServerCompletionQueue* notification_cq,
            void* tag
        );
    };
    
    // ========== Stub 类（完整版） ==========
    class Stub final : public StubInterface {
    public:
        explicit Stub(const std::shared_ptr<grpc::ChannelInterface>& channel);
        
        // ---------- 同步接口 ----------
        std::unique_ptr<grpc::ClientWriter<FileChunk>> UploadFile(
            grpc::ClientContext* context,
            UploadResult* response
        );
        
        // ---------- 异步接口 ----------
        std::unique_ptr<grpc::ClientAsyncWriter<FileChunk>> AsyncUploadFile(
            grpc::ClientContext* context,
            UploadResult* response,
            grpc::CompletionQueue* cq,
            void* tag
        );
        
        // PrepareAsync 版本（延迟启动，需手动调用 StartCall）
        std::unique_ptr<grpc::ClientAsyncWriter<FileChunk>> PrepareAsyncUploadFile(
            grpc::ClientContext* context,
            UploadResult* response,
            grpc::CompletionQueue* cq
        );
        
    private:
        std::shared_ptr<grpc::ChannelInterface> channel_;
        const grpc::internal::RpcMethod rpcmethod_UploadFile_;
    };
    
    static std::unique_ptr<Stub> NewStub(
        const std::shared_ptr<grpc::Channel>& channel,
        const grpc::StubOptions& options = grpc::StubOptions()
    );
};

} // namespace storage
```
```cpp
// ==================== file.grpc.pb.cc（实现伪代码） ====================
namespace storage {

// Stub 实现
std::unique_ptr<grpc::ClientWriter<FileChunk>> FileService::Stub::UploadFile(
    grpc::ClientContext* context,
    UploadResult* response) 
{
    // 1. 创建调用
    auto call = channel_->CreateCall("/storage.FileService/UploadFile", context);
    
    // 2. 返回 Writer，让调用者循环写入
    // 响应指针保存，在 Finish() 时填充
    return std::make_unique<grpc::ClientWriter<FileChunk>>(
        std::move(call), context, response
    );
}

// Service 默认实现
grpc::Status FileService::Service::UploadFile(
    grpc::ServerContext* context,
    grpc::ServerReader<FileChunk>* reader,
    UploadResult* response) 
{
    return grpc::Status(grpc::UNIMPLEMENTED, "Method not implemented");
}

} // namespace storage
```

### （4）使用示例
**服务端实现**
```cpp
// ==================== server.cpp ====================
#include <grpcpp/grpcpp.h>
#include "file.grpc.pb.h"
#include <fstream>

class FileServiceImpl final : public storage::FileService::Service {
public:
    grpc::Status UploadFile(
        grpc::ServerContext* context,
        grpc::ServerReader<storage::FileChunk>* reader,
        storage::UploadResult* response) override 
    {
        storage::FileChunk chunk;
        std::string filename;
        std::ofstream file;
        int64_t total_bytes = 0;
        
        // 循环读取客户端发来的数据块
        while (reader->Read(&chunk)) {
            // 检查取消
            if (context->IsCancelled()) {
                return grpc::Status(grpc::CANCELLED, "上传被取消");
            }
            
            // 第一个块时创建文件
            if (filename.empty()) {
                filename = "/tmp/" + chunk.filename();
                file.open(filename, std::ios::binary);
                std::cout << "开始接收文件: " << filename << std::endl;
            }
            
            // 写入数据
            file.write(chunk.data().data(), chunk.data().size());
            total_bytes += chunk.data().size();
            
            std::cout << "  收到块 " << chunk.chunk_index() 
                      << ", " << chunk.data().size() << " 字节" << std::endl;
        }
        
        file.close();
        
        // 填充响应
        response->set_success(true);
        response->set_file_id("FILE-" + std::to_string(time(nullptr)));
        response->set_total_bytes(total_bytes);
        response->set_message("上传完成: " + filename);
        
        return grpc::Status::OK;
    }
};

int main() {
    std::string server_address("0.0.0.0:50051");
    FileServiceImpl service;
    
    grpc::ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);
    builder.SetMaxReceiveMessageSize(100 * 1024 * 1024);  // 100MB
    
    auto server = builder.BuildAndStart();
    std::cout << "服务器启动: " << server_address << std::endl;
    server->Wait();
    return 0;
}
```
**客户端调用**
```cpp
// ==================== client.cpp ====================
#include <grpcpp/grpcpp.h>
#include "file.grpc.pb.h"
#include <fstream>

int main() {
    auto channel = grpc::CreateChannel("localhost:50051", 
                                       grpc::InsecureChannelCredentials());
    auto stub = storage::FileService::NewStub(channel);
    
    // 创建 Context
    grpc::ClientContext context;
    context.set_deadline(
        std::chrono::system_clock::now() + std::chrono::minutes(5)
    );
    
    // 准备响应对象（Finish 后填充）
    storage::UploadResult result;
    
    // 获取 Writer
    auto writer = stub->UploadFile(&context, &result);
    
    // 读取本地文件并分块上传
    std::ifstream file("test.txt", std::ios::binary);
    const size_t CHUNK_SIZE = 64 * 1024;  // 64KB 每块
    char buffer[CHUNK_SIZE];
    int chunk_index = 0;
    
    while (file.read(buffer, CHUNK_SIZE) || file.gcount() > 0) {
        storage::FileChunk chunk;
        chunk.set_filename("test.txt");
        chunk.set_data(buffer, file.gcount());
        chunk.set_chunk_index(chunk_index++);
        chunk.set_is_last(file.eof());
        
        if (!writer->Write(chunk)) {
            std::cerr << "写入失败！" << std::endl;
            break;
        }
        
        std::cout << "发送块 " << chunk_index << std::endl;
    }
    file.close();
    
    // ⚠️ 必须调用！告诉服务端发送完毕
    writer->WritesDone();
    
    // 等待响应并获取状态
    grpc::Status status = writer->Finish();
    
    if (status.ok()) {
        std::cout << "✓ 上传成功！" << std::endl;
        std::cout << "  文件ID: " << result.file_id() << std::endl;
        std::cout << "  总大小: " << result.total_bytes() << " 字节" << std::endl;
    } else {
        std::cout << "✗ 上传失败: " << status.error_message() << std::endl;
    }
    
    return 0;
}
```

### （5）数据流图
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                    Client Streaming 调用流程                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   客户端                                              服务端             │
│                                                                         │
│   auto writer = stub->UploadFile(&ctx, &result);                       │
│         │                                                               │
│         │  ┌──────────────────┐                                        │
│         └─→│  POST /storage.FileService/UploadFile                     │
│            └────────────────────────────────────────→ 开始接收          │
│                                                      │                  │
│   writer->Write(chunk1);                             │                  │
│         │  ┌──────────────────┐                      │                  │
│         └─→│  DATA: [FileChunk 1]                    │                  │
│            └───────────────────────────────────────→ reader->Read()    │
│                                                      保存数据            │
│   writer->Write(chunk2);                             │                  │
│         │  ┌──────────────────┐                      │                  │
│         └─→│  DATA: [FileChunk 2]                    │                  │
│            └───────────────────────────────────────→ reader->Read()    │
│                                                      保存数据            │
│   writer->Write(chunk3);                             │                  │
│         │  ┌──────────────────┐                      │                  │
│         └─→│  DATA: [FileChunk 3]                    │                  │
│            └───────────────────────────────────────→ reader->Read()    │
│                                                      保存数据            │
│   writer->WritesDone();                              │                  │
│         │  ┌──────────────────┐                      │                  │
│         └─→│  END_STREAM flag                        │                  │
│            └───────────────────────────────────────→ Read() 返回 false │
│                                                      填充 response      │
│                                                      return Status::OK │
│            ┌────────────────────────────────────────←┘                  │
│            │  DATA: [UploadResult]                                     │
│            │  HEADERS: grpc-status=0                                   │
│   writer->Finish() ←─┘                                                 │
│   result 已填充                                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### （6）Client Streaming 模式总结
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122153548149.png)

## 7.Bidirectional Streaming（双向流）
### （1）功能设计
```cpp
场景：实时聊天室

┌─────────────┐       ChatMessage             ┌─────────────┐
│   客户端     │ ←──────────────────────────→  │   服务端     │
│             │       ChatMessage             │             │
│             │ ←──────────────────────────→  │             │
│             │       ChatMessage             │             │
│             │ ←──────────────────────────→  │             │
│             │       ...                     │             │
└─────────────┘   （双向独立，可同时读写）      └─────────────┘

特点：多个请求，多个响应，全双工通信
```

### （2）Proto 定义
```cpp
// chat.proto
syntax = "proto3";

package im;

message ChatMessage {
    string sender = 1;
    string content = 2;
    int64 timestamp = 3;
}

service ChatService {
    // Bidirectional Streaming：请求和响应都加 stream
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

### （3）生成的 C++ 关键伪代码
```cpp
// ==================== chat.grpc.pb.h ====================
namespace im {

class ChatService final {
public:
    
    // ========== Service 基类（服务端继承） ==========
    class Service : public grpc::Service {
    public:
        // 服务端用 ServerReaderWriter 同时读写
        virtual grpc::Status Chat(
            grpc::ServerContext* context,
            grpc::ServerReaderWriter<ChatMessage, ChatMessage>* stream
            //                       ↑ 写出类型    ↑ 读入类型
        );
    };
    
    // ========== Stub 类（客户端使用） ==========
    class Stub final : public StubInterface {
    public:
        explicit Stub(const std::shared_ptr<grpc::ChannelInterface>& channel);
        
        // 返回 ClientReaderWriter，可同时读写
        std::unique_ptr<grpc::ClientReaderWriter<ChatMessage, ChatMessage>> Chat(
            grpc::ClientContext* context
            //                    ↑ 写出类型       ↑ 读入类型
        );
        
        // 异步版本
        std::unique_ptr<grpc::ClientAsyncReaderWriter<ChatMessage, ChatMessage>> 
        AsyncChat(
            grpc::ClientContext* context,
            grpc::CompletionQueue* cq,
            void* tag
        );
        
    private:
        std::shared_ptr<grpc::ChannelInterface> channel_;
    };
    
    static std::unique_ptr<Stub> NewStub(
        const std::shared_ptr<grpc::Channel>& channel
    );
};

} // namespace im
```
```cpp
// ==================== chat.grpc.pb.cc（实现伪代码） ====================
namespace im {

// Stub 实现
std::unique_ptr<grpc::ClientReaderWriter<ChatMessage, ChatMessage>> 
ChatService::Stub::Chat(grpc::ClientContext* context) 
{
    // 创建调用并返回双向流对象
    auto call = channel_->CreateCall("/im.ChatService/Chat", context);
    return std::make_unique<grpc::ClientReaderWriter<ChatMessage, ChatMessage>>(
        std::move(call), context
    );
}

// Service 默认实现
grpc::Status ChatService::Service::Chat(
    grpc::ServerContext* context,
    grpc::ServerReaderWriter<ChatMessage, ChatMessage>* stream) 
{
    return grpc::Status(grpc::UNIMPLEMENTED, "Method not implemented");
}

} // namespace im
```

### （4）使用示例
**服务端实现**
```cpp
// ==================== server.cpp ====================
#include <grpcpp/grpcpp.h>
#include "chat.grpc.pb.h"
#include <thread>
#include <mutex>
#include <vector>

class ChatServiceImpl final : public im::ChatService::Service {
private:
    // 存储所有活跃的流（简化版，实际需要更复杂的管理）
    std::mutex streams_mutex_;
    std::vector<grpc::ServerReaderWriter<im::ChatMessage, im::ChatMessage>*> streams_;

public:
    grpc::Status Chat(
        grpc::ServerContext* context,
        grpc::ServerReaderWriter<im::ChatMessage, im::ChatMessage>* stream) override 
    {
        // 注册这个流
        {
            std::lock_guard<std::mutex> lock(streams_mutex_);
            streams_.push_back(stream);
        }
        
        std::cout << "新用户连接: " << context->peer() << std::endl;
        
        im::ChatMessage msg;
        
        // 循环读取客户端消息
        while (stream->Read(&msg)) {
            if (context->IsCancelled()) break;
            
            std::cout << "[" << msg.sender() << "]: " << msg.content() << std::endl;
            
            // 广播给所有客户端（包括发送者）
            im::ChatMessage broadcast;
            broadcast.set_sender(msg.sender());
            broadcast.set_content(msg.content());
            broadcast.set_timestamp(time(nullptr));
            
            std::lock_guard<std::mutex> lock(streams_mutex_);
            for (auto* s : streams_) {
                s->Write(broadcast);
            }
        }
        
        // 移除这个流
        {
            std::lock_guard<std::mutex> lock(streams_mutex_);
            streams_.erase(
                std::remove(streams_.begin(), streams_.end(), stream),
                streams_.end()
            );
        }
        
        std::cout << "用户断开: " << context->peer() << std::endl;
        return grpc::Status::OK;
    }
};

int main() {
    std::string server_address("0.0.0.0:50051");
    ChatServiceImpl service;
    
    grpc::ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);
    
    auto server = builder.BuildAndStart();
    std::cout << "聊天服务器启动: " << server_address << std::endl;
    server->Wait();
    return 0;
}
```
**客户端调用**
```cpp
// ==================== client.cpp ====================
#include <grpcpp/grpcpp.h>
#include "chat.grpc.pb.h"
#include <thread>
#include <atomic>
#include <iostream>

int main() {
    auto channel = grpc::CreateChannel("localhost:50051", 
                                       grpc::InsecureChannelCredentials());
    auto stub = im::ChatService::NewStub(channel);
    
    // 创建 Context
    grpc::ClientContext context;
    
    // 获取双向流
    auto stream = stub->Chat(&context);
    
    std::atomic<bool> running{true};
    
    // ===== 读取线程：接收服务端消息 =====
    std::thread reader_thread([&stream, &running]() {
        im::ChatMessage msg;
        while (stream->Read(&msg)) {
            std::cout << "\r[" << msg.sender() << "]: " 
                      << msg.content() << std::endl;
            std::cout << "> " << std::flush;  // 重新显示输入提示
        }
        running = false;
        std::cout << "连接已断开" << std::endl;
    });
    
    // ===== 主线程：发送用户输入 =====
    std::string username;
    std::cout << "请输入用户名: ";
    std::getline(std::cin, username);
    
    std::cout << "开始聊天（输入 'quit' 退出）" << std::endl;
    std::cout << "> ";
    
    std::string line;
    while (running && std::getline(std::cin, line)) {
        if (line == "quit") break;
        if (line.empty()) {
            std::cout << "> ";
            continue;
        }
        
        im::ChatMessage msg;
        msg.set_sender(username);
        msg.set_content(line);
        msg.set_timestamp(time(nullptr));
        
        if (!stream->Write(msg)) {
            std::cerr << "发送失败！" << std::endl;
            break;
        }
        
        std::cout << "> ";
    }
    
    // 标记写入完成
    stream->WritesDone();
    
    // 等待读取线程结束
    reader_thread.join();
    
    // 获取最终状态
    grpc::Status status = stream->Finish();
    
    if (status.ok()) {
        std::cout << "正常退出" << std::endl;
    } else {
        std::cout << "错误: " << status.error_message() << std::endl;
    }
    
    return 0;
}
```

### （5）数据流图
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                  Bidirectional Streaming 调用流程                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   客户端                                              服务端             │
│                                                                         │
│   auto stream = stub->Chat(&ctx);                                      │
│         │                                                               │
│         │  ┌──────────────────┐                                        │
│         └─→│  POST /im.ChatService/Chat                                │
│            └────────────────────────────────────────→ 连接建立          │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │         双向独立通信（全双工）                                     │   │
│   │                                                                  │   │
│   │   stream->Write(msg1);                                          │   │
│   │         ─────────── [ChatMessage] ───────────→ stream->Read()   │   │
│   │                                                                  │   │
│   │                                                 stream->Write()  │   │
│   │   stream->Read() ←─────────── [ChatMessage] ───────────          │   │
│   │                                                                  │   │
│   │   stream->Write(msg2);                                          │   │
│   │         ─────────── [ChatMessage] ───────────→ stream->Read()   │   │
│   │                                                                  │   │
│   │                                                 stream->Write()  │   │
│   │   stream->Read() ←─────────── [ChatMessage] ───────────          │   │
│   │                                                                  │   │
│   │   （读和写可以同时进行，互不阻塞）                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│   stream->WritesDone();                                                 │
│         ─────────── END_STREAM ───────────→ Read() 返回 false          │
│                                            return Status::OK           │
│         ←─────────── grpc-status=0 ───────────                         │
│   stream->Finish();                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### （6）Bidirectional Streaming 模式总结
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260122153842158.png)


## 8.gRPC的异步调用
### （0）概述
gRPC C++的异步调用主要涉及：
- CompletionQueue - 核心事件队列

- ClientAsyncResponseReader - 异步Unary客户端
- ClientAsyncReader - 异步Server Streaming客户端
- ClientAsyncWriter - 异步Client Streaming客户端
- ClientAsyncReaderWriter - 异步双向流客户端

- ServerAsyncResponseWriter - 异步Unary服务端
- ServerAsyncReader - 异步Client Streaming服务端
- ServerAsyncWriter - 异步Server Streaming服务端
- ServerAsyncReaderWriter - 异步双向流服务端

- Alarm - 定时


**异步 vs 同步**
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                        同步 vs 异步                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   同步调用（阻塞）                                                        │
│   ┌─────────┐                                                           │
│   │ Thread  │──Call()──→ [等待...等待...等待...] ──→ 返回结果           │
│   └─────────┘            （线程被阻塞）                                   │
│                                                                         │
│   异步调用（非阻塞）                                                      │
│   ┌─────────┐                                                           │
│   │ Thread  │──AsyncCall()──→ 立即返回，继续做其他事                     │
│   └─────────┘                    │                                      │
│        │                         ▼                                      │
│        │              ┌──────────────────┐                              │
│        └──轮询────────→│ CompletionQueue │←── 结果就绪时通知             │
│                       └──────────────────┘                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**核心类总览**
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                     gRPC 异步核心类                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    CompletionQueue                               │   │
│   │                   （核心事件队列）                                 │   │
│   │                                                                  │   │
│   │   所有异步操作完成后，都会向 CQ 投递事件                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                          ▲                                              │
│          ┌───────────────┼───────────────┐                              │
│          │               │               │                              │
│   ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐                       │
│   │   客户端     │ │   服务端     │ │   工具类    │                       │
│   ├─────────────┤ ├─────────────┤ ├─────────────┤                       │
│   │AsyncResp..  │ │AsyncResp..  │ │   Alarm     │                       │
│   │AsyncReader  │ │AsyncReader  │ │  (定时器)   │                       │
│   │AsyncWriter  │ │AsyncWriter  │ │             │                       │
│   │AsyncRW      │ │AsyncRW      │ │             │                       │
│   └─────────────┘ └─────────────┘ └─────────────┘                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### （1）CompletionQueue — 核心事件队列
#### <1>伪代码
```cpp
namespace grpc {

class CompletionQueue {
public:
    CompletionQueue();
    ~CompletionQueue();
    
    // ========== 事件获取 ==========
    
    // 阻塞等待，直到有事件或队列关闭
    // 返回值：GOT_EVENT / SHUTDOWN / TIMEOUT
    NextStatus Next(void** tag, bool* ok);
    
    // 带超时的等待
    NextStatus AsyncNext(void** tag, bool* ok, 
                         const std::chrono::system_clock::time_point& deadline);
    
    // ========== 关闭队列 ==========
    void Shutdown();  // 关闭后，Next() 返回 SHUTDOWN

    // ========== 返回状态枚举 ==========
    enum NextStatus {
        GOT_EVENT,   // 成功获取到事件
        SHUTDOWN,    // 队列已关闭
        TIMEOUT      // 超时（仅 AsyncNext）
    };
};

// 服务端专用（支持请求注册）
class ServerCompletionQueue : public CompletionQueue {
public:
    // 服务端专用功能...
};

} // namespace grpc
```
#### <2>使用示例
```cpp
grpc::CompletionQueue cq;

// 事件循环
void* tag;
bool ok;

while (true) {
    // 阻塞等待事件
    grpc::CompletionQueue::NextStatus status = cq.Next(&tag, &ok);
    
    if (status == grpc::CompletionQueue::SHUTDOWN) {
        break;  // 队列关闭
    }
    
    if (status == grpc::CompletionQueue::GOT_EVENT) {
        // tag: 用户自定义指针，用于识别是哪个操作完成
        // ok:  操作是否成功
        //      - true: 操作成功完成
        //      - false: 操作失败（如连接断开、取消等）
        
        ProcessEvent(tag, ok);
    }
}
```

### （2）Alarm — 定时器
#### <1>伪代码
```cpp
namespace grpc {

class Alarm {
public:
    Alarm();
    ~Alarm();
    
    // 设置定时器：到期后 tag 被投递到 cq
    void Set(CompletionQueue* cq, 
             const std::chrono::system_clock::time_point& deadline,
             void* tag);
    
    // 取消定时器
    void Cancel();
};

} // namespace grpc
```

#### <2>使用示例
```cpp
grpc::Alarm alarm;
grpc::CompletionQueue cq;

// 5秒后触发
alarm.Set(&cq, 
          std::chrono::system_clock::now() + std::chrono::seconds(5),
          (void*)"timer_tag");

void* tag;
bool ok;
cq.Next(&tag, &ok);  // 5秒后返回
// tag == "timer_tag"
// ok == true (正常触发) 或 false (被取消)
```

### （3）客户端异步类
#### <1>ClientAsyncResponseReader — 异步 Unary
```cpp
namespace grpc {

template <class R>  // R = 响应类型 (LoginResponse)
class ClientAsyncResponseReader {
public:
    // 启动调用（PrepareAsync 版本需要手动调用）
    void StartCall();
    
    // 读取服务端发送的初始元数据
    void ReadInitialMetadata(void* tag);
    
    // 完成调用，获取响应和状态
    void Finish(R* msg, grpc::Status* status, void* tag);
};

} // namespace grpc
```

#### <2>ClientAsyncReader — 异步 Server Streaming
```cpp
namespace grpc {

// ==================== 客户端异步读取器 ====================
template <class R>  // R = 读取类型 (Order)
class ClientAsyncReader {
public:
    // 启动调用（PrepareAsync 版本需要手动调用）
    void StartCall(void* tag);
    
    // 读取服务端发送的初始元数据
    void ReadInitialMetadata(void* tag);
    
    // 异步读取一条消息
    void Read(R* msg, void* tag);
    
    // 完成调用，获取最终状态
    void Finish(grpc::Status* status, void* tag);
};

} // namespace grpc
```

#### <3>ClientAsyncWriter — 异步 Client Streaming
```cpp
namespace grpc {

template <class W>  // W = 写入类型 (FileChunk)
class ClientAsyncWriter {
public:
    // 启动调用（PrepareAsync 版本需要手动调用）
    void StartCall(void* tag);
    
    // 异步写入一条消息
    void Write(const W& msg, void* tag);
    
    // 带选项的写入
    void Write(const W& msg, grpc::WriteOptions options, void* tag);
    
    // 写完所有消息后调用（告诉服务器不再写了）
    void WritesDone(void* tag);
    
    // 完成调用，获取响应和状态
    // response 在创建 Writer 时已经传入，Finish 后会被填充
    void Finish(grpc::Status* status, void* tag);
    
    // 读取服务端发送的初始元数据
    void ReadInitialMetadata(void* tag);
};

} // namespace grpc
```

#### <4>ClientAsyncReaderWriter — 异步双向流
```cpp
namespace grpc {

template <class W, class R>
class ClientAsyncReaderWriter {
public:
    // 异步读取一条消息
    void Read(R* msg, void* tag);
    
    // 异步写入一条消息
    void Write(const W& msg, void* tag);
    
    // 带 WriteOptions 的写入
    void Write(const W& msg, grpc::WriteOptions options, void* tag);
    
    // 写完所有消息后调用（半关闭）
    void WritesDone(void* tag);
    
    // 完成调用，获取最终状态
    void Finish(grpc::Status* status, void* tag);
    
    // 读写同时进行
    void ReadInitialMetadata(void* tag);
};

} // namespace grpc
```

### （4）服务端异步类
#### <1>ServerAsyncResponseWriter — 异步 Unary
```cpp
namespace grpc {

// ==================== 服务端异步响应写入器 ====================
template <class W>  // W = 响应类型 (LoginResponse)
class ServerAsyncResponseWriter {
public:
    // 发送初始元数据
    void SendInitialMetadata(void* tag);
    
    // 发送响应并结束调用
    void Finish(const W& msg, const grpc::Status& status, void* tag);
    
    // 结束调用但不发送响应（出错时使用）
    void FinishWithError(const grpc::Status& status, void* tag);
};

} // namespace grpc
```

#### <2>ServerAsyncReader — 异步 Client Streaming
```cpp
namespace grpc {

// ==================== 服务端异步读取器 ====================
template <class W, class R>  // W = 响应类型, R = 请求类型
class ServerAsyncReader {
public:
    // 发送初始元数据
    void SendInitialMetadata(void* tag);
    
    // 异步读取一条消息
    void Read(R* msg, void* tag);
    
    // 结束调用并发送响应
    void Finish(const W& msg, const grpc::Status& status, void* tag);
    
    // 结束调用但不发送响应（出错时使用）
    void FinishWithError(const grpc::Status& status, void* tag);
};

} // namespace grpc
```

#### <3>ServerAsyncWriter — 异步 Server Streaming
```cpp
namespace grpc {

// ==================== 服务端异步写入器 ====================
template <class W>  // W = 写入类型 (Order)
class ServerAsyncWriter {
public:
    // 发送初始元数据
    void SendInitialMetadata(void* tag);
    
    // 异步写入一条消息
    void Write(const W& msg, void* tag);
    
    // 带选项的写入
    void Write(const W& msg, grpc::WriteOptions options, void* tag);
    
    // 结束调用（发送状态码）
    void Finish(const grpc::Status& status, void* tag);
    
    // 写入最后一条消息并结束（合并操作，更高效）
    void WriteAndFinish(const W& msg, grpc::WriteOptions options,
                        const grpc::Status& status, void* tag);
};

} // namespace grpc
```

#### <4>ServerAsyncReaderWriter — 异步双向流
```cpp
namespace grpc {

template <class W, class R>
class ServerAsyncReaderWriter {
public:
    // 异步读取
    void Read(R* msg, void* tag);
    
    // 异步写入
    void Write(const W& msg, void* tag);
    void Write(const W& msg, grpc::WriteOptions options, void* tag);
    
    // 结束调用
    void Finish(const grpc::Status& status, void* tag);
    
    // 发送初始元数据
    void SendInitialMetadata(void* tag);
};

} // namespace grpc
```

### （5）异步调用过程详解
#### <1>示例：异步登录
```cpp
#include <grpcpp/grpcpp.h>
#include "auth.grpc.pb.h"  // protoc 生成的头文件

// ============================================================
// CallData 结构体：封装一次 RPC 调用的所有状态
// 每个异步请求都需要独立的状态，因为多个请求可能同时进行
// ============================================================
struct CallData {
    auth::LoginRequest request;     // 请求消息
    auth::LoginResponse response;   // 响应消息（Finish 后被填充）
    grpc::ClientContext context;    // 调用上下文（超时、元数据等）
    grpc::Status status;            // RPC 状态（Finish 后被填充）
    std::unique_ptr<grpc::ClientAsyncResponseReader<auth::LoginResponse>> reader;  // 异步读取器
};

int main() {
    // ============================================================
    // 1. 创建 Channel 和 Stub
    // ============================================================
    // Channel：到服务端的连接（可复用）
    auto channel = grpc::CreateChannel("localhost:50051", 
                                       grpc::InsecureChannelCredentials());
    // Stub：RPC 调用的代理对象
    auto stub = auth::AuthService::NewStub(channel);
    
    // CompletionQueue：异步操作完成通知的队列
    // 所有异步操作完成后都会往这里投递事件
    grpc::CompletionQueue cq;

    // ============================================================
    // 2. 发起多个异步请求（非阻塞，立即返回）
    // ============================================================
    for (int i = 0; i < 3; i++) {
        // 为每个请求分配独立的状态（堆上分配，生命周期跨越异步操作）
        CallData* call = new CallData();
        
        // 填充请求数据
        call->request.set_username("user" + std::to_string(i));
        call->request.set_password("pass" + std::to_string(i));
        
        // AsyncLogin()：发起异步 RPC，立即返回
        // - 请求已发送到服务端
        // - 返回 ClientAsyncResponseReader 用于后续操作
        call->reader = stub->AsyncLogin(&call->context, call->request, &cq);
        
        // Finish()：注册"完成回调"
        // - 告诉 gRPC：当响应到达时，把结果填入 response 和 status
        // - tag 参数：完成时用于识别是哪个请求（这里传 call 指针）
        // - 此时请求还没完成，只是注册了回调
        call->reader->Finish(&call->response, &call->status, (void*)call);
        
        // 注意：循环立即继续，不等待响应
        // 3 个请求几乎同时发出
    }

    // ============================================================
    // 3. 事件循环：等待并处理完成事件
    // ============================================================
    void* tag;   // 完成事件的标识（就是 Finish 时传入的 call 指针）
    bool ok;     // 操作是否成功
    int completed = 0;
    
    // cq.Next()：阻塞等待下一个完成事件
    // 返回 false 表示队列已关闭
    while (completed < 3 && cq.Next(&tag, &ok)) {
        // 从 tag 恢复出 CallData 指针
        CallData* call = static_cast<CallData*>(tag);
        
        // 检查结果
        if (ok && call->status.ok()) {
            // ok=true：gRPC 层面操作成功
            // status.ok()：业务层面 RPC 成功
            std::cout << "成功: " << call->response.token() << std::endl;
        } else {
            // 失败原因可能：网络错误、服务端错误等
            std::cout << "失败: " << call->status.error_message() << std::endl;
        }
        
        // 释放堆内存
        delete call;
        completed++;
    }

    return 0;
}
```
**执行流程图**
```cpp
时间线 ─────────────────────────────────────────────────────────────────►

主线程:
    │
    ├─ AsyncLogin(user0) ──► 请求0发出 ─────────────┐
    ├─ AsyncLogin(user1) ──► 请求1发出 ────────┐    │
    ├─ AsyncLogin(user2) ──► 请求2发出 ───┐    │    │
    │                                     │    │    │
    ├─ cq.Next() [阻塞等待]               │    │    │
    │      ◄────────────────────────────────────────┘ 响应0到达
    ├─ 处理 user0 结果                    │    │
    │                                     │    │
    ├─ cq.Next() [阻塞等待]               │    │
    │      ◄─────────────────────────────────────┘ 响应1到达  
    ├─ 处理 user1 结果                    │
    │                                     │
    ├─ cq.Next() [阻塞等待]               │
    │      ◄──────────────────────────────┘ 响应2到达
    ├─ 处理 user2 结果
    │
    └─ 退出

注意：响应到达顺序可能与发送顺序不同！

```

#### <2>gRPC 异步调用关键过程
- **调用 AsyncLogin() 时**：
	- 传入事件队列 cq，内部通过 Channel 的网络连接将序列化后的请求发送到服务端，并返回一个 ClientAsyncResponseReader 对象。此时请求已发出，但还没注册"完成通知"。
- **调用 Finish(response, status, tag) 时**：
	告诉框架"当响应回来时，把结果写入 response 和 status，然后把 tag 放入事件队列通知我"。此时 tag 被绑定到这次调用。
- **网络层收到响应时（由 gRPC 内部的 I/O 线程处理）**：
	- 框架将响应反序列化后写入用户提供的 response，将状态写入 status，然后把 (tag, ok) 放入事件队列。
- **如果网络层响应先到达（Finish 还没调用）**：
	- gRPC 内部的 I/O 线程会把响应数据暂存在内部缓冲区（绑定在 grpc_call 结构中），不会入队，也不会丢弃，只是静静等待。
	- **调用 Finish(response, status, tag) 时**：框架检查内部缓冲区，发现响应已经到了，于是立即将数据写入 response 和 status，然后把 (tag, ok) 放入事件队列。（如果响应还没到，则注册回调等待）
- **用户调用 cq.Next(&tag, &ok) 时**：
	- 从事件队列中取出完成的事件，**通过 tag 找到对应的调用上下文**，处理结果。

#### <3>tag 的作用
响应确实已经写入 response 和 status 了。那 tag 的作用是什么？	
**🎯 核心问题**：你怎么知道哪个调用完成了？
```cpp
// 假设你同时发起了 100 个异步调用
for (int i = 0; i < 100; i++) {
    CallData* call = new CallData();
    call->id = i;
    call->reader = stub->AsyncLogin(&call->context, call->request, &cq);
    call->reader->Finish(&call->response, &call->status, (void*)call);  // tag = call
}

// 事件循环
void* tag;
bool ok;
while (cq.Next(&tag, &ok)) {
    // 问题来了：这是哪个调用完成了？？
    // response 在哪？status 在哪？
    
    // 答案：通过 tag 找回来！
    CallData* call = static_cast<CallData*>(tag);
    
    // 现在你知道了：
    // - 这是第 call->id 个调用
    // - 响应在 call->response 里
    // - 状态在 call->status 里
}
```
- **tag 不是用来传递数据的，而是用来"认领"的**——当事件队列告诉你"有个调用完成了"，你需要通过 tag 知道"是哪个调用完成了"，从而找到对应的 response 和 status。
- 如果只有一个调用，tag 确实显得多余。但异步模型的意义就在于并发多个调用，tag 是区分它们的唯一标识。


<a id="Proto"></a>
# 附录：Proto文件
## 0.概述
### <1>简单演示
**初学先了解以下几个部分即可，其它的遇到时再看**
```cppServer Streaming（服务端流）
syntax = "proto3"; // 1. 语法版本声明（必须在第一行非注释处）

// 2. 包名声明（可选，防止命名冲突）
package user;

// 3. 导入其他 proto 文件
import "google/protobuf/timestamp.proto";
import "other/message.proto";
import public "shared/common.proto";  // 传递导入

// 4. 消息、枚举、服务定义
message User { ... }
enum Status { ... }
service UserService { ... }

```

### <2>编译命令
通过protoc命令进行编译
```bash
# 基本编译
protoc --cpp_out=./output user.proto

# 指定搜索路径
protoc -I=./protos --cpp_out=./output user.proto

# 生成 gRPC 代码
protoc --cpp_out=. --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` user.proto
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119235923782.png)

## 1.Proto 文件核心语法详解：syntax、package、import、option
### （1）syntax：指定语法版本
#### <1>核心作用
声明 .proto 文件的语法版本，是文件**第一行必须写**的内容，直接决定编译器生成的 C++ 代码结构、字段访问方式和默认行为

#### <2>语法格式
```cpp
syntax = "proto3"; // 推荐使用 proto3
// 或
syntax = "proto2"; // 旧版，不推荐新项目使用
```

### （2）package：定义命名空间（C++ 命名空间映射）
#### <1>核心作用
避免不同 .proto 文件的类型重名，**直接映射为 C++ 的命名空间**，是 C++ 中隔离类型的核心方式。

#### <2>语法格式 + C++ 映射示例
```cpp
// .proto 文件
syntax = "proto3";
package com.example.chat; // 多层级包名

message ChatMsg {
  string content = 1;
}
```
编译后生成的 C++ 代码结构：
```cpp
// 自动生成命名空间，与 package 完全一致
namespace com {
namespace example {
namespace chat {
  class ChatMsg : public ::google::protobuf::Message {
    // ... 成员方法（get_content()、set_content() 等）
  };
} // namespace chat
} // namespace example
} // namespace com

// C++ 代码中使用
com::example::chat::ChatMsg msg;
msg.set_content("hello");
```

#### <3>C++ 实战注意事项
- package 命名建议与 C++ 项目的命名空间规范一致（如 company.module.feature），避免冲突；
- 若多个 .proto 文件使用同一 package，编译后会合并到同一个 C++ 命名空间下；
- 禁止在 package 中使用 C++ 关键字（如 class、namespace），否则生成代码会编译报错。


### （3）import：导入其他 Proto 文件（C++ 类型复用）
#### <1>核心作用
复用其他 .proto 文件定义的 message/enum/service，C++ 中可直接使用导入的类型，无需重复定义。

#### <2>基础语法 + C++ 示例
```cpp
// 1. 基础导入（导入自定义 proto）
import "common/error_code.proto"; 
// 2. 导入官方 Well-Known Type（C++ 常用，如时间戳）
import "google/protobuf/timestamp.proto";

syntax = "proto3";
package com.example.chat;

message ChatMsg {
  string content = 1;
  // 使用导入的枚举（来自 error_code.proto）
  com.example.common.ErrorCode code = 2;
  // 使用官方 Timestamp 类型（C++ 映射为 google::protobuf::Timestamp）
  google::protobuf::Timestamp send_time = 3;
}
```
#### <3>C++ 场景高级用法
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119234919030.png)

#### <4>C++ 编译关键
编译时需通过 -I（或 --proto_path）指定 Proto 文件的搜索根目录，否则编译器找不到导入的文件
```bash
# 示例：指定根目录为 proto/，编译 chat.proto 生成 C++ 代码
protoc -I=proto/ --cpp_out=src/ proto/com/example/chat.proto
```

### （4）option：配置编译行为（C++ 专属配置）
#### <1>核心作用
设置 Protobuf 编译器生成 C++ 代码的规则、命名、优化策略等，分为「文件级 option」（全局）和「局部 option」（字段/枚举）。

#### <2>C++ 常用文件级 Option
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119235233461.png)

#### <3>C++ 常用局部 Option
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119235355637.png)

#### <4>C++ 实战示例（完整 Option 配置）
```cpp
syntax = "proto3";
package com.example.chat;

// C++ 专属全局配置
option cc_namespace = "chat::v1"; // 覆盖默认命名空间
option optimize_for = SPEED;      // 优先执行速度
option cc_enable_arenas = true;   // 启用 Arena 内存分配

enum MsgType {
  option allow_alias = true; // 允许枚举别名
  TEXT = 0;
  TEXT_MSG = 0; // 别名（C++ 中可混用）
  VOICE = 1;
}

message ChatMsg {
  string content = 1;
  // 废弃字段（C++ 使用时会报警告）
  string old_content = 2 [deprecated = true];
  // 自定义 C++ 类型（默认 int32_t → 改为 uint32_t）
  int32 msg_id = 3 [cc_type = "uint32_t"];
}
```


# 附录：Protobuf 学习路线（从基础到高级）
---

## 第一阶段：基础入门

| 序号 | 知识点 | 说明 |
|------|--------|------|
| 1 | Proto 文件语法 | `syntax`、`package`、`import`、`option` |
| 2 | 标量类型 | int32/sint32/fixed32、string/bytes、bool、float/double |
| 3 | 复合类型 | message、enum、嵌套定义 |
| 4 | 字段规则 | singular、`optional`、`repeated`、`map` |
| 5 | 字段编号 | 1-15 优化、reserved、编号不可复用 |
| 6 | 编译与生成 | `protoc` 命令、生成的 C++ 类结构 |
| 7 | 序列化/反序列化 | `SerializeToString`、`ParseFromString` |

---

## 第二阶段：核心应用

| 序号 | 知识点 | 说明 |
|------|--------|------|
| 8 | 默认值机制 | Proto3 默认值、optional 与 has_xxx() |
| 9 | oneof | 互斥字段、union 存储原理 |
| 10 | Well-Known Types | `Any`、`Timestamp`、`Duration`、`Wrapper`、`Struct` |
| 11 | **Service 定义** | RPC 接口定义、生成的 Stub 类 |
| 12 | **gRPC 集成** | 同步/异步调用、四种 RPC 模式（Unary/Stream） |
| 13 | 错误处理 | Status、错误码、metadata |

---

## 第三阶段：进阶优化

| 序号 | 知识点 | 说明 |
|------|--------|------|
| 14 | **编码原理** | Varint、ZigZag、Wire Type、Tag-Length-Value |
| 15 | 性能优化 | Arena 内存池、对象复用、预分配 |
| 16 | 大消息处理 | 分块传输、流式 RPC、CodedInputStream |
| 17 | Proto2 vs Proto3 | 差异对比、迁移注意事项 |
| 18 | 版本兼容 | 字段新增/删除/修改的兼容性规则 |

---

## 第四阶段：高级特性

| 序号 | 知识点 | 说明 |
|------|--------|------|
| 19 | **反射机制** | Descriptor、FieldDescriptor、Reflection |
| 20 | **动态消息** | DynamicMessage、运行时构造消息 |
| 21 | 自定义 Option | extend、自定义注解、代码生成器读取 |
| 22 | 插件开发 | `protoc` 插件机制、CodeGenerator 接口 |
| 23 | JSON 互转 | `util::JsonStringToMessage`、JsonPrintOptions |

---

## 第五阶段：工程实践

| 序号 | 知识点 | 说明 |
|------|--------|------|
| 24 | 项目组织 | proto 文件管理、包结构设计、CI/CD 集成 |
| 25 | API 设计规范 | Google API 设计指南、命名规范、版本管理 |
| 26 | 安全考虑 | 大小限制、递归深度、未知字段处理 |
| 27 | 调试工具 | `protoc --decode`、grpcurl、Wireshark 插件 |
| 28 | 与其他序列化对比 | vs JSON、vs FlatBuffers、vs MessagePack |

---

## 可视化路线图

```
基础 ──────────────────────────────────────────────► 高级

[语法] → [类型] → [序列化] → [Service/gRPC] → [编码原理]
                                    │
                                    ▼
                            [Arena 优化]
                                    │
                                    ▼
                    [反射] → [动态消息] → [插件开发]
                                    │
                                    ▼
                            [工程实践/规范]
```


