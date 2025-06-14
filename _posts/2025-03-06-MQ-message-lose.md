---
title: MQ怎么保证消息不丢失
date: 2025-03-06 11:54:00 +0800
categories: [后端, MQ]
tags: [后端, MQ]
music-id: 1808943074
---

## **序言**

对于业务系统来说，保证业务的正确可靠很重要，如果在业务架构中引入了消息中间件，必须要**保证数据安全，做到消息不丢失**。一旦丢消息，就意味着丢数据，在业务上是完全不能接受的，特别是资产相关的业务，比如订单、交易、支付等，数据一旦不可靠，就会产生资损。

主流`MQ`都提供了保证消息可靠性的机制，在消息传递过程中，即使发生网络中断或者宕机，也能做到不丢消息。要做到这一点，得**先知道消息从生产到消费过程中，哪些地方可能丢消息，才能知道`MQ`怎么避免丢消息**。

## **一条MQ消息的生命周期**
![Desktop View](/assets/img/20250306/mq_msg_flow_simply.png){: width="400" height="200" }
_MQ消息传输流程简图_

一条消息的生命周期分为生产、存储、消费三个阶段。
- 生产阶段，消息在`Producer`创建出来，经过网络发送到`Broker`。
- 存储阶段，消息在`Broker`存储，如果是集群，消息还会被复制到其他副本上。
- 消费阶段，`Consumer`从`Broker`拉取消息，经过网络传输到`Consumer`上。

## **消息丢失原因**
1. **生产者发送失败**：生产者在发送消息时遇到网络异常或`Broker`不可用，会导致消息发送失败。
2. **Broker存储失败**：`Broker`接收到消息后没能持久化到磁盘。
3. **Broker故障**：`Broker`发生故障或重启时，没能持久化消息。
4. **Broker主从切换**：`Broker`集群主从复制异步时，主从切换可能导致消息丢失。
5. **消费者消费失败**：消费者在消费消息时遇到异常，没能正确处理消息，导致消息没能被确认消费。

## **怎么保证消息不丢失**

### **生产者层面**
#### **消息ACK**

在生产阶段，`MQ`通过请求确认机制（`ACK`）保证消息可靠传递。`Producer`客户端把消息发送到`Broker`，`Broker`收到消息后给`Producer`返回一个确认响应，`Producer`收到响应，这时就完成了一次消息发送过程。`Producer`收到`Broker`的确认响应，就能保证消息在生产阶段不会丢失。

#### **重试**

`Producer`发送消息时遇到网络异常或`Broker`不可用，长时间没收到确认响应，会自动重试，如果重试失败，就会以返回值或异常的方式告知用户，`Producer`可以选择记录日志或将消息写入数据库进行补偿。**正确处理返回值或捕获异常，就可以保证生产阶段的消息不会丢失。比如同步发送只需捕获或异常即可，异步发送需要在回调方法里检查发送结果**。`Producer`可以配置重试次数和重试间隔，确保消息最终能成功发送。
```java
// 以RocketMQ设置为例
// 设置发送超时时间
producer.setSendMsgTimeout(5000);
// 设置重试次数
producer.setRetryTimesWhenSendFailed(3);
// 设置在发送失败时重试其他Broker
producer.setRetryAnotherBrokerWhenNotStoreOK(true);
// 设置默认队列数
producer.setDefaultTopicQueueNums(4);
```

### **Broker层面**

#### **存储策略**

`MQ`消息存储分为内存和磁盘，内存是指在`Broker`内存中读写，磁盘是指保存在磁盘上，`MQ`支持**同步刷盘**和**异步刷盘**。在同步刷盘模式下，消息写入磁盘时，会等待磁盘写入完成才返回成功响应，这样在`Broker`宕机时消息也不会丢失。在异步刷盘模式下，消息写入磁盘后立即返回成功响应，不等待磁盘写入完成。

