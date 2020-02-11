---
title: "mongodb集群数据基于flink同步"
author: PengXu
date: 2020-02-11 19:00:00 +0800
categories: mongodb flink
---

MongoDB集群之间数据同步，市面上常见的是使用MongoShake，如果数据量非常大，或者数据操作的很频繁，比如有大量的ttl删除操作，基于MongoShake会有大量的滞后并最终导致同步失败。本文提供基于Flink来实现集群间同步的思路和实现步骤。

## 同步原理OpLog
MongoDB的操作在ReplicaSet集群或者是分片集群中，都会记录在OpLog这个Collection中，具体来说就是 __local.oplog.rs__ 

只要将oplog中的内容在目标集群上重演一次，即可实现操作同步。

## RichParallelSourceFunction

用**Flink**来写同步程序，需要实现RichParallelSourceFunction，并发读取多个分片上的操作日志。
- 先判定集群中有多少分片
- 在RichParallelSourceFunction中读取相应分片上的OpLog，分片对应的地址通过读取 __config.shards__ 这个collection来得到

## RichSinkFunction

自定义实现HashFunction，相同ObjectId的记录分发到同一个Sink中，保证操作有序

## 坑点

为了避免Insert操作中出现duplicated id错误， 可以将**Insert**转换为**Upsert**，然后再写入到目标集群。
