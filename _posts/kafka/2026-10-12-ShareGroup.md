---
layout: post
title: "2-2-Kafka-ShareGroup 共享消费组"
category: kafka
---

ShareGroup 的引入理念与ConsumerGroup对比

## 简介

本文记录 Kafka ShareGroup 的消息分配方式、消息状态与确认流程，整理共享消费组按消息粒度消费的机制。内容包括消息锁、消费窗口以及显式和隐式提交方式。

打破一个分区只能被一个消费者消费的规矩，支持**消息队列**，而不是**数据流**

相比ConsumerGroup是以Partition粒度进行分配，ShareGroup 是以单个消息粒度进行划分。

## ShareParitionManager

协调消息分配，通过有时间的锁来实现消息独占（**锁有过期时间**）

- 消息状态
  Available: 可分配
- Acquire：已分配，加锁了

- Acknowledged: 已被处理
  - 锁过期之前被成功处理，从Acquire 到 Acknowledged
- Archived：已归档，不会再被分配
  - 消息重试超过上限，从Acquire 到 Archived
  - 锁过期之前被拒绝，从Acquire 到 Archived
  - 消息被滑动窗口左指针扫描到，从 Acknowledged 变成 Archived

![image-20261012163512041]({{ "/assets/kafka/ShareGroup/image-20261012163512041.png" | absolute_url }})

通过区间来存储消息状态，而不是每个消息记录，左侧一定是Archived，右侧一定是Available，中间就是即将给Consumer消费的，维护这个窗口来记录消息状态

## Rebalance

> Consumer 会先随便连接一个 Broker，然后发送请求，从内部元数据 topic（__consumer_offsets），根据group.id计算获取到某个Partition，从这个Partition保存这个Group 的 ShareParitionManager 信息。

ShareParitionManager 把Parition 分配给各个Consumer，可能一个Consumer被分配了多个Parition 。

## Consumer拉取机制

poll周期性执行

acknowledge 确认每条消息的处理结果

ACCEPT:处理成功

RELEASE：本次失败可重试

REJECT：彻底失败

RENEW：锁延期

### 依靠kafka存储offset

 share.acknoewledgement.mode

#### 显示提交（explicit）

手动调用 acknowledge  方法设置每条消息的消费状态，然后再调用commitSync 把结果同步提交给 Broker

```text
poll
 ↓
record1 -> ACCEPT
record2 -> RELEASE
record3 -> REJECT
 ↓
commitSync()
 ↓
同步提交这些状态
```

#### 隐式提交（implicit）

无需 acknowledge ，commitSync 会把消息默认标记成 **ACCEPT**
