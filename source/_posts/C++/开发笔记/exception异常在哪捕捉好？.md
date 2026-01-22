---
title: exception异常在哪捕捉好？
date: 2026-01-19 11:45:48
tags:
	- 笔记
categories:
	- C++
	- 开发笔记
	- exception异常在哪捕捉好？
---

# 1.错误处理分层原则（以Rdkafka为例）
**核心思想**：封装层是技术细节和业务逻辑之间的隔离带。
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                         错误处理职责分层                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐
│   业务层        │  ← 只关心"能不能用"，决定降级/重试/告警策略
│ (UserService)   │     不关心底层技术细节
└────────┬────────┘
         │ 接收封装后的错误 (KafkaResult / std::expected)
         │
┌────────▼────────┐
│   封装层        │  ← 捕获所有底层错误，转换为统一的错误类型
│ (KafkaProducer) │     记录详细日志，隐藏技术细节
└────────┬────────┘
         │ 调用底层 API
         │
┌────────▼────────┐
│   库层          │  ← 返回原始错误码/抛出异常
│ (librdkafka)    │
└─────────────────┘
```

# 2.实际代码对比
## （1）❌ 不好的做法：让异常穿透
```cpp
// 业务层需要了解 librdkafka 的异常类型
void UserService::CreateUser(const User& user) {
    try {
        kafka_producer_.Send("user-events", user.id, Serialize(user));
    } catch (const RdKafka::Exception& e) {  // 业务层耦合了 librdkafka
        // ...
    } catch (const std::runtime_error& e) {
        // ...
    }
}
```

## （2）✅ 推荐做法：封装层捕获并转换
```cpp
// ==================== 封装层：捕获一切，返回统一错误 ====================
class KafkaProducer {
public:
    KafkaResult Send(const std::string& topic, 
                     const std::string& key, 
                     const std::string& value) {
        // 所有错误在这里捕获和转换
        if (!producer_) {
            return KafkaResult::Error(KafkaError::NOT_INITIALIZED, "Producer not init");
        }

        RdKafka::ErrorCode err = producer_->produce(...);
        
        if (err != RdKafka::ERR_NO_ERROR) {
            // 记录详细技术日志（封装层职责）
            LOG_ERROR("Kafka send failed: topic={}, err={}", topic, RdKafka::err2str(err));
            // 返回业务可理解的错误
            return KafkaResult::Error(MapError(err), RdKafka::err2str(err));
        }
        
        return KafkaResult::Ok();
    }
};

// ==================== 业务层：只关心成功/失败 ====================
void UserService::CreateUser(const User& user) {
    auto result = kafka_producer_.Send("user-events", user.id, Serialize(user));
    
    if (!result) {
        // 业务层只关心"失败了"，决定业务策略
        LOG_WARN("Event publish failed, saving to retry queue");
        retry_queue_.Push(user);  // 降级策略
    }
}
```

## （3）封装层的职责清单
```cpp
class KafkaProducer {
public:
    KafkaResult Send(...) {
        // 1️⃣ 前置检查（状态、参数）
        if (!producer_) {
            return KafkaResult::Error(KafkaError::NOT_INITIALIZED, "...");
        }
        if (value.size() > max_message_size_) {
            return KafkaResult::Error(KafkaError::MESSAGE_TOO_LARGE, "...");
        }

        // 2️⃣ 调用底层 API
        RdKafka::ErrorCode err = producer_->produce(...);

        // 3️⃣ 处理特定错误（如队列满重试）
        if (err == RdKafka::ERR__QUEUE_FULL) {
            // 内部重试，业务层无感知
            for (int i = 0; i < 3 && err == RdKafka::ERR__QUEUE_FULL; i++) {
                producer_->poll(100);
                err = producer_->produce(...);
            }
        }

        // 4️⃣ 错误转换 + 日志
        if (err != RdKafka::ERR_NO_ERROR) {
            LOG_ERROR("Kafka error: {}", RdKafka::err2str(err));  // 详细日志
            return KafkaResult::Error(MapError(err), RdKafka::err2str(err));
        }

        // 5️⃣ 成功
        return KafkaResult::Ok();
    }
};
```