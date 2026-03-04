---
title: Kafka实战笔记
date: 2026-01-18 23:29:45
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- Kafka
---

# 一、实战技巧
## 1.msg_opaque 的使用

### （1）msg_opaque 传递函数指针
```cpp
//@param msg_opaque 用户自定义指针，会在投递回调中返回
virtual ErrorCode produce(Topic *topic,
                          int32_t partition,
                          int msgflags,
                          void *payload,
                          size_t len,
                          const std::string *key,
                          void *msg_opaque) = 0;
```
**注意**：
- 这里传递的函数指针不是“回调函数”，是需要我们在函数中手动调用的
- RdKafka 只有一种设置回调函数的方式 ——> 通过 Conf::set() 设置，Producer::create() 只接收 Conf 对象

#### <1>为什么用 msg_opaque 传递函数指针？
```cpp
┌─────────────────────────────────────────────────────────────────────┐
│  问题：dr_cb 是全局唯一的，但每条消息可能需要不同的回调逻辑          │
└─────────────────────────────────────────────────────────────────────┘

    消息A ──→ 发送成功后：更新数据库
    消息B ──→ 发送成功后：通知用户  
    消息C ──→ 不需要回调
                    │
                    ▼
        ┌─────────────────────┐
        │  dr_cb（全局唯一）   │  ← 它怎么知道每条消息该做什么？
        └─────────────────────┘
                    │
                    ▼
        答案：通过 msg_opaque 携带每条消息的"专属回调"
```
**注意**：
- librdkafka 是 C 库的 C++ 封装，保留了 C 风格的 void* 回调机制。
```cpp
// ❌ 理想中的 API（不存在）
producer->produce(topic, payload, [](bool ok) {
    // 每条消息直接传 lambda
});

// ✅ 实际的 API（C 风格设计）
producer->produce(topic, ..., msg_opaque);  // 只能传 void* 指针

```

#### <2>工作原理图解
- 虽然produce不支持lambda表达式，但可以自己封装Produce支持lambda
```cpp
                        produce() 调用时
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   // 用户代码                                                     │
│   producer->Produce("topic", "msg1", [](bool ok, auto& err) {    │
│       UpdateDatabase();  // 消息1的专属逻辑                       │
│   });                                                            │
│                                                                  │
│   producer->Produce("topic", "msg2", [](bool ok, auto& err) {    │
│       NotifyUser();      // 消息2的专属逻辑                       │
│   });                                                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│   // Produce 内部实现                                             │
│                                                                  │
│   auto* cb = new ProduceCallback(std::move(callback));           │
│   producer_->produce(..., cb);  // cb 作为 msg_opaque 传入       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                        dr_cb() 回调时
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   void dr_cb(Message& msg) {                                     │
│       auto* cb = static_cast<ProduceCallback*>(msg.msg_opaque());│
│       (*cb)(success, error);  // 执行这条消息专属的回调           │
│       delete cb;                                                 │
│   }                                                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 2.producer的封装
### （1）kafka_producer.h
```cpp
#pragma once


#include <librdkafka/rdkafkacpp.h>
#include <memory>
#include <string>
#include <functional>
#include <atomic>
#include <unordered_map>
#include "config/config.h"
#include "common/result.h"


namespace user_service{

using ProduceCallback=std::function<void(bool success, const std::string& error)>;

class KafkaProducer{
public:
    explicit KafkaProducer(const KafkaConfig& config);
    ~KafkaProducer();

    // 禁止拷贝
    KafkaProducer(const KafkaProducer&) = delete;
    KafkaProducer& operator=(const KafkaProducer&) = delete;

    // ==================== 发送消息 ====================
    // 异步发送（fire-and-forget：发射后不管）
    Result<void> Send(const std::string& topic, 
        const std::string& key,
        const std::string& message) noexcept;

    // 异步发送（带回调）
    // produce本身是C API，不支持lambda，但通过封装后可以支持lamdba
    Result<void> SendAsync(const std::string& topic,
                const std::string& key,
                const std::string& message,
                ProduceCallback callback) noexcept;

    // ==================== 管理方法 ====================
    // 

    /// @brief 等待所有消息发送完成
    /// @param timeout_ms 等待时间，默认5000ms
    /// @return 返回消息队列中剩余还未发送的消息数量
    Result<int> Flush(int timeout_ms = 5000) noexcept;
    
    // 

    /// @brief 触发回调处理（需要定期调用，或在单独线程中调用）
    /// @param timeout_ms 阻塞时间，默认“非阻塞”
    /// @return 返回处理的事件数量
    int Poll(int timeout_ms = 0);   // 几乎不会失败,没必要用Result

    // 获取待发送队列长度
    int OutQueueLength() const;

    // 是否健康
    bool IsHealthy() const noexcept { return healthy_.load(); }

    // 统计信息
    struct Stats {
        uint64_t messages_sent = 0;
        uint64_t messages_failed = 0;
        uint64_t bytes_sent = 0;
    };
    Stats GetStats() const noexcept;


private:
    class DeliveryCallback;

    std::unique_ptr<RdKafka::Producer> producer_;
    std::unique_ptr<RdKafka::Conf> config_;
    std::unique_ptr<DeliveryCallback> delivery_cb_;

    std::atomic<bool> healthy_{false};
    mutable std::atomic<uint64_t> messages_sent_{0};
    mutable std::atomic<uint64_t> messages_failed_{0};
    mutable std::atomic<uint64_t> bytes_sent_{0};
};

}
```

### （2）kafka_producer.cpp
```cpp
#include "kafka_producer.h"
#include "detail/detail.h"
#include "common/logger.h"
#include "common/error_codes.h"

