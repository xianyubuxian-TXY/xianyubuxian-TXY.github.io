---
title: kafka使用教程
date: 2026-01-15 15:56:39
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- Kafka
---

# 一、Kafka介绍
Kafka 是一款分布式、高吞吐量、高可靠性的**分布式事件流平台**（原定位为：**分布式消息队列**），基于**发布/订阅模式**，主要用于处理实时数据管道、流处理、数据集成等场景，能够高效地收集、存储和分发大规模的实时数据流。
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115205205304.png)

## 1.应用场景
- **消息队列**：替代传统消息队列（如 RabbitMQ），实现服务间的异步通信、解耦和削峰填谷（如秒杀场景的流量缓冲）。
- **实时数据管道**：在分布式系统间构建数据传输通道，实现不同系统（如数据库、缓存、业务系统）之间的实时数据同步。
- **日志收集**：集中收集分布式系统的日志数据（如 ELK 架构中的日志传输环节），便于日志分析和故障排查。
- **流处理**：作为流处理框架（如 Flink、Spark Streaming）的数据源和数据Sink，支撑实时计算场景（如实时监控、实时报表、风控预警）。
- **事件溯源**：记录系统中的关键事件，支持业务状态回溯和历史数据审计。

## 2.kafka作为“消息队列”
### （1）为什么需要kafka，它的作用是什么？
下面我将以“**用户注册场景**”为例，简要讲解为什么要引入kafka作为消息队列，它作为工作流程的哪一个环节，如何使用。

#### <1>工作流程图对比
- **1.没有使用Kafka或任何任务队列的传统同步处理流程图**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115161939911.png)

- **2.使用kafka的异步处理流程图**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115161159031.png)

**看到这，敏锐的人可能想到：“不使用kafka，用多线程异步处理不是也可以吗？” 是的，也是可以的，但使用kafka有很多优势，这个之后在介绍。**

#### <2>完整的流程对比（场景：用户注册）
##### 1）前景知识介绍
- **用户注册的“核心业务”**：
	- 向数据库中插入新注册的用户信息（这里其实就“注册成功了！”）
- **用户注册的“附加业务”**：
	- 发送邮件到注册用户的邮箱提示注册成功
	- 注册成功后，返回后台数据用于渲染前端页面（如：现实中你注册成功时，显示“3s 跳转页面”，就是前端在等待后端数据进行渲染）
	- 给你一些“信息提示”（如：在购物平台，提示你有优惠券可领）
	- ...
- 不同平台/软件用户注册的“附加业务”各有不同，单“核心业务”都是在“向数据库插入新注册用户的信息”，插入成功其实就代表“注册成功”了，“附加业务”即使没有也可以。


##### 2)没有Kafka（传统同步）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115163026489.png)
**问题**：
- ❌ 用户等待 3+ 秒才看到响应（延迟高）
- ❌ 任一个环节出现问题（如：邮件服务挂了），注册失败（可靠性较低）
- ❌ 高并发时，线程全部阻塞“附加业务”上，“附加业务”是主要的“性能瓶颈”

**再次提示**：这里虽然也可以将“附加业务”再次用异步线程处理，但我们先不讨论它。

##### 3)有 Kafka（异步解耦）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115164615280.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115164736803.png)

<a id="P1"></a>
#### <3>后台服务器、kafka、后台任务服务 三者的关系
- **类比进程池**：
	- **后台服务器** ——> **master进程**
	- **kafka**	——> **任务队列**
	- **后台任务服务** ——> **worker工作进程**
- 客户端发送请求到达“后台服务器”（类比master进程）时，后台服务器完成“核心业务”（如：将新用户数据插入数据库），并将“附加业务”请求（高耗时）发送到kafka消息队列（类比任务队列）中，然后立即向前端回复执行结果。“后台任务服务”（类比worker进程）则不断从kafka的消息队列中取出“附加业务”请求进行处理
	- **解耦了“核心业务”与“附加业务”，从而使“后台服务器”可以快速响应前端请求**

#### <4>“kafka消息队列” vs “进程池/线程池”
从前面可以知道，使用kafka的模式与Reactor模式很像，但kafka模式有很多优点：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115204712186.png)

# 二、kafka 核心概念
**推荐阅读《Apache Kafka实战》了解详情**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115213709054.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115213729718.png)

**message格式**：
```plaintext
┌───────────────────────────────────────────────────────────────────────────┐
│                            消息头部 (message header)                      │
├────────┬──────┬──────┬──────────┬──────────┬─────────┬──────────┬─────────┤
│ CRC    │版本号 │ 属性 │ 时间戳   │ Key 长度 │ Key     │ Value 长度│ Value   │
│ 4B     │ 1B   │ 1B   │ 8B       │ 4B       │ k bytes │ 4B       │ v bytes │
└────────┴──────┴──────┴──────────┴──────────┴─────────┴──────────┴─────────┘
```
- **key**：消息键，在对消息进行partition时使用 ——>决定消息被存储在 该 Topic 的哪个 Partition。
- **Value**：消息体，消息的实际数据。
- **Timestamp**：消息发送时间戳。
- 每个消息由 **<topic,partition,offset>三元组 唯一标识**


# 三、librdkafka的使用（C++ 2.13.0）


## 1.Producer 与 Consumer 使用概述
### （1）Producer、kafka服务器、Consumer三者的关系
- **Producer**（消息生产者）：
    - **后台服务器** 连接 kafka服务器的句柄 ——>根据 “topic,partition” 向kafka服务器中写入消息
- **Consumer**（消息消费者）：
    - **后台任务服务** 连接 kafka服务器的句柄 ——> 根据 “<topic,partition,offset>” 从kafka中消费消息
- 无论是**Producer**还是**Consumer**，都是**kafka的客户端**，kafka的服务器是两者的中介

**如果对“后台服务器”、“后台任务服务”不了解**，参考[后台服务器、kafka、后台任务服务 三者的关系](#P1)
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                          Kafka 架构角色                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────┐                         ┌─────────────────┐      │
│   │    Producer     │                         │    Consumer     │      │
│   │   (消息生产者)   │                         │   (消息消费者)   │      │
│   │                 │                         │                 │      │
│   │  后台服务器     │                         │  后台任务服务    │      │
│   │  连接 Kafka 句柄 │                         │  连接 Kafka 句柄 │      │
│   └────────┬────────┘                         └────────▲────────┘      │
│            │                                           │               │
│            │  写入消息                        消费消息  │               │
│            │  <topic, partition>    <topic, partition, offset>         │
│            │                                           │               │
│            ▼                                           │               │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                      Kafka Server                           │      │
│   │                       (中介/Broker)                          │      │
│   │                                                             │      │
│   │    ┌─────────┐    ┌─────────┐    ┌─────────┐               │      │
│   │    │ Topic A │    │ Topic B │    │ Topic C │    ...        │      │
│   │    │ ┌─────┐ │    │ ┌─────┐ │    │ ┌─────┐ │               │      │
│   │    │ │ P0  │ │    │ │ P0  │ │    │ │ P0  │ │               │      │
│   │    │ │ P1  │ │    │ │ P1  │ │    │ │ P1  │ │               │      │
│   │    │ │ P2  │ │    │ │ P2  │ │    │ │ P2  │ │               │      │
│   │    │ └─────┘ │    │ └─────┘ │    │ └─────┘ │               │      │
│   │    └─────────┘    └─────────┘    └─────────┘               │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
│   ※ Producer 和 Consumer 都是 Kafka 的客户端                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
```cpp
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Producer   │  ────►  │    Kafka     │  ────►  │   Consumer   │
│  (后台服务器) │  写入    │   Server     │  消费    │ (后台任务服务) │
└──────────────┘         │   (Broker)   │         └──────────────┘
                         └──────────────┘
                         
写入定位：topic + partition
消费定位：topic + partition + offset
```

### （2）类层次总览
```cpp
┌─────────────────────────────────────────────────────────────────────────────┐
│                        librdkafka C++ 类层次结构                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         基础配置层                                   │   │
│  │   ┌────────┐    ┌────────────────┐    ┌──────────────────────┐     │   │
│  │   │  Conf  │    │  TopicPartition │    │      Headers        │     │   │
│  │   └────────┘    └────────────────┘    └──────────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          核心句柄层                                  │   │
│  │                      ┌──────────────┐                               │   │
│  │                      │    Handle    │ (抽象基类)                    │   │
│  │                      └──────┬───────┘                               │   │
│  │               ┌─────────────┴─────────────┐                         │   │
│  │               ▼                           ▼                         │   │
│  │        ┌──────────────┐           ┌───────────────┐                 │   │
│  │        │   Producer   │           │ KafkaConsumer │                 │   │
│  │        └──────────────┘           └───────────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          消息数据层                                  │   │
│  │   ┌──────────┐    ┌──────────────────┐    ┌────────────────────┐   │   │
│  │   │ Message  │    │ MessageTimestamp │    │       Error        │   │   │
│  │   └──────────┘    └──────────────────┘    └────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          回调接口层                                  │   │
│  │  ┌─────────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │   │
│  │  │DeliveryReportCb │ │ RebalanceCb │ │OffsetCommit │ │  EventCb  │ │   │
│  │  │                 │ │             │ │     Cb      │ │           │ │   │
│  │  └─────────────────┘ └─────────────┘ └─────────────┘ └───────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          高级功能层                                  │   │
│  │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────────────┐  │   │
│  │   │ Metadata │    │  Queue   │    │  Topic   │    │ AdminClient │  │   │
│  │   └──────────┘    └──────────┘    └──────────┘    └─────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### （3）Producer的使用流程图
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                         初始化阶段                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────────┐    ┌───────────────────┐    ┌────────────────┐ │
│   │    创建 Conf      │───→│    设置参数       │───→│   注册回调     │ │
│   │   (全局配置)      │    │  bootstrap.servers│    │                │ │
│   │                   │    │  acks, retries 等 │    │ DeliveryReport │ │
│   │ ┌───────────────┐ │    └───────────────────┘    │      Cb        │ │
│   │ │  Conf (GLOBAL)│ │                             │    EventCb     │ │
│   │ └───────────────┘ │                             └───────┬────────┘ │
│   └───────────────────┘                                     │          │
│                                                             ▼          │
│                                                 ┌───────────────────┐  │
│                                                 │  创建 Producer    │  │
│                                                 │ Producer::create()│  │
│                                                 │                   │  │
│                                                 │ ┌───────────────┐ │  │
│                                                 │ │   Producer    │ │  │
│                                                 │ └───────────────┘ │  │
│                                                 └─────────┬─────────┘  │
└───────────────────────────────────────────────────────────┼─────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         生产阶段（循环）                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────────┐    ┌───────────────────┐    ┌────────────────┐ │
│   │    构造消息       │───→│    produce()      │───→│    poll()      │ │
│   │  topic/key/value  │    │   放入内部队列    │    │   触发回调     │ │
│   │                   │    │                   │    │   处理事件     │ │
│   │ ┌───────────────┐ │    │ ┌───────────────┐ │    └───────┬────────┘ │
│   │ │    Topic      │ │    │ │   Producer    │ │            │        ◄─┐
│   │ │    Headers    │ │    │ │   ::produce() │ │            │          │
│   │ └───────────────┘ │    │ └───────────────┘ │            │          │
│   └───────────────────┘    └───────────────────┘            │          │
│                                                             │          │
│            ┌────────────────────────────────────────────────┘          │
│            │                                                           │
│            │    ┌─────────────────────────────────────────┐            │
│            │    │  回调触发时涉及的类：                    │            │
│            │    │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │            │
│            │    │  │ Message │  │  Event  │  │ErrorCode│  │            │
│            │    │  └─────────┘  └─────────┘  └─────────┘  │            │
│            │    └─────────────────────────────────────────┘            │
│            │                                                           │
│            └───────────────────────────────────────────────────────────┘
│                                            (持续生产则循环)              │
└───────────────────────────────────────────────────────────┬─────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         清理阶段                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────────┐         ┌───────────────────┐                  │
│   │     flush()       │────────→│  delete producer  │                  │
│   │   等待队列清空    │         │                   │                  │
│   │                   │         │ ┌───────────────┐ │                  │
│   │ ┌───────────────┐ │         │ │   Producer    │ │                  │
│   │ │   Producer    │ │         │ │   (销毁)      │ │                  │
│   │ │   ::flush()   │ │         │ └───────────────┘ │                  │
│   │ └───────────────┘ │         └───────────────────┘                  │
│   └───────────────────┘                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

```

### （4）Consumer的使用流程图
```cpp
┌─────────────────────────────────────────────────────────────────────────┐
│                         初始化阶段                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────────┐    ┌───────────────────┐    ┌────────────────┐ │
│   │    创建 Conf      │───→│    设置参数       │───→│   注册回调     │ │
│   │   (全局配置)      │    │  bootstrap.servers│    │                │ │
│   │                   │    │  group.id (必需)  │    │  RebalanceCb   │ │
│   │ ┌───────────────┐ │    │  auto.offset.reset│    │OffsetCommitCb  │ │
│   │ │  Conf (GLOBAL)│ │    │  enable.auto.     │    │    EventCb     │ │
│   │ └───────────────┘ │    │      commit 等    │    └───────┬────────┘ │
│   └───────────────────┘    └───────────────────┘            │          │
│                                                             ▼          │
│                                                 ┌───────────────────┐  │
│                                                 │ 创建 KafkaConsumer│  │
│                                                 │ KafkaConsumer::   │  │
│                                                 │    create()       │  │
│                                                 │ ┌───────────────┐ │  │
│                                                 │ │ KafkaConsumer │ │  │
│                                                 │ └───────────────┘ │  │
│                                                 └─────────┬─────────┘  │
└───────────────────────────────────────────────────────────┼─────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         订阅阶段                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                       两种订阅方式（二选一）                      |  |
    |               用于确定从哪个（topic，partition）获取消息           │  │
│   ├─────────────────────────────────┬───────────────────────────────┤  │
│   │                                 │                               │  │
│   │   ┌───────────────────────┐     │     ┌───────────────────────┐ │  │
│   │   │    subscribe()        │     │     │    assign()           │ │  │
│   │   │    订阅主题列表       │     │     │    手动分配分区       │ │  │
│   │   │                       │     │     │                       │ │  │
│   │   │ ┌───────────────────┐ │     │     │ ┌───────────────────┐ │ │  │
│   │   │ │ vector<string>    │ │     │     │ │ TopicPartition    │ │ │  │
│   │   │ │ {"topic1","topic2"}│ │     │     │ │ (topic, partition)│ │ │  │
│   │   │ └───────────────────┘ │     │     │ └───────────────────┘ │ │  │
│   │   │                       │     │     │                       │ │  │
│   │   │ ✓ 自动负载均衡       │     │     │ ✗ 无自动负载均衡     │ │  │
│   │   │ ✓ 触发 RebalanceCb   │     │     │ ✗ 不触发 RebalanceCb │ │  │
│   │   └───────────────────────┘     │     └───────────────────────┘ │  │
│   │                                 │                               │  │
│   └─────────────────────────────────┴───────────────────────────────┘  │
│                                                                         │
└───────────────────────────────────────────────────────────┬─────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         消费阶段（循环）                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────────┐    ┌───────────────────┐    ┌────────────────┐ │
│   │    consume()      │───→│    处理消息       │───→│   提交 offset  │ │
│   │   拉取消息(阻塞)  │    │   业务逻辑处理    │    │  (可选手动)    │ │
│   │                   │    │                   │    │ 用于确定下次读取的位置│ │
│   │ ┌───────────────┐ │    │ ┌───────────────┐ │    │ ┌────────────┐ │ │
│   │ │ KafkaConsumer │ │    │ │    Message    │ │    │ │commitSync()│ │ │
│   │ │  ::consume()  │ │    │ │  (返回消息)   │ │    │ │   或       │ │ │
│   │ └───────────────┘ │    │ └───────────────┘ │    │ │commitAsync│ │ │
│   └───────────────────┘    └───────────────────┘    │ └────────────┘ │ │
│            ▲                                        └───────┬────────┘ │
│            │                                                │          │
│            │         ┌──────────────────────────────────────┘          │
│            │         │                                                 │
│            │         ▼                                                 │
│            │    ┌─────────────────────────────────────────────────┐   │
│            │    │  消费过程中涉及的类：                             │   │
│            │    │  ┌─────────┐  ┌───────────────┐  ┌───────────┐  │   │
│            │    │  │ Message │  │TopicPartition │  │  Headers  │  │   │
│            │    │  │ payload │  │ offset 管理   │  │ (消息头)  │  │   │
│            │    │  │ key     │  └───────────────┘  └───────────┘  │   │
│            │    │  │ offset  │                                    │   │
│            │    │  └─────────┘                                    │   │
│            │    └─────────────────────────────────────────────────┘   │
│            │                                                           │
│            └───────────────────────────────────────────────────────────┘
│                                            (持续消费则循环)              │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                    重平衡触发时                                   │  │
│   │                                                                  │  │
│   │   消费者加入/离开组 ───→ RebalanceCb 触发 ───→ 重新分配分区     │  │
│   │                                                                  │  │
│   │   ┌─────────────────────────────────────────────────────────┐   │  │
│   │   │  ERR__ASSIGN_PARTITIONS  →  assign() / incremental_assign│   │  │
│   │   │  ERR__REVOKE_PARTITIONS  →  unassign() / incremental_    │   │  │
│   │   │                             unassign()                   │   │  │
│   │   └─────────────────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└───────────────────────────────────────────────────────────────────────┬─┘
                                                                        │
                                                                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         清理阶段                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌───────────────────┐         ┌───────────────────┐                  │
│   │     close()       │────────→│ delete consumer   │                  │
│   │   关闭消费者      │         │                   │                  │
│   │                   │         │ ┌───────────────┐ │                  │
│   │ ┌───────────────┐ │         │ │ KafkaConsumer │ │                  │
│   │ │ KafkaConsumer │ │         │ │   (销毁)      │ │                  │
│   │ │   ::close()   │ │         │ └───────────────┘ │                  │
│   │ └───────────────┘ │         └───────────────────┘                  │
│   │                   │                                                │
│   │ • 提交最终offset  │                                                │
│   │ • 离开消费者组    │                                                │
│   │ • 触发重平衡      │                                                │
│   └───────────────────┘                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

```

## 2.Conf类（配置类）
dKafka::Conf 是 librdkafka 中**配置管理的核心类**，用于统一管理 Kafka 客户端（生产者 / 消费者）的**全局配置**、**主题级配置**，是创建生产者 / 消费者实例的基础。
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116100012806.png)

