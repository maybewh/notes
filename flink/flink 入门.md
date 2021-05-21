# Flink 入门

## Flink简介

​		flink是一个分布式、高性能随时可用的以及准确的流处理计算框架，flink可以对**无界数据**（流处理）和**有界数据**（批处理）进行有状态计算（flink天生支持状态计算）

### Flink基石

* CheckPoint机制

  基于Chandy-Lamport算法实现了一个分布式一致性快照，从而提供了一致性的语义

* State API（状态API）

  主要包括ValueState、ListState、MapState、BroadcastState，使用State API能够自动享受到一致性。state可以认为程序的中间计算结果或者是历史计算结果；

* time：支持事件时间和处理时间进行计算，spark streaming只能按照process time进行处理。基于事件时间计算我们可以解决数据迟到和乱序等问题。

* window：提供了更加丰富的window，基于数量，session window，同样支持滚动和窗口的计算。

### 批处理与流处理

+ 流处理：无界，实时性有要求，只需对经过程序的每条数据进行处理。

+ 批处理：有界，持久，需要对全部数据进行访问处理。

spark vs flink

spark: spark生态中把所有的计算都当做批处理，spark streaming中流处理本质上也是批处理（micro batch)

flink：flink中是把批处理（有界数据集的处理）看成是一个特殊的流处理场景；flink中所有的计算都是流式计算。

## Flink集群搭建

## DataSet开发

## DataStream开发

## Flink容错