namespace user_service{

KafkaProducer::KafkaProducer(const KafkaConfig& config){
    try{
        // 1.设置配置
        config_=KafkaConfBuilder()
                .ApplyForProducer(config)
                .SetCallBack("dr_cb",delivery_cb_.get())
                .Build();
        
        // 2.生成producer
        std::string errstr;
        producer_.reset(RdKafka::Producer::create(config_.get(),errstr));
        if(!producer_){
            throw std::runtime_error("创建 Kafka Producer 失败: " + errstr);
        }
        healthy_ = true;
        LOG_INFO("[Kafka] Producer 初始化成功, brokers={}", config.brokers);
    }catch(const std::exception& e){
        // 捕捉KafkaConfBuilder()构造配置过程的错误信息
        throw;
    }
}

KafkaProducer::~KafkaProducer(){
    if (producer_) {
        // 等待所有消息发送完成
        LOG_INFO("[Kafka] Producer 关闭中，等待消息发送完成...");
        Flush(10000);
        LOG_INFO("[Kafka] Producer 已关闭");
    }
}

Result<void> KafkaProducer::Send(const std::string& topic, 
    const std::string& key,
    const std::string& message){

    // 健康检查
    if (!healthy_) {
        LOG_WARN("[Kafka] Producer 不健康，消息丢弃");
        return Result<void>::Fail(ErrorCode::Internal,"服务器内部异常");
    }

    RdKafka::ErrorCode err = producer_->produce(
        topic,                                    // topic
        RdKafka::Topic::PARTITION_UA,            // 自动分区
        RdKafka::Producer::RK_MSG_COPY,          // 复制消息
        const_cast<char*>(message.c_str()),      // payload
        message.size(),                           // payload size
        key.empty() ? nullptr : key.c_str(),     // key
        key.size(),                               // key size
        0,                                        // timestamp (0=使用当前时间)
        nullptr                                   // opaque (回调参数)
    );

    if(err!=RdKafka::ERR_NO_ERROR){
        if(err==RdKafka::ERR__QUEUE_FULL){ // 发送队列满了
            LOG_INFO("[Kafka] 队列满，等待后重试");
            producer_->poll(100);
            // 重试一次
            err = producer_->produce(
                topic, RdKafka::Topic::PARTITION_UA, RdKafka::Producer::RK_MSG_COPY,
                const_cast<char*>(message.c_str()), message.size(),
                key.empty() ? nullptr : key.c_str(), key.size(), 0, nullptr
            );
        }

        // 还是没有成功
        if (err != RdKafka::ERR_NO_ERROR) {
            LOG_ERROR("[Kafka] 发送失败: topic={}, error={}", 
                         topic, RdKafka::err2str(err));
            messages_failed_++;
            
            return Result<void>::Fail(ErrorCode::Internal,"内部服务器异常");
        }

    }

    // 触发回调
    producer_->poll(0);
    return Result<void>::Ok();

}

Result<void> KafkaProducer::SendAsync(const std::string& topic,
    const std::string& key,
    const std::string& message,
    ProduceCallback callback){
    
    // 健康检查
    if (!healthy_) {
        LOG_WARN("[Kafka] Producer 不健康，消息丢弃");
        return Result<void>::Fail(ErrorCode::Internal,"服务器内部异常");
    }

    // 在堆上分配回调，由 delivery callback 释放
    // 同时可以将lambda表达式转为 void *
    auto* cb= new ProduceCallback(std::move(callback));

    RdKafka::ErrorCode err = producer_->produce(
        topic,
        RdKafka::Topic::PARTITION_UA,
        RdKafka::Producer::RK_MSG_COPY,
        const_cast<char*>(message.c_str()), message.size(),
        key.empty() ? nullptr : key.c_str(), key.size(),
        0,
        cb  // opaque 参数，传递给回调
    );

    if(err!=RdKafka::ERR_NO_ERROR){
        if(err==RdKafka::ERR__QUEUE_FULL){ // 发送队列满了
            LOG_INFO("[Kafka] 队列满，等待后重试");
            producer_->poll(100);
            // 重试一次
            err = producer_->produce(
                topic, RdKafka::Topic::PARTITION_UA, RdKafka::Producer::RK_MSG_COPY,
                const_cast<char*>(message.c_str()), message.size(),
                key.empty() ? nullptr : key.c_str(), key.size(), 0, nullptr
            );
        }

        // 还是没有成功
        if (err != RdKafka::ERR_NO_ERROR) {
            LOG_ERROR("[Kafka] 发送失败: topic={}, error={}", 
                         topic, RdKafka::err2str(err));
            messages_failed_++;
            
            return Result<void>::Fail(ErrorCode::Internal,"内部服务器异常");
        }

    }

    // 触发回调
    producer_->poll(0);
    return Result<void>::Ok();
}


// 等待所有消息发送完成（默认等待5s）
Result<int> KafkaProducer::Flush(int timeout_ms){
    if (!producer_) {
        return Result<int>::Fail(ErrorCode::Internal, "Producer not initialized");
    }

    // 等待消息发送
    RdKafka::ErrorCode err=producer_->flush(timeout_ms);
    
    // 获取消息队列中剩余消息数量
    int remaining=producer_->outq_len();

    if (err == RdKafka::ERR__TIMED_OUT) {
        // 超时：不算错误，返回剩余消息数，让调用方决定如何处理
        return Result<int>::Ok(remaining);
    }

    if (err != RdKafka::ERR_NO_ERROR) {
        // 其他真正的错误
        LOG_WARN("Flush failed: " + RdKafka::err2str(err));
        return Result<int>::Fail(ErrorCode::Internal,"内部服务器异常");
    }

    // 成功，队列已清空
    return Result<int>::Ok(0);
}

// 触发回调处理（默认“非阻塞”）（需要定期调用，或在单独线程中调用）
int KafkaProducer::Poll(int timeout_ms){
    return producer_ ? producer_->poll(timeout_ms) : 0;
}

// 获取待发送队列长度
int KafkaProducer::OutQueueLength() const{
    return producer_ ? producer_->outq_len() : 0;
}

KafkaProducer::Stats KafkaProducer::GetStats() const{
    return Stats{
        messages_sent_.load(),
        messages_failed_.load(),
        bytes_sent_.load()
    };
}

}
```

### （3）kafka_callbacks.h
```cpp
#pragma once
#include <librdkafka/rdkafkacpp.h>
#include "common/logger.h"
#include "kafka/kafka_producer.h"
namespace user_service{

class KafkaProducer::DeliveryCallback: public RdKafka::DeliveryReportCb{
public:
    void dr_cb(RdKafka::Message &message) override{
        if(message.err()){
            LOG_ERROR("[Kafka] 消息投递失败: topic={}, partition={}, error={}",
                message.topic_name(), 
                message.partition(),
                message.errstr());
            producer_->messages_failed_++;
        
            // 如果有回调，执行回调
            if(message.msg_opaque()){
                auto* cb=static_cast<ProduceCallback*>(message.msg_opaque());
                (*cb)(false,message.errstr());
                delete cb;
            }
        }else{
            LOG_TRACE("[Kafka] 消息投递成功: topic={}, partition={}, offset={}",
                message.topic_name(),
                message.partition(),
                message.offset());
            producer_->messages_sent_++;
            producer_->bytes_sent_ += message.len();
            
            if (message.msg_opaque()) {
                auto* cb = static_cast<ProduceCallback*>(message.msg_opaque());
                (*cb)(true, "");
                delete cb;
            }
        }
    }
private:
    KafkaProducer* producer_;
};
}
```

### （4）kafka_conf_builder.h
```cpp
// ============ Kafka Producer 配置 ============
struct KafkaProducerConfig {
    std::optional<std::string> acks;                  // 消息发送确认级别，可选0/1/all(-1)，控可靠性
    std::optional<bool> enable_idempotence;           // 是否启用幂等性，保证消息仅投递一次
    std::optional<int> retries;                       // 发送失败最大重试次数，0为不重试
    std::optional<int> retry_backoff_ms;              // 重试退避间隔(ms)，避免频繁重试压垮broker
    std::optional<int> delivery_timeout_ms;           // 消息投递总超时(ms)，含传输/重试所有耗时
    std::optional<int> batch_size;                    // 批量发送批次大小(字节)，达阈值立即发送
    std::optional<int> linger_ms;                     // 批量等待最大延迟(ms)，超时强制发送批次
    std::optional<std::string> compression_codec;     // 消息压缩算法，可选none/gzip/snappy/lz4/zstd
    std::optional<int> queue_buffering_max_messages;  // 生产者本地队列最大消息数，满则阻塞/丢弃
    std::optional<int> queue_buffering_max_kbytes;    // 生产者本地队列最大容量(KB)，满则阻塞/丢弃

    std::string ToString() const;
};

// ============ Kafka Consumer 配置 ============
struct KafkaConsumerConfig {
    std::optional<std::string> group_id;
    std::optional<std::string> auto_offset_reset;
    std::optional<bool> enable_auto_commit;
    std::optional<int> max_poll_records;
    std::optional<int> session_timeout_ms;
    std::optional<int> heartbeat_interval_ms;

    std::string ToString() const;
};

// ============ Kafka 网络配置 ============
struct KafkaNetworkConfig {
    std::optional<int> socket_timeout_ms;
    std::optional<int> reconnect_backoff_ms;
    std::optional<int> reconnect_backoff_max_ms;

    std::string ToString() const;
};


// ============ Kafka 主配置 ============
struct KafkaConfig {
    std::string brokers = "localhost:9092";
    std::string user_events="user-events";
    std::string client_id= "user-service";
    
    KafkaProducerConfig producer;
    KafkaConsumerConfig consumer;
    KafkaNetworkConfig network;

    std::string ToString() const;
};
```

```cpp
#pragma once
#include <librdkafka/rdkafkacpp.h>
#include <memory>
#include <stdexcept>
#include <string>
#include <vector>
#include <sstream>
#include <type_traits>
#include <utility>

#include "config/config.h"