### （1）Conf类的定义
```cpp
class RD_EXPORT Conf {
public:
    // 配置类型
    enum ConfType {
        CONF_GLOBAL, // 全局配置
        CONF_TOPIC   // Topic配置
    };

    // set() 返回码
    enum ConfResult {
        CONF_UNKNOWN = -2, // 未知属性
        CONF_INVALID = -1, // 无效值
        CONF_OK      = 0   // 成功
    };

    // 创建配置对象
    static Conf *create(ConfType type);

    virtual ~Conf() {}

  	/**
   	* @brief 设置字符串类型的配置属性
   	* @param name 配置属性名称
   	* @param value 配置值
   	* @param errstr 错误时返回的错误描述
   	* @return 配置结果码
   	*/
    virtual ConfResult set(const std::string &name,
                           const std::string &value,
                           std::string &errstr) = 0;

    // 设置默认 Topic 配置 (name = "default_topic_conf")
    virtual ConfResult set(const std::string &name, const Conf *topic_conf, std::string &errstr) = 0;

    // ===================================== 设置回调函数配置 =================================================
    /*	1.参数：
    		- name：配置名（固定）
				- 特征很明显，如DeliveryReportCb *dr_cb ——> name："dr_cb"
			- xxxCb：回调对象（成员函数：回调函数）
			- errstr：错误时返回的错误描述

		2.返回值：
			- ConfResult：配置结果码（见上）
    */

    /*  1.DeliveryReportCb - 消息投递报告回调（⭐⭐⭐⭐⭐ 极高（生产者常用））
            - 功能：生产者发送消息后，通知消息是否成功投递到 Broker 
    		- 使用场景：确认消息发送成功/失败，处理发送失败的重试逻辑*/
    virtual ConfResult set(const std::string &name, DeliveryReportCb *dr_cb, std::string &errstr) = 0;
    
    /* 2.EventCb - 事件回调（⭐⭐⭐⭐⭐ 极高（生产者/消费者都常用））
    		- 功能：接收 Kafka 客户端的各种事件（错误、统计信息、日志、限流等）
    		- 使用场景：监控连接状态、记录错误日志、获取统计数据*/
    virtual ConfResult set(const std::string &name, EventCb *event_cb, std::string &errstr) = 0;
    
    /*	3.RebalanceCb - 消费者重平衡回调
    		- 功能：消费者组发生分区重新分配时触发（成员加入/离开）（⭐⭐⭐⭐ 高（消费者组常用））
    		- 使用场景：在重平衡前保存 offset、释放资源；重平衡后初始化状态*/
    virtual ConfResult set(const std::string &name, RebalanceCb *rebalance_cb, std::string &errstr) = 0;

    /*	4.OffsetCommitCb - 偏移量提交回调（⭐⭐⭐ 中等（需要精确控制 offset 时使用））
			- 功能：offset 提交完成后的通知（成功或失败）
			- 使用场景：确认 offset 提交状态，处理提交失败
    */
    virtual ConfResult set(const std::string &name, OffsetCommitCb *cb, std::string &errstr) = 0;

    /*	5.PartitionerCb - 自定义分区器回调（⭐⭐⭐ 中等（默认分区器够用时不需要））
    		- 功能：自定义消息路由到哪个分区的逻辑
			- 使用场景：需要特殊分区策略（如按业务 ID 分区、地理位置分区）
    */
    virtual ConfResult set(const std::string &name, PartitionerCb *partitioner_cb, std::string &errstr) = 0;
    
    /*	6.ConsumeCb - 消息消费回调（⭐⭐⭐ 中等（通常直接用 poll() 更常见））
		- 功能：消费到消息时的回调处理
		- 使用场景：回调式消费模式（替代 poll 循环）
    */
    virtual ConfResult set(const std::string &name, ConsumeCb *consume_cb, std::string &errstr) = 0;

    //下面使用频率较低，就不详细注释了
    virtual ConfResult set(const std::string &name, OAuthBearerTokenRefreshCb *cb, std::string &errstr) = 0; //令牌刷新回调
    virtual ConfResult set(const std::string &name, SslCertificateVerifyCb *cb, std::string &errstr) = 0; //SSL 证书验证回调
    virtual ConfResult set(const std::string &name, SocketCb *socket_cb, std::string &errstr) = 0; //Socket 创建回调
    virtual ConfResult set(const std::string &name, OpenCb *open_cb, std::string &errstr) = 0; //文件打开回调
    virtual ConfResult set(const std::string &name, PartitionerKeyPointerCb *cb, std::string &errstr) = 0; // 键指针分区器回调

    // 设置 SSL 证书/密钥
    virtual ConfResult set_ssl_cert(RdKafka::CertificateType cert_type,
                                    RdKafka::CertificateEncoding cert_enc,
                                    const void *buffer, size_t size,
                                    std::string &errstr) = 0;

    // 查询字符串配置值
    virtual ConfResult get(const std::string &name, std::string &value) const = 0;

    // ========== 获取回调函数 ==========
    virtual ConfResult get(DeliveryReportCb *&dr_cb) const = 0;
    virtual ConfResult get(OAuthBearerTokenRefreshCb *&cb) const = 0;
    virtual ConfResult get(EventCb *&event_cb) const = 0;
    virtual ConfResult get(PartitionerCb *&partitioner_cb) const = 0;
    virtual ConfResult get(PartitionerKeyPointerCb *&cb) const = 0;
    virtual ConfResult get(SocketCb *&socket_cb) const = 0;
    virtual ConfResult get(OpenCb *&open_cb) const = 0;
    virtual ConfResult get(RebalanceCb *&rebalance_cb) const = 0;
    virtual ConfResult get(OffsetCommitCb *&cb) const = 0;
    virtual ConfResult get(SslCertificateVerifyCb *&cb) const = 0;

    // 导出所有配置 (name, value) 列表
    virtual std::list<std::string> *dump() = 0;

    // 获取底层 C 句柄（不推荐直接使用）
    virtual struct rd_kafka_conf_s *c_ptr_global() = 0;
    virtual struct rd_kafka_topic_conf_s *c_ptr_topic() = 0;

    // 设置 SSL 引擎回调数据
    virtual ConfResult set_engine_callback_data(void *value, std::string &errstr) = 0;

    // 启用 SASL 专用队列
    virtual ConfResult enable_sasl_queue(bool enable, std::string &errstr) = 0;
};
```

### （2）Conf类的使用
#### <1>创建 RdKafka::Conf
- RdKafka::Conf 的创建非常简单，使用**静态工厂方法 create()**
- **推荐**：使用“智能指针”自动管理内存，避免手动 delete

##### 1）使用智能指针直接创建
```cpp
#include <iostream>
#include <string>
#include <memory>
#include <librdkafka/rdkafkacpp.h>

int main() {
    // ==================== 1. 使用 unique_ptr 创建全局配置 ====================
    std::unique_ptr<RdKafka::Conf> global_conf(
        RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL)
    );
    
    if (!global_conf) {
        std::cerr << "创建全局配置失败!" << std::endl;
        return -1;
    }
    std::cout << "全局配置创建成功" << std::endl;
    
    // ==================== 2. 使用 unique_ptr 创建 Topic 配置 ====================
    std::unique_ptr<RdKafka::Conf> topic_conf(
        RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC)
    );
    
    if (!topic_conf) {
        std::cerr << "创建 Topic 配置失败!" << std::endl;
        return -1;
    }
    std::cout << "Topic 配置创建成功" << std::endl;

    return 0;
}
```

##### 2）封装为工厂函数
```cpp
#include <iostream>
#include <string>
#include <memory>
#include <librdkafka/rdkafkacpp.h>

// 工厂函数：创建全局配置
std::unique_ptr<RdKafka::Conf> createGlobalConf() {
    return std::unique_ptr<RdKafka::Conf>(
        RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL)
    );
}

// 工厂函数：创建 Topic 配置
std::unique_ptr<RdKafka::Conf> createTopicConf() {
    return std::unique_ptr<RdKafka::Conf>(
        RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC)
    );
}

int main() {
    auto global_conf = createGlobalConf();
    auto topic_conf = createTopicConf();
    
    if (!global_conf || !topic_conf) {
        std::cerr << "配置创建失败!" << std::endl;
        return -1;
    }
    
    std::cout << "配置创建并设置成功" << std::endl;
    
    return 0;
}

```

##### 3）不使用“智能指针”（不推荐！）
在“RdKafka::Conf类的结构”部分可知：create返回 “裸指针”，所以需要“**手动释放资源**”
```cpp
#include <iostream>
#include <string>
#include <librdkafka/rdkafkacpp.h>

int main() {
    std::string errstr;
    
    // ==================== 1. 创建配置（裸指针） ====================
    RdKafka::Conf *global_conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    RdKafka::Conf *topic_conf = RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC);
    
    // 必须检查是否创建成功
    if (!global_conf || !topic_conf) {
        std::cerr << "配置创建失败!" << std::endl;
        // ⚠️ 问题1：如果一个成功一个失败，需要处理部分释放
        delete global_conf;  // 可能是 nullptr，但 delete nullptr 是安全的
        delete topic_conf;
        return -1;
    }

    return 0;
}
```

#### <2>设置 RdKafka::Conf
```cpp
// Producer 最小配置
RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
conf->set("bootstrap.servers", "localhost:9092", errstr);
conf->set("acks", "all", errstr);
conf->set("dr_cb", &delivery_cb, errstr);  // 投递回调

// Consumer 最小配置
RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
conf->set("bootstrap.servers", "localhost:9092", errstr);
conf->set("group.id", "my-consumer-group", errstr);
conf->set("enable.auto.commit", "false", errstr);
```
**实战使用**：
```cpp
#pragma once
#include <iostream>
#include <librdkafka/rdkafkacpp.h>
#include <memory>
#include <string>
#include <stdexcept>


//Kafka配置构建器（Builder模式）
class KafkaConfBuilder {
public:
    explicit KafkaConfBuilder(RdKafka::Conf::ConfType type = RdKafka::Conf::CONF_GLOBAL) 
        : conf_(RdKafka::Conf::create(type)) {
        if (!conf_) {
            throw std::runtime_error("RdKafka::Conf create failed");
        }
    }
    
    // 禁止拷贝
    KafkaConfBuilder(const KafkaConfBuilder&) = delete;
    KafkaConfBuilder& operator=(const KafkaConfBuilder&) = delete;
    
    // 允许移动
    KafkaConfBuilder(KafkaConfBuilder&&) = default;
    KafkaConfBuilder& operator=(KafkaConfBuilder&&) = default;
    
    /**
     * @brief 设置字符串类型配置项
     * @param key 配置键名
     * @param value 配置值
     * @return 当前Builder引用（支持链式调用）
     * @throws std::runtime_error 配置失败时抛出
     */
    KafkaConfBuilder& set(const std::string& key, const std::string& value) {
        std::string errstr;
        if (conf_->set(key, value, errstr) != RdKafka::Conf::CONF_OK) {
            throw std::runtime_error("Kafka Conf [" + key + "] error: " + errstr);
        }
        return *this;
    }
    
    /**
     * @brief 设置回调对象
     * @tparam T 回调类型（如 RdKafka::DeliveryReportCb*）
     * @param key 配置键名（如 "dr_cb"）
     * @param cb 回调对象指针
     * @return 当前Builder引用
     * @throws std::runtime_error 配置失败时抛出
     */
    template<typename T>
    KafkaConfBuilder& setCallback(const std::string& key, T* cb) {
        std::string errstr;
        if (conf_->set(key, cb, errstr) != RdKafka::Conf::CONF_OK) {
            throw std::runtime_error("Kafka Conf [" + key + "] error: " + errstr);
        }
        return *this;
    }
    
    /**
     * @brief 构建并返回配置对象
     * @return 配置对象的unique_ptr（所有权转移）
     * @note 调用后当前Builder失效，不可再使用
     */
    std::unique_ptr<RdKafka::Conf> build() {
        return std::move(conf_);
    }
    
    /**
     * @brief 检查Builder是否有效
     */
    explicit operator bool() const {
        return conf_ != nullptr;
    }
    
private:
    std::unique_ptr<RdKafka::Conf> conf_;
};


// ==================== 使用示例 ====================
int main() {
    try {
        auto producer_conf = KafkaConfBuilder()
            .set("bootstrap.servers", "localhost:9092")
            .set("client.id", "my-producer")
            .set("acks", "all")
            .set("retries", "3")
            .set("batch.size", "16384")
            .set("linger.ms", "5")
            .build();
        
        auto consumer_conf = KafkaConfBuilder()
            .set("bootstrap.servers", "localhost:9092")
            .set("group.id", "my-consumer-group")
            .set("client.id", "my-consumer")
            .set("enable.auto.commit", "false")
            .set("auto.offset.reset", "earliest")
            .build();
        
        std::cout << "配置创建成功！" << std::endl;
        return 0;
        
    } catch (const std::exception& e) {
        std::cerr << "错误: " << e.what() << std::endl;
        return -1;
    }
}

```

**“回调类”参考**：[回调类](#cb)

##### 1）生产者常用配置
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116161810771.png)

##### 2）消费者常用配置
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116161420955.png)

