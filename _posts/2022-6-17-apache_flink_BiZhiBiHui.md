---
layout: post
title: 《Apache Flink 必知必会》笔记
date: 2022-06-16
tags: 计算机基础
---

## 《Apache Flink 必知必会》
### 一、走进Apache Flink
#### 概念
- Flink是一个*框架*和*分布式*处理引擎，用来对*无界*和*有界*的数据流进行*有状态*的计算。
#### 流引擎演进
- Apache Storm：纯流设计。延迟非常低，无法避免消息重复处理。
- Spark：以批为核心。解决了流计算语义正确性问题，但导致延迟比较高，10s级别延迟。
- Flink：第三代流引擎。低延迟、保证一致性语义、内置状态管理。

#### 使用场景
1. 事件驱动型应用
- 事件驱动表示一个事件会触发另一个或者是很多个后续的事件，然后这一系列事件会形成一些信息，基于这些信息需要做一定处理。
- 事件驱动型应用是一类具有状态的应用，会根据事件流中的事件触发计算、更新状态或进行外部系统操作。事件驱动型应用常见于实时计算业务中，比如：实时推荐，金融反欺诈，实时规则预警等。
2. 数据分析型应用
- 如双 11 成交额实时汇总，包括PV、UV 的统计，环比、同比的比较，这些背后都涉及到大量信息实时的分析和聚合，这些都是 Flink 非常典型的使用场景。
3. 数据管道型应用 (ETL)
- ETL（Extract-Transform-Load）是从数据源抽取/转换/加载/数据至目的端的过程。
- Flink 有非常丰富的 Connector，支持多种数据源和数据 Sink，囊括了所有主流的存储系统。另外它也有一些非常通用的内置聚合函数来完成 ETL 程序的编写，因此 ETL 类型的应用也是它非常适合的应用场景。
![](/images/apacheFlink/flnik-etl.png){:height="500px" style="margin:initial"}

#### 基本概念
##### (一) 核心概念
1. Event Streams：即事件流，事件流可以是实时的也可以是历史的。Flink 是基于流的，但它不止能处理流，也能处理批，而流和批的输入都是事件流，差别在于实时与批量。
2. State：Flink 擅长处理有状态的计算。通常的复杂业务逻辑都是有状态的，它不仅要处理单一的事件，而且需要记录一系列历史的信息，然后进行计算或者判断。
3. （Event）Time：最主要处理的问题是数据乱序的时候，一致性如何保证。
4. Snapshots：实现了数据的快照、故障的恢复，保证数据一致性和作业的升级迁移等。

##### （二）Flink 作业描述和逻辑拓扑
逻辑拓扑里面有 4 个称为算子或者是运算的单元，分别是 Source、Map 、KeyBy/Window/Apply 、Sink，我们把逻辑拓扑称为 Streaming Dataflow.

##### （三）Flink 物理拓扑
逻辑拓扑对应物理拓扑，它的每一个算子都可以并发进行处理，进行负载均衡与处理加速等。

### 二、Stream Processing with Apache Flin
#### DataStream API 概览及简单应用
1. 逻辑层次
2. 转换
3. 分区
4. 连接器

#### Flink 中的状态和时间
如果想要深入地了解 DataStream API，状态和时间是必须掌握的要点。
所有的计算都可以简单地分为无状态计算和有状态计算。无状态计算相对而言比
较容易。假设这里有个加法算子，每进来一组数据，都把它们全部加起来，然后把结
果输出去，有点纯函数的味道。纯函数指的是每一次计算结果只和输入数据有关，之



## 《剑指大数据——Flink学习精要（Java版）》
### 一、初识Flink
#### 1. 框架演变
1. lamda架构
- 使用双框架，流处理实时但不准确，批处理准确但延迟时间长。
- 使用流处理实时更新中间数据（低延迟），到某个时间点换成批处理的最终数据（准确率）。
![](/images/apacheFlink/flnik-lamda.png){:height="500px" style="margin:initial"}

2. Flnik
- 高吞吐、低延迟、结果准确、状态一致性、能与众多常用存储系统连接、高可用、动态扩展

#### 2. 分层API
1. 越顶层越抽象（易用、语义），越低层越具体（能力丰富、灵活）
2. 从高到低：SQL(最高层)、TABLE API(声明式领域)、DataStream/DataSet API（核心API）、有状态流处理（底层API）

### 二、Flink快速上手（maven）

#### 1. pom依赖
```
<properties>
    <flink.version>1.12.7</flink.version>
</properties>

<!-- artifactId里的2.12是scala版本 -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-java</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-java_2.12</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-clients_2.12</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-runtime-web_2.12</artifactId>
    <version>${flink.version}</version>
</dependency>
```

#### 2. 批处理