namespace user_service{

/**
 * @brief Kafka 配置构建器（Builder 模式）
 * 
 * @example Producer 配置示例
 * @code
 * // 方式1: 使用完整配置结构
 * auto conf = KafkaConfBuilder()
 *     .ApplyForProducer(kafka_config)
 *     .SetCallBack("dr_cb", &delivery_cb)      // 设置投递回调
 *     .SetCallBack("event_cb", &event_cb)      // 设置事件回调
 *     .Build();
 * 
 * // 方式2: 手动设置各项参数
 * auto conf = KafkaConfBuilder()
 *     .Set("bootstrap.servers", "localhost:9092")
 *     .Set("client.id", "user-service-producer")
 *     .Set("acks", "all")
 *     .Set("enable.idempotence", "true")
 *     .Set("retries", "3")
 *     .Set("compression.codec", "snappy")
 *     .Build();
 * 
 * // 创建 Producer
 * std::string errstr;
 * auto producer = std::unique_ptr<RdKafka::Producer>(
 *     RdKafka::Producer::create(conf.get(), errstr)
 * );
 * @endcode
 * 
 * @example Consumer 配置示例
 * @code
 * auto conf = KafkaConfBuilder()
 *     .ApplyForConsumer(kafka_config)
 *     .Set("group.id", "user-event-consumer-group")
 *     .SetCallBack("rebalance_cb", &rebalance_cb)
 *     .Build();
 * 
 * // 创建 Consumer
 * std::string errstr;
 * auto consumer = std::unique_ptr<RdKafka::KafkaConsumer>(
 *     RdKafka::KafkaConsumer::create(conf.get(), errstr)
 * );
 * @endcode
 * 
 * @example 错误处理示例
 * @code
 * KafkaConfBuilder builder;
 * builder.Set("bootstrap.servers", "localhost:9092")
 *        .Set("invalid.key", "value");  // 无效配置
 * 
 * if (builder.HasErrors()) {
 *     for (const auto& err : builder.GetErrors()) {
 *         LOG_ERROR("Config error: {}", err);
 *     }
 *     return;
 * }
 * 
 * // 或者直接 Build()，会抛出异常
 * try {
 *     auto conf = builder.Build();
 * } catch (const std::runtime_error& e) {
 *     LOG_ERROR("Build failed: {}", e.what());
 * }
 * @endcode
 * 
 * @example 配合配置文件使用
 * @code
 * // config.yaml:
 * // kafka:
 * //   brokers: "kafka:9092"
 * //   client_id: "user-service"
 * //   producer:
 * //     acks: "all"
 * //     enable_idempotence: true
 * //     retries: 3
 * //   topics:
 * //     user_events: "user-events"
 * 
 * auto config = Config::LoadFromFile("config.yaml");
 * auto conf = KafkaConfBuilder()
 *     .ApplyForProducer(config.kafka)
 *     .Build();
 * @endcode
 */

//RdKafka::Conf构造器
class KafkaConfBuilder{
public:
    KafkaConfBuilder(RdKafka::Conf::ConfType conf_type = RdKafka::Conf::CONF_GLOBAL) {
        conf_.reset(RdKafka::Conf::create(conf_type));

        if(!conf_){
            throw std::runtime_error("Failed to create RdKafka::Conf object");
        }
    }

    // 基础设置方法
    KafkaConfBuilder& Set(const std::string& key, const std::string& value) {
        // 空指针保护
        if (!conf_) {
            errors_.push_back("Conf object is null, cannot set key: " + key);
            return *this;
        }

        std::string errstr;
        if (conf_->set(key, value, errstr) != RdKafka::Conf::CONF_OK) {
            errors_.push_back("Failed to set " + key + ": " + errstr);
        }
        return *this;
    }

    // 设置回调
    template<typename T>
    KafkaConfBuilder& SetCallBack(const std::string& key, T* cb) {
        // 空指针保护
        if (!conf_) {
            errors_.push_back("Conf object is null, cannot set key: " + key);
            return *this;
        }

        // 静态断言约束回调类型，提前拦截错误
        static_assert(
            std::is_base_of<RdKafka::DeliveryReportCb, T>::value ||
            std::is_base_of<RdKafka::EventCb, T>::value ||
            std::is_base_of<RdKafka::RebalanceCb, T>::value ||
            std::is_base_of<RdKafka::OffsetCommitCb, T>::value ||
            std::is_base_of<RdKafka::PartitionerCb, T>::value ||
            std::is_base_of<RdKafka::ConsumeCb, T>::value   ||
            std::is_base_of<RdKafka::OAuthBearerTokenRefreshCb, T>::value   ||
            std::is_base_of<RdKafka::SslCertificateVerifyCb, T>::value  ||
            std::is_base_of<RdKafka::SocketCb, T>::value    ||
            std::is_base_of<RdKafka::OpenCb, T>::value  ||
            std::is_base_of<RdKafka::PartitionerKeyPointerCb, T>::value,
            "T must inherit from RdKafka callback base class (e.g. DeliveryReportCb, ErrorCb)"
        );

        // 空回调指针检查
        if (!cb) {
            errors_.push_back("Callback pointer is null for key: " + key);
            return *this;
        }

        std::string errstr;
        if (conf_->set(key, cb, errstr) != RdKafka::Conf::CONF_OK) {
            errors_.push_back("Failed to set callback " + key + ": " + errstr);
        }
        return *this;
    }

    // 应用 Producer 配置
    KafkaConfBuilder& ApplyProducerConfig(const KafkaProducerConfig& config) {
        if (config.acks) Set("acks", *config.acks);
        if (config.enable_idempotence) Set("enable.idempotence", *config.enable_idempotence ? "true" : "false");
        if (config.retries) Set("retries", std::to_string(*config.retries));
        if (config.retry_backoff_ms) Set("retry.backoff.ms", std::to_string(*config.retry_backoff_ms));
        if (config.delivery_timeout_ms) Set("delivery.timeout.ms", std::to_string(*config.delivery_timeout_ms));
        if (config.batch_size) Set("batch.size", std::to_string(*config.batch_size));
        if (config.linger_ms) Set("queue.buffering.max.ms", std::to_string(*config.linger_ms));
        if (config.compression_codec) Set("compression.codec", *config.compression_codec);
        if (config.queue_buffering_max_messages) Set("queue.buffering.max.messages", std::to_string(*config.queue_buffering_max_messages));
        if (config.queue_buffering_max_kbytes) Set("queue.buffering.max.kbytes", std::to_string(*config.queue_buffering_max_kbytes));
        return *this;
    }

    // 应用 Consumer 配置
    KafkaConfBuilder& ApplyConsumerConfig(const KafkaConsumerConfig& config) {
        if (config.group_id) Set("group.id", *config.group_id);
        if (config.auto_offset_reset) Set("auto.offset.reset", *config.auto_offset_reset);
        if (config.enable_auto_commit) Set("enable.auto.commit", *config.enable_auto_commit ? "true" : "false");
        if (config.max_poll_records) Set("max.poll.records", std::to_string(*config.max_poll_records));
        if (config.session_timeout_ms) Set("session.timeout.ms", std::to_string(*config.session_timeout_ms));
        if (config.heartbeat_interval_ms) Set("heartbeat.interval.ms", std::to_string(*config.heartbeat_interval_ms));
        return *this;
    }

    // 应用 Network 配置
    KafkaConfBuilder& ApplyNetworkConfig(const KafkaNetworkConfig& config) {
        if (config.socket_timeout_ms) Set("socket.timeout.ms", std::to_string(*config.socket_timeout_ms));
        if (config.reconnect_backoff_ms) Set("reconnect.backoff.ms", std::to_string(*config.reconnect_backoff_ms));
        if (config.reconnect_backoff_max_ms) Set("reconnect.backoff.max.ms", std::to_string(*config.reconnect_backoff_max_ms));
        return *this;
    }

    // 应用完整配置（用于 Producer）
    KafkaConfBuilder& ApplyForProducer(const KafkaConfig& config) {
        Set("bootstrap.servers", config.brokers);
        Set("client.id", config.client_id);
        ApplyProducerConfig(config.producer);
        ApplyNetworkConfig(config.network);
        return *this;
    }

    // 应用完整配置（用于 Consumer）
    KafkaConfBuilder& ApplyForConsumer(const KafkaConfig& config) {
        Set("bootstrap.servers", config.brokers);
        Set("client.id", config.client_id);
        ApplyConsumerConfig(config.consumer);
        ApplyNetworkConfig(config.network);
        return *this;
    }

    // 构建
    std::unique_ptr<RdKafka::Conf> Build() {
        // 防止重复 Build
        if (build_called_) {
            throw std::runtime_error("KafkaConfBuilder::Build() called multiple times (conf ownership moved)");
        }

        if (!errors_.empty()) {
            std::ostringstream oss;
            oss << "KafkaConfBuilder errors:\n";
            for (const auto& err : errors_) {
                oss << "  - " << err << "\n";
            }
            throw std::runtime_error(oss.str());
        }
        
        if (!conf_) {
            throw std::runtime_error("KafkaConfBuilder: Conf object is null");
        }

        build_called_ = true;
        return std::move(conf_);
    }

    // 检查是否有错误
    bool HasErrors() const { return !errors_.empty(); }
    const std::vector<std::string>& GetErrors() const { return errors_; }