想了解更多配置项，**参考**：[附录1：常用配置项](#config)


## 3.TopicPartition类（主题分区类）
### （1）概述
#### <1>作用
- 封装**<topic, partition, offset>**三元组的核心类
- 客户端与 Kafka 集群交互「**分区级操作**」的基础载体
- 所有和「**指定分区、控制消费位置、提交偏移量**」相关的操作都依赖该类

#### <2>核心定位
- **元数据载体**：
    - 本质是存储「主题名（string）、分区号（int）、偏移量（int64_t）」的轻量级类，仅记录 “要操作的目标分区 + 位置”，**不包含任何集群交互逻辑**；
- **操作桥梁**：
    - 作为 assign()（手动分配分区）、commitSync()/commitAsync()（提交 offset）、seek()（重置消费位置）等核心 API 的入参 / 返回值，是**客户端向 Kafka 传递 “分区操作指令” 的标准化格式**；
- **无状态特性**：
    - 该类对象仅存储静态元数据，不关联 Kafka 连接状态，**修改对象属性（如 offset）仅影响本地，需调用 API 才能同步到集群**。

### （2）TopicPartition类的定义
```cpp
class RD_EXPORT TopicPartition {
public:
    //==================== 高频使用（必用） ====================
    
    /**
     * @brief 创建 topic+partition 关联对象（无 offset）
     * 需使用 delete 释放
     */
    static TopicPartition *create(const std::string &topic, int partition);
    
    /**
     * @brief 创建 topic+partition+offset 关联对象
     * 需使用 delete 释放
     */
    static TopicPartition *create(const std::string &topic,
                                  int partition,
                                  int64_t offset);

    virtual const std::string &topic() const = 0;    // 获取 topic 名
    virtual int partition() const = 0;               // 获取分区 ID
    virtual int64_t offset() const = 0;              // 获取 offset

    //==================== 中频使用（常用） ====================
    
    static void destroy(std::vector<TopicPartition *> &partitions);  // 批量销毁
    virtual ErrorCode err() const = 0;               // 获取错误码
    virtual void set_offset(int64_t offset) = 0;     // 设置 offset
    virtual ~TopicPartition() = 0;                   // 析构函数

    //==================== 低频使用（高级场景） ====================
    
    virtual int32_t get_leader_epoch() = 0;          // 获取 leader epoch
    virtual void set_leader_epoch(int32_t leader_epoch) = 0;  //设置 leader epoch

    //==================== 极少使用（特殊场景） ====================
    

    virtual std::vector<unsigned char> get_metadata() = 0;  // 获取分区元数据
    

    virtual void set_metadata(std::vector<unsigned char> &metadata) = 0; // 设置分区元数据
};

// 特殊 offset 常量（按使用频率）
const int64_t OFFSET_STORED    = -1000; // ★★★ 最常用：从上次提交位置继续
const int64_t OFFSET_END       = -1;    // ★★  常用：只消费新消息
const int64_t OFFSET_BEGINNING = -2;    // ★★  常用：从头消费所有消息
const int64_t OFFSET_INVALID   = -1001; // ★   少用：表示无效状态
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116211851723.png)

### （3）典型使用场景（结合核心 API）
#### <1> 手动分配分区
无需指定偏移量，仅关联「主题 + 分区」，依赖 auto.offset.reset 配置确定消费位置：    
```cpp
std::vector<RdKafka::TopicPartition*> partitions;
partitions.push_back(RdKafka::TopicPartition::create("topic1", 0));
partitions.push_back(RdKafka::TopicPartition::create("topic1", 1));
consumer->assign(partitions); // 仅指定要消费的分区
```

#### <2>指定偏移量消费
创建时指定偏移量，重置消费位置：
```cpp
// 从 topic1-0 分区的第 100 条消息开始消费
RdKafka::TopicPartition *tp = RdKafka::TopicPartition::create("topic1", 0, 100);
consumer->seek(tp, 5000); // 5000ms 超时时间
delete tp;
```

#### <3>手动提交指定 offset
修改对象偏移量后提交，实现精准的 offset 管理：
```cpp
std::vector<RdKafka::TopicPartition*> partitions;
partitions.push_back(RdKafka::TopicPartition::create("topic1", 0));
partitions[0]->set_offset(200); // 提交 topic1-0 的 offset=200
consumer->commitSync(partitions); // 同步提交
```

#### <4>批量操作所有分区
使用 partition=-1 代表主题的所有分区：
```cpp
// 提交 topic1 所有分区的 offset=300
RdKafka::TopicPartition *tp = RdKafka::TopicPartition::create("topic1", -1, 300);
consumer->commitSync({tp});
delete tp;
```

#### <5>destroy() 的使用示例
```cpp
// 批量创建
std::vector<RdKafka::TopicPartition*> partitions;
partitions.push_back(RdKafka::TopicPartition::create("topic1", 0));
partitions.push_back(RdKafka::TopicPartition::create("topic1", 1));
partitions.push_back(RdKafka::TopicPartition::create("topic2", 0));

consumer->assign(partitions);

// ... 使用完毕后 ...

// 方式1：手动逐个删除
for (auto p : partitions) 
    delete p;
partitions.clear();

// 方式2：使用 destroy() 一次性销毁并清空（推荐）
RdKafka::TopicPartition::destroy(partitions);  // 删除所有元素 + 清空 vector
// 此时 partitions.size() == 0，且原指针已被 delete
// ❌ 不能再访问 partitions 中的元素
```

### （4）内存管理（核心注意事项）
**必须释放资源**：
- create 方法返回的是动态分配的裸指针，未释放会导致内存泄漏，推荐两种方式：
    - 手动 delete（基础方式）
    - 封装智能指针（生产环境推荐）
```cpp
//手动 delete
for (auto p : partitions) {
    delete p;
    p = nullptr; // 避免野指针
}

//使用智能指针
using TopicPartitionPtr = std::unique_ptr<RdKafka::TopicPartition>;
TopicPartitionPtr tp(RdKafka::TopicPartition::create("topic1", 0));
// 自动释放，无需手动 delete
```

### （5）核心易错点与最佳实践
#### <1>易错点
- **混淆「客户端对象创建」和「集群物理创建」**：
    - TopicPartition::create 仅创建**本地对象**，不会在 Kafka 集群创建主题 / 分区；
- **忽略内存释放**：
    - 生产环境需严格管理 TopicPartition 指针，**建议封装智能指针**；
- **偏移量设置后未调用 API**：
    - **修改 offset 仅改本地值**，需调用 commitSync()/seek() 等 API 才能同步到集群；
- **分区号越界**：
    - 指定的分区号超出主题实际分区数，会导致 assign()/commitSync() 返回 ERR_UNKNOWN_PARTITION 错误。


## 4.Headers类（消息头类）
### （1）概述
#### <1>作用
- RdKafka::Headers 是 librdkafka 中**封装 Kafka 消息头（Header）的核心类**
    - 用于为消息附**加键值对**形式的元数据（如业务标识、追踪 ID、消息类型等）
    - 是消息内容之外的 “**辅助信息载体**”，不影响消息的分区路由，仅用于业务侧的消息溯源、过滤、扩展。

#### <2>核心定位
- **元数据扩展**：
    - **弥补 Kafka 消息 Key/Value 仅能存储业务数据的不足**
    - 支持附加多组键值对元数据（如 trace-id: 123456、msg-type: order）；
- **可写 / 只读特性**：
    - 生产者侧可创建并写入 Headers
    - 消费者侧仅能读取 Headers（**消息一旦发送，Headers 不可修改**）；
- **轻量级设计**：
    - Headers 存储的是字符串键值对，**建议仅存放轻量元数据**（避免增大消息体积）。
- **跨系统兼容**：
    - 消息头是 Kafka 协议标准特性，可与其他 Kafka 客户端（如 Java、Python）互通，便于多语言系统协作；    

### （2）Headers类的定义
```cpp
class RD_EXPORT Headers {
public:
    virtual ~Headers() = 0;

    /**
     * @brief 单个 Header 对象（键值对 + 错误码）
     * @remark 不支持动态分配（new）
     */
    class Header {
    public:
        /**
         * @brief 构造 Header
         * @param key    Header 的键（字符串）
         * @param value  Header 的值（二进制，可为 NULL）
         * @param value_size  值的字节长度
         * @remark key 和 value 会被复制
         */
        Header(const std::string &key, const void *value, size_t value_size);

        /**
         * @brief 构造 Header（带错误码）
         * @param key    Header 的键
         * @param value  Header 的值
         * @param value_size  值的字节长度
         * @param err    错误码（用于 get_last() 内部构造失败结果）
         * @remark 若 err != ERR_NO_ERROR，value 和 value_size 字段未定义
         */
        Header(const std::string &key, const void *value, size_t value_size, 
               const RdKafka::ErrorCode err);

        Header(const Header &other);              // 拷贝构造
        Header &operator=(const Header &other);   // 赋值运算符
        ~Header();

        std::string key() const;                  // 获取键名
        const void *value() const;                // 获取二进制值（可能为 NULL）
        const char *value_string() const;         // 获取值并转为 C 字符串（可能为 NULL）
        size_t value_size() const;                // 获取值的字节长度
        RdKafka::ErrorCode err() const;           // 获取错误码（通常为 ERR_NO_ERROR）

    private:
        char *copy_value(const void *value, size_t value_size);
        std::string key_;
        RdKafka::ErrorCode err_;
        char *value_;
        size_t value_size_;
        void *operator new(size_t);               // 禁止动态分配
    };

    /**
     * @brief 创建空的 Headers 对象
     * @returns 空 Headers 列表
     */
    static Headers *create();

    /**
     * @brief 从 std::vector 创建 Headers 对象
     * @param headers  Header 向量（会被复制，非引用）
     * @returns Headers 列表
     */
    static Headers *create(const std::vector<Header> &headers);

    /**
     * @brief 添加 Header（二进制值）
     * @param key    键名
     * @param value  二进制值（可为 NULL）
     * @param value_size  值的字节长度
     * @returns 错误码
     */
    virtual ErrorCode add(const std::string &key, const void *value, size_t value_size) = 0;

    /**
     * @brief 添加 Header（字符串值，便捷方法）
     * @param key    键名
     * @param value  字符串值
     * @returns 错误码
     */
    virtual ErrorCode add(const std::string &key, const std::string &value) = 0;

    /**
     * @brief 添加 Header（复制已有 Header 对象）
     * @param header  要复制的 Header
     * @returns 错误码
     */
    virtual ErrorCode add(const Header &header) = 0;

    /**
     * @brief 移除指定 key 的所有 Header
     * @param key  要移除的键名
     * @returns 错误码
     */
    virtual ErrorCode remove(const std::string &key) = 0;

    /**
     * @brief 获取指定 key 的所有 Header
     * @param key  键名
     * @returns 包含所有匹配 Header 的向量（支持重复 key）
     */
    virtual std::vector<Header> get(const std::string &key) const = 0;

    /**
     * @brief 获取指定 key 的最后一个 Header
     * @param key  键名
     * @returns 找到则返回 Header，未找到则返回 err=ERR__NOENT 的 Header
     */
    virtual Header get_last(const std::string &key) const = 0;

    /**
     * @brief 获取所有 Header
     * @returns 包含所有 Header 的向量
     */
    virtual std::vector<Header> get_all() const = 0;

    /**
     * @brief 获取 Header 数量
     * @returns Header 个数
     */
    virtual size_t size() const = 0;
};
```

### （3）使用示例
#### <1>生产者写入 Headers 示例
```cpp
// 1. 创建 Headers 对象
RdKafka::Headers *headers = RdKafka::Headers::create();
if (!headers) {
    std::cerr << "创建 Headers 失败" << std::endl;
    return -1;
}

// 2. 添加键值对（支持多组）
headers->add("trace-id", "trace-789012");              // 字符串重载（推荐）
headers->add("msg-type", "payment");                   // 字符串重载
headers->add("timestamp", std::to_string(time(nullptr)));  // 字符串重载

// 或使用二进制形式
const char* binary_data = "binary-value";
headers->add("binary-key", binary_data, strlen(binary_data));

// 3. 发送消息时附加 Headers
std::string payload = "order-payload";
std::string key = "order-key";
RdKafka::ErrorCode err = producer->produce(
    "my-topic",
    RdKafka::Topic::PARTITION_UA,
    RdKafka::Producer::RK_MSG_COPY,
    (void*)payload.data(), 13,
    key.data(), 9,
    0,
    headers,    // 附加消息头
    nullptr     // opaque
);

if (err != RdKafka::ERR_NO_ERROR) {
    std::cerr << "发送失败：" << RdKafka::err2str(err) << std::endl;
    delete headers;  // ⚠️ produce() 失败时需手动释放
} 
// ⚠️ produce() 成功后，headers 所有权转移给 librdkafka，无需手动 delete
```

#### <2>消费者读取 Headers 示例
```cpp
// 消费消息并读取 Headers
RdKafka::Message *msg = consumer->consume(1000);
if (!msg || msg->err() != RdKafka::ERR_NO_ERROR) {
    delete msg;
    return;
}

// 1. 获取消息的 Headers 对象（无 Headers 则返回 nullptr）
RdKafka::Headers *headers = msg->headers();
if (headers) {
    std::cout << "消息头数量：" << headers->size() << std::endl;  // ✅ 用 size()
    
    // 2. 读取指定 key 的所有值（支持重复 key）
    std::vector<RdKafka::Headers::Header> trace_headers = headers->get("trace-id");
    for (const auto& h : trace_headers) {
        if (h.err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "trace-id：" << h.value_string() << std::endl;
        }
    }

    // 3. 读取指定 key 的最后一个值
    RdKafka::Headers::Header last_trace = headers->get_last("trace-id");
    if (last_trace.err() == RdKafka::ERR_NO_ERROR) {
        std::cout << "最后一个 trace-id：" << last_trace.value_string() << std::endl;
    } else {
        std::cout << "未找到 trace-id" << std::endl;
    }

    // 4. 遍历所有 Headers
    std::vector<RdKafka::Headers::Header> all_headers = headers->get_all();
    for (size_t i = 0; i < all_headers.size(); i++) {
        const auto& h = all_headers[i];
        std::cout << "Header[" << i << "]：" << h.key() << " = ";
        if (h.value()) {
            std::cout << h.value_string() << std::endl;
        } else {
            std::cout << "(null)" << std::endl;
        }
    }
} else {
    std::cout << "该消息无消息头" << std::endl;
}

delete msg;
```

### （4）核心注意事项
- **内存管理**：
    - **生产者侧**：create() 创建的 Headers
        - 若 produce() **成功**：则所有权转移给 librdkafka，**无需手动释放**；
        - 若 produce() **失败**：则**需手动 delete**；
    - **消费者**侧从 msg->headers() 获取的 Headers 由 Message 对象管理
        - **无需手动释放**（释放 Message 时自动释放）；
- **空值处理**：
    - Header 的 Value 可以为空（value=nullptr，value_len=0），需在读取时判断 val 是否为 nullptr；
- **get_last() 错误处理**：
    - 未找到时返回 err=ERR__NOENT 的 Header，需检查 h.err() 后再使用 value()；
- **性能影响**：
    - Headers 会增加消息体积，建议单条消息的 Headers 总大小不超过 1KB，避免影响吞吐量。


## 5.Handle类（句柄基类）
### （1）概述
#### <1>作用
- RdKafka::Handle 是 librdkafka 中**生产者和消费者客户端**（Producer / Consumer / KafkaConsumer）**的抽象基类**
    - **定义了客户端的通用能力**（事件轮询（poll）、元数据/偏移量查询、流控（pause/resume）、集群信息获取等），是整个 librdkafka 客户端的 “基础骨架”
    - **生产者和消费者客户端**（如 Producer、Consumer、KafkaConsumer）均继承自 Handle，因此**具备统一的基础行为**。

#### <2>核心定位
- **抽象通用能力**：
    - 提取生产者、消费者的共性操作（如配置获取、错误码转换、资源销毁），避免代码冗余；
- **生命周期管理**：
    - 定义客户端的创建 / 销毁规范，统一管理与 Kafka 集群的连接资源；
- **状态与配置统一**：
    - 提供全局的配置查询、错误处理、日志相关接口，保证不同客户端的使用体验一致；
- **纯抽象类特性**：
    - Handle 包含纯虚函数，无法直接实例化，仅能通过子类（Producer/Consumer/KafkaConsumer）使用其接口。

### （2）Handle 类的定义
```cpp
class RD_EXPORT Handle {
public:
    virtual ~Handle() {
    }

    //=============================================================================
    // 【第一梯队】几乎每个应用都会用到
    //=============================================================================

    /**
     * @brief 轮询事件
     * @param timeout_ms 超时时间（毫秒）
     	- 0：非阻塞，立即返回（处理已就绪的事件）
     	- 大于0：最多等待 timeout_ms 毫秒，有事件则提前返回 （推荐：100 ~ 1000）
     	- -1：无限阻塞，直到有事件发生
     * @returns 处理的事件数量
     * 
     * @remark 触发已注册的回调函数（如 DeliveryCb、EventCb）
     	- 1. 检查内部事件队列
     	- 2. 如果有待处理事件 → 触发对应回调 → 立即返回
     	- 3. 如果没有事件 → 等待最多 timeout 毫秒
     	- 4. 返回本次处理的事件数量
     * @warning 禁止在 KafkaConsumer 中使用，应使用 consume() 代替
     */
    virtual int poll(int timeout_ms) = 0;

    /**
     * @brief 获取客户端名称
     * @returns 客户端名称字符串
     */
    virtual std::string name() const = 0;

    /**
     * @brief 获取待发送/待确认的消息队列长度
     * @returns 队列中的消息数量
     * 
     * @remark 常用于优雅关闭时等待所有消息发送完成
     */
    virtual int outq_len() = 0;

    //=============================================================================
    // 【第二梯队】元数据与偏移量查询（常用）
    //=============================================================================

    /**
     * @brief 从 Broker 请求元数据
     * @param all_topics   true=请求所有 Topic，false=仅本地已知 Topic
     * @param only_rkt     仅请求指定 Topic 的元数据（可为 NULL）
     * @param metadatap    [out] 元数据结果指针，需用 delete 释放
     * @param timeout_ms   超时时间（毫秒）
     * @returns ERR_NO_ERROR 成功，ERR__TIMED_OUT 超时，其他为错误码
     */
    virtual ErrorCode metadata(bool all_topics,
                               const Topic *only_rkt,
                               Metadata **metadatap,
                               int timeout_ms) = 0;

    /**
     * @brief 查询分区的水位偏移量（从 Broker 实时查询）
     * @param topic      Topic 名称
     * @param partition  分区号
     * @param low        [out] 最低偏移量（最早消息）
     * @param high       [out] 最高偏移量（最新消息）
     * @param timeout_ms 超时时间
     * @returns ERR_NO_ERROR 成功，其他为错误码
     */
    virtual ErrorCode query_watermark_offsets(const std::string &topic,
                                              int32_t partition,
                                              int64_t *low,
                                              int64_t *high,
                                              int timeout_ms) = 0;

    /**
     * @brief 获取缓存的分区水位偏移量（本地缓存，不请求 Broker）
     * @param topic      Topic 名称
     * @param partition  分区号
     * @param low        [out] 最低偏移量
     * @param high       [out] 最高偏移量
     * @returns ERR_NO_ERROR 成功，其他为错误码
     * 
     * @remark 仅适用于活跃的消费者实例
     * @remark 无缓存时返回 OFFSET_INVALID
     */
    virtual ErrorCode get_watermark_offsets(const std::string &topic,
                                            int32_t partition,
                                            int64_t *low,
                                            int64_t *high) = 0;

    /**
     * @brief 根据时间戳查找偏移量
     * @param offsets    [in/out] TopicPartition 列表，输入 offset 字段为时间戳，
     *                   输出 offset 字段为对应偏移量
     * @param timeout_ms 超时时间
     * @returns ERR_NO_ERROR 成功（可能有部分分区失败，需检查各分区的 err()）
     * 
     * @remark 时间戳为毫秒级 Unix 时间戳（UTC）
     */
    virtual ErrorCode offsetsForTimes(std::vector<TopicPartition *> &offsets,
                                      int timeout_ms) = 0;

    //=============================================================================
    // 【第三梯队】流控与集群信息
    //=============================================================================

    /**
     * @brief 暂停指定分区的生产/消费
     * @param partitions 分区列表，每个分区的结果会写入其 err() 字段
     * @returns ERR_NO_ERROR
     */
    virtual ErrorCode pause(std::vector<TopicPartition *> &partitions) = 0;

    /**
     * @brief 恢复指定分区的生产/消费
     * @param partitions 分区列表，每个分区的结果会写入其 err() 字段
     * @returns ERR_NO_ERROR
     */
    virtual ErrorCode resume(std::vector<TopicPartition *> &partitions) = 0;

    /**
     * @brief 获取 Kafka 集群 ID
     * @param timeout_ms 无缓存时的最大等待时间，0=非阻塞
     * @returns 集群 ID 字符串，失败返回空字符串
     * 
     * @remark 要求 Broker 版本 >= 0.10.0 且 api.version.request=true
     */
    virtual std::string clusterid(int timeout_ms) = 0;

    /**
     * @brief 获取当前 Controller Broker 的 ID
     * @param timeout_ms 无缓存时的最大等待时间
     * @returns Controller ID，失败返回 -1
     * 
     * @remark 要求 Broker 版本 >= 0.10.0 且 api.version.request=true
     */
    virtual int32_t controllerid(int timeout_ms) = 0;

    /**
     * @brief 获取消费者的组成员 ID
     * @returns 成员 ID，非组成员时返回空字符串
     * 
     * @remark 仅适用于高级消费者（KafkaConsumer）
     */
    virtual std::string memberid() const = 0;

    //=============================================================================
    // 【第四梯队】队列管理与回调控制
    //=============================================================================

    /**
     * @brief 获取指定分区的 Fetch 队列
     * @param partition 分区对象
     * @returns 队列指针，失败返回 NULL
     * 
     * @remark 仅适用于消费者
     */
    virtual Queue *get_partition_queue(const TopicPartition *partition) = 0;

    /**
     * @brief 将日志转发到指定队列
     * @param queue 目标队列，NULL 表示转发到主队列
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark 需同时设置配置项 log.queue=true
     */
    virtual ErrorCode set_log_queue(Queue *queue) = 0;

    /**
     * @brief 获取后台线程队列
     * @returns 后台队列指针
     */
    virtual Queue *get_background_queue() = 0;

    /**
     * @brief 取消当前的回调分发（立即从 poll/consume 返回）
     * 
     * @remark 只能在 librdkafka 回调函数内部调用
     */
    virtual void yield() = 0;

    //=============================================================================
    // 【第五梯队】错误处理与认证（特定场景）
    //=============================================================================

    /**
     * @brief 获取首个致命错误
     * @param errstr [out] 错误描述字符串
     * @returns ERR_NO_ERROR 表示无致命错误，其他为错误码
     * 
     * @remark 主要用于幂等生产者的错误检测
     */
    virtual ErrorCode fatal_error(std::string &errstr) const = 0;

    /**
     * @brief 设置 SASL PLAIN/SCRAM 认证凭据
     * @param username 用户名
     * @param password 密码
     * @returns NULL 成功，否则返回 Error 对象
     * 
     * @remark 新凭据在下次认证时生效，不会断开现有连接
     */
    virtual Error *sasl_set_credentials(const std::string &username,
                                        const std::string &password) = 0;

    /**
     * @brief 设置 SASL/OAUTHBEARER Token
     * @param token_value      Token 值（通常为 JWS 格式）
     * @param md_lifetime_ms   Token 过期时间（毫秒级 Unix 时间戳）
     * @param md_principal_name Kafka Principal 名称
     * @param extensions       扩展键值对列表（key1, value1, key2, value2...）
     * @param errstr           [out] 错误描述
     * @returns ERR_NO_ERROR 成功，ERR__INVALID_ARG 参数错误，
     *          ERR__NOT_IMPLEMENTED 不支持，ERR__STATE 未配置 OAUTHBEARER
     */
    virtual ErrorCode oauthbearer_set_token(
        const std::string &token_value,
        int64_t md_lifetime_ms,
        const std::string &md_principal_name,
        const std::list<std::string> &extensions,
        std::string &errstr) = 0;

    /**
     * @brief 标记 SASL/OAUTHBEARER Token 刷新失败
     * @param errstr 失败原因描述
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode oauthbearer_set_token_failure(const std::string &errstr) = 0;

    /**
     * @brief 在后台线程启用 SASL OAUTHBEARER 刷新回调
     * @returns NULL 成功，否则返回 Error 对象
     * 
     * @remark 适用于不定期调用 poll() 的应用
     */
    virtual Error *sasl_background_callbacks_enable() = 0;

    /**
     * @brief 获取 SASL 回调队列
     * @returns 队列指针，未启用时返回 NULL
     */
    virtual Queue *get_sasl_queue() = 0;

    //=============================================================================
    // 【第六梯队】底层操作（高级用法）
    //=============================================================================

    /**
     * @brief 获取底层 C 语言句柄
     * @returns rd_kafka_t* 指针
     * 
     * @warning 不推荐直接调用 C API，无官方支持
     * @remark 需在 rdkafkacpp.h 之前包含 rdkafka.h
     */
    virtual struct rd_kafka_s *c_ptr() = 0;

    /**
     * @brief 使用 librdkafka 的分配器申请内存
     * @param size 字节数
     * @returns 内存指针
     * 
     * @remark 必须使用 mem_free() 释放
     */
    virtual void *mem_malloc(size_t size) = 0;

    /**
     * @brief 释放 librdkafka 返回的内存
     * @param ptr 内存指针
     * 
     * @remark 仅用于 API 明确说明需使用此函数释放的指针
     */
    virtual void mem_free(void *ptr) = 0;
};

```

### （3）核心特性与注意事项
- **继承关系**：
```cpp
RdKafka::Handle（抽象基类）
├─ RdKafka::Producer          // 生产者
├─ RdKafka::Consumer          // 低级消费者（Simple Consumer，很少用）
└─ RdKafka::KafkaConsumer     // 高级消费者（High-Level Consumer，90%+用它）

RdKafka::AdminClient          // 管理客户端（独立，不继承 Handle）
```
- **资源管理**：
    - 客户端实例由 Conf::create() + Producer/KafkaConsumer::create() 创建
    - 必须通过 delete 释放
- Handle 内部管理与 Broker 的连接池、IO 线程、定时器，销毁时会自动清理所有资源，避免内存泄漏。

- **线程模型**：
    - Handle 子类会启动内部 IO 线程（默认 1 个，可通过 io_threads 配置），处理网络通信、消息发送 / 接收；
    - 业务线程调用 poll()/produce() 等 API 时，仅触发逻辑调度，核心 IO 由内部线程处理。


## 6.Producer类（生产者类）
### （1）概述
#### <1>作用
- RdKafka::Producer 是 librdkafka 中实现 Kafka **“消息生产逻辑”的核心类**，继承自 RdKafka::Handle 抽象基类
- 封装了消息异步发送、分区路由、回调通知、资源管理等全量生产能力，是**客户端向 Kafka 集群发送消息的唯一入口**。

#### <2>核心定位
- **异步发送**：
    - 默认采用 “**异步非阻塞**” 模式，**produce()** 仅将消息放入**本地队列**，由内部 IO 线程发送至 Broker
    - 仅 flush()/ 同步发送会阻塞
- **回调驱动**：
    - 通过 **DeliveryReportCb** “**异步通知**”消息发送结果（成功 / 失败）
- **分区路由**：
    - 支持**内置分区器**（按 Key 哈希 / 随机）或**自定义分区器**（PartitionerCb）
- **资源管理**：
    - 继承 Handle 基类的生命周期管理能力，通过 **delete** 或 **flush() + delete** 安全释放连接、线程、队列等资源
- **高可用设计**：
    - 支持**消息重试**、**批量发送**、**压缩**（gzip/snappy/lz4），提升吞吐量和可靠性
- **可靠性**：
    - 支持消息重试（retries）、ACK 确认（acks）、幂等 / 事务（需配置）；
- **高性能**：
    - 支持批量发送（batch.size）、异步 IO，减少网络请求次数；

#### <3>继承关系
- Producer 继承自 Handle 抽象基类，因此具备以下 Handle 通用能力：
    - **poll()**：轮询事件，触发回调（**生产者必须定期调用**）
    - **name()**：获取客户端名称
    - **outq_len()**：获取待发送消息队列长度
    - 元数据查询、流控等其他 Handle 方法

### （2）Producer类的定义
```cpp
class RD_EXPORT Producer : public virtual Handle {
public:
    //=============================================================================
    // 【第一梯队】几乎每个生产者应用都会用到
    //=============================================================================

    /**
     * @brief 创建 Kafka 生产者实例
     * @param conf 配置对象（可选），不传则使用默认配置；配置对象可复用
     * @param errstr [out] 错误描述字符串
     * @returns 成功返回生产者实例，失败返回 NULL
     */
    static Producer *create(const Conf *conf, std::string &errstr);

    virtual ~Producer() = 0;

    /**
     * @brief 消息标志位（用于 produce() 的 msgflags 参数）
     */
    enum {
    	// ⚠️ 详情见类定义末尾
        RK_MSG_FREE  = 0x1,  /**< librdkafka 使用完 payload 后会自动 free()
                              *   与 RK_MSG_COPY 互斥 */
        RK_MSG_COPY  = 0x2,  /**< librdkafka 会复制 payload 数据，
                              *   调用返回后不再使用原指针
                              *   与 RK_MSG_FREE 互斥 */
        RK_MSG_BLOCK = 0x4   /**< 当消息队列满时阻塞 produce() 调用
                              *   警告：使用时必须在另一个线程调用 poll()
                              *   否则会导致无限阻塞 */
    };

    /**
     * @brief 生产并发送单条消息到 Broker（异步非阻塞）
     * @param topic      Topic 对象
     * @param partition  目标分区：PARTITION_UA 自动分区(不手动指定分区，让 librdkafka 根据 key 自动决定)，
     	或 0..N 手动指定分区
     * @param msgflags   消息标志：RK_MSG_FREE / RK_MSG_COPY / RK_MSG_BLOCK (⚠️ 详情见类定义末尾)
     * @param payload    消息内容指针
     * @param len        消息长度（字节）
     * @param key        消息 Key（可选），用于分区选择和传递给消费者
     * @param msg_opaque 用户自定义指针，会在投递回调中返回
     * @returns ERR_NO_ERROR 成功入队
     *          ERR__QUEUE_FULL 队列满
     *          ERR_MSG_SIZE_TOO_LARGE 消息过大
     *          ERR__UNKNOWN_PARTITION / ERR__UNKNOWN_TOPIC 未知分区/Topic
     * 
     * @remark 若返回错误且使用了 RK_MSG_FREE，调用方仍需负责释放 payload
     */
    virtual ErrorCode produce(Topic *topic,
                              int32_t partition,
                              int msgflags,
                              void *payload,
                              size_t len,
                              const std::string *key,
                              void *msg_opaque) = 0;

    /**
     * @brief produce() 重载：Key 使用指针+长度形式
     */
    virtual ErrorCode produce(Topic *topic,
                              int32_t partition,
                              int msgflags,
                              void *payload,
                              size_t len,
                              const void *key,
                              size_t key_len,
                              void *msg_opaque) = 0;

    /**
     * @brief produce() 重载：Topic 使用字符串，支持指定时间戳（推荐）
     * @param topic_name Topic 名称（无需创建 Topic 对象）
     * @param timestamp  消息时间戳（毫秒级 Unix 时间戳，UTC）
     */
    virtual ErrorCode produce(const std::string topic_name,
                              int32_t partition,
                              int msgflags,
                              void *payload,
                              size_t len,
                              const void *key,
                              size_t key_len,
                              int64_t timestamp,
                              void *msg_opaque) = 0;

    /**
     * @brief produce() 重载：支持消息头（Headers）
     * @param headers 消息头对象，成功时会被释放，失败时保持不变
     * @warning 成功调用后 headers 会被自动释放，失败时需调用方处理
     */
    virtual ErrorCode produce(const std::string topic_name,
                              int32_t partition,
                              int msgflags,
                              void *payload,
                              size_t len,
                              const void *key,
                              size_t key_len,
                              int64_t timestamp,
                              RdKafka::Headers *headers,
                              void *msg_opaque) = 0;

    /**
     * @brief produce() 重载：使用 vector 传递 key 和 payload（自动复制）
     */
    virtual ErrorCode produce(Topic *topic,
                              int32_t partition,
                              const std::vector<char> *payload,
                              const std::vector<char> *key,
                              void *msg_opaque) = 0;

    /**
     * @brief 等待所有待发送消息完成（刷新）
     * @param timeout_ms 超时时间（毫秒）
     * @returns ERR_NO_ERROR 全部完成，ERR__TIMED_OUT 超时
     * 
     * @remark 销毁生产者前应调用此方法，确保所有消息发送完成
     * @remark 会忽略 linger.ms 配置，立即发送队列中的消息
     * @remark 内部会调用 poll()，因此会触发回调
     */
    virtual ErrorCode flush(int timeout_ms) = 0;

    //=============================================================================
    // 【第二梯队】特殊场景使用
    //=============================================================================

    /**
     * @brief 清除生产者当前处理的消息
     * @param purge_flags 清除标志
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark 清除后需调用 poll() 或 flush() 处理投递报告回调
     * @remark 队列中的消息会收到 ERR__PURGE_QUEUE 错误
     * @remark 传输中的消息会收到 ERR__PURGE_INFLIGHT 错误
     * @warning 清除传输中消息会导致无法确认消息是否成功，可能产生重复
     */
    virtual ErrorCode purge(int purge_flags) = 0;

    /**
     * @brief 清除标志位
     */
    enum {
        PURGE_QUEUE        = 0x1,  /**< 清除内部队列中的消息 */
        PURGE_INFLIGHT     = 0x2,  /**< 清除传输中的消息（可能导致重复） */
        PURGE_NON_BLOCKING = 0x4   /**< 不等待后台队列清除完成 */
    };

    //=============================================================================
    // 【第三梯队】事务 API（需要 Broker >= 0.11.0）
    //=============================================================================

    /**
     * @brief 初始化事务
     * @param timeout_ms 最大阻塞时间，超时后可在后台继续，允许重试
     * @returns NULL 成功，否则返回 Error 对象（需 delete）
     * 
     * @remark 使用事务前必须先调用此方法（仅需一次）
     * @remark 通过 Error::is_retriable() 判断是否可重试
     * @remark 通过 Error::is_fatal() 判断是否为致命错误
     */
    virtual Error *init_transactions(int timeout_ms) = 0;

    /**
     * @brief 开始新事务
     * @returns NULL 成功，否则返回 Error 对象（需 delete）
     * 
     * @remark 必须先调用 init_transactions() 成功后才能使用
     */
    virtual Error *begin_transaction() = 0;

    /**
     * @brief 提交当前事务
     * @param timeout_ms 最大阻塞时间，-1 表示使用剩余事务超时时间（推荐）
     * @returns NULL 成功，否则返回 Error 对象（需 delete）
     * 
     * @remark 强烈建议 timeout_ms 传 -1
     * @remark 提交前会自动 flush 所有待发送消息
     * @remark 通过 Error::txn_requires_abort() 判断是否需要中止事务
     */
    virtual Error *commit_transaction(int timeout_ms) = 0;

    /**
     * @brief 中止当前事务
     * @param timeout_ms 最大阻塞时间，-1 表示使用剩余事务超时时间（推荐）
     * @returns NULL 成功，否则返回 Error 对象（需 delete）
     * 
     * @remark 用于从可中止的事务错误中恢复
     * @remark 会清除所有待发送消息，触发 ERR__PURGE_INFLIGHT/QUEUE
     */
    virtual Error *abort_transaction(int timeout_ms) = 0;

    /**
     * @brief 发送消费偏移量到事务（Consume-Transform-Produce 模式）
     * @param offsets        要提交的偏移量列表（应为下一条要消费的偏移量）
     * @param group_metadata 消费者组元数据（从 KafkaConsumer::groupMetadata() 获取）
     * @param timeout_ms     最大阻塞时间
     * @returns NULL 成功，否则返回 Error 对象（需 delete）
     * 
     * @remark 必须在生产者实例上调用，不是消费者
     * @remark 消费者必须禁用自动提交（enable.auto.commit=false）
     * @remark 在 commit_transaction() 之前调用
     */
    virtual Error *send_offsets_to_transaction(
        const std::vector<TopicPartition *> &offsets,
        const ConsumerGroupMetadata *group_metadata,
        int timeout_ms) = 0;
};
```
```cpp
┌─────────────────────────────────────────────────────────────────┐
│  RK_MSG_COPY（推荐）                                            │
│  ─────────────────                                              │
│  • librdkafka 立即复制你的数据到内部缓冲区                       │
│  • produce() 返回后，你的 payload 可以立即释放/修改              │
│  • 代价：多一次内存拷贝                                          │
│                                                                  │
│  你的代码               librdkafka                              │
│  ┌────────┐   复制    ┌────────────┐                            │
│  │payload │ ───────► │ 内部缓冲区 │ ────► Kafka                 │
│  └────────┘          └────────────┘                             │
│      ↓                                                          │
│  produce()返回后                                                 │
│  你可以 delete payload                                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  RK_MSG_FREE（零拷贝，但危险）                                   │
│  ──────────────────────────                                      │
│  • librdkafka 不复制，直接使用你的指针                           │
│  • 发送完成后，librdkafka 会调用 free(payload)                   │
│  • 你必须用 malloc() 分配，不能用 new/栈/string                  │
│  • produce() 返回后，你不能再访问 payload！                      │
│                                                                  │
│  你的代码               librdkafka                              │
│  ┌────────┐   直接使用  ┌────────────┐                          │
│  │payload │ ──────────►│   待发送   │ ────► Kafka              │
│  └────────┘            └────────────┘                           │
│      ↑                       ↓                                  │
│  不能再访问！            发送完成后                              │
│                         free(payload) ← librdkafka 自动调用     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  RK_MSG_BLOCK（阻塞模式）                                        │
│  ──────────────────────                                          │
│  • 与上面两个正交，可组合使用                                     │
│  • 当内部队列满时：                                              │
│    - 不加此标志：produce() 立即返回错误 ERR__QUEUE_FULL         │
│    - 加此标志：produce() 阻塞等待队列有空间                      │
│  • ⚠️ 必须有另一个线程调用 poll()，否则死锁！                    │
└─────────────────────────────────────────────────────────────────┘
```

### （3）核心配置
[全局配置](#config)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116154712623.png)



### （4）使用示例
#### <1>异步发送示例（生产环境首选）
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <string>
#include <csignal>
#include <unistd.h>
// 全局标志：控制生产者退出
volatile sig_atomic_t run = 1;

// 自定义投递结果回调（继承 DeliveryReportCb）
class MyDeliveryReportCb : public RdKafka::DeliveryReportCb {
public:
    void dr_cb(RdKafka::Message &msg) override {
        // 处理消息发送结果
        if (msg.err() != RdKafka::ERR_NO_ERROR) {
            // 发送失败
            std::cerr << "[发送失败] Topic：" << msg.topic_name()
                      << " 分区：" << msg.partition()
                      << " 错误：" << msg.errstr() << std::endl;
            // 可选：获取失败消息的 Key/内容（需设置 RK_MSG_COPY）
            std::string key_str = msg.key() ? *msg.key() : "无Key";
            std::cerr << "  Key：" << key_str << std::endl;
        } else {
            // 发送成功
            std::cout << "[发送成功] Topic：" << msg.topic_name()
                      << " 分区：" << msg.partition()
                      << " 偏移量：" << msg.offset() << std::endl;
        }
    }
};

// 信号处理：优雅退出
void sigterm(int sig) {
    run = 0;
}

int main() {
    // ========== 1. 初始化配置 ==========
    std::string errstr;
    // 创建全局配置（CONF_GLOBAL）
    RdKafka::Conf *global_conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    // 创建主题配置（CONF_TOPIC，可选）
    RdKafka::Conf *topic_conf = RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC);

    // 配置核心参数
    global_conf->set("bootstrap.servers", "localhost:9092", errstr); // 必填
    global_conf->set("client.id", "order-producer-01", errstr);      // 生产者标识
    global_conf->set("acks", "1", errstr);                           // ACK级别：Leader确认
    global_conf->set("retries", "3", errstr);                        // 重试3次
    global_conf->set("batch.size", "16384", errstr);                 // 批量大小16KB
    global_conf->set("linger.ms", "5", errstr);                      // 延迟5ms发送（批量）

    // 绑定投递回调（核心：异步获取发送结果）
    MyDeliveryReportCb dr_cb;
    global_conf->set("dr_cb", &dr_cb, errstr);

    // ========== 2. 创建生产者实例 ==========
    RdKafka::Producer *producer = RdKafka::Producer::create(global_conf, errstr);
    if (!producer) {
        std::cerr << "创建生产者失败：" << errstr << std::endl;
        delete global_conf;
        delete topic_conf;
        return -1;
    }

    // 释放配置对象（生产者创建后，配置已拷贝，可释放）
    delete global_conf;
    delete topic_conf;

    // ========== 3. 注册信号处理（优雅退出） ==========
    signal(SIGINT, sigterm);
    signal(SIGTERM, sigterm);

    // ========== 4. 循环发送消息 ==========
    int msg_count = 0;
    while (run && msg_count < 10) { // 发送10条测试消息
        std::string msg_payload = "order_" + std::to_string(msg_count); // 消息内容
        std::string msg_key = "user_1001";                              // 消息Key（用于分区路由）

        // 创建消息头（可选，附加元数据）
        RdKafka::Headers *headers = RdKafka::Headers::create();
        std::string trace_id = "trace_789012";
        std::string msg_type = "order_create";
        headers->add("trace-id", trace_id.c_str(), trace_id.size());
        headers->add("msg-type", msg_type.c_str(), msg_type.size());

        // 异步发送消息
        // 参数说明：
        // - "order_topic"：目标Topic
        // - RdKafka::Topic::PARTITION_UA（-1）：自动分配分区（由分区器决定）
        // - RK_MSG_COPY：拷贝消息内容（避免本地内存释放导致问题）
        // - payload/len：消息内容及长度
        // - key/key_len：消息Key及长度
        // - 0：自定义时间戳（0=使用Broker时间）
        // - headers：消息头
        // - nullptr：透传上下文（msg_opaque）

        // 异步发送消息
        RdKafka::ErrorCode err = producer->produce(
            "order_topic",
            RdKafka::Topic::PARTITION_UA,
            RdKafka::Producer::RK_MSG_COPY,
            (void*)msg_payload.data(), msg_payload.size(),
            msg_key.data(), msg_key.size(),
            0,       // 时间戳
            headers, // 消息头
            nullptr  // msg_opaque
        );

        // ⚠️ produce成功调用后 headers 会被自动释放，失败时需手动释放
        if (err != RdKafka::ERR_NO_ERROR) {
            std::cerr << "发送消息失败（即时）：" << RdKafka::err2str(err) << std::endl;
            delete headers;  // produce 失败时需手动释放 headers
            
            // 队列满时，等待并重试
            if (err == RdKafka::ERR__QUEUE_FULL) {
                producer->poll(100); // 处理内部事件，腾出队列空间
                continue;
            }
        }
        // produce() 成功：headers 已被 librdkafka 接管，禁止再 delete
        // ⚠️ 此时 headers 指针已失效，不要再使用

        // 定期调用poll：驱动回调执行、处理重试（核心！异步发送必须调用）
        producer->poll(0); // 0ms超时：非阻塞处理

        msg_count++;
        // 模拟业务延迟
        usleep(100000);
    }

    // ========== 5. 优雅关闭生产者 ==========
    std::cout << "等待所有消息发送完成..." << std::endl;
    // flush：阻塞等待所有未发送的消息完成（超时5秒）
    producer->flush(5000);

    // 检查是否有未发送完成的消息
    if (producer->outq_len() > 0) {
        std::cerr << "仍有 " << producer->outq_len() << " 条消息未发送完成" << std::endl;
    }

    // ========== 6. 释放资源 ==========
    delete producer; // 释放生产者资源
    return 0;
}
```

#### <2>同步发送示例（特殊场景使用）
```cpp
// 同步发送：通过 msg_opaque 阻塞等待结果
void sync_produce(RdKafka::Producer *producer, const std::string &topic, const std::string &payload) {
    std::string errstr;
    RdKafka::ErrorCode err;
    // 用于阻塞的信号量
    bool done = false;

    // 发送消息（msg_opaque 传递上下文）
    err = producer->produce(
        topic,
        RdKafka::Topic::PARTITION_UA,
        RdKafka::Producer::RK_MSG_COPY,
        (void*)payload.data(), payload.size(),
        nullptr, 0,
        0,        // timestamp
        nullptr,  // headers
        &done     // msg_opaque
    );

    if (err != RdKafka::ERR_NO_ERROR) {
        std::cerr << "同步发送失败：" << RdKafka::err2str(err) << std::endl;
        return;
    }

    // 阻塞等待发送结果（轮询+sleep）
    int timeout = 5000; // 5秒超时
    int poll_interval = 10;
    while (!done && timeout > 0) {
        producer->poll(poll_interval);
        timeout -= poll_interval;
        usleep(poll_interval * 1000);
    }

    if (!done) {
        std::cerr << "同步发送超时" << std::endl;
    }
}

// 需修改投递回调，设置 done 标志：
// void dr_cb(RdKafka::Message &msg) override {
//     bool *done = static_cast<bool*>(msg.opaque());
//     *done = true;
//     // ... 原有逻辑
// }
```

### （5）关键注意事项
- **异步发送**：
    - produce() **仅**将消息写入**本地环形队列**，内部 IO 线程异步发送
    - 返回 ERR_NO_ERROR **仅代表 “入队成功”**，**不代表 “发送成功”**
- **poll () 必须调用**：
    - 异步发送时，poll() 是**驱动回调执行、处理重试、清理队列**的核心，需定期调用（建议每次 produce() 后或定时调用）；
    - 若不调用 poll()，**回调不会执行，消息重试也会停滞**，最终导致发送队列满（ERR__QUEUE_FULL）
- **消息可靠性**：
    - 关闭生产者前必须调用 flush()，否则队列中的消息会丢失；
    - acks=all + retries=N 是最高可靠性配置，但会降低吞吐量；
    - 启用 enable.idempotence=true 可避免重复发送（幂等性）；

- **分区策略**：
    - 指定 partition 数值：直接发送到该分区；
    - PARTITION_UA（-1）：按 key 哈希分区（无 key 则随机）；
    - 自定义 PartitionerCb：按业务规则路由（如用户 ID 取模）；
- **消息重试**：
    - 配置 retries（重试次数）、retry.backoff.ms（重试间隔），处理网络抖动导致的发送失败；
- **压缩配置**：
    - 通过 compression.type 配置（gzip/snappy/lz4），降低网络传输量。


## 7.KafkaConsumer类（消费者类）
### （1）概述
#### <1>作用
- RdKafka::KafkaConsumer 是 librdkafka 中最常用的消费者实现类，继承自 RdKafka::Handle
- 封装了 Kafka **消费者组模式**下的消息消费能力，包括：
  - **subscribe()** 自动分区模式（依赖消费者组协调器自动分配分区）
  - **assign()** 手动分区模式（精准指定要消费的分区列表）
  - **offset 管理**（自动/手动提交）
  - **Rebalance 处理**（分区重分配）
- 是客户端从 Kafka 集群消费消息的核心入口

#### <2>核心定位
- **消费者组支持**：
    - 实现 Kafka 消费者组协议，支持自动/手动分区分配
    - 依赖 Broker 协调器（Coordinator）实现分区重平衡
- **消费模式**：
    - **subscribe()** 自动分区模式：由消费者组自动分配，支持 Rebalance
    - **assign()** 手动分区模式：精准控制分区，无自动 Rebalance
- **偏移量管理**：
    - 支持自动提交（enable.auto.commit=true）
    - 支持手动提交（commitSync/commitAsync）
- **事件驱动**：
    - **consume()** 是核心消费方法，同时触发 Rebalance、心跳、回调等事件
    - **禁止使用 poll()**，KafkaConsumer 的 consume() 方法已整合了 poll() 的功能，因此无需单独调用 poll()
- **资源管理**：
    - 继承 Handle 基类，通过 **close() + delete** 安全释放资源
- **线程安全**：
    - **非线程安全**，所有 API 需在单线程调用（或加锁）

### （2）KafkaConsumer类的定义
```cpp
class RD_EXPORT KafkaConsumer : public virtual Handle {
public:
    //=============================================================================
    // 【第一梯队】几乎每个消费者应用都会用到
    //=============================================================================

    /**
     * @brief 创建 KafkaConsumer 实例
     * @param conf 配置对象，必须设置 group.id（消费者组 ID）
     * @param errstr [out] 错误描述字符串
     * @returns 成功返回消费者实例，失败返回 NULL
     * 
     * @remark 使用 close() 关闭消费者
     * @sa RebalanceCb, CONFIGURATION.md 中的 group.id, session.timeout.ms 等配置
     */
    static KafkaConsumer *create(const Conf *conf, std::string &errstr);

    virtual ~KafkaConsumer() = 0;

    /**
     * @brief 订阅 Topic 列表
     * @param topics 要订阅的 Topic 名称列表
     * @returns ERR_NO_ERROR 成功，否则返回错误码
     * 
     * @remark 之前的订阅会被自动取消
     * @remark 支持正则表达式（以 "^" 开头，如 "^myPfx[0-9]_.*"）
     * @remark 订阅是异步操作，不可用的 Topic 会通过 consume() 返回错误
     * @remark 不存在的 Topic 返回 ERR_UNKNOWN_TOPIC_OR_PART
     * @remark 无权限的 Topic 返回 ERR_TOPIC_AUTHORIZATION_FAILED
     */
    virtual ErrorCode subscribe(const std::vector<std::string> &topics) = 0;

    /**
     * @brief 消费消息或获取错误事件（核心消费方法）
     * @param timeout_ms 超时时间（毫秒）
     * @returns Message 对象，需用 delete 释放
     * 
     * @remark 会自动触发已注册的回调（RebalanceCb、EventCb、OffsetCommitCb 等）
     * @remark 即使没有消息，也应定期调用以处理回调（尤其是 RebalanceCb）
     * @remark 返回值判断：
     *         - Message::err() == ERR_NO_ERROR：正常消息
     *         - Message::err() == ERR__TIMED_OUT：超时，无消息
     *         - 其他：错误事件
     * 
     * @warning KafkaConsumer 禁止调用 poll()，只能用 consume()
     */
    virtual Message *consume(int timeout_ms) = 0;

    /**
     * @brief 关闭消费者（同步阻塞）
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark 会依次执行：触发 Rebalance → 停止消费 → 提交偏移量 → 离开消费者组
     * @remark 最大阻塞时间约为 session.timeout.ms
     * @remark 关闭过程中可能触发回调（RebalanceCb、OffsetCommitCb 等）
     * @remark 关闭后必须用 delete 释放消费者对象
     */
    virtual ErrorCode close() = 0;

    /**
     * @brief 同步提交当前分配的所有分区偏移量
     * @returns ERR_NO_ERROR 成功，否则返回错误码
     * 
     * @remark 阻塞直到提交完成或失败
     * @remark 若注册了 OffsetCommitCb，会在后续 consume() 中回调
     */
    virtual ErrorCode commitSync() = 0;

    /**
     * @brief 异步提交当前分配的所有分区偏移量
     * @returns ERR_NO_ERROR 成功入队
     * 
     * @remark 非阻塞，提交结果通过 OffsetCommitCb 回调
     */
    virtual ErrorCode commitAsync() = 0;

    //=============================================================================
    // 【第二梯队】常用但非必需
    //=============================================================================

    /**
     * @brief 同步提交单条消息的偏移量
     * @param message 要提交的消息
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark 提交的偏移量 = message.offset() + 1（即下一条待消费位置）
     * @remark 阻塞直到提交完成
     */
    virtual ErrorCode commitSync(Message *message) = 0;

    /**
     * @brief 异步提交单条消息的偏移量
     * @param message 要提交的消息
     * @returns ERR_NO_ERROR 成功入队
     * 
     * @remark 提交的偏移量 = message.offset() + 1
     */
    virtual ErrorCode commitAsync(Message *message) = 0;

    /**
     * @brief 同步提交指定分区列表的偏移量
     * @param offsets 分区偏移量列表（offset 应为下一条待消费位置）
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode commitSync(std::vector<TopicPartition *> &offsets) = 0;

    /**
     * @brief 异步提交指定分区列表的偏移量
     * @param offsets 分区偏移量列表
     * @returns ERR_NO_ERROR 成功入队
     */
    virtual ErrorCode commitAsync(const std::vector<TopicPartition *> &offsets) = 0;

    /**
     * @brief 同步提交并通过回调获取结果
     * @param offset_commit_cb 提交完成后的回调函数
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark 回调会在此函数内部被调用（同步）
     */
    virtual ErrorCode commitSync(OffsetCommitCb *offset_commit_cb) = 0;

    /**
     * @brief 同步提交指定分区偏移量并通过回调获取结果
     * @param offsets 分区偏移量列表
     * @param offset_commit_cb 提交完成后的回调函数
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode commitSync(std::vector<TopicPartition *> &offsets,
                                 OffsetCommitCb *offset_commit_cb) = 0;

    /**
     * @brief 取消当前订阅
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode unsubscribe() = 0;

    /**
     * @brief 获取当前订阅的 Topic 列表
     * @param topics [out] 输出当前订阅的 Topic 名称
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode subscription(std::vector<std::string> &topics) = 0;

    /**
     * @brief 获取当前分配的分区列表
     * @param partitions [out] 输出当前分配的分区
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode assignment(std::vector<RdKafka::TopicPartition *> &partitions) = 0;

    /**
     * @brief 查询已提交的偏移量（从 Broker 获取）
     * @param partitions [in/out] 输入分区列表，输出时填充 offset 或 err
     * @param timeout_ms 超时时间
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode committed(std::vector<TopicPartition *> &partitions,
                                int timeout_ms) = 0;

    /**
     * @brief 获取当前消费位置（内存中的偏移量）
     * @param partitions [in/out] 输入分区列表，输出时填充 offset 或 err
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark 与 committed() 区别：position() 返回内存中的位置，committed() 查询 Broker
     */
    virtual ErrorCode position(std::vector<TopicPartition *> &partitions) = 0;

    //=============================================================================
    // 【第三梯队】手动分区管理（不使用消费者组自动分配时）
    //=============================================================================

    /**
     * @brief 手动分配分区（不使用消费者组）
     * @param partitions 要消费的分区列表
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark 与 subscribe() 互斥，手动分配时不参与消费者组 Rebalance
     */
    virtual ErrorCode assign(const std::vector<TopicPartition *> &partitions) = 0;

    /**
     * @brief 取消当前分区分配并停止消费
     * @returns ERR_NO_ERROR 成功
     */
    virtual ErrorCode unassign() = 0;

    /**
     * @brief 跳转到指定偏移量位置
     * @param partition 目标分区及偏移量
     * @param timeout_ms 超时时间（0=异步，>0=同步等待）
     * @returns ERR_NO_ERROR 成功，ERR__TIMED_OUT 超时
     * 
     * @remark 必须先通过 assign() 分配分区后才能 seek
     */
    virtual ErrorCode seek(const TopicPartition &partition, int timeout_ms) = 0;

    /**
     * @brief 存储偏移量（不立即提交，等待自动提交或手动 commit）
     * @param offsets 分区偏移量列表
     * @returns ERR_NO_ERROR 成功
     * 
     * @remark offset 值直接存储，不会 +1
     * @remark 必须设置 enable.auto.offset.store=false 才能使用
     */
    virtual ErrorCode offsets_store(std::vector<TopicPartition *> &offsets) = 0;

    //=============================================================================
    // 【第四梯队】协作式 Rebalance（COOPERATIVE 协议）
    //=============================================================================

    /**
     * @brief 增量添加分区到当前分配（协作式 Rebalance）
     * @param partitions 要添加的分区列表
     * @returns NULL 成功，否则返回 Error 对象（需 delete）
     * 
     * @remark 用于 COOPERATIVE 协议的 Rebalance 回调中处理 ERR__ASSIGN_PARTITIONS
     * @remark 也可在 Rebalance 回调外使用
     */
    virtual Error *incremental_assign(const std::vector<TopicPartition *> &partitions) = 0;

    /**
     * @brief 增量移除分区从当前分配（协作式 Rebalance）
     * @param partitions 要移除的分区列表
     * @returns NULL 成功，否则返回 Error 对象（需 delete）
     * 
     * @remark 用于 COOPERATIVE 协议的 Rebalance 回调中处理 ERR__REVOKE_PARTITIONS
     */
    virtual Error *incremental_unassign(const std::vector<TopicPartition *> &partitions) = 0;

    /**
     * @brief 获取当前 Rebalance 协议类型
     * @returns "NONE"（未加入组）、"EAGER"（急切式）、"COOPERATIVE"（协作式）
     */
    virtual std::string rebalance_protocol() = 0;

    /**
     * @brief 检查当前分配是否为非自愿丢失
     * @returns true 表示分区分配已丢失（可能已被其他消费者接管）
     * 
     * @remark 仅在 Rebalance 回调中有意义
     * @remark 分配丢失后提交偏移量可能失败
     */
    virtual bool assignment_lost() = 0;

    //=============================================================================
    // 【第五梯队】事务支持 & 高级功能
    //=============================================================================

    /**
     * @brief 获取消费者组元数据（用于事务）
     * @returns ConsumerGroupMetadata 对象（需 delete），未配置 group.id 时返回 NULL
     * 
     * @remark 用于 Producer::send_offsets_to_transaction() 的参数
     * @sa Producer::send_offsets_to_transaction()
     */
    virtual ConsumerGroupMetadata *groupMetadata() = 0;

    /**
     * @brief 异步关闭消费者（后台执行）
     * @param queue 用于接收 Rebalance 等事件的队列
     * @returns NULL 成功，否则返回 Error 对象
     * 
     * @remark 必须持续 poll 该 queue 直到 closed() 返回 true
     */
    virtual Error *close(Queue *queue) = 0;

    /**
     * @brief 检查消费者是否已关闭
     * @returns true 已关闭，false 未关闭
     * 
     * @sa close(Queue *queue)
     */
    virtual bool closed() = 0;
};

```

### （3）核心配置
[全局配置](#config)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116155332043.png)

### （4）使用示例
#### <1>基础消费示例（消费者组模式，最常用）
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <string>
#include <csignal>

// 全局标志：控制消费者退出
volatile sig_atomic_t run = 1;

// 信号处理：优雅退出
void sigterm(int sig) {
    run = 0;
}

// 自定义 Rebalance 回调（可选，用于监控分区分配变化）
class MyRebalanceCb : public RdKafka::RebalanceCb {
public:
    void rebalance_cb(RdKafka::KafkaConsumer *consumer,
                      RdKafka::ErrorCode err,
                      std::vector<RdKafka::TopicPartition *> &partitions) override {
        if (err == RdKafka::ERR__ASSIGN_PARTITIONS) {
            // 分区分配
            std::cout << "[Rebalance] 分配分区: ";
            for (auto *tp : partitions) {
                std::cout << tp->topic() << "[" << tp->partition() << "] ";
            }
            std::cout << std::endl;
            consumer->assign(partitions);
        } else if (err == RdKafka::ERR__REVOKE_PARTITIONS) {
            // 分区撤销
            std::cout << "[Rebalance] 撤销分区" << std::endl;
            consumer->unassign();
        } else {
            std::cerr << "[Rebalance] 错误: " << RdKafka::err2str(err) << std::endl;
        }
    }
};

int main() {
    std::string errstr;

    // ========== 1. 创建配置 ==========
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);

    // 必填配置
    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "my-consumer-group", errstr);  // 消费者组 ID（必填）

    // 可选配置
    conf->set("auto.offset.reset", "earliest", errstr);  // 无偏移量时从头消费
    conf->set("enable.auto.commit", "true", errstr);     // 自动提交偏移量
    conf->set("auto.commit.interval.ms", "5000", errstr); // 自动提交间隔 5秒

    // 绑定 Rebalance 回调（可选）
    MyRebalanceCb rebalance_cb;
    conf->set("rebalance_cb", &rebalance_cb, errstr);

    // ========== 2. 创建消费者实例 ==========
    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    if (!consumer) {
        std::cerr << "创建消费者失败: " << errstr << std::endl;
        delete conf;
        return -1;
    }
    delete conf;  // 配置已拷贝，可释放

    // ========== 3. 订阅 Topic ==========
    std::vector<std::string> topics = {"order_topic", "user_topic"};
    RdKafka::ErrorCode err = consumer->subscribe(topics);
    if (err != RdKafka::ERR_NO_ERROR) {
        std::cerr << "订阅失败: " << RdKafka::err2str(err) << std::endl;
        delete consumer;
        return -1;
    }
    std::cout << "已订阅 Topic: order_topic, user_topic" << std::endl;

    // ========== 4. 注册信号处理 ==========
    signal(SIGINT, sigterm);
    signal(SIGTERM, sigterm);

    // ========== 5. 消费循环 ==========
    while (run) {
        // consume() 是核心方法，会触发所有回调
        RdKafka::Message *msg = consumer->consume(1000);  // 超时 1 秒

        switch (msg->err()) {
            case RdKafka::ERR_NO_ERROR:
                // 正常消息
                std::cout << "[消息] Topic: " << msg->topic_name()
                          << " 分区: " << msg->partition()
                          << " 偏移量: " << msg->offset()
                          << " Key: " << (msg->key() ? msg->key()->c_str() : "null")
                          << " 内容: " << std::string(static_cast<const char*>(msg->payload()), msg->len())
                          << std::endl;
                break;

            case RdKafka::ERR__TIMED_OUT:
                // 超时，无消息（正常情况）
                break;

            case RdKafka::ERR__PARTITION_EOF:
                // 到达分区末尾
                std::cout << "[EOF] 分区 " << msg->partition() << " 已读取完毕" << std::endl;
                break;

            default:
                // 错误
                std::cerr << "[错误] " << msg->errstr() << std::endl;
                break;
        }

        delete msg;  // ⚠️ 必须释放消息对象
    }

    // ========== 6. 优雅关闭 ==========
    std::cout << "正在关闭消费者..." << std::endl;
    consumer->close();  // 阻塞等待：Rebalance → 提交偏移量 → 离开组
    delete consumer;

    std::cout << "消费者已关闭" << std::endl;
    return 0;
}
```

#### <2>手动提交偏移量示例（精确控制）
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>

int main() {
    std::string errstr;
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);

    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "manual-commit-group", errstr);
    conf->set("enable.auto.commit", "false", errstr);  // ⚠️ 禁用自动提交
    conf->set("auto.offset.reset", "earliest", errstr);

    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    delete conf;

    consumer->subscribe({"order_topic"});

    int msg_count = 0;
    bool running = true;

    while (running && msg_count < 100) {
        RdKafka::Message *msg = consumer->consume(1000);

        if (msg->err() == RdKafka::ERR_NO_ERROR) {
            // 处理消息
            std::cout << "处理消息: " << msg->offset() << std::endl;
            msg_count++;

            // 方式1：每条消息立即同步提交（最安全，但性能差）
            // consumer->commitSync(msg);

            // 方式2：每 10 条消息批量提交
            if (msg_count % 10 == 0) {
                RdKafka::ErrorCode err = consumer->commitSync();
                if (err == RdKafka::ERR_NO_ERROR) {
                    std::cout << "已提交偏移量（批次 " << msg_count / 10 << "）" << std::endl;
                } else {
                    std::cerr << "提交失败: " << RdKafka::err2str(err) << std::endl;
                }
            }

            // 方式3：异步提交（高性能，但可能丢失确认）
            // consumer->commitAsync(msg);
        }

        delete msg;
    }

    // 退出前提交剩余偏移量
    consumer->commitSync();

    consumer->close();
    delete consumer;
    return 0;
}
```