>以RocketMQ为例，<br/>**异步刷盘**<br/>默认刷盘方式。消息写入`CommitLog`时，并不直接写入磁盘，而是先写入`PageCache`缓存后返回成功，然后用后台线程异步把消息刷入磁盘。异步刷盘提高了消息吞吐量，但可能会丢消息，比如断点导致机器停机，`PageCache`中没来得及刷盘的消息就会丢失。<br/>**同步刷盘**<br/>消息写入内存后，立刻请求刷盘线程进行刷盘，如果消息未在约定的时间内(默认`5s`)刷盘成功，就返回`FLUSH_DISK_TIMEOUT`，`Producer`收到这个响应后，可以进行重试。同步刷盘策略保证了消息的可靠性，降低了吞吐量，但增加了延迟。要开启同步刷盘，需要增加下面配置，
```java
// 配置Broker持久化策略
broker.setMessageStoreEnable(true);
broker.setFlushDiskType(SyncFlush);
```
{: .prompt-tip }

#### **主从复制**

如果`Broker`是由多个节点组成的`Master-Slave`集群，需要把`Broker`集群配置成**至少将消息发送到`2`个节点，再给客户端发送确认响应**，通过主从复制来保证数据的高可用性。这样当某个`Broker`宕机时，其他`Broker`可以替代宕机的`Broker`，也不会丢消息。主从复制有两种方式，分别是同步和异步，

- **异步复制**
    - **性能与可靠性平衡**：消息先写入主节点，然后异步地复制到从节点。主节点宕机后，从节点还没有完成消息复制，主从切换后就会有少量消息丢失，但这种方式可以在性能和可靠性之间取得平衡。
- **同步复制**
    - **高可靠性**：消息在主节点和从节点上同时进行写入操作。只有当消息成功写入从节点后，才会返回给生产者。这种模式下，即使主节点宕机，从节点也有完整的数据，不会丢失消息。

```java
// 以RocketMQ设置为例
// 配置Broker集群
broker.setMasterAddr("master-broker-host:port");
broker.setSlaveSynchronization(true);
```

>以RocketMQ设置为例，同步复制流程如下，
- `slave`初始化后，跟`master`建立连接，并向`master`发送自己的`offset`；
- `master`收到`slave`发送的`offset`后，将`offset`后面的消息批量发送给`slave`；
- `slave`把收到的消息写入`CommitLog`文件，并给`master`发送新的`offset`；
- `master`收到新的`offset`后，如果`offset >= producer`发送消息后的`offset`，给`Producer`返回`SEND_OK`。
{: .prompt-tip }

### **消费者层面**

#### **消息确认**

`Consumer`客户端从`Broker`拉取消息后，执行用户的消费业务逻辑成功后，才会给`Broker`发送消费确认响应。如果`Broker`没收到响应，下次拉消息还会返回同一条消息，确保消息不会在网络传输过程中丢失，也不会因为客户端在执行消费逻辑时出错而导致丢失。**不要在收到消息后就立即发送消费确认，应该在执行完所有消费业务逻辑后，再发送确认响应**。

>以RocketMQ为例，RocketMQ提供两种消费确认模式，<br/>
**自动确认**：消费者在处理完消息后会自动确认消费情况。<br/>
**手动确认**：消费者需要手动确认消息消费情况。只有在消费者确认后，RocketMQ才会将消息标记为已消费。
```java
// 手动提交
return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
```
{: .prompt-tip }

#### **重试、死信队列**

如果消费者在处理消息时发生异常，`MQ`会自动将该消息重新放回队列，等待下次重试。消费者可以通过配置最大重试次数（`RocketMQ`默认`16`次）来控制消息的重试策略。如果消息经过多次重试仍无法被成功消费，`MQ`会将该消息放入死信队列。死信队列是一种特殊的队列，用于存储处理失败的消息，以便后续人工干预或定期处理。