    //explicit：只允许在条件判断中使用（if/while/!/&&/||），禁止意外的隐式转换到其他类型。
    explicit operator bool() const{
        return conf_!=nullptr;
    }

private: 
    std::unique_ptr<RdKafka::Conf> conf_;
    std::vector<std::string> errors_;
    bool build_called_ = false;  // 标记是否已调用 Build
};

}
```



# 一、特殊情况的应对策略
## 1.Producer侧“消息队列”满的处理策略
实际开发中，队列满（ERR__QUEUE_FULL）的处理取决于业务对可靠性 vs 性能的权衡
📋 **策略选择决策树**
```cpp
                    消息丢失可接受吗？
                          │
            ┌────Yes──────┴──────No────┐
            │                          │
            ▼                          ▼
      丢弃+监控告警              延迟敏感吗？
     （日志/埋点）                    │
                       ┌───Yes───────┴───────No───┐
                       │                          │
                       ▼                          ▼
                有限重试+退避              本地持久化兜底
               （交易/订单）               （核心业务）
```


### （1）问题描述
使用producer调用send时，如果出现“消息队列已满”，该如何处理？
```cpp
bool KafkaProducer::Send(const std::string& topic,const std::string& key,const std::string& value){
    RdKafka::ErrorCode err = producer_->produce(
        topic,
        RdKafka::Topic::PARTITION_UA,
        RdKafka::Producer::RK_MSG_COPY,
        (void*)value.data(),value.size(),
        key.data(),key.size(),
        0,nullptr 
    );

    if(err!=RdKafka::ERR_NO_ERROR){
        if(err==RdKafka::ERR__QUEUE_FULL){
        	//TODO: 如何处理“消息队列已满”比较好？
        }
        LOG_ERROR("kafka produce failed: topic={},error{}",topic,RdKafka::err2str(err));
        return false;
    }

    //触发回调
    producer_->poll(0);

    LOG_INFO("Kafka message produced: topic={},key={}",topic,RdKafka::err2str(err));
    return true;
}
```

### （2）kafka的配置项无法解决此场景吗？
```cpp
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              配置项关联关系                                              │
└─────────────────────────────────────────────────────────────────────────────────────────┘

                           ┌─────────────────┐
                           │   produce()     │
                           └────────┬────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
        ┌───────────────┐  ┌──────────────┐  ┌─────────────────┐
        │ 消息大小检查   │  │  队列容量检查 │  │   入队成功       │
        │ message.max.  │  │ queue.buffer │  │                 │
        │ bytes         │  │ ing.max.*    │  │                 │
        └───────┬───────┘  └──────┬───────┘  └────────┬────────┘
                │                 │                   │
        ┌───────┴───────┐  ┌──────┴───────┐          │
        │ ERR_MSG_SIZE  │  │ERR__QUEUE_   │          │
        │ _TOO_LARGE    │  │FULL          │          │
        │               │  │              │          │
        │ ❌ 不触发重试  │  │ ❌ 不触发重试 │          │
        └───────────────┘  └──────────────┘          │
                                                     ▼
                                          ┌──────────────────┐
                                          │   批量组装       │
                                          │ batch.size      │
                                          │ batch.num.msgs  │
                                          │ queue.buffering │
                                          │ .max.ms         │
                                          └────────┬─────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │   压缩处理       │
                                          │ compression.    │
                                          │ codec/level     │
                                          └────────┬─────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │   发送请求       │
                                          │ acks            │
                                          │ request.timeout │
                                          │ .ms             │
                                          └────────┬─────────┘
                                                   │
                          ┌────────────────────────┼────────────────────────┐
                          │                        │                        │
                          ▼                        ▼                        ▼
                   ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
                   │  发送成功    │         │ 可重试错误   │         │ 不可重试错误 │
                   │             │         │             │         │             │
                   │ delivery    │         │ ✅ 触发重试  │         │ ❌ 不触发重试│
                   │ report(OK)  │         │             │         │             │
                   └─────────────┘         └──────┬──────┘         └─────────────┘
                                                  │
                                                  ▼
                                          ┌──────────────────┐
                                          │   重试控制       │
                                          │ retries         │
                                          │ retry.backoff.  │
                                          │ ms/max.ms       │
                                          └────────┬─────────┘
                                                   │
                                    ┌──────────────┴──────────────┐
                                    │                             │
                                    ▼                             ▼
                            ┌─────────────┐               ┌─────────────┐
                            │ 重试次数>0   │               │ 重试次数=0   │
                            │ 继续重试     │               │ 放弃重试     │
                            └──────┬──────┘               └──────┬──────┘
                                   │                             │
                                   ▼                             │
                            ┌─────────────┐                      │
                            │delivery.    │                      │
                            │timeout.ms   │                      │
                            │超时检查     │                      │
                            └──────┬──────┘                      │
                                   │                             │
                          ┌────────┴────────┐                    │
                          │                 │                    │
                          ▼                 ▼                    ▼
                    ┌──────────┐      ┌──────────┐        ┌──────────┐
                    │ 未超时    │      │ 已超时    │        │delivery  │
                    │ 继续重试  │      │ 放弃      │        │report    │
                    └──────────┘      └────┬─────┘        │(FAIL)    │
                                           │              └──────────┘
                                           └──────────────────┘
```

### （3）配置层面预防
调整 librdkafka 配置减少队列满的概率
```cpp
conf_ = KafkaConfBuilder()
    .Set("bootstrap.servers", brokers)
    .Set("queue.buffering.max.messages", "1000000")  // 增大队列容量
    .Set("queue.buffering.max.kbytes", "1048576")    // 1GB 缓冲区
    .Set("batch.num.messages", "10000")              // 批量发送
    .Set("queue.buffering.max.ms", "5")                           // 等待聚合
    .Set("message.send.max.retries", "3")       // librdkafka 内部重试
    .Set("retry.backoff.ms", "100")             // 内部重试退避
    .Set("delivery.timeout.ms", "120000")       // 投递总超时
    .Set("acks", "all")                         // 可靠性要求高时使用
    .Build();
```

### （4）常见策略对比
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118233422535.png)
#### <1>有限重试 + 指数退避（推荐）
```cpp
bool KafkaProducer::Send(const std::string& topic, 
                          const std::string& key, 
                          const std::string& value) {
    constexpr int kMaxRetries = 3;       // 最大重试次数（队列满时）
    constexpr int kMaxBackoffMs = 1000;  // 退避时间上限，防止等待过久
    int backoff_ms = 50;                 // 初始退避时间

    // 重试循环：retry=0 是首次发送，1~3 是重试
    for (int retry = 0; retry <= kMaxRetries; ++retry) {
        
        // 调用 librdkafka 发送消息
        // RK_MSG_COPY：librdkafka 会复制 value/key，函数返回后原数据可释放
        // PARTITION_UA：由 librdkafka 根据 key 哈希自动选择分区
        RdKafka::ErrorCode err = producer_->produce(
            topic,
            RdKafka::Topic::PARTITION_UA,
            RdKafka::Producer::RK_MSG_COPY,
            (void*)value.data(), value.size(),
            key.data(), key.size(),
            0,        // timestamp: 0 表示使用当前时间
            nullptr   // msg_opaque: 传递给回调的用户数据，这里不需要
        );

        // 发送成功（仅表示入队成功，非真正送达 Kafka）
        if (err == RdKafka::ERR_NO_ERROR) {
            producer_->poll(0);  // 非阻塞触发回调，处理之前的发送结果
            return true;
        }

        // 非队列满错误（如 topic 不存在、消息过大等），不可恢复，直接失败
        if (err != RdKafka::ERR__QUEUE_FULL) {
            LOG_ERROR("Kafka produce failed: topic={}, err={}", 
                      topic, RdKafka::err2str(err));
            return false;
        }

        // 队列满，准备重试
        if (retry < kMaxRetries) {
            LOG_WARN("Kafka queue full, retry {}/{} after {}ms", 
                     retry + 1, kMaxRetries, backoff_ms);
            
            // poll 的双重作用：
            // 1. 阻塞等待 backoff_ms 毫秒
            // 2. 触发投递回调，释放已发送消息占用的队列空间
            producer_->poll(backoff_ms);
            
            // 指数退避 + 随机抖动
            // - 指数退避(x2)：50 → 100 → 200 → 400...
            // - 随机抖动(+rand%50)：避免多个生产者同时重试造成"惊群效应"
            // - min 限制上限：防止退避时间无限增长
            backoff_ms = std::min(backoff_ms * 2 + rand() % 50, kMaxBackoffMs);
        }
    }

    // 重试耗尽，消息发送失败
    // 调用方可选择：丢弃、写入 WAL、或其他兜底策略
    LOG_ERROR("Kafka queue full after {} retries, topic={}", kMaxRetries, topic);
    return false;
}
```



#### <2>丢弃 + 监控告警（日志/埋点场景）
```cpp
bool KafkaProducer::SendBestEffort(const std::string& topic,
                                    const std::string& key,
                                    const std::string& value) {
    RdKafka::ErrorCode err = producer_->produce(
        topic,
        RdKafka::Topic::PARTITION_UA,
        RdKafka::Producer::RK_MSG_COPY,
        (void*)value.data(), value.size(),
        key.data(), key.size(),
        0, nullptr
    );

    producer_->poll(0);

    if (err == RdKafka::ERR_NO_ERROR) {
        return true;
    }

    if (err == RdKafka::ERR__QUEUE_FULL) {
        ++queue_full_count_;  // 统计
        
        // 每 1000 次告警一次，避免日志爆炸
        if (queue_full_count_ % 1000 == 1) {
            LOG_WARN("Kafka queue full, {} messages dropped", queue_full_count_);
            // 可以发送告警到监控系统
            // AlertManager::Send("kafka_queue_full", queue_full_count_);
        }
        return false;  // 直接丢弃
    }

    LOG_ERROR("Kafka produce failed: {}", RdKafka::err2str(err));
    return false;
}
```

#### <3>本地持久化兜底（比较复杂）
核心逻辑：发送失败 → 写文件 → 后台线程定期重试 → 成功后删除记录。
**核心流程图**
```
业务调用 SendReliable()
         │
         ▼
    ┌─────────────┐
    │ 尝试发送x3  │
    └─────────────┘
         │
    成功? ──Yes──► return true
         │
        No
         ▼
    ┌─────────────┐
    │ 写入 WAL    │  ← 持久化到磁盘
    └─────────────┘
         │
         ▼
    return true（消息已被接管）