#### <3> 手动分区分配示例（不使用消费者组）
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>

int main() {
    std::string errstr;
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);

    conf->set("bootstrap.servers", "localhost:9092", errstr);
    // ⚠️ 手动分配模式下 group.id 可选（不参与消费者组协调）
    conf->set("group.id", "manual-assign-group", errstr);

    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    delete conf;

    // 手动指定要消费的分区和起始偏移量
    std::vector<RdKafka::TopicPartition *> partitions;
    partitions.push_back(RdKafka::TopicPartition::create("order_topic", 0, 0));     // 分区0，从头开始
    partitions.push_back(RdKafka::TopicPartition::create("order_topic", 1, 100));   // 分区1，从偏移量100开始

    // ⚠️ 使用 assign() 而非 subscribe()
    RdKafka::ErrorCode err = consumer->assign(partitions);
    if (err != RdKafka::ERR_NO_ERROR) {
        std::cerr << "分配分区失败: " << RdKafka::err2str(err) << std::endl;
    }

    // 释放 TopicPartition 对象
    for (auto *tp : partitions) {
        delete tp;
    }

    // 消费循环
    for (int i = 0; i < 50; i++) {
        RdKafka::Message *msg = consumer->consume(1000);
        if (msg->err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "分区 " << msg->partition() 
                      << " 偏移量 " << msg->offset() 
                      << " 内容: " << std::string((char*)msg->payload(), msg->len()) 
                      << std::endl;
        }
        delete msg;
    }

    consumer->unassign();  // 取消分配
    consumer->close();
    delete consumer;
    return 0;
}
```

#### <4>Seek 跳转偏移量示例
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>

void seek_to_offset(RdKafka::KafkaConsumer *consumer, 
                    const std::string &topic, int partition, int64_t offset) {
    // 创建 TopicPartition 对象
    RdKafka::TopicPartition *tp = RdKafka::TopicPartition::create(topic, partition, offset);

    // 同步 seek（等待 5 秒）
    RdKafka::ErrorCode err = consumer->seek(*tp, 5000);
    if (err == RdKafka::ERR_NO_ERROR) {
        std::cout << "已跳转到 " << topic << "[" << partition << "] 偏移量 " << offset << std::endl;
    } else {
        std::cerr << "跳转失败: " << RdKafka::err2str(err) << std::endl;
    }

    delete tp;
}

int main() {
    std::string errstr;
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "seek-demo-group", errstr);

    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    delete conf;

    // 先分配分区
    std::vector<RdKafka::TopicPartition *> partitions;
    partitions.push_back(RdKafka::TopicPartition::create("order_topic", 0));
    consumer->assign(partitions);
    for (auto *tp : partitions) delete tp;

    // 消费几条消息后跳转
    for (int i = 0; i < 5; i++) {
        RdKafka::Message *msg = consumer->consume(1000);
        if (msg->err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "当前偏移量: " << msg->offset() << std::endl;
        }
        delete msg;
    }

    // 跳转到偏移量 100
    seek_to_offset(consumer, "order_topic", 0, 100);

    // 继续消费
    for (int i = 0; i < 5; i++) {
        RdKafka::Message *msg = consumer->consume(1000);
        if (msg->err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "跳转后偏移量: " << msg->offset() << std::endl;
        }
        delete msg;
    }

    consumer->close();
    delete consumer;
    return 0;
}
```

