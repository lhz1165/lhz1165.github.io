---
layout: post
title: "2-1-Kafka-ConsumerGroup 消费者组"
category: kafka
---

Kafka-ConsumerGroup的原理

## 简介

Consumer Group 是 Kafka 对消费者进行逻辑分组的机制。同一个 Consumer Group 内的多个 Consumer 会共同消费 Topic 的 Partition，Kafka 会进行分区分配，实现负载均衡；同一个 Partition 在同一个 Group 内同一时刻只能由一个 Consumer 消费。不同 Consumer Group 之间消费进度相互独立，因此可以分别消费同一份 Topic 数据。



## GroupCoordinator

> Consumer 会先随便连接一个 Broker，然后发送请求，从内部元数据 topic（__consumer_offsets），根据group.id计算获取到某个Partition，从这个Partition保存这个Group 的 Coordinator信息。

- 管理 Consumer 加入、退出 Consumer Group
- 维护 Group 成员状态
- 触发和协调 Rebalance
- 选出 Group Leader
- 接收并保存 Consumer 提交的 offset
- 检测 Consumer 是否存活
- 每个broker都有一个

```text
GroupCoordinator
负责：
- 成员管理
- Rebalance 协调
- Leader 选举
- 分发最终分配结果

Group Leader Consumer
负责：
- 根据 assignor 算出 Partition 分配方案
- 然后把方案交给 Coordinator，再由 Coordinator 分发给所有 Consumer。
```

## Reblance

> Coordinator 发现成员发生变化，于是触发 Rebalance，实现partition重新分配，实现一个Consumer一个Partition

### 全局屏障

所有 consumer 暂停消费，撤销已拥有的 Partition

#### 1. 信息收集

consumer 向 Coordinator 发送 JoinRequest，上报自己的信息

Coordinator 回复响应，包括，partition列表，分配算法

#### 2. 分配结果结算

consumerLeader 收到响应，根据assignor，计算如何分配 partition

算法有

RoundRobin: 如果consumer变化会重新分配partition，影响效率

Sticky：减少partition变化

Range：按顺序分配

#### 3. 分配结果同步

所有consumer发送报告SyncRequest

Coordinator 从consumerLeader 收到分配方案

Coordinator 回复SyncResponse 把分配发送给 consumer

#### 缺陷

如果某个 partition Reblance后一直都归属某个 Consumer，每次还是要暂停消费，导致低效

### Incremental  Reblance

第一轮 找出哪些 partition 不会变化

第二轮 按照全局屏障剩余 partition 分配出去

### Static Membership

减少Consumer 重启的时候 发生Reblance

### Next Generation of Consumer Reblance

1. 分配策略不在委托consumer 执行，由Coordinator 来计算
2. 不再使用JoinRequest，SyncRequest等协议，而是使用心跳通信
   1. Consuemr 上报元数据
   2. Coordinator  告知 Consuemr 的partition 集合

### 总结

group.protocol配置是否使用新版协议，genertic（旧），consumer(新)

## Consumer拉取机制

获取数据最小单位是“**消息批次**”

### 健康度检测

普通心跳检测只能保证连通，不能证明消费正常因为有活锁等bug，如果检测到上层应用太久没调用poll() 方法(**max.poll.interval.ms**)，让Coordinator 把Consumer 逐出 Group

## Offset提交

### 依靠kafka存储offset

 enable.auto.commit

#### 自动提交（true）

根据上次提交的时间(auto.commit.offset.interval)，如果太久poll会顺便提交拉到消息的offset

可能出现出现消息还没处理完毕，但是已经被自动提交offset了

#### 手动提交（false）

需要手动调用commitSync方法

### 依靠外部存储 offset

业务逻辑执行和offset的提交，变成原子操作

首先从外部数据库获取offset，让后配置到kafka，将两操作封装成事务，自己更新offset