后台线程（每5秒）
         │
         ▼
    ┌─────────────┐
    │ 读取 WAL    │
    └─────────────┘
         │
         ▼
    ┌─────────────┐
    │ 重试发送    │
    └─────────────┘
         │
    成功? ──Yes──► MarkDone() 删除记录
         │
        No ──────► 保留，下次继续
```

#### <4>阻塞模式
```cpp
// ⚠️ 阻塞模式：队列满时无限等待直到入队成功
// 适用场景：批处理/离线任务，对延迟不敏感但要求消息必须入队
// 风险：如果 Kafka 持续不可用，会永久阻塞调用线程
bool KafkaProducer::SendBlocking(const std::string& topic,
                                  const std::string& key,
                                  const std::string& value) {
    while (true) {
        RdKafka::ErrorCode err = producer_->produce(
            topic,
            RdKafka::Topic::PARTITION_UA,      // 自动分区：按 key 哈希选择
            RdKafka::Producer::RK_MSG_COPY,    // 复制模式：librdkafka 复制数据，调用后可释放原数据
            (void*)value.data(), value.size(),
            key.data(), key.size(),
            0,        // timestamp: 0 = 使用当前时间
            nullptr   // msg_opaque: 回调透传数据，这里不需要
        );

        // 入队成功（注意：仅入队，非真正送达 Kafka）
        if (err == RdKafka::ERR_NO_ERROR) {
            producer_->poll(0);  // 非阻塞触发回调，处理历史投递结果
            return true;
        }

        // 队列满：等待 + 重试（阻塞的核心逻辑）
        if (err == RdKafka::ERR__QUEUE_FULL) {
            // poll(100) 的双重作用：
            // 1. 阻塞等待 100ms，降低 CPU 空转
            // 2. 触发 delivery report 回调，已发送的消息出队，腾出空间
            producer_->poll(100);
            continue;  // 队列满可恢复，继续重试
        }

        // 其他错误（topic 不存在、消息过大等），不可恢复，直接失败
        LOG_ERROR("Kafka produce failed: {}", RdKafka::err2str(err));
        return false;
    }
}

```

# 附录1：配置项详解
## 1.Producer端配置项生效场景
### （1）网络与连接相关（Global通用配置）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116144611415.png)

**触发时机总结表**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119091618544.png)

**生产环境配置建议**
```cpp
// 稳定网络环境
conf->set("socket.timeout.ms", "60000", errstr);
conf->set("socket.connection.setup.timeout.ms", "30000", errstr);
conf->set("socket.keepalive.enable", "true", errstr);
conf->set("connections.max.idle.ms", "600000", errstr);
conf->set("reconnect.backoff.ms", "100", errstr);
conf->set("reconnect.backoff.max.ms", "10000", errstr);

// 不稳定网络/跨机房场景
conf->set("socket.timeout.ms", "30000", errstr);
conf->set("socket.connection.setup.timeout.ms", "50000", errstr);
conf->set("socket.keepalive.enable", "true", errstr);
conf->set("connections.max.idle.ms", "300000", errstr);
conf->set("reconnect.backoff.ms", "50", errstr);
conf->set("reconnect.backoff.max.ms", "5000", errstr);

```

**生效机制（指数退避算法）**
```cpp
第1次重试: 等待 100ms (reconnect.backoff.ms)
第2次重试: 等待 200ms (100 * 2)
第3次重试: 等待 400ms (200 * 2)
第4次重试: 等待 800ms (400 * 2)
   ...
第N次重试: 等待 min(计算值, reconnect.backoff.max.ms)
```

#### <1>socket.connection.setup.timeout.ms触发点
```cpp
// 触发函数: RdKafka::Producer::create() 和内部连接建立
RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
conf->set("socket.connection.setup.timeout.ms", "30000", errstr);

RdKafka::Producer *producer = RdKafka::Producer::create(conf, errstr);
// ↑ 此时触发与 bootstrap.servers 的首次连接
//   内部调用 rd_kafka_broker_connect() → rd_kafka_transport_connect()

// ⚠️ 注意：此超时不仅在 create() 时触发
// 运行期间如果需要连接新 Broker（如 partition leader 变更），也会触发

// 场景举例：
// 1. 初始连接 bootstrap.servers
// 2. metadata 更新发现新 broker
// 3. leader 切换需要连接新 broker
```
**内部触发链路**：
```cpp
Producer::create()
    └→ rd_kafka_new()
        └→ rd_kafka_brokers_add()
            └→ rd_kafka_broker_connect()
                └→ rd_kafka_transport_connect()  ← 超时由此配置控制
                    └→ connect() 系统调用
```

#### <2>socket.timeout.ms触发点
```cpp
// 触发函数: produce() + poll() + flush()

// 1. produce() - 消息入队（不直接触发网络IO，但会唤醒后台线程）
producer->produce(...);

// 2. poll() - 处理回调，驱动网络IO ← 这里会触发！
producer->poll(1000);  // ← socket读写受 socket.timeout.ms 控制

// 3. flush() - 内部循环调用 poll()
producer->flush(10000);  // ← 本质是 while(outq_len > 0) poll()
```
**内部触发路径**
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                     socket.timeout.ms 触发路径                   │
└─────────────────────────────────────────────────────────────────┘

produce()                     poll()                    flush()
    │                           │                          │
    │ 消息入队列                 │                          │
    ▼                           │                          │
┌─────────┐                     │                    ┌─────────┐
│ 队列    │◄────────────────────┼────────────────────│ while() │
└────┬────┘                     │                    │ poll()  │
     │                          ▼                    └────┬────┘
     │                    ┌───────────┐                   │
     │                    │  poll()   │◄──────────────────┘
     │                    └─────┬─────┘
     │                          │
     ▼                          ▼
┌─────────────────────────────────────────┐
│         后台 IO 线程                      │
│   rd_kafka_broker_serve()               │
│         │                               │
│         ├→ rd_kafka_send() ──┐          │
│         │                    │          │
│         └→ rd_kafka_recv() ──┼─→ socket.timeout.ms 生效
│                              │          │
└─────────────────────────────────────────┘
```