#### <5>协作式 Rebalance 示例（COOPERATIVE 协议）
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>

// 协作式 Rebalance 回调
class CooperativeRebalanceCb : public RdKafka::RebalanceCb {
public:
    void rebalance_cb(RdKafka::KafkaConsumer *consumer,
                      RdKafka::ErrorCode err,
                      std::vector<RdKafka::TopicPartition *> &partitions) override {
        
        std::string protocol = consumer->rebalance_protocol();
        std::cout << "[Rebalance] 协议: " << protocol << std::endl;

        if (err == RdKafka::ERR__ASSIGN_PARTITIONS) {
            std::cout << "[Rebalance] 增量分配分区: ";
            for (auto *tp : partitions) {
                std::cout << tp->topic() << "[" << tp->partition() << "] ";
            }
            std::cout << std::endl;

            if (protocol == "COOPERATIVE") {
                // 协作式：增量添加
                RdKafka::Error *error = consumer->incremental_assign(partitions);
                if (error) {
                    std::cerr << "增量分配失败: " << error->str() << std::endl;
                    delete error;
                }
            } else {
                // 急切式：全量替换
                consumer->assign(partitions);
            }

        } else if (err == RdKafka::ERR__REVOKE_PARTITIONS) {
            std::cout << "[Rebalance] 增量撤销分区" << std::endl;

            if (protocol == "COOPERATIVE") {
                // 协作式：增量移除
                RdKafka::Error *error = consumer->incremental_unassign(partitions);
                if (error) {
                    std::cerr << "增量撤销失败: " << error->str() << std::endl;
                    delete error;
                }
            } else {
                // 急切式：全量撤销
                consumer->unassign();
            }
        }
    }
};

int main() {
    std::string errstr;
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);

    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "cooperative-group", errstr);
    // 使用协作式分配策略
    conf->set("partition.assignment.strategy", "cooperative-sticky", errstr);

    CooperativeRebalanceCb rebalance_cb;
    conf->set("rebalance_cb", &rebalance_cb, errstr);

    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    delete conf;

    consumer->subscribe({"order_topic"});

    // 消费循环...
    for (int i = 0; i < 100; i++) {
        RdKafka::Message *msg = consumer->consume(1000);
        if (msg->err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "消息: " << msg->offset() << std::endl;
        }
        delete msg;
    }

    consumer->close();
    delete consumer;
    return 0;
}
```

### （5）对比图
#### <1>KafkaConsumer vs Producer
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117103143018.png)

#### <2>subscribe vs assign 
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117103427689.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117103455545.png)

### （6）关键注意事项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117103734641.png)


## 8.RdKafka::Message类（消息类）

### （1）概述
#### <1>作用
- RdKafka::Message 是 librdkafka 中**消息的核心载体类**,承载消息的元数据、状态（如错误码）等关键信息，用于：
    - 生产者端：在 DeliveryReportCb 回调中获取投递结果
    - 消费者端：通过 consume() 获取拉取到的消息
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                      Message 使用场景                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Producer                              Consumer                │
│      │                                      │                   │
│      │ produce()                            │ consume()         │
│      ▼                                      ▼                   │
│   ┌──────────┐                        ┌──────────┐             │
│   │ 内部队列 │                        │ Message  │◄── 拉取结果  │
│   └────┬─────┘                        └──────────┘             │
│        │ 发送完成                                               │
│        ▼                                                       │
│   ┌────────────────┐                                           │
│   │DeliveryReportCb│                                           │
│   │   dr_cb()      │                                           │
│   │  ┌──────────┐  │                                           │
│   │  │ Message  │◄─┼── 投递结果                                 │
│   │  └──────────┘  │                                           │
│   └────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116203247930.png)

#### <2>核心定位
- **消息数据载体**：
    - 存储消息的「**内容、Key、分区、偏移量**」等“核心业务数据”，是生产者 / 消费者处理消息的直接对象；
- **元数据容器**：
    - 附带消息的「**主题名、时间戳、生产 / 消费状态、错误码**」等“元数据”，支撑消息溯源、异常排查；
- **状态标识**：
    - 不仅封装正常消息，也可表示 “空消息”“错误消息”（如消费超时、分区 EOF），是 API 反馈操作结果的重要形式；
- **内存管理差异**：
    - 消费者侧：consume() 返回的 Message 需手动 delete
    - 生产者侧：回调中的 Message 由 librdkafka 管理，无需释放

### （2）Message类的定义
```cpp
class RD_EXPORT Message {
public:
    // ================================= 错误相关 =================================
    /** @returns 如果对象表示错误事件则返回错误字符串，否则返回空字符串。 */
    virtual std::string errstr() const = 0;

    /** @returns 如果对象表示错误事件则返回错误码，否则返回 ERR_NO_ERROR（0）。 */
    virtual ErrorCode err() const = 0;
    
    // ================================= 位置信息 =================================
    /** @returns 消息对应的 RdKafka::Topic 对象（如果适用），
    *           如果没有通过 RdKafka::Topic::create() 显式创建对应的
    *           RdKafka::Topic 对象则返回 NULL。
    *           在这种情况下请改用 topic_name()。 */
    virtual Topic *topic() const = 0;

    /** @returns 主题名称（如果适用，否则返回空字符串） */
    virtual std::string topic_name() const = 0;

    /** @returns 分区号（如果适用） */
    virtual int32_t partition() const = 0;

    /** @returns 消息或错误的偏移量（如果适用） */
    virtual int64_t offset() const = 0;


    // ================================= 消息内容 =================================
    virtual void *payload() const = 0;           // 消息体指针（二进制数据，消息体指针，需转换类型）
    virtual size_t len() const = 0;              // 消息体长度
    virtual const std::string *key() const = 0;  // 消息 Key（可能为 NULL）
    virtual size_t key_len() const = 0;          // Key 长度
    virtual const void *key_pointer() const = 0; // Key 指针（二进制 Key）


    // ================================= 时间戳 =================================
    virtual MessageTimestamp timestamp() const = 0;  // 时间戳（含类型）

    virtual int64_t latency() const = 0;         // 消息延迟（微秒，仅生产者）
    
    // ================================= 消息头 =================================
    virtual Headers *headers() = 0;              // 获取消息头（可修改）
    virtual Headers *headers(ErrorCode *err) = 0;// 获取消息头，并返回错误码
    
    // ================================= 其他 =================================
    virtual int32_t broker_id() const = 0;       // 处理该消息的 Broker ID
  
    /**
    * @brief 为已消费的消息存储 offset + 1。
    *
    * 消息的 offset + 1 将根据 \c `auto.commit.interval.ms` 配置
    * 或手动无偏移量的 commit() 调用提交到 broker。
    *
    * @warning 此方法只能对当前已分配的分区调用。
    *          对未分配的分区调用将失败并返回 ERR__STATE。
    *
    * @warning 避免在调用 seek() 等方法后存储偏移量，
    *          因为这可能会干扰后续恢复暂停的分区，
    *          应在调用 seek 之前存储偏移量。
    *
    * @remark 使用此 API 时必须将 \c `enable.auto.offset.store` 设置为 "false"。
    *
    * @returns 成功时返回 NULL，失败时返回错误对象。
    */
    virtual Error *offset_store() = 0;

    /** @brief 消息持久化状态，应用程序可以用它来判断生产的消息是否已被持久化到主题日志中。 */
    enum Status {
        /** 消息从未发送到 broker，或者发送失败且错误表明消息未写入日志。
        *  应用程序重试可能导致乱序，但不会导致重复。 */
        MSG_STATUS_NOT_PERSISTED = 0,

