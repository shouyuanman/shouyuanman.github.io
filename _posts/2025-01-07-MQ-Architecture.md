---
title: MQ架构
date: 2025-01-07 16:42:20 +0800
categories: [后端, MQ]
tags: [后端, MQ]
music-id: 1824564461
---

## **MQ基本模块**

一个消息队列至少要有**通信协议、网络、存储、生产者、消费者**最基础的`5`个模块，不同的`MQ`架构就是对这`5`个模块的选型设计。

![Desktop View](/assets/img/20250107/MQArchitect.png){: width="500" height="300" }
_消息队列组成部分简图_

消息队列中存储的数据分为两大类，分别是元数据和消息数据。元数据是集群维度的资源数据，包括`Topic`、`Group`、`User`、`ACL`、`Config`；消息数据是客户端写入的用户业务数据。

## **RocketMQ**

![Desktop View](/assets/img/20250107/RocketMQArchitect.png){: width="750" height="450" }
_RocketMQ_

## **Kafka**

![Desktop View](/assets/img/20250107/KafkaArchitect.png){: width="750" height="450" }
_Kafka_

## **RabbitMQ**

![Desktop View](/assets/img/20250107/RabbitMQArchitect.png){: width="750" height="450" }
_RabbitMQ_