#### <3>socket.keepalive.enable触发点
```cpp
// 在 create() 时设置，连接建立后立即生效
conf->set("socket.keepalive.enable", "true", errstr);
RdKafka::Producer *producer = RdKafka::Producer::create(conf, errstr);
// 后续所有连接自动启用 TCP keepalive
```
**内部触发链路**
```cpp
rd_kafka_transport_connect()
    └→ rd_kafka_transport_post_connect_setup()
        └→ setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, ...)  ← 此处生效
```

#### <4>connections.max.idle.ms触发点
```cpp
// 触发场景: 长时间不调用 produce() 或 poll()
producer->produce(...);  // 最后一次发送

// ... 10分钟没有任何操作 ...

producer->produce(...);  // 此时连接已被关闭，需要重建
```
**内部触发链路**
```cpp
rd_kafka_thread_main()  // 后台轮询线程
    └→ rd_kafka_broker_serve()
        └→ rd_kafka_broker_idle_check()  ← 检查空闲时间
            └→ 当前时间 - 最后活动时间 > connections.max.idle.ms
                └→ rd_kafka_broker_fail()  ← 主动断开连接
```

#### <5> reconnect.backoff.ms / reconnect.backoff.max.ms 触发点
```cpp
// 触发场景: Broker 故障或网络断开后自动重连
producer->produce(...);  // 发送时发现连接断开

// 内部自动触发重连，无需用户调用任何函数
// 可通过回调观察重连过程:

class MyEventCb : public RdKafka::EventCb {
public:
    void event_cb(RdKafka::Event &event) override {
        switch (event.type()) {
            case RdKafka::Event::EVENT_ERROR:
                if (event.err() == RdKafka::ERR__TRANSPORT) {
                    // 连接断开，即将触发 reconnect.backoff.ms
                    std::cerr << "Connection lost, reconnecting..." << std::endl;
                }
                break;
            case RdKafka::Event::EVENT_LOG:
                // 可看到重连日志
                break;
        }
    }
};

conf->set("event_cb", &my_event_cb, errstr);
```
**内部触发链路**
```cpp
rd_kafka_broker_fail()  // 连接失败
    └→ rd_kafka_broker_set_state(BROKER_STATE_INIT)
        └→ rd_kafka_broker_reconnect_schedule()
            └→ 计算退避时间 = min(backoff * 2^n, backoff_max)
                └→ 定时器触发 rd_kafka_broker_connect()
```

### （2）消息发送与可靠性配置（Producer专属配置）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119095109007.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119095142663.png)

#### <1>可靠性配置
##### 1）acks
**触发函数**：
```cpp
// 配置阶段
conf->set("acks", "all", errstr);  // 或 "0", "1", "-1"

// 触发阶段: produce() 发送消息后，后台线程等待 Broker 确认
producer->produce(...);

// poll() 中接收确认结果，触发 delivery report 回调
producer->poll(1000);
```

**内部触发链路**
```cpp
produce()
    └→ 消息入队列
        └→ 后台线程 rd_kafka_broker_produce_toppar()
            └→ 发送 ProduceRequest (携带 acks 参数)
                └→ Broker 处理
                    │
        ┌───────────┴───────────────────────────────────┐
        │                                               │
        ▼ acks=0                 ▼ acks=1              ▼ acks=all
   不等待响应              等待 Leader 确认        等待 ISR 全部确认
        │                       │                      │
        └───────────────────────┴──────────────────────┘
                                │
                                ▼
                    poll() → delivery report 回调
```

**三种模式对比**
```
┌─────────┬────────────────────┬──────────────┬──────────────┐
│  acks   │      确认机制       │    可靠性    │     性能     │
├─────────┼────────────────────┼──────────────┼──────────────┤
│    0    │ 不等待任何确认      │    最低      │    最高      │
│    1    │ Leader 写入即确认   │    中等      │    中等      │
│ all/-1  │ ISR 全部写入确认    │    最高      │    最低      │
└─────────┴────────────────────┴──────────────┴──────────────┘
```

##### 2）enable.idempotence
**触发函数**
```cpp
// 配置阶段
conf->set("enable.idempotence", "true", errstr);

// 触发阶段1: Producer::create() 时初始化 PID
RdKafka::Producer *producer = RdKafka::Producer::create(conf, errstr);
// ↑ 内部向 Broker 请求 Producer ID (PID) 和 Epoch

// 触发阶段2: produce() 时自动附加序列号
producer->produce(...);
// ↑ 每条消息自动携带 (PID, Epoch, SequenceNumber)
```

**内部触发链路**
```cpp
Producer::create()
    └→ rd_kafka_new()
        └→ rd_kafka_idemp_init()
            └→ 向 Broker 发送 InitProducerId 请求
                └→ 获取 PID + Epoch
                    └→ 初始化 SequenceNumber = 0

produce()
    └→ rd_kafka_produce()
        └→ 消息附加 (PID, Epoch, SeqNum++)
            └→ 后台线程发送到 Broker
                └→ Broker 检测
                    │
        ┌───────────┴───────────────────────┐
        │                                   │
        ▼ SeqNum 连续                       ▼ SeqNum 重复/乱序
      正常写入                          丢弃/返回错误
        │                                   │
        └───────────────────────────────────┘
                        │
                        ▼
                poll() → delivery report
```

**幂等性保障机制**
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                     幂等性工作原理                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer                         Broker                        │
│     │                                │                          │
│     │─── msg(PID=1, Seq=0) ─────────→│ 写入成功                 │
│     │                                │                          │
│     │─── msg(PID=1, Seq=1) ─────────→│ 写入成功                 │
│     │                                │                          │
│     │─── msg(PID=1, Seq=1) ─────────→│ 检测重复，丢弃但返回成功  │
│     │    (网络重试导致)              │                          │
│     │                                │                          │
│     │─── msg(PID=1, Seq=3) ─────────→│ 检测乱序，返回错误       │
│     │    (Seq=2 丢失)                │                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

##### 3）max.in.flight.requests.per.connection
**触发函数**
```cpp
// 配置阶段
conf->set("max.in.flight.requests.per.connection", "5", errstr);

// 触发阶段: produce() 后台线程发送前检查
producer->produce(...);
// ↑ 后台线程检查当前未确认请求数

// poll() 收到确认后释放槽位
producer->poll(1000);
```

**内部触发链路**
```cpp
produce() → 消息入队列
              │
              ▼
    后台线程准备发送
              │
              ▼
    检查 in_flight_count < max_in_flight (5)?
              │
    ┌─────────┴─────────┐
    │                   │
    ▼ 是                ▼ 否
 发送请求            暂不发送，等待确认
 in_flight++              │
    │                     │
    └─────────────────────┘
              │
              ▼
    poll() → 收到 Broker 确认
              │
              ▼
    in_flight_count--
              │
              ▼
    触发 delivery report 回调
```

**与幂等性的关系**
```cpp
┌─────────────────────────────────────────────────────────────────┐
│              max.in.flight 与幂等性的约束关系                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  enable.idempotence = false:                                    │
│      max.in.flight 可设任意值                                    │
│      但值过大可能导致重试时消息乱序                               │
│                                                                 │
│  enable.idempotence = true:                                     │
│      max.in.flight 自动限制 ≤ 5                                  │
│      即使配置更大值也会被覆盖                                     │
│      原因: Broker 端只缓存最近5个序列号用于去重                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### <2>重试配置
##### 1）retries
**触发函数**
```cpp
// 配置阶段
conf->set("retries", "5", errstr);

// 触发阶段: 发送失败后自动重试（无需用户调用）
producer->produce(...);

// poll() 中处理重试结果
producer->poll(1000);
```

**内部触发链路**
```cpp
produce()
    └→ 消息入队列
        └→ 后台线程发送
            └→ Broker 返回错误
                │
                ▼
            rd_kafka_handle_produce_err()
                │
                ▼
            错误类型判断
                │
    ┌───────────┴───────────────┐
    │                           │
    ▼ 可重试错误                ▼ 不可重试错误
检查 retries_remaining         直接触发 delivery report (失败)
    │
    ├→ > 0: retries_remaining--
    │       等待 retry.backoff.ms
    │       重新发送
    │
    └→ = 0: 触发 delivery report (失败)