        /** 消息已发送到 broker，但未收到确认。
        *  应用程序重试可能导致乱序和重复。 */
        MSG_STATUS_POSSIBLY_PERSISTED = 1,

        /** 消息已写入日志并被完全确认。
        *  应用程序无需重试。
        *  注意：此值仅在 \c acks=all 时才可完全信任。 */
        MSG_STATUS_PERSISTED = 2,
    };

    //@brief 返回消息在主题日志中的持久化状态。（生产者专用）
    virtual Status status() const = 0;

    virtual ~Message() = 0;
};
```

```cpp
┌─────────────────────────────────────────────────────────────┐
│                    Message 核心属性                          │
├──────────────┬──────────────────────────────────────────────┤
│   内容       │  payload() + len()  /  key() + key_len()     │
├──────────────┼──────────────────────────────────────────────┤
│   位置       │  topic_name() + partition() + offset()       │
├──────────────┼──────────────────────────────────────────────┤
│   状态       │  err() + errstr() + status()                 │
├──────────────┼──────────────────────────────────────────────┤
│   元数据     │  timestamp() + headers() + broker_id()       │
└──────────────┴──────────────────────────────────────────────┘
```

### （3）常见错误码 ErrorCode
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116205721363.png)

### （4）使用示例
#### <1>消费者接收并处理消息（核心场景）
```cpp
while (running) {
    // consume() 方法返回 Message 对象，超时时间 1000ms
    RdKafka::Message *msg = consumer->consume(1000);
    
    // 通过 err() 判断消息状态
    switch (msg->err()) {
        case RdKafka::ERR_NO_ERROR: {
            // 正常消息
            const char *payload = static_cast<const char*>(msg->payload());
            size_t len = msg->len();
            
            // key() 返回 const std::string*
            const std::string *key = msg->key();
            std::string key_str = key ? *key : "无Key";
            
            // timestamp() 返回 MessageTimestamp 结构体
            RdKafka::MessageTimestamp ts = msg->timestamp();
            std::string ts_type = (ts.type == RdKafka::MessageTimestamp::MSG_TIMESTAMP_CREATE_TIME) 
                                  ? "生产时间" : "写入时间";
            
            std::cout << "收到消息：" << std::endl
                      << "  主题：" << msg->topic_name() << std::endl
                      << "  分区：" << msg->partition() << std::endl
                      << "  偏移量：" << msg->offset() << std::endl
                      << "  Key：" << key_str << std::endl
                      << "  内容：" << std::string(payload, len) << std::endl
                      << "  时间戳：" << ts.timestamp << "（" << ts_type << "）" << std::endl;
            break;
        }
        case RdKafka::ERR__TIMED_OUT:
            // 超时，无消息，继续
            break;
            
        case RdKafka::ERR__PARTITION_EOF:
            // 到达分区末尾
            std::cout << "分区 EOF：" << msg->topic_name() 
                      << "-" << msg->partition() << std::endl;
            break;
            
        default:
            // 其他错误
            std::cerr << "消费错误：" << msg->errstr() << std::endl;
            break;
    }
    
    // ⚠️ 必须释放（关键：避免内存泄漏）
    delete msg;
}
```

#### <2>生产者发送消息后处理回调（异步发送）
```cpp
class MyDeliveryReportCb : public RdKafka::DeliveryReportCb {
public:
    // 参数是引用
    void dr_cb(RdKafka::Message &msg) override {
        if (msg.err() != RdKafka::ERR_NO_ERROR) {
            // 发送失败
            const std::string *key = msg.key();
            std::cerr << "消息发送失败：" << msg.errstr() << std::endl
                      << "  Key：" << (key ? *key : "无Key") << std::endl;
        } else {
            // 发送成功
            std::cout << "消息发送成功：" << std::endl
                      << "  主题：" << msg.topic_name() << std::endl  // ✅ 用 . 不用 ->
                      << "  分区：" << msg.partition() << std::endl
                      << "  偏移量：" << msg.offset() << std::endl;
        }
        // ⚠️ 回调中的 msg 由 librdkafka 管理，不要 delete
    }
};
```

### （5）核心特性与注意事项
#### <1>核心特征
- **只读性**：
    - 消费者侧的 Message 对象所有属性均为只读；
    - 生产者侧仅发送回调中的 Message 包含发送结果，无法修改消息内容；
- **分区号特殊值**：
    - RdKafka::Topic::PARTITION_UA（等价于 -1）代表 "让 Kafka 自动分配分区"（生产者发送时使用）；
- **内存管理**：
    - **消费者** consume() 返回的 Message 指针**需手动 delete，否则内存泄漏**；
    - **生产者回调中**的 Message 对象由 librdkafka 管理，**无需手动释放**；
- **错误消息判断**：
    - consume() 始终返回 Message 对象（不会返回 nullptr）
    - 通过 **err()** 判断状态：
        - **ERR_NO_ERROR**：正常消息
        - **ERR__TIMED_OUT**：超时，无可用消息
        - **ERR__PARTITION_EOF**：到达分区末尾

#### <2>易错点与最佳实践
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116221010502.png)

## 9.ErrorCode枚举（错误码枚举）
### （1）概述
#### <1>作用
- 是 librdkafka 中封装所有操作结果状态的**核心枚举类型**（本质是整型枚举）
- 是客户端与 Kafka 集群交互的“**状态反馈中枢**”——所有核心 API（如 produce()/assign()/commitSync()/poll()）的返回值、回调函数的入参均通过该枚举标识操作成功/失败原因
- 是错误处理、问题排查的核心依据。

#### <2>核心定位
- **类型本质**：
    - typedef int RdKafka::ErrorCode，底层是**整型枚举**（librdkafka 定义了数百个错误码常量）；
- **分层设计**：
  - **0**：
    - **正常状态**（ERR_NO_ERROR）；
  - **负数**：
    - **客户端内部错误/事件**（librdkafka 本地产生，如队列满、超时、重平衡事件）
  - **正数**：
    - **真正的错误**（如 ERR_UNKNOWN_TOPIC_OR_PART/ERR_LEADER_NOT_AVAILABLE）。

### （2）核心分类（按业务场景）
ErrorCode 可按“场景+严重程度”分为 6 大类，覆盖客户端全生命周期：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117113059844.png)

### （3）核心错误码详解
#### <1>基础通用错误码（所有场景必知）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117112530324.png)

#### <2>生产者核心错误码
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117112653492.png)

#### <3>消费者核心错误码
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117112755227.png)

#### <4> 权限/集群核心错误码
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117112844123.png)

### （4）核心注意事项
- **负数不一定是错误**：
    - 如 ERR__ASSIGN_PARTITIONS（-175）、ERR__REVOKE_PARTITIONS（-174）、ERR__TIMED_OUT（-185）是"事件标识"，仅用于业务逻辑分支，无需告警；
- **0 一定是成功**：
    - 所有 API 返回 ERR_NO_ERROR（0）均代表操作成功；
- **正数一定是错误**：
    - 如 ERR_UNKNOWN_TOPIC_OR_PART（3）、ERR_MSG_SIZE_TOO_LARGE（10）是真正的错误，需处理/告警。


## 10.Error类（错误类）
### （1）概述
#### <1>作用
- RdKafka::Error 是 librdkafka 中**封装错误详情的核心类**（区别于 RdKafka::ErrorCode 枚举）
- 它不仅包含错误码（ErrorCode），还提供了错误描述、错误来源、额外上下文等维度的信息，是对“原始错误码”的增强版封装
- 在复杂场景（如消息发送失败、重平衡异常）中，Error 类能提供比单纯错误码更丰富的排查依据，是精细化错误处理的关键。
- 该类主要出现在 Message 对象、回调函数、高级 API 返回值中，与 ErrorCode 枚举是“配套使用”的关系
    - ErrorCode 是“错误类型标识”，Error 是“错误详情载体”。

#### <2>核心定位
- **类的定位**：
    - 是**对 ErrorCode 的“包装器”**，补充错误的可读描述、来源、额外信息；
- **核心价值**：
    - 统一错误信息的获取方式（无需手动调用 err2str()）；
    - 提供错误的“上下文维度”（如错误来源的 Broker 节点、错误发生的 Topic/分区）；
    - 支持错误链（嵌套错误），便于排查多层级异常；

- **使用场景**：
  - Message::error()：返回消费/生产消息的 Error 对象；
  - 高级 API（如事务、AdminClient）的返回值；
  - 部分回调函数的入参（如事务回调）。

#### <3>与 ErrorCode 的核心区别
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117114455324.png)

### （2）Error类的定义
```cpp
class RD_EXPORT Error {
 public:
  /**
   * @brief 创建错误对象（静态工厂方法）
   * @param code 错误码枚举值
   * @param errstr 错误描述字符串指针（可为空）
   * @returns 错误对象指针，需调用方负责释放内存
   */
  static Error *create(ErrorCode code, const std::string *errstr);

  /**
   * @brief 虚析构函数（支持多态释放）
   */
  virtual ~Error() {
  }

  /*

   * ========================================
   *        错误访问器方法（按使用频率排序）
   * ========================================
   */

  /**
   * 【使用频率：★★★★★ 最高】
   * @brief 获取错误码枚举值
   * @returns 错误码，如 RdKafka::ERR_UNKNOWN_MEMBER_ID
   * @note 最常用——用于 switch/if 分支判断错误类型
   */
  virtual ErrorCode code() const = 0;

  /**
   * 【使用频率：★★★★☆ 高】
   * @brief 获取人类可读的错误描述字符串
   * @returns 错误描述，如 "Broker: Unknown member"
   * @note 常用——用于日志记录、错误提示
   */
  virtual std::string str() const = 0;

  /**
   * 【使用频率：★★★☆☆ 中】
   * @brief 判断是否为致命错误
   * @returns true 表示客户端实例不可用，需重建；false 表示可恢复
   * @note 用于决定是否需要重启客户端
   */
  virtual bool is_fatal() const = 0;

  /**
   * 【使用频率：★★★☆☆ 中】
   * @brief 判断操作是否可重试
   * @returns true 表示可重试；false 表示不可重试
   * @note 用于实现重试逻辑
   */
  virtual bool is_retriable() const = 0;

  /**
   * 【使用频率：★★☆☆☆ 较低】
   * @brief 获取错误码名称字符串
   * @returns 错误码常量名，如 "ERR_UNKNOWN_MEMBER_ID"
   * @note 主要用于调试日志、监控指标上报
   */
  virtual std::string name() const = 0;

  /**
   * 【使用频率：★☆☆☆☆ 低】
   * @brief 判断是否为需要中止的事务错误
   * @returns true 表示必须调用 abort_transaction() 中止事务；false 表示无需中止
   * @note 仅在使用事务 API 时有意义，普通生产/消费场景无需关注
   * @remark 返回值仅对事务 API 返回的错误有效
   */
  virtual bool txn_requires_abort() const = 0;
};
```

### （3）使用示例
```cpp
#include <iostream>
#include <librdkafka/rdkafkacpp.h>

// ========================================
// 示例1：生产者事务场景中的错误处理
// ========================================
void handleProducerError(RdKafka::Producer* producer) {
    RdKafka::Error* error = producer->begin_transaction();
    
    if (error) {
        // 【最常用】获取错误码进行分支判断
        RdKafka::ErrorCode code = error->code();
        
        // 【常用】记录错误日志
        std::cerr << "事务开始失败: " << error->str() << std::endl;
        std::cerr << "错误码名称: " << error->name() << std::endl;
        
        // 【中频】判断是否为致命错误
        if (error->is_fatal()) {
            std::cerr << "致命错误！需要重建生产者实例" << std::endl;
            // 重建生产者逻辑...
        }
        // 【中频】判断是否可重试
        else if (error->is_retriable()) {
            std::cerr << "可重试错误，稍后重试..." << std::endl;
            // 重试逻辑...
        }
        // 【低频】事务场景：判断是否需要中止事务
        else if (error->txn_requires_abort()) {
            std::cerr << "需要中止事务" << std::endl;
            producer->abort_transaction(10000);
        }
        
        // 释放错误对象（重要！避免内存泄漏）
        delete error;
    }
}

// ========================================
// 示例2：提交事务的完整错误处理
// ========================================
bool commitTransaction(RdKafka::Producer* producer, int timeout_ms) {
    RdKafka::Error* error = producer->commit_transaction(timeout_ms);
    
    if (!error) {
        std::cout << "事务提交成功" << std::endl;
        return true;
    }
    
    // 根据错误码进行精细化处理
    switch (error->code()) {
        case RdKafka::ERR__TIMED_OUT:
            std::cerr << "事务提交超时: " << error->str() << std::endl;
            if (error->is_retriable()) {
                delete error;
                return commitTransaction(producer, timeout_ms);  // 重试
            }
            break;
            
        case RdKafka::ERR__FENCED:
            std::cerr << "生产者被隔离，需要重建实例" << std::endl;
            break;
            
        default:
            std::cerr << "事务提交失败 [" << error->name() << "]: " 
                      << error->str() << std::endl;
            
            if (error->txn_requires_abort()) {
                producer->abort_transaction(timeout_ms);
            }
            break;
    }
    
    delete error;
    return false;
}

// ========================================
// 示例3：封装通用错误处理函数
// ========================================
enum class ErrorAction {
    SUCCESS,      // 成功，无需处理
    RETRY,        // 可重试
    ABORT_TXN,    // 需中止事务
    FATAL,        // 致命错误，需重建客户端
    FAIL          // 失败，不可恢复
};

ErrorAction analyzeError(RdKafka::Error* error) {
    if (!error) {
        return ErrorAction::SUCCESS;
    }
    
    // 记录错误信息
    std::cerr << "[ERROR] Code: " << error->code() 
              << " | Name: " << error->name()
              << " | Desc: " << error->str() << std::endl;
    
    if (error->is_fatal()) {
        return ErrorAction::FATAL;
    }
    if (error->is_retriable()) {
        return ErrorAction::RETRY;
    }
    if (error->txn_requires_abort()) {
        return ErrorAction::ABORT_TXN;
    }
    
    return ErrorAction::FAIL;
}

// 使用示例
void exampleUsage(RdKafka::Producer* producer) {
    RdKafka::Error* error = producer->begin_transaction();
    
    ErrorAction action = analyzeError(error);
    
    switch (action) {
        case ErrorAction::SUCCESS:
            std::cout << "操作成功" << std::endl;
            break;
        case ErrorAction::RETRY:
            std::cout << "准备重试..." << std::endl;
            break;
        case ErrorAction::ABORT_TXN:
            producer->abort_transaction(10000);
            break;
        case ErrorAction::FATAL:
            std::cerr << "致命错误，退出程序" << std::endl;
            exit(1);
        case ErrorAction::FAIL:
            std::cerr << "操作失败" << std::endl;
            break;
    }
    
    if (error) {
        delete error;  // 始终记得释放
    }
}
```

### （4）核心注意事项
- **内存管理**：
    - Error* 指针由 API 返回（如 begin_transaction()、commit_transaction()），**调用方负责 delete**，否则内存泄漏；
    - **核心原则**：凡是 API 返回 Error* 的，用完必须 delete。
- **关键使用原则**：
    - **常规场景优先使用 ErrorCode**：
        - 高频调用（如消息循环）中，整型比较开销极低；
    - **复杂场景使用 Error 类**：
        - 事务 API、需要判断 is_fatal()/is_retriable() 时，Error 类更合适；
    - **日志记录**：
        - 简单错误用 err2str(code)，需要上下文时用 Error::str()。
- **致命错误处理**：
    - is_fatal()=true 时，客户端无法恢复，需记录核心日志并重启客户端；


## 11.MessageTimestamp类（时间戳类）
### （1）概述
#### <1>作用
- RdKafka::MessageTimestamp 是 librdkafka 中封装 Kafka **消息时间戳**的专用类，用于统一管理消息的「**时间戳值 + 时间戳类型**」，是消费/生产场景中**获取消息时间维度信息的核心入口**。
- Kafka 消息的时间戳是其元数据的重要组成部分（如生产时间、日志追加时间），该类通过清晰的 API 封装了时间戳的类型判断、数值获取能力，解决了不同时间戳语义的区分问题。
- 该类**仅用于读取**（生产者发送消息时可指定时间戳，但最终由 Message 对象承载），**所有方法均为只读，线程安全**，是处理消息时间相关逻辑（如按时间过滤、数据回溯、延时消费）的基础。

#### <2>核心定位
- **类的定位**：
    - 轻量级值对象（Value Object），仅包含「**时间戳类型（枚举）+ 时间戳数值（毫秒级）**」两个核心属性；
- **核心价值**：
  - 区分时间戳的语义（生产时间/日志追加时间），避免业务误解；
    - 统一时间戳的单位（毫秒级 Unix 时间戳），简化跨语言/跨版本兼容；
    - 提供类型安全的 API，替代原始的“数值+标记位”模式；
- **使用场景**：
  - 消费者 consume() 获取消息后，通过 Message::timestamp() 获取该类实例；
  - 生产者 produce() 时指定自定义时间戳（最终体现在消费端的 MessageTimestamp）；
  - 按时间戳过滤消息（如仅处理近1小时的消息）、定位历史数据。

### （2）MessageTimestamp类的定义
```cpp
class RD_EXPORT MessageTimestamp {
 public:
  /**
   * @brief 消息时间戳类型枚举
   * @details 标识时间戳的来源/含义
   */
  enum MessageTimestampType {
    MSG_TIMESTAMP_NOT_AVAILABLE,  /**< 时间戳不可用（消息无时间戳信息） */
    MSG_TIMESTAMP_CREATE_TIME,    /**< 消息创建时间（由生产者在发送时设置） */
    MSG_TIMESTAMP_LOG_APPEND_TIME /**< 日志追加时间（由 Broker 在写入日志时设置） */
  };

  MessageTimestampType type; /**< 时间戳类型 */
  int64_t timestamp;         /**< 时间戳值：自 Unix 纪元（1970-01-01 00:00:00 UTC）以来的毫秒数 */
};
```

### （3）使用示例
```cpp
// 示例1：基本使用
void processMessage(RdKafka::Message* message) {
    // 获取消息时间戳
    RdKafka::MessageTimestamp ts = message->timestamp();
    
    switch (ts.type) {
        case RdKafka::MessageTimestamp::MSG_TIMESTAMP_CREATE_TIME:
            std::cout << "消息创建时间: " << ts.timestamp << " ms" << std::endl;
            break;
            
        case RdKafka::MessageTimestamp::MSG_TIMESTAMP_LOG_APPEND_TIME:
            std::cout << "Broker 追加时间: " << ts.timestamp << " ms" << std::endl;
            break;
            
        case RdKafka::MessageTimestamp::MSG_TIMESTAMP_NOT_AVAILABLE:
            std::cout << "时间戳不可用" << std::endl;
            break;
    }
    
    // 转换为可读时间（示例）
    if (ts.type != RdKafka::MessageTimestamp::MSG_TIMESTAMP_NOT_AVAILABLE) {
        time_t seconds = ts.timestamp / 1000;
        std::cout << "可读时间: " << std::ctime(&seconds);
    }
}

// 示例2：时间戳单位与转换
#include <ctime>
#include <string>
#include <iomanip>
#include <sstream>

/**
 * @brief 将毫秒级时间戳转换为本地时间字符串
 * @param ts_ms 毫秒级 Unix 时间戳
 * @returns 格式化字符串，如 "2025-01-01 08:00:00.123"
 * @note 线程安全版本
 */
std::string timestamp_to_local(int64_t ts_ms) {
    time_t ts_sec = ts_ms / 1000;
    tm local_tm;
    
#ifdef _WIN32
    localtime_s(&local_tm, &ts_sec);  // Windows
#else
    localtime_r(&ts_sec, &local_tm);  // POSIX (Linux/macOS)
#endif
    
    std::ostringstream oss;
    oss << std::put_time(&local_tm, "%Y-%m-%d %H:%M:%S") 
        << "." << std::setfill('0') << std::setw(3) << (ts_ms % 1000);
    return oss.str();
}
```

### （4）注意事项
- **时间戳单位**：
    - 毫秒级 Unix 时间戳，转秒级需 “/1000”
- **时区**：
    - 存储的是 UTC 时间，显示时需转换本地时区
- **配置依赖**：
    - Topic 级别 **message.timestamp.type** 决定使用 **CreateTime** 还是 LogAppendTime**

<a id="cb"></a>
## 12.RdKafka::xxxCb回调类
- RdKafka::xxxCb 是 librdkafka 中一组**抽象回调基类**的统称，用于实现客户端与 Kafka 集群交互的 “**异步事件通知” 机制**
    - 客户端通过继承这些回调类并实现纯虚函数，**可监听生产 / 消费过程中的关键事件**
    - 如:消息发送结果、消费错误、重平衡、日志输出等，是实现**异步化、定制化业务**逻辑的核心。

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116110908752.png)
### （1）DeliveryReportCb（生产者投递结果回调）
- **作用**：
    - 生产者**异步发送消息（produce()）**后，通过该回调获取最终发送结果，是**处理消息发送失败**的核心入口；
- **注意**：
    - **仅异步发送生效**，同步发送（produce()+flush()）无需该回调；

#### <1>定义
```cpp
class RD_EXPORT DeliveryReportCb {
public:
    // @brief 消息投递报告回调
    virtual void dr_cb(Message &message) = 0;
    virtual ~DeliveryReportCb() {}
};
```
#### <2>使用示例
```cpp
// 1. 自定义投递结果回调类
class MyDeliveryReportCb : public RdKafka::DeliveryReportCb {
public:
    void dr_cb(RdKafka::Message &msg) override {
        if (msg.err() != RdKafka::ERR_NO_ERROR) {
            std::cerr << "[发送失败] 主题：" << msg.topic_name()
                      << " 分区：" << msg.partition() 
                      << " 错误：" << msg.errstr() << std::endl;
        } else {
            std::cout << "[发送成功] 主题：" << msg.topic_name() 
                      << " 分区：" << msg.partition() 
                      << " 偏移量：" << msg.offset() << std::endl;
        }
    }
};