```java
// 以RocketMQ为例
// 创建死信队列
consumer.subscribe("DeadLetterQueue", "*");
// 创建重试队列
consumer.subscribe("RetryQueue", "*");
```

>注意
- `Broker`默认最多重试`16`次，如果重试`16`次都失败，就把这条消息放入死信队列，`Consumer`可以订阅死信队列进行消费。
- 重试只有在集群模式（`MessageModel.CLUSTERING`）下生效，在广播模式（`MessageModel.BROADCASTING`）下是不生效的。
- `Consumer`端一定要做好幂等处理。
- 其实重试`3`次都失败就可以说明代码有问题，这时`Consumer`可以把消息存入本地，给`Broker`返回`CONSUME_SUCCESS`来结束重试。
{: .prompt-tip }

## **总结**

可以看到，主流`MQ`都提供了完善的机制来保证消息的可靠性，但需要开发者对以上这些机制非常熟悉，且能正确使用和配置MQ，**绝大多数丢消息都是因为开发者不熟悉这些特性而导致姿势不对。每个阶段都需要正确使用并配置MQ，这样才能配合MQ的可靠性机制来确保消息不会丢失。**值得注意的是，为了保证消息的可靠性，`MQ`发送消息的速度可能会受到限制，需要在消息可靠性和性能之间`Trade-off balance`。

## **附录**

### **RocketMQ实现消息不丢失的步骤**

![Desktop View](/assets/img/20250306/rocketmq_msg_analysis.png){: width="800" height="600" }
_RocketMQ消息可靠性分析_

#### **生产者端配置**

在生产者端，我们需要配置消息发送确认机制和重试机制，确保消息能够成功发送到`Broker`。

```java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.common.message.Message;

public class Producer {
    public static void main(String[] args) throws Exception {
        // 创建生产者实例
        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroup");
        producer.setNamesrvAddr("localhost:9876");
        producer.start();

        // 发送消息
        for (int i = 0; i < 10; i++) {
            Message msg = new Message("TopicTest", "TagA", ("Hello RocketMQ " + i).getBytes());
            SendResult sendResult = producer.send(msg);
            System.out.printf("%s Send Result: %s%n", msg, sendResult);
        }

        producer.shutdown();
    }
}
```

#### **Broker端配置**

在`Broker`端，我们需要配置消息持久化机制和主从复制机制，确保消息能够持久化到磁盘，并且在主`Broker`故障时能够由从`Broker`接管。

```markdown
# Broker配置文件
# 持久化策略
messageStoreEnable=true
flushDiskType=SyncFlush

# 主从复制配置
masterAddr=masterBrokerHost:port
slaveSynchronization=true
```

#### **消费者端配置**

在消费者端，我们需要配置消息确认机制，并处理消费失败的消息。

```java
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.message.MessageExt;

public class Consumer {
    public static void main(String[] args) throws Exception {
        // 创建消费者实例
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("ConsumerGroup");
        consumer.setNamesrvAddr("localhost:9876");
        consumer.start();

        // 订阅主题
        consumer.subscribe("TopicTest", "*");

        // 设置消息监听器
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), new String(msg.getBody()));
                }

                // 手动提交
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        System.out.println("Consumer Started.");
    }
}
```

### **性能与扩展性考虑**

>**性能**
- 生产者性能：通过消息发送确认机制和重试机制，可以提高生产者的可靠性，但也可能导致一定的性能开销。
- `Broker`性能：通过消息持久化机制和主从复制机制，可以提高`Broker`的可靠性，但也可能导致一定的性能开销。
- 消费者性能：通过消息确认机制和重试队列机制，可以提高消费者的可靠性，但也可能导致一定的性能开销。
{: .prompt-tip }

>**扩展性**
- 水平扩展：通过增加`Broker`的数量和队列的数量，可以实现水平扩展，提高系统的处理能力。
-  垂直扩展：通过提升单个`Broker`的硬件性能，可以提高单个`Broker`的处理能力。
{: .prompt-tip }