```

**可重试 vs 不可重试错误**
```
┌─────────────────────────────────────┬─────────────────────────────────────┐
│           ✅ 可重试错误              │           ❌ 不可重试错误            │
├─────────────────────────────────────┼─────────────────────────────────────┤
│ • REQUEST_TIMED_OUT                 │ • MSG_SIZE_TOO_LARGE                │
│ • NOT_LEADER_FOR_PARTITION          │ • TOPIC_AUTHORIZATION_FAILED        │
│ • LEADER_NOT_AVAILABLE              │ • CLUSTER_AUTHORIZATION_FAILED      │
│ • NOT_ENOUGH_REPLICAS               │ • UNSUPPORTED_FOR_MESSAGE_FORMAT    │
│ • NOT_ENOUGH_REPLICAS_AFTER_APPEND  │ • INVALID_REQUIRED_ACKS             │
│ • NETWORK_EXCEPTION                 │ • UNKNOWN_TOPIC_OR_PARTITION        │
│ • BROKER_NOT_AVAILABLE              │   (且 allow.auto.create.topics=false)│
└─────────────────────────────────────┴─────────────────────────────────────┘
```

##### 2）retry.backoff.ms / retry.backoff.max.ms
**触发函数**
```cpp
// 配置阶段
conf->set("retry.backoff.ms", "200", errstr);
conf->set("retry.backoff.max.ms", "1000", errstr);

// 触发阶段: 发送失败后自动计算退避时间
producer->produce(...);
// ↑ 失败后自动触发退避重试，无需用户调用
```

**内部触发链路（指数退避算法）**
```cpp
发送失败（可重试错误）
    │
    ▼
计算退避时间 = min(retry.backoff.ms * 2^(N-1), retry.backoff.max.ms)
    │
    │   N=1: min(200 * 1, 1000) = 200ms
    │   N=2: min(200 * 2, 1000) = 400ms
    │   N=3: min(200 * 4, 1000) = 800ms
    │   N=4: min(200 * 8, 1000) = 1000ms  ← 达到上限
    │   N=5: min(200 * 16, 1000) = 1000ms ← 保持上限
    │
    ▼
等待退避时间
    │
    ▼
重新发送
```

**退避时间示意图**
```cpp
retry.backoff.ms = 200, retry.backoff.max.ms = 1000

时间线:
0      200ms    600ms   1400ms   2400ms   3400ms
│       │        │        │        │        │
▼       ▼        ▼        ▼        ▼        ▼
发送   重试1    重试2    重试3    重试4    重试5
失败    │        │        │        │        │
│       │        │        │        │        │
└──200ms┘        │        │        │        │
        └──400ms─┘        │        │        │
                 └──800ms─┘        │        │
                          └─1000ms─┘        │
                                   └─1000ms─┘
```

#### <3>超时配置
##### 1）delivery.timeout.ms
**触发函数**
```cpp
// 配置阶段
conf->set("delivery.timeout.ms", "60000", errstr);

// 触发阶段: produce() 开始计时
producer->produce(...);
// ↑ 从此刻开始计时，包含排队、发送、重试全过程

// poll() 中检测超时或收到结果
producer->poll(1000);
```

**内部触发链路**
```cpp
produce() 调用，记录 timestamp_enqueue
    │
    ▼
消息入队列等待发送
    │
    ▼
后台线程定期检查
    │
    ▼
now() - timestamp_enqueue > delivery.timeout.ms ?
    │
    ├→ 否: 继续等待/发送/重试
    │
    └→ 是: 触发 delivery report (ERR__MSG_TIMED_OUT)
            消息从队列移除
```

##### 2）request.timeout.ms
**触发函数**
```cpp
// 配置阶段
conf->set("request.timeout.ms", "30000", errstr);

// 触发阶段: 后台线程发送单次请求时
producer->produce(...);
// ↑ 后台线程发送 ProduceRequest 时开始计时

// poll() 中处理超时
producer->poll(1000);
```

**内部触发链路**
```cpp
后台线程发送 ProduceRequest
    │
    ▼
记录请求发送时间
    │
    ▼
等待 Broker 响应
    │
    ├→ 在 30s 内收到响应: 处理响应
    │
    └→ 超过 30s 未响应: 
            │
            ▼
        触发 REQUEST_TIMED_OUT
            │
            ▼
        检查 retries > 0 ?
            │
            ├→ 是: 进入重试流程
            │
            └→ 否: delivery report (失败)
```

##### 3）message.timeout.ms
**触发函数**
```cpp
// 配置阶段
conf->set("message.timeout.ms", "60000", errstr);

// 触发阶段: 消息在本地队列中等待时
producer->produce(...);
// ↑ 消息入队后开始计时
```

**与 delivery.timeout.ms 的区别**
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                    三种超时的作用范围                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  produce() ─→ [本地队列等待] ─→ [发送] ─→ [等待确认] ─→ delivery report  │
│     │              │              │           │              │          │
│     │◄─────────────┼──────────────┼───────────┼──────────────┤          │
│     │     message.timeout.ms (消息存活时间)                   │          │
│     │              │              │           │              │          │
│     │◄─────────────┼──────────────┼───────────┼──────────────┤          │
│     │        delivery.timeout.ms (投递总超时，含重试)         │          │
│     │              │              │           │              │          │
│     │              │              │◄──────────┤              │          │
│     │              │              │ request.  │              │          │
│     │              │              │ timeout.ms│              │          │
│     │              │              │ (单次请求)│              │          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

建议: message.timeout.ms >= delivery.timeout.ms
      delivery.timeout.ms >= request.timeout.ms
```

#### <4>队列配置
##### 1）queue.buffering.max.messages
**触发函数**
```cpp
// 配置阶段
conf->set("queue.buffering.max.messages", "500000", errstr);

// 触发阶段: produce() 入队前检查
RdKafka::ErrorCode err = producer->produce(...);
// ↑ 入队前检查当前消息数

if (err == RdKafka::ERR__QUEUE_FULL) {
    // 队列已满
    producer->poll(100);  // 等待释放空间
    // 重试 produce
}
```

**内部触发链路**
```cpp
producer->produce(...)
    │
    ▼
rd_kafka_produce()
    │
    ▼
rd_kafka_curr_msgs_cnt() >= queue.buffering.max.messages ?
    │
    ├→ 否: 消息入队成功
    │       rd_kafka_curr_msgs_cnt()++
    │
    └→ 是: 返回 ERR__QUEUE_FULL
            消息不入队
```

##### 2）queue.buffering.max.kbytes
**触发函数**
```cpp
// 配置阶段
conf->set("queue.buffering.max.kbytes", "524288", errstr);  // 512MB

// 触发阶段: produce() 入队前检查
RdKafka::ErrorCode err = producer->produce(...);
// ↑ 入队前检查当前内存占用
```

**内部触发链路**
```cpp
producer->produce(payload, len, ...)
    │
    ▼
rd_kafka_produce()
    │
    ▼
rd_kafka_curr_msgs_size() + len > queue.buffering.max.kbytes * 1024 ?
    │
    ├→ 否: 消息入队成功
    │       rd_kafka_curr_msgs_size() += len
    │
    └→ 是: 返回 ERR__QUEUE_FULL
```

**双重队列限制**
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                   队列容量双重检查                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  produce() 入队检查:                                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  条件1: 消息数 < queue.buffering.max.messages (500000)  │    │
│  │                       AND                               │    │
│  │  条件2: 内存 < queue.buffering.max.kbytes (512MB)       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                      │
│              ┌───────────┴───────────┐                          │
│              │                       │                          │
│              ▼ 均满足                ▼ 任一不满足                │
│         入队成功                 返回 ERR__QUEUE_FULL           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### <5>批量配置
##### 1）batch.num.messages
**触发函数**
```cpp
// 配置阶段
conf->set("batch.num.messages", "10000", errstr);

// 触发阶段: produce() 后，后台线程检查批次消息数
producer->produce(...);  // 多次调用
// ↑ 累计消息数达到 10000 时触发批量发送
```

##### 2）batch.size
**触发函数**
```cpp
// 配置阶段
conf->set("batch.size", "1000000", errstr);

// 触发阶段: produce() 后，后台线程检查批次字节数
producer->produce(...);  // 多次调用
// ↑ 累计字节数达到 1MB 时触发批量发送
```