// 2. 绑定到生产者配置
MyDeliveryReportCb dr_cb;
producer_conf->set("dr_cb", &dr_cb, errstr);
```

### （2）EventCb（客户端底层事件回调）
- **作用**：
    - 监听 librdkafka 底层核心事件（如 Broker 连接 / 断开、元数据更新、分区 Leader 切换、限流），是精细化监控客户端状态、调试底层问题的关键；
- **区别于 ErrorCb**：
    - ErrorCb 仅监听 “错误事件”，EventCb 覆盖 “错误 + 正常事件”（如连接成功、元数据更新），粒度更细
- **事件类型**：
    - 通过 **Event::type()** 获取，核心类型包括：
        - **EVENT_ERROR**：错误事件（等价于 ErrorCb）；
        - **EVENT_REBALANCE**：重平衡事件（等价于 RebalanceCb）；
        - **EVENT_LOG**：日志事件（等价于 LogCb）；
        - **EVENT_THROTTLE**：限流事件（等价于 ThrottleCb）；
        - **EVENT_CONNECT**：Broker 连接成功；
        - **EVENT_DISCONNECT**：Broker 连接断开；
        - **EVENT_METADATA_UPDATED**：元数据更新（如 Topic 分区变化）；

#### <1>定义
```cpp
class RD_EXPORT EventCb {
public:
    // @brief 事件回调
    virtual void event_cb(Event &event) = 0;
    virtual ~EventCb() {}
};
```

#### <2>使用示例
```cpp
// 1. 自定义底层事件回调类
class MyEventCb : public RdKafka::EventCb {
public:
    // 纯虚函数实现：event=事件对象（包含事件类型、详情、关联的 Broker/分区）
    void event_cb(RdKafka::Event &event) override {
        // 根据事件类型分类处理
        switch (event.type()) {
            case RdKafka::Event::EVENT_CONNECT:
                // Broker 连接成功
                std::cout << "[事件] 连接 Broker 成功：" << event.broker_name() << std::endl;
                break;
            case RdKafka::Event::EVENT_DISCONNECT:
                // Broker 连接断开
                std::cerr << "[事件] 断开 Broker 连接：" << event.broker_name() << std::endl;
                break;
            case RdKafka::Event::EVENT_METADATA_UPDATED:
                // 元数据更新（如 Topic 分区数变化、Leader 切换）
                std::cout << "[事件] 元数据更新：" << event.str() << std::endl;
                break;
            case RdKafka::Event::EVENT_THROTTLE:
                // 被 Broker 限流（等价于 ThrottleCb）
                std::cerr << "[事件] 被 Broker 限流：" << event.broker_name()
                          << " 限流时长：" << event.throttle_time_ms() << "ms" << std::endl;
                break;
            case RdKafka::Event::EVENT_ERROR:
                // 错误事件（等价于 ErrorCb）
                std::cerr << "[事件] 错误：" << event.str()
                          << " 是否致命：" << (event.fatal() ? "是" : "否") << std::endl;
                break;
            case RdKafka::Event::EVENT_REBALANCE:
                // 重平衡事件（等价于 RebalanceCb）
                std::cout << "[事件] 重平衡：" << event.str() << std::endl;
                break;
            default:
                // 其他事件（如日志、统计）
                std::cout << "[事件] 未知类型：" << event.type() << " 详情：" << event.str() << std::endl;
                break;
        }
    }
};

// 2. 绑定到全局配置（生产者/消费者均可）
MyEventCb event_cb;
RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
if (conf->set("event_cb", &event_cb, errstr) != RdKafka::CONF_OK) {
    std::cerr << "绑定底层事件回调失败：" << errstr << std::endl;
    exit(1);
}
```


### （3）RebalanceCb（消费者重平衡回调）
- **作用**：
    - 消费者组发生“**重平衡**”时触发，是保证消费不丢数据、清理分区资源的关键；
- **重平衡场景**：
    - 消费者加入 / 退出组、Topic 分区数调整、Broker 重选举；
#### <1>定义
```cpp
class RD_EXPORT RebalanceCb {
public:
 /**
  * @brief 消费者组重平衡回调，配合 RdKafka::KafkaConsumer 使用
  *
  * 注册 rebalance_cb 后，librdkafka 将关闭自动分区分配/撤销机制，
  * 改由应用程序的 rebalance_cb 全权负责。
  *
  * 重平衡回调需要根据两种事件更新 librdkafka 的分区分配集合：
  * - RdKafka::ERR__ASSIGN_PARTITIONS（分配分区）
  * - RdKafka::ERR__REVOKE_PARTITIONS（撤销分区）
  * 同时也要能处理其他任意的重平衡错误（err 不是上述两种的情况）。
  * 
  * @remark 对于其他错误情况，应用程序必须调用 unassign() 来同步状态。
  *
  * 【Eager 全量分配策略】（如 range、roundrobin）：
  *   - 分配时调用 assign() 设置全部分区
  *   - 撤销时调用 unassign() 清空全部分区
  *
  * 【Cooperative 增量分配策略】（如 cooperative-sticky）：
  *   - 分配时调用 incremental_assign() 增量添加分区
  *   - 撤销时调用 incremental_unassign() 增量移除分区
  *
  * 不注册回调时，librdkafka 会自动完成上述操作。
  * 注册回调后，应用程序可以在分配/撤销时执行额外操作，例如：
  *   - 分配时：从外部存储加载自定义 offset
  *   - 撤销时：手动提交 offset（当 auto.commit.enable=false 时）
  *
  * @sa RdKafka::KafkaConsumer::assign()
  * @sa RdKafka::KafkaConsumer::incremental_assign()
  * @sa RdKafka::KafkaConsumer::incremental_unassign()
  * @sa RdKafka::KafkaConsumer::assignment_lost()
  * @sa RdKafka::KafkaConsumer::rebalance_protocol()
  */
 virtual void rebalance_cb(RdKafka::KafkaConsumer *consumer,
                           RdKafka::ErrorCode err,
                           std::vector<TopicPartition *> &partitions) = 0;

 virtual ~RebalanceCb() {
 }
};
```

#### <2>使用示例
```cpp
class MyRebalanceCb : public RdKafka::RebalanceCb {
public:
    void rebalance_cb(RdKafka::KafkaConsumer *consumer,
                      RdKafka::ErrorCode err,
                      std::vector<RdKafka::TopicPartition*> &partitions) override {
        if (err == RdKafka::ERR__ASSIGN_PARTITIONS) {
            std::cout << "[重平衡] 分配分区数：" << partitions.size() << std::endl;
            for (auto tp : partitions) {
                tp->set_offset(RdKafka::Topic::OFFSET_STORED);
            }
            consumer->assign(partitions);
        } else if (err == RdKafka::ERR__REVOKE_PARTITIONS) {
            std::cout << "[重平衡] 回收分区数：" << partitions.size() << std::endl;
            consumer->commitSync();  // 提交当前 offset
            consumer->unassign();   //取消分区分配
        } else {
            std::cerr << "[重平衡失败] 错误：" << RdKafka::err2str(err) << std::endl;
            consumer->unassign();   //取消分区分配
        }
    }
};

// 绑定到消费者配置
MyRebalanceCb rebalance_cb;
consumer_conf->set("rebalance_cb", &rebalance_cb, errstr);
```

### （4）OffsetCommitCb（消费者 Offset 提交结果回调）
- **作用**：
    - 消费者**异步提交 offset**（commitAsync()）后，通过该回调获取提交结果；
    - 同步提交（commitSync()）无需此回调，可直接通过返回值判断结果；
- **关键场景**：
    - 异步提交 offset 是生产环境首选（非阻塞），该回调是监控 offset 提交失败的核心入口，避免因提交失败导致重复消费；

#### <1>定义
```cpp
class RD_EXPORT OffsetCommitCb {
public:
 /**
  * @brief 设置 offset 提交回调，用于消费者组
  *
  * 自动提交或手动提交 offset 的结果会被调度到此回调，
  * 并通过 RdKafka::KafkaConsumer::consume() 触发执行。
  *
  * 如果没有分区有有效的 offset 可提交，此回调会以 err == ERR__NO_OFFSET 被调用，
  * 这不应被视为错误。
  *
  * offsets 列表包含每个分区的信息：
  *   - topic      提交的 Topic 名称
  *   - partition  提交的分区号
  *   - offset     已提交的 offset（尝试提交的值）
  *   - err        提交错误码
  */
 virtual void offset_commit_cb(RdKafka::ErrorCode err,
                               std::vector<TopicPartition *> &offsets) = 0;

 virtual ~OffsetCommitCb() {
 }
};
```

#### <2>使用示例
**自动/手动提交 offset 完成后触发**
- 提交重试、消费进度监控、lag 告警
```cpp
// 1. 自定义 Offset 提交结果回调类
class MyOffsetCommitCb : public RdKafka::OffsetCommitCb {
public:
    // 纯虚函数实现：
    // err=整体提交错误码，partitions=提交的分区列表（含每个分区的提交结果）
    void offset_commit_cb(RdKafka::ErrorCode err,
                          std::vector<RdKafka::TopicPartition*> &partitions) override {
        if (err != RdKafka::ERR_NO_ERROR) {
            // 整体提交失败（如网络异常、无权限）
            std::cerr << "[Offset 提交失败] 整体错误：" << RdKafka::err2str(err) << std::endl;
        }

        // 遍历分区，检查每个分区的提交结果（精细化处理）
        for (auto tp : partitions) {
            if (tp->err() != RdKafka::ERR_NO_ERROR) {
                // 单个分区提交失败
                std::cerr << "[Offset 提交失败] 主题：" << tp->topic()
                          << " 分区：" << tp->partition()
                          << " 错误：" << RdKafka::err2str(tp->err()) << std::endl;
                // 业务逻辑：记录失败的 offset、触发重试、告警
            } else {
                // 单个分区提交成功
                std::cout << "[Offset 提交成功] 主题：" << tp->topic()
                          << " 分区：" << tp->partition()
                          << " 提交的 offset：" << tp->offset() << std::endl;
            }
        }
    }
};

// 2. 绑定到消费者配置
MyOffsetCommitCb offset_commit_cb;
std::string errstr;
RdKafka::Conf *consumer_conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
// 关键：将回调对象绑定到 "offset_commit_cb" 配置项
if (consumer_conf->set("offset_commit_cb", &offset_commit_cb, errstr) != RdKafka::CONF_OK) {
    std::cerr << "绑定 Offset 提交回调失败：" << errstr << std::endl;
    exit(1);
}

// 3. 异步提交 Offset（触发回调）
std::vector<RdKafka::TopicPartition*> partitions;
partitions.push_back(RdKafka::TopicPartition::create("topic1", 0));
partitions[0]->set_offset(1000);
// 异步提交：非阻塞，结果通过 OffsetCommitCb 返回
consumer->commitAsync(partitions, nullptr); // 第二个参数为 opaque（透传上下文）
```

### （5）回调类通用规则与注意事项
#### <1>核心通用规则
- **抽象类特性**：
    - 所有 xxxCb 类均包含纯虚函数，必须实现所有纯虚函数才能实例化（未实现会编译失败）；
- **线程安全**：
    - 回调函数由 librdkafka 内部线程调用（**非业务线程**），需保证： 
        - 回调内操作（如写日志、更新变量）需**加锁**（std::mutex）；
        - **避免**在回调内执行**耗时操作**（如网络请求、磁盘 IO），会阻塞客户端核心逻辑；
- **生命周期管理**：
    - **回调对象的生命周期必须 ≥ 客户端对象（生产者 / 消费者）**，否则会导致野指针崩溃；
    - **禁止**在回调内销毁客户端对象（如 delete consumer）；
- **配置项命名**：
    - 绑定回调时的配置项名与回调类名对应（如 DeliveryReportCb → dr_cb，RebalanceCb → rebalance_cb），需严格匹配 librdkafka 规范。

#### <2>高频易错点
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116224028039.png)


## 13.Topic类（主题类）
### （1）概述
#### <1>作用
- 是 librdkafka 中封装 Kafka **Topic（主题）句柄**的类，提供主题名称访问、
偏移量常量定义及分区可用性检查等功能，是客户端与**具体 Kafka 主题**交互的 “具象化载体”
- 它不仅定义了主题的名称、分区数、专属配置等核心属性，还提供了消费偏移量控制的关键常量（如 OFFSET_BEGINNING），是**生产者指定发送目标、消费者控制消费范围**的基础类。

#### <2>核心定位
- **类的定位**：
    - 轻量级封装类，代表一个具体的 Kafka 主题，是 “主题元数据 + 主题级配置 + 核心常量” 的集合体；
- **核心特性**：
    - **不可修改**：
        - Topic 实例创建后，名称、配置等核心属性只读；
    - **轻量级**：
        - 无复杂内部状态，拷贝 / 销毁成本极低；
    - **工厂模式创建**：
        - 仅能通过 Topic::create() 静态方法实例化，避免手动构造。
- **核心价值**：
    - **分离 “全局配置” 与 “主题级配置”**：
        - 支持为不同 Topic 设置差异化配置（如 Topic A 用 acks=all，Topic B 用 acks=1）；
    - **提供消费偏移量的标准化常量**：
        - 统一消费起始位置的控制逻辑（如从最开始 / 最新消息消费）；
    - **封装主题元数据查询**：
        - 简化分区数、配置等信息的获取；
- ⚠️**注意**：
    - 大多数常用配置（如 acks、retries、compression.type），实际是**全局配置**，真正的 Topic 级配置项较少（如 partitioner）。
    - **现代用法中，Topic 配置的使用场景已大幅减少。**       


### （2）Topic类的定义
```cpp
class RD_EXPORT Topic {
 public:
  //==========================================================================
  // 静态常量（使用频率：⭐⭐⭐⭐⭐）
  //==========================================================================

  /**
   * @brief 未分配分区标识
   * @details 生产者发送消息时使用此值，表示由分区器自动选择目标分区
   * @note 值为 -1，是 produce() 最常用的分区参数
   */
  static const int32_t PARTITION_UA;

  /**
   * @brief 从分区最早的消息开始消费
   * @details 值为 -2，用于数据回溯、全量消费历史数据
   */
  static const int64_t OFFSET_BEGINNING;

  /**
   * @brief 从分区最新的消息开始消费
   * @details 值为 -1，仅消费后续新增消息，用于实时业务场景
   */
  static const int64_t OFFSET_END;

  /**
   * @brief 使用已存储的偏移量
   * @details 值为 -1000，从 Broker 中恢复上次消费位置，用于断点续传
   */
  static const int64_t OFFSET_STORED;

  /**
   * @brief 无效偏移量标识
   * @details 值为 -1001，用于标记异常场景或偏移量转换失败
   */
  static const int64_t OFFSET_INVALID;

  //==========================================================================
  // 静态工厂方法（使用频率：⭐⭐⭐⭐⭐）
  //==========================================================================

  /**
   * @brief 创建 Topic 句柄
   * 
   * @param base      关联的 Producer 或 Consumer 句柄
   * @param topic_str Topic 名称
   * @param conf      可选的 Topic 配置（可为 nullptr 使用默认配置）
   * @param errstr    [out] 错误信息输出
   * 
   * @returns 成功返回 Topic 指针，失败返回 nullptr
   * 
   * @note conf 对象在调用后可复用
   * @note 调用者负责 delete 返回的 Topic 对象
   * 
   * @code
   * std::string errstr;
   * RdKafka::Topic* topic = RdKafka::Topic::create(
   *     producer, "my-topic", nullptr, errstr);
   * if (!topic) {
   *     std::cerr << "创建 Topic 失败: " << errstr << std::endl;
   * }
   * @endcode
   */
  static Topic *create(Handle *base,
                       const std::string &topic_str,
                       const Conf *conf,
                       std::string &errstr);

  //==========================================================================
  // 实例方法（使用频率：⭐⭐⭐⭐）
  //==========================================================================

  /**
   * @brief 获取 Topic 名称
   * @returns Topic 名称字符串
   */
  virtual std::string name() const = 0;

  //==========================================================================
  // 实例方法（使用频率：⭐⭐）
  //==========================================================================

  /**
   * @brief 检查指定分区是否可用（有 Leader）
   * 
   * @param partition 分区编号
   * @returns true 表示分区可用，false 表示不可用
   * 
   * @warning 仅能在 PartitionerCb 回调函数内部调用！
   */
  virtual bool partition_available(int32_t partition) const = 0;

  //==========================================================================
  // 已废弃方法（使用频率：⭐ - 不推荐使用）
  //==========================================================================

  /**
   * @brief 存储偏移量（已废弃）
   * 
   * @param partition 分区编号
   * @param offset    偏移量（实际存储 offset + 1）
   * 
   * @returns ERR_NO_ERROR 成功，其他值表示错误
   * 
   * @deprecated 此 API 不支持分区 Leader Epoch，存在日志截断风险
   *             请使用 KafkaConsumer::offsets_store() 或 Message::offset_store() 替代
   * 
   * @note 使用前必须设置 enable.auto.offset.store = false
   */
  virtual ErrorCode offset_store(int32_t partition, int64_t offset) = 0;

  //==========================================================================
  // 底层访问（使用频率：⭐ - 仅特殊场景）
  //==========================================================================

  /**
   * @brief 获取底层 C API 句柄
   * 
   * @returns rd_kafka_topic_t* 指针
   * 
   * @warning 直接调用 C API 无官方支持，仅在 C++ API 无法满足需求时使用
   * @note 返回指针生命周期与 Topic 对象相同
   * @note 使用前需 #include <rdkafka/rdkafka.h>
   */
  virtual struct rd_kafka_topic_t *c_ptr() = 0;

  //==========================================================================
  // 析构函数
  //==========================================================================

  /**
   * @brief 虚析构函数
   * @note 纯虚函数，Topic 为抽象基类
   */
  virtual ~Topic() = 0;
};
```

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117160922667.png)
- **注意**：OFFSET_STORED 的行为依赖配置项 auto.offset.reset：
    - 若无已存储 offset，会根据 auto.offset.reset 决定回退到 earliest 或 latest
    - 默认值为 latest

### （3）使用示例
#### <1>Producer 创建 Topic 句柄并查询元数据
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <string>

int main() {
    std::string errstr;

    // 1. 创建全局配置（客户端级）
    RdKafka::Conf *global_conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    global_conf->set("bootstrap.servers", "localhost:9092", errstr);
    global_conf->set("client.id", "topic-demo-producer", errstr);
    
    global_conf->set("acks", "all", errstr); 
    global_conf->set("retries", "5", errstr); 
    global_conf->set("compression.type", "lz4", errstr); 

    // 2. 创建 Topic 级配置（仅包含真正的 Topic 级配置项）
    RdKafka::Conf *topic_conf = RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC);
    // Topic 级配置示例（实际可用的配置项）：
    // topic_conf->set("partitioner", "consistent_random", errstr);  // 分区策略

    // 3. 将 Topic 配置绑定到全局配置（可选）
    global_conf->set("default_topic_conf", topic_conf, errstr);

    // 4. 创建 Producer 实例
    RdKafka::Producer *producer = RdKafka::Producer::create(global_conf, errstr);
    if (!producer) {
        std::cerr << "创建 Producer 失败：" << errstr << std::endl;
        delete topic_conf;
        delete global_conf;
        return -1;
    }
    delete global_conf;  // Producer 创建后可释放
    delete topic_conf;   // 已绑定到 global_conf，可释放

    // 5. 创建 Topic 实例
    RdKafka::Topic *topic = RdKafka::Topic::create(producer, "order_topic", nullptr, errstr);
    if (!topic) {
        std::cerr << "创建 Topic 实例失败：" << errstr << std::endl;
        delete producer;
        return -1;
    }

    // 6. 验证 Topic 元数据
    std::cout << "Topic 名称：" << topic->name() << std::endl;
    
    // 获取分区数需要通过 Metadata API
    RdKafka::Metadata *metadata = nullptr;
    if (producer->metadata(false, topic, &metadata, 5000) == RdKafka::ERR_NO_ERROR) {
        // 遍历 Topic 元数据获取分区数
        for (const RdKafka::TopicMetadata* t : *metadata->topics()){
            if (t->topic() == topic->name()) {
                std::cout << "Topic 分区数：" << t->partitions()->size() << std::endl;
                break;
            }
        }
        delete metadata;
    } else {
        std::cerr << "加载 Topic 元数据失败" << std::endl;
    }

    // 7. 销毁资源
    delete topic;
    delete producer;
    
    // 8. 等待后台线程清理（可选但推荐）
    RdKafka::wait_destroyed(5000);
    
    return 0;
}
```

#### <2> Consumer 使用 Topic 偏移量常量控制消费位置
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <vector>
#include <string>
#include <csignal>
#include <cstring>

// 全局运行标志，用于优雅退出
static volatile sig_atomic_t running = 1;