### **常见问题与解决方案**

>**消息发送失败**<br/>
**问题描述**：生产者在发送消息时遇到网络异常或`Broker`不可用的情况，导致消息未能成功发送。<br/>
**解决方案**：
- 确保生产者配置了正确的`Broker`地址。
- 检查网络连接是否正常。
- 适当增加重试次数和重试间隔。
{: .prompt-tip }


>**`Broker`存储失败**<br/>
**问题描述**：`Broker`接收到消息后未能成功持久化到磁盘，导致消息丢失。<br/>
**解决方案**：
- 确保`Broker`配置了正确的持久化策略。
- 检查磁盘空间是否充足。
- 调整`Broker`的写盘策略。
{: .prompt-tip }

>**消费者消费失败**<br/>
**问题描述**：消费者在消费消息时遇到异常，未能正确处理消息，导致消息未能被确认消费。<br/>
**解决方案**：
- 确保消费者配置了正确的消息确认机制。
- 检查消费者的消费逻辑，确保没有跨线程或跨进程的干扰。
- 使用死信队列或重试队列机制来处理消费失败的消息。
{: .prompt-tip }

### **检测消息丢失**

使用`MQ`要尽最大努力去保证消息不丢，还要能感知丢失消息的情况。对于一个刚上线的新系统，各方面都不稳定，会有一段磨合期，在这个阶段特别需要监控系统中是否有丢消息的情况。

#### **日志、监控和报警**

`MQ`提供了完善的监控和日志系统，能够监控消息的发送、存储和消费情况。当出现异常时，可以及时报警并采取相应的措施。如果有分布式链路追踪系统就更好了，可以方便地追踪每一条消息。

如果没有链路追踪，可以利用`MQ`的有序性来验证是否有丢消息。原理是在`Producer`端给每个发出的消息附加一个连续递增的序号，然后在`Consumer`端来检查这个序号的连续性。如果没有消息丢失，`Consumer`收到消息的序号必然是连续递增的，或者说收到的消息，其中的序号必然是上一条消息的序号加`1`。如果检测到序号不连续，那就是丢消息了。还可以通过缺失的序号来确定丢失的是哪条消息，方便进一步排查原因。

>大多数`MQ`的客户端都支持拦截器机制，可以利用这个拦截器机制，在`Producer`发送消息之前的拦截器中将序号注入到消息中，在`Consumer`收到消息的拦截器中检测序号的连续性，这样实现的好处是消息检测代码不会侵入到业务代码中，待系统稳定后，也方便将这部分检测的逻辑关闭或者删除。
{: .prompt-tip }
>如果是在一个分布式系统中实现这个检测方法，有几个问题需要你注意。
- 像`Kafka`和`RocketMQ`这样的消息队列，不保证在`Topic`上的严格顺序，只能保证分区上的消息是有序的，所以在发消息时必须要指定分区，并且在每个分区单独检测消息序号的连续性。
- 如果系统中`Producer`是多实例的，由于并不好协调多个`Producer`之间的发送顺序，所以也需要每个`Producer`分别生成各自的消息序号，并且需要附加上`Producer`的标识，在`Consumer`端按照每个`Producer`分别来检测序号的连续性。
- `Consumer`实例的数量最好和分区数量一致，做到`Consumer`和分区一一对应，这样比较方便在`Consumer`内检测消息序号的连续性。
{: .prompt-tip }

### **其他机制**

`RocketMQ`还提供了多种机制来保证消息不丢失，例如事务消息、延迟消息、顺序消息等，可以根据业务需求选择和使用。

### **极端情况需要降级处理**

如果对消息丢失零容忍，必须要考虑极端情况，比如整个`RocketMQ`集群挂了，这时`Producer`发送消息一定会失败，可以考虑在`Producer`端做降级，把要发送的消息保存到数据库或磁盘，等`RocketMQ`恢复以后再把本地消息推送出去。