##### 3）queue.buffering.max.ms
**触发函数**
```cpp
// 配置阶段
conf->set("queue.buffering.max.ms", "5", errstr);

// 触发阶段: produce() 后，后台定时器检查
producer->produce(...);
// ↑ 首条消息入队后 5ms 内触发批量发送

// poll() 驱动后台线程执行
producer->poll(0);
```

**批量发送触发条件（三选一）**
```cpp
produce() → produce() → produce() → ...
    │           │           │
    ▼           ▼           ▼
┌────────────────────────────────────────┐
│          本地消息队列 (per partition)   │
│  [msg1] [msg2] [msg3] [msg4] ...       │
└────────────────────┬───────────────────┘
                     │
    ┌────────────────┼────────────────┬─────────────┐
    │                │                │             │
    ▼                ▼                ▼             ▼
 消息数>=10000    字节数>=1MB     时间>=5ms      flush()调用
    │                │                │             │
    └────────────────┴────────────────┴─────────────┘
                     │
                     ▼ 任一条件满足
            ┌────────────────────┐
            │  rd_kafka_produce  │
            │  _batch()          │
            └─────────┬──────────┘
                      │
                      ▼
            ┌────────────────────┐
            │ 压缩 (如有配置)     │
            └─────────┬──────────┘
                      │
                      ▼
            ┌────────────────────┐
            │ 发送到 Broker      │
            └────────────────────┘
```

**批量效果示意**
```cpp
queue.buffering.max.ms = 5ms
batch.size = 1MB
batch.num.messages = 10000

场景1: 时间触发
0ms     1ms     2ms     3ms     4ms     5ms
 │       │       │       │       │       │
 ▼       ▼       ▼       ▼       ▼       ▼
[msg1] [msg2] [msg3]                  ──→ 触发发送（5ms到期）
                                           3条消息组成批次

场景2: 消息数触发
0ms      ...     Xms
 │               │
 ▼               ▼
[msg1]...[msg10000] ──→ 触发发送（达到10000条）
                        无需等待5ms

场景3: 字节数触发
0ms          Xms
 │            │
 ▼            ▼
[大消息1][大消息2] ──→ 触发发送（达到1MB）
                       2条消息组成批次
```

#### <6>压缩配置
##### 1）compression.codec
**触发函数**
```cpp
// 配置阶段
conf->set("compression.codec", "lz4", errstr);

// 触发阶段: 后台线程批量发送前压缩
producer->produce(...);  // 消息入队（原始数据）
// ↑ 后台线程收集批次后触发压缩
```

**内部触发链路**
```cpp
produce() → 消息入队列（原始数据）
              │
              ▼
    后台线程收集批次（满足发送条件）
              │
              ▼
    rd_kafka_msgset_writer_write()
              │
              ▼
    rd_kafka_compress(codec, level, batch_data)
              │
    ┌─────────┴─────────────────────────────────────┐
    │           │           │           │           │
    ▼ none      ▼ lz4       ▼ zstd      ▼ gzip      ▼ snappy
  不压缩     LZ4压缩     ZSTD压缩    GZIP压缩   Snappy压缩
    │           │           │           │           │
    └─────────┬─────────────────────────────────────┘
              │
              ▼
    发送压缩后的数据到 Broker
```

**压缩算法对比**
```cpp
┌──────────┬─────────────┬─────────────┬─────────────┬──────────────┐
│   算法   │   压缩比    │  压缩速度   │  解压速度   │   适用场景   │
├──────────┼─────────────┼─────────────┼─────────────┼──────────────┤
│   none   │     1:1     │     -       │     -       │ 极低延迟     │
│   lz4    │    ~2.5:1   │    最快     │    最快     │ 生产首选     │
│   zstd   │    ~3.5:1   │    较快     │    较快     │ 大消息/带宽敏感│
│   gzip   │    ~3:1     │    较慢     │    较慢     │ 兼容旧系统   │
│  snappy  │    ~2:1     │    快       │    快       │ 兼容旧系统   │
└──────────┴─────────────┴─────────────┴─────────────┴──────────────┘
```

##### 2）compression.level
**触发函数**
```cpp
// 配置阶段
conf->set("compression.level", "-1", errstr);  // 使用算法默认级别

// 触发阶段: 与 compression.codec 同时生效
producer->produce(...);
```

**各压缩算法级别范围**
```cpp
┌──────────┬─────────────┬─────────────┬─────────────────────────────┐
│   算法   │  级别范围   │  默认级别   │           说明              │
├──────────┼─────────────┼─────────────┼─────────────────────────────┤
│   lz4    │   1 ~ 12    │     1       │ 级别越高压缩比越好，速度越慢 │
│   zstd   │   1 ~ 22    │     3       │ 推荐 1~5，更高级别收益递减   │
│   gzip   │   1 ~ 9     │     6       │ 9=最佳压缩，1=最快          │
│  snappy  │     -       │     -       │ 无可调级别                  │
└──────────┴─────────────┴─────────────┴─────────────────────────────┘

-1 = 使用算法默认级别（推荐）
```

#### <7>消息配置
##### 1）message.max.bytes
**触发函数**
```cpp
// 配置阶段
conf->set("message.max.bytes", "1000000", errstr);

// 触发阶段: produce() 入队前检查消息大小
RdKafka::ErrorCode err = producer->produce(
    topic, partition, flags,
    payload, len,  // ← len 在此处被检查
    key, key_len,
    timestamp, headers, opaque
);

// 消息过大时立即返回错误
if (err == RdKafka::ERR_MSG_SIZE_TOO_LARGE) {
    // 消息超过 1MB
}
```

**内部触发链路**
```cpp
producer->produce(payload, len, ...)
    │
    ▼
rd_kafka_produce()
    │
    ▼
len > message.max.bytes ?
    │
    ├→ 否: 继续处理，消息入队
    │
    └→ 是: 立即返回 ERR_MSG_SIZE_TOO_LARGE
            消息不入队
            不触发 delivery report
            不计入 retries
```

**与 Broker 端配置的关系**
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                    消息大小配置链路                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Producer                    Broker                     Consumer        │
│      │                         │                            │           │
│  message.max.bytes         message.max.bytes           fetch.max.bytes  │
│  (客户端配置)              (server.properties)          (客户端配置)    │
│      │                         │                            │           │
│      └─────────────────────────┴────────────────────────────┘           │
│                                │                                        │
│                    三者需保持一致或:                                     │
│                    Broker >= Producer                                   │
│                    Consumer >= Broker                                   │
│                                                                         │
│  否则可能出现:                                                          │
│  • Producer 发送成功但 Broker 拒绝                                      │
│  • Broker 存储成功但 Consumer 无法消费                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### <8>生产环境配置建议
##### 1）核心业务（金融/交易）
```cpp
// 最高可靠性，允许牺牲部分性能
conf->set("acks", "all", errstr);
conf->set("enable.idempotence", "true", errstr);
conf->set("max.in.flight.requests.per.connection", "5", errstr);
conf->set("retries", "5", errstr);
conf->set("retry.backoff.ms", "200", errstr);
conf->set("delivery.timeout.ms", "60000", errstr);
conf->set("compression.codec", "lz4", errstr);
conf->set("batch.size", "32768", errstr);         // 32KB，降低延迟
conf->set("queue.buffering.max.ms", "5", errstr);

```

##### 2）日志采集（高吞吐）
```cpp
// 高吞吐，可接受少量丢失
conf->set("acks", "1", errstr);                    // 仅 Leader 确认
conf->set("enable.idempotence", "false", errstr);
conf->set("retries", "3", errstr);
conf->set("compression.codec", "zstd", errstr);   // 最高压缩比
conf->set("batch.size", "1048576", errstr);       // 1MB 大批次
conf->set("batch.num.messages", "50000", errstr);
conf->set("queue.buffering.max.ms", "50", errstr);// 等待更多消息
conf->set("queue.buffering.max.messages", "1000000", errstr);
```

##### 3）监控指标（低延迟）
```cpp
// 最低延迟，实时性优先
conf->set("acks", "1", errstr);
conf->set("enable.idempotence", "false", errstr);
conf->set("retries", "0", errstr);                // 不重试
conf->set("compression.codec", "none", errstr);   // 不压缩
conf->set("batch.size", "16384", errstr);         // 16KB 小批次
conf->set("batch.num.messages", "100", errstr);
conf->set("queue.buffering.max.ms", "0", errstr); // 立即发送
```