// 信号处理函数
static void sigterm_handler(int sig) {
    running = 0;
}

int main() {
    std::string errstr;

    // 注册信号处理（支持 Ctrl+C 优雅退出）
    signal(SIGINT, sigterm_handler);
    signal(SIGTERM, sigterm_handler);

    // 1. 创建消费者配置
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    if (conf->set("bootstrap.servers", "localhost:9092", errstr) != RdKafka::Conf::CONF_OK ||
        conf->set("group.id", "offset-demo-group", errstr) != RdKafka::Conf::CONF_OK ||
        conf->set("enable.auto.commit", "false", errstr) != RdKafka::Conf::CONF_OK ||
        conf->set("auto.offset.reset", "earliest", errstr) != RdKafka::Conf::CONF_OK) {
        std::cerr << "配置设置失败：" << errstr << std::endl;
        delete conf;
        return -1;
    }

    // 2. 创建消费者
    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    if (!consumer) {
        std::cerr << "创建消费者失败：" << errstr << std::endl;
        delete conf;
        return -1;
    }
    delete conf;
    conf = nullptr;

    // 3. 构建分区列表，使用 Topic 常量指定偏移量
    std::vector<RdKafka::TopicPartition*> partitions;

    partitions.push_back(
        RdKafka::TopicPartition::create("order_topic", 0, RdKafka::Topic::OFFSET_BEGINNING));
    partitions.push_back(
        RdKafka::TopicPartition::create("order_topic", 1, RdKafka::Topic::OFFSET_END));
    partitions.push_back(
        RdKafka::TopicPartition::create("order_topic", 2, RdKafka::Topic::OFFSET_STORED));

    // 4. 应用分区分配
    RdKafka::ErrorCode err = consumer->assign(partitions);
    if (err != RdKafka::ERR_NO_ERROR) {
        std::cerr << "分配分区失败：" << RdKafka::err2str(err) << std::endl;
        for (auto tp : partitions) delete tp;
        delete consumer;
        return -1;
    }

    std::cout << "开始消费，按 Ctrl+C 退出..." << std::endl;

    // 5. 消费循环（添加退出条件）
    while (running) {
        RdKafka::Message *msg = consumer->consume(1000);
        
        if (!msg) {
            continue;  // 防御性检查
        }

        switch (msg->err()) {
            case RdKafka::ERR_NO_ERROR:
                // 安全处理 payload（可能为空或不以 null 结尾）
                if (msg->payload() && msg->len() > 0) {
                    std::cout << "分区 " << msg->partition() 
                              << " | 偏移量 " << msg->offset()
                              << " | Key: " << (msg->key() ? *msg->key() : "(null)")
                              << " | 内容: " 
                              << std::string(static_cast<const char*>(msg->payload()), msg->len())
                              << std::endl;
                }
                break;

            case RdKafka::ERR__TIMED_OUT:
                // 超时是正常的，静默处理
                break;

            case RdKafka::ERR__PARTITION_EOF:
                // 到达分区末尾（可选：输出提示）
                std::cout << "分区 " << msg->partition() << " 已到达末尾" << std::endl;
                break;

            default:
                // 其他错误
                std::cerr << "消费错误：" << msg->errstr() << std::endl;
                break;
        }
        
        delete msg;
    }

    std::cout << "\n正在关闭消费者..." << std::endl;

    // 6. 清理资源（正确顺序）
    for (auto tp : partitions) {
        delete tp;
    }
    partitions.clear();

    consumer->close();  // 先 close
    delete consumer;    // 再 delete
    
    // 等待后台线程清理
    RdKafka::wait_destroyed(5000);
    
    std::cout << "消费者已关闭" << std::endl;
    return 0;
}
```
- ⚠️**注意**：
    - OFFSET_STORED 依赖 auto.offset.reset 配置,若 Broker 中**无已存储的 offset**，会根据 auto.offset.reset 回退：
   - **earliest** → 等效于 OFFSET_BEGINNING
   - **latest** → 等效于 OFFSET_END（默认值）

### （4）注意事项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117161830580.png)

## 14.Queue类（队列类）
### （1）概述
#### <1>作用
- 是 librdkafka 中**封装客户端异步操作队列**的核心类，用于管理生产者 / 消费者的消息队列、事件队列（如投递报告、重平衡事件）
- 是实现异步 IO、回调触发的底层核心 ——> 所有异步操作（如消息发送、事件处理）均通过 Queue 类的队列机制完成，**其行为直接影响客户端的性能、可靠性和资源占用**。

#### <2>核心定位
- **类的定位**：
    - 底层封装 librdkafka 的 rd_kafka_queue_t 结构体，是**消费者端**的 "消息/事件缓冲区" 封装类；
- **核心价值**：
    - **分区级消费隔离**：
        - 可为不同分区创建独立队列，实现并行消费，避免相互阻塞；
    - **事件转发与隔离**：
        - 支持将消费者回调事件（如重平衡）转发到独立队列，便于自定义处理逻辑；
    - **细粒度 poll 控制**：
        - 支持对单个队列进行阻塞/非阻塞轮询（poll() / consume()）；

- **主要使用场景**：
    - **分区队列**：
        - 通过 Consumer::get_partition_queue() 获取指定分区的消息队列；
    - **事件队列**：
        - 通过 Handle::get_queue() 或 Queue::create() 获取/创建事件队列；
    - **独立轮询**：
        - 使用 queue->poll(timeout) 单独处理该队列的消息/事件；

### （2）Queue类的定义
```cpp
class RD_EXPORT Queue {
 public:
  //==========================================================================
  // 静态工厂方法（使用频率：⭐⭐⭐⭐⭐）
  //==========================================================================

  /**
   * @brief 创建独立队列对象
   * 
   * @param handle 关联的 Producer 或 Consumer 句柄
   * 
   * @returns 成功返回 Queue 指针，失败返回 nullptr
   * 
   * @note 调用者负责 delete 返回的 Queue 对象
   * @note 主要用于消费者端创建分区级队列或事件队列
   * 
   * @code
   * RdKafka::Queue *queue = RdKafka::Queue::create(consumer);
   * if (!queue) {
   *     std::cerr << "创建队列失败" << std::endl;
   * }
   * @endcode
   */
  static Queue *create(Handle *handle);

  //==========================================================================
  // 核心消费方法（使用频率：⭐⭐⭐⭐⭐）
  //==========================================================================

  /**
   * @brief 从队列中消费消息或获取错误事件
   * 
   * @param timeout_ms 超时时间（毫秒），0 表示非阻塞，-1 表示无限等待
   * 
   * @returns 返回以下情况之一：
   *   - 正常消息：Message::err() == ERR_NO_ERROR
   *   - 错误事件：Message::err() != ERR_NO_ERROR
   *   - 超时：Message::err() == ERR__TIMED_OUT
   * 
   * @note 使用 delete 释放返回的 Message 对象
   * @note 此方法用于包含消息的队列（消费者分区队列）
   * 
   * @code
   * RdKafka::Message *msg = queue->consume(1000);
   * if (msg->err() == RdKafka::ERR_NO_ERROR) {
   *     // 处理消息
   * }
   * delete msg;
   * @endcode
   */
  virtual Message *consume(int timeout_ms) = 0;

  //==========================================================================
  // 事件轮询方法（使用频率：⭐⭐⭐⭐）
  //==========================================================================

  /**
   * @brief 轮询队列，处理队列中的回调事件
   * 
   * @param timeout_ms 超时时间（毫秒）
   * 
   * @returns 处理的事件数量，超时返回 0
   * 
   * @warning 禁止用于包含消息的队列！仅用于事件队列
   * @note 用于驱动回调事件（如投递报告、统计信息等）
   * 
   * @code
   * // 用于事件队列
   * int events = event_queue->poll(100);
   * @endcode
   */
  virtual int poll(int timeout_ms) = 0;

  //==========================================================================
  // 队列转发方法（使用频率：⭐⭐⭐）
  //==========================================================================

  /**
   * @brief 将当前队列的内容转发/路由到目标队列
   * 
   * @param dst 目标队列，传入 nullptr 可取消转发
   * 
   * @returns ERR_NO_ERROR 成功，其他值表示错误
   * 
   * @note 调用后，两个队列的内部引用计数都会增加
   * @note 无论 dst 是否为 nullptr，调用后当前队列都不会再向消费者队列转发消息
   * 
   * @remark 使用场景：
   *   - 将多个分区队列合并到一个队列统一处理
   *   - 将事件队列转发到自定义队列
   * 
   * @code
   * // 将分区队列转发到统一队列
   * partition_queue->forward(unified_queue);
   * 
   * // 取消转发
   * partition_queue->forward(nullptr);
   * @endcode
   */
  virtual ErrorCode forward(Queue *dst) = 0;

  //==========================================================================
  // IO 事件集成方法（使用频率：⭐⭐ - 高级用法）
  //==========================================================================

  /**
   * @brief 启用队列的 IO 事件触发机制
   * 
   * @param fd      文件描述符，传入 -1 可禁用事件触发
   * @param payload 当队列有新元素入队时写入 fd 的数据
   * @param size    payload 的字节大小
   * 
   * @details 用于与基于 IO 的事件循环（如 epoll/select/libevent）集成：
   *   - 当空队列收到新元素时，librdkafka 会向 fd 写入 payload
   *   - 应用程序监听 fd 的可读事件，触发后调用 consume()/poll()
   * 
   * @note librdkafka 会维护 payload 的副本
   * @warning 使用转发队列时，仅在最终目标队列上启用 IO 事件
   * 
   * @code
   * // 创建 eventfd 用于通知
   * int efd = eventfd(0, EFD_NONBLOCK);
   * uint64_t notify_val = 1;
   * queue->io_event_enable(efd, &notify_val, sizeof(notify_val));
   * 
   * // 在事件循环中监听 efd
   * // 当 efd 可读时，调用 queue->consume() 获取消息
   * 
   * // 禁用
   * queue->io_event_enable(-1, nullptr, 0);
   * @endcode
   */
  virtual void io_event_enable(int fd, const void *payload, size_t size) = 0;

  //==========================================================================
  // 析构函数
  //==========================================================================

  /**
   * @brief 虚析构函数
   * @note 纯虚函数，Queue 为抽象基类
   */
  virtual ~Queue() = 0;
};
```
**注意**：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117163118116.png)

### （3）使用示例
#### <1>基础用法 - 创建队列并消费消息
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <string>
#include <csignal>

static volatile sig_atomic_t running = 1;

static void sigterm_handler(int sig) {
    running = 0;
}

int main() {
    std::string errstr;
    
    signal(SIGINT, sigterm_handler);
    signal(SIGTERM, sigterm_handler);

    // 1. 创建消费者配置
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "queue-demo-group", errstr);
    conf->set("auto.offset.reset", "earliest", errstr);

    // 2. 创建消费者
    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    if (!consumer) {
        std::cerr << "创建消费者失败：" << errstr << std::endl;
        delete conf;
        return -1;
    }
    delete conf;

    // 3. 订阅主题
    consumer->subscribe({"order_topic"});

    // 4. 创建独立队列
    RdKafka::Queue *queue = RdKafka::Queue::create(consumer);
    if (!queue) {
        std::cerr << "创建队列失败" << std::endl;
        delete consumer;
        return -1;
    }

    std::cout << "开始消费（使用独立队列）..." << std::endl;

    // 5. 消费循环
    while (running) {
        RdKafka::Message *msg = queue->consume(1000);
        
        if (!msg) continue;

        if (msg->err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "收到消息 | 分区: " << msg->partition()
                      << " | 偏移量: " << msg->offset()
                      << " | 内容: " 
                      << std::string(static_cast<const char*>(msg->payload()), msg->len())
                      << std::endl;
        } else if (msg->err() != RdKafka::ERR__TIMED_OUT) {
            std::cerr << "错误：" << msg->errstr() << std::endl;
        }

        delete msg;
    }

    // 6. 清理
    delete queue;
    consumer->close();
    delete consumer;
    RdKafka::wait_destroyed(5000);

    return 0;
}
```

#### <2>分区队列 - 并行消费不同分区
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <vector>
#include <thread>
#include <string>
#include <csignal>

static volatile sig_atomic_t running = 1;

static void sigterm_handler(int sig) {
    running = 0;
}

// 分区消费线程函数
void consume_partition(RdKafka::Queue *queue, int partition_id) {
    std::cout << "[线程-" << partition_id << "] 开始消费分区 " << partition_id << std::endl;
    
    while (running) {
        RdKafka::Message *msg = queue->consume(500);
        
        if (!msg) continue;

        if (msg->err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "[线程-" << partition_id << "] "
                      << "分区: " << msg->partition()
                      << " | 偏移量: " << msg->offset()
                      << " | 内容: " 
                      << std::string(static_cast<const char*>(msg->payload()), msg->len())
                      << std::endl;
        } else if (msg->err() != RdKafka::ERR__TIMED_OUT) {
            // 非超时错误才输出
            std::cerr << "[线程-" << partition_id << "] 错误: " << msg->errstr() << std::endl;
        }

        delete msg;
    }
    
    std::cout << "[线程-" << partition_id << "] 退出" << std::endl;
}

int main() {
    std::string errstr;
    
    signal(SIGINT, sigterm_handler);
    signal(SIGTERM, sigterm_handler);

    // 1. 创建消费者配置
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "parallel-queue-group", errstr);
    conf->set("auto.offset.reset", "earliest", errstr);

    // 2. 创建消费者
    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    delete conf;

    if (!consumer) {
        std::cerr << "创建消费者失败：" << errstr << std::endl;
        return -1;
    }

    // 3. 手动分配分区
    const int partition_count = 3;
    std::vector<RdKafka::TopicPartition*> partitions;
    for (int i = 0; i < partition_count; ++i) {
        partitions.push_back(RdKafka::TopicPartition::create("order_topic", i));
    }
    
    RdKafka::ErrorCode err = consumer->assign(partitions);
    if (err != RdKafka::ERR_NO_ERROR) {
        std::cerr << "分配分区失败：" << RdKafka::err2str(err) << std::endl;
        for (auto tp : partitions) delete tp;
        delete consumer;
        return -1;
    }

    // 4. 为每个分区获取独立队列（核心：使用 get_partition_queue）
    std::vector<RdKafka::Queue*> queues;
    std::vector<std::thread> threads;

    for (int i = 0; i < partition_count; ++i) {
        //通过 get_partition_queue 获取分区专属队列
        RdKafka::Queue *queue = consumer->get_partition_queue(partitions[i]);
        
        if (queue) {
            queues.push_back(queue);
            // 启动独立线程消费该分区
            threads.emplace_back(consume_partition, queue, i);
        } else {
            std::cerr << "获取分区 " << i << " 的队列失败" << std::endl;
        }
    }

    std::cout << "已启动 " << threads.size() << " 个消费线程，按 Ctrl+C 退出" << std::endl;

    // 5. 主线程等待退出信号
    while (running) {
        // 主线程需要定期 poll 以处理心跳等内部事件
        // 注意：消息已被分区队列接管，这里的 consume 主要用于保持连接
        RdKafka::Message *msg = consumer->consume(100);
        if (msg) {
            // 如果有消息到达主队列（通常不会），也处理掉
            if (msg->err() == RdKafka::ERR_NO_ERROR) {
                std::cout << "[主线程] 收到消息: " << msg->offset() << std::endl;
            }
            delete msg;
        }
    }

    // 6. 等待所有消费线程结束
    for (auto &t : threads) {
        if (t.joinable()) {
            t.join();
        }
    }

    // 7. 清理资源
    for (auto q : queues) delete q;
    for (auto tp : partitions) delete tp;
    
    consumer->close();
    delete consumer;
    RdKafka::wait_destroyed(5000);

    std::cout << "程序退出" << std::endl;
    return 0;
}


```

#### <3>队列转发 - 合并多个队列
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <string>

int main() {
    std::string errstr;

    // 1. 创建消费者
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "forward-demo-group", errstr);

    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    delete conf;

    if (!consumer) {
        std::cerr << "创建消费者失败" << std::endl;
        return -1;
    }

    // 2. 创建统一目标队列
    RdKafka::Queue *unified_queue = RdKafka::Queue::create(consumer);

    // 3. 创建多个源队列并转发到统一队列
    RdKafka::Queue *queue_a = RdKafka::Queue::create(consumer);
    RdKafka::Queue *queue_b = RdKafka::Queue::create(consumer);

    // 将 queue_a 和 queue_b 的内容转发到 unified_queue
    RdKafka::ErrorCode err_a = queue_a->forward(unified_queue);
    RdKafka::ErrorCode err_b = queue_b->forward(unified_queue);

    if (err_a == RdKafka::ERR_NO_ERROR && err_b == RdKafka::ERR_NO_ERROR) {
        std::cout << "队列转发设置成功" << std::endl;
    }

    // 4. 从统一队列消费（可以收到来自 queue_a 和 queue_b 的消息）
    for (int i = 0; i < 10; ++i) {
        RdKafka::Message *msg = unified_queue->consume(1000);
        if (msg && msg->err() == RdKafka::ERR_NO_ERROR) {
            std::cout << "收到消息: " 
                      << std::string(static_cast<const char*>(msg->payload()), msg->len())
                      << std::endl;
        }
        delete msg;
    }

    // 5. 取消转发
    queue_a->forward(nullptr);
    queue_b->forward(nullptr);

    // 6. 清理
    delete queue_a;
    delete queue_b;
    delete unified_queue;
    consumer->close();
    delete consumer;
    RdKafka::wait_destroyed(5000);

    return 0;
}
```

#### <4>IO 事件集成 - 与 epoll 配合使用
```cpp
#include <librdkafka/rdkafkacpp.h>
#include <iostream>
#include <string>
#include <sys/eventfd.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <csignal>

static volatile sig_atomic_t running = 1;

static void sigterm_handler(int sig) {
    running = 0;
}

int main() {
    std::string errstr;
    
    signal(SIGINT, sigterm_handler);
    signal(SIGTERM, sigterm_handler);

    // 1. 创建消费者
    RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
    conf->set("bootstrap.servers", "localhost:9092", errstr);
    conf->set("group.id", "io-event-demo-group", errstr);
    conf->set("auto.offset.reset", "earliest", errstr);

    RdKafka::KafkaConsumer *consumer = RdKafka::KafkaConsumer::create(conf, errstr);
    delete conf;

    if (!consumer) {
        std::cerr << "创建消费者失败" << std::endl;
        return -1;
    }

    consumer->subscribe({"order_topic"});

    // 2. 创建队列
    RdKafka::Queue *queue = RdKafka::Queue::create(consumer);

    // 3. 创建 eventfd 用于 IO 事件通知
    int efd = eventfd(0, EFD_NONBLOCK);
    if (efd < 0) {
        std::cerr << "创建 eventfd 失败" << std::endl;
        delete queue;
        delete consumer;
        return -1;
    }

    // 4. 启用 IO 事件触发
    uint64_t notify_value = 1;
    queue->io_event_enable(efd, &notify_value, sizeof(notify_value));

    // 5. 创建 epoll 实例
    int epfd = epoll_create1(0);
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = efd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &ev);

    std::cout << "使用 epoll + eventfd 监听队列事件..." << std::endl;

    // 6. 事件循环
    struct epoll_event events[10];
    while (running) {
        int nfds = epoll_wait(epfd, events, 10, 1000);
        
        for (int i = 0; i < nfds; ++i) {
            if (events[i].data.fd == efd) {
                // 读取并清除 eventfd
                uint64_t val;
                read(efd, &val, sizeof(val));

                // 处理队列中所有消息
                while (true) {
                    RdKafka::Message *msg = queue->consume(0);  // 非阻塞
                    if (!msg || msg->err() == RdKafka::ERR__TIMED_OUT) {
                        delete msg;
                        break;
                    }

                    if (msg->err() == RdKafka::ERR_NO_ERROR) {
                        std::cout << "[IO Event] 收到消息: "
                                  << std::string(static_cast<const char*>(msg->payload()), msg->len())
                                  << std::endl;
                    }
                    delete msg;
                }
            }
        }
    }

    // 7. 清理
    queue->io_event_enable(-1, nullptr, 0);  // 禁用 IO 事件
    close(epfd);
    close(efd);
    delete queue;
    consumer->close();
    delete consumer;
    RdKafka::wait_destroyed(5000);

    return 0;
}
```

### （4）注意事项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260117163442568.png)
**poll() 与 consume() 的区别（关键！）**
```cpp
// ❌ 错误：对消息队列使用 poll()
RdKafka::Queue *msg_queue = RdKafka::Queue::create(consumer);
msg_queue->poll(1000);  // 禁止！poll() 不处理消息

// ✅ 正确：消息队列使用 consume()
RdKafka::Message *msg = msg_queue->consume(1000);

// ✅ 正确：事件队列使用 poll()
RdKafka::Queue *event_queue = RdKafka::Queue::create(consumer);
event_queue->poll(1000);  // 处理回调事件
```

<a id="config"></a>
# 附录1：常用配置项
## 1.Global通用配置（生产者/消费者共用）
### （1）关键配置
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116144249346.png)

### （2）网络与连接相关（网络优化）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116144611415.png)

### （3）安全认证相关（生产环境）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116153411714.png)
### （4）元数据相关
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116151134440.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116151233728.png)


## 2.生产者专用
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119095109007.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260119095142663.png)

## 3.消费者专用
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116155332043.png)

## 4.Cb回调配置项
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260118103329248.png)