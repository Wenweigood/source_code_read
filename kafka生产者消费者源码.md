# Kafka客户端源码简析
- [Kafka客户端源码简析](#kafka客户端源码简析)
  - [前置依赖版本信息](#前置依赖版本信息)
  - [生产者](#生产者)
    - [创建生产者](#创建生产者)
    - [生产者发送消息](#生产者发送消息)
    - [Sender线程发送消息](#sender线程发送消息)
  - [消费者](#消费者)
    - [1. KafkaConsumer#poll](#1-kafkaconsumerpoll)
      - [1.1 KafkaConsumer#updateAssignmentMetadataIfNeeded](#11-kafkaconsumerupdateassignmentmetadataifneeded)
        - [1.1.1 ConsumerCoordinator#poll](#111-consumercoordinatorpoll)
          - [1.1.1.1 AbstractCoordinator#ensureActiveGroup](#1111-abstractcoordinatorensureactivegroup)
      - [1.2 KafkaConsumer#pollForFetches](#12-kafkaconsumerpollforfetches)
        - [1.2.1 AbstractFetch#collectFetch](#121-abstractfetchcollectfetch)
    - [2. KafkaConsumer#commitAsync](#2-kafkaconsumercommitasync)
      - [2.1 SubscriptionState#allConsumed](#21-subscriptionstateallconsumed)
      - [2.2 ConsumerCoordinator#commitOffsetsAsync](#22-consumercoordinatorcommitoffsetsasync)
    - [3. KafkaConsumer#commitSync()](#3-kafkaconsumercommitsync)
      - [3.1 ConsumerCoordinator#commitOffsetsSync](#31-consumercoordinatorcommitoffsetssync)
    - [4. AbstractCoordinator.HeartbeatThread](#4-abstractcoordinatorheartbeatthread)
      - [4.1 AbstractCoordinator.HeartbeatThread#run](#41-abstractcoordinatorheartbeatthreadrun)



## 前置依赖版本信息
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.6.1</version>
</dependency>
```

## 生产者

### 创建生产者

```java
Properties properties = new Properties();
properties.put("bootstrap.servers", "localhost:9092");
properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
properties.put("batch.size", 16384); // 16KB
properties.put("linger.ms", 10); // 等待10ms
Producer<String, String> producer = new KafkaProducer<>(properties);
```
1. 设置基本信息字段
2. `org.apache.kafka.clients.producer.internals.RecordAccumulator`：新建RecordAccumulator，存放一批一批的待发消息
3. `org.apache.kafka.clients.producer.KafkaProducer#newSender`：新建发送线程（实现`Runnable`接口）
   1. `org.apache.kafka.clients.ClientUtils#createNetworkClient`：创建Kafka客户端（实现`KafkaClient`接口）
      1. `org.apache.kafka.common.network.Selector`：**创建NIO selector**，入参包含关键参数`connections.max.idle.ms`，Kafka客户端和Broker的最大空闲时间，默认9分钟，超时会关闭连接。通过 **Java NIO** 实现单线程高效处理大量连接，避免线程阻塞

### 生产者发送消息
1. `org.apache.kafka.clients.producer.KafkaProducer#send(org.apache.kafka.clients.producer.ProducerRecord<K,V>, org.apache.kafka.clients.producer.Callback)`：入参为待发送的消息（包含主题、key、消息），异常回调函数
2. `org.apache.kafka.clients.producer.KafkaProducer#doSend`：真正处理待发消息的方法
   1. `org.apache.kafka.clients.producer.KafkaProducer#waitOnMetadata`：获取主题、分区的元数据
   2. `org.apache.kafka.clients.producer.Partitioner#partition`：**根据粘性分配策略计算分区（如有key则根据key计算）**
   3. `org.apache.kafka.clients.producer.internals.RecordAccumulator#append`：追加待发消息到最新的一批消息（ProducerBatch），如果无法追加则新开一个消息批次
      1. `org.apache.kafka.clients.producer.internals.RecordAccumulator#tryAppend`：尝试追加消息到分区的缓冲队列（ArrayDeque）的最新一批消息中
         1. `org.apache.kafka.common.record.MemoryRecordsBuilder#isFull`：判断这批消息是否满了（`估算当前已缓冲大小 >= writeLimit`, writeLimit和一批消息的缓冲区大小buffer有关）
      2. 新开一批消息，其最大缓冲区大小计算方式如下，涉及关键参数`batch.size`：
            ```java
            int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, this.compression, key, value, headers));
            buffer = this.free.allocate(size, maxTimeToBlock);
            ```
   4. `org.apache.kafka.clients.producer.internals.Sender#wakeup`：如果添加消息后，一批消息满了，则唤醒发送线程
      1. `org.apache.kafka.clients.KafkaClient#wakeup`
         1. `org.apache.kafka.common.network.Selector#wakeup`
            1. `java.nio.channels.Selector#wakeup`


### Sender线程发送消息
1. `org.apache.kafka.clients.producer.internals.Sender#run`：线程主方法，持续发送消息
   1. `org.apache.kafka.clients.producer.internals.Sender#runOnce`：发送消息核心方法
      1. `org.apache.kafka.clients.producer.internals.Sender#sendProducerData`：负责**​​预发送**消息​​的准备工作，包括数据批次的准备（按目标Broker分组）、元数据检查（分区Leader是否可用，不可用触发元数据更新）、节点可用性验证（剔除不可用的Broker节点）等
         ```java
         //存在不可用的Leader时触发元数据更新，获取其Leader数据
         if (!result.unknownLeaderTopics.isEmpty()) {
            iter = result.unknownLeaderTopics.iterator();
         
            while(iter.hasNext()) {
                String topic = (String)iter.next();
                this.metadata.add(topic, now);
            }
         
            this.log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}", result.unknownLeaderTopics);
            this.metadata.requestUpdate();
         }
         
         //移除不可用的Broker节点
         while(iter.hasNext()) {
            Node node = (Node)iter.next();
            if (!this.client.ready(node, now)) {
                  this.accumulator.updateNodeLatencyStats(node.id(), now, false);
                  iter.remove();
                  notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
            } else {
                  this.accumulator.updateNodeLatencyStats(node.id(), now, true);
            }
         }
         ```
         1. `org.apache.kafka.clients.producer.internals.RecordAccumulator#drain`：按目标Broker分组待发数据
         2. `org.apache.kafka.clients.KafkaClient#send`：将高层请求封装为 NetworkSend（包含目的地、请求头、请求数据）对象
            1. `org.apache.kafka.clients.NetworkClient#canSendRequest`：基于连接状态和未完成请求数（in-flight requests）的限制（`max.in.flight.requests.per.connection`）判断是否可发送
               ```java
               Deque<NetworkClient.InFlightRequest> queue = (Deque)this.requests.get(node);
               return queue == null || queue.isEmpty() || ((NetworkClient.InFlightRequest)queue.peekFirst()).send.completed() && queue.size() < this.maxInFlightRequestsPerConnection;
               ```
            2. `org.apache.kafka.common.network.Selectable#send`：网络层中用于​​预存待发送数据​​，将 NetworkSend 对象暂存到 KafkaChannel 中，并注册 OP_WRITE事件
      2. `org.apache.kafka.clients.KafkaClient#poll`：负责**执行实际的网络 I/O 操作**（发送请求和接收响应）
         1. `org.apache.kafka.common.network.Selectable#poll`：触发 **NIO** 的 select()，处理就绪的 I/O 事件


## 消费者

```java
public static void main(String[] args) {
   Properties props = new Properties();
   props.put("bootstrap.servers", "localhost:9092");
   props.put("group.id", "test-group");
   props.put("enable.auto.commit", "false"); // 关闭自动提交
   props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
   props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

   KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
   consumer.subscribe(Arrays.asList("your-topic"));

   try {
      while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            
            // 处理消息
            for (ConsumerRecord<String, String> record : records) {
               System.out.printf("topic=%s, partition=%d, offset=%d, value=%s\n",
                        record.topic(), record.partition(), record.offset(), record.value());
            }

            // 异步提交（提高吞吐量）
            consumer.commitAsync((offsets, exception) -> {
               if (exception != null) {
                  System.err.println("异步提交失败: " + offsets + ", 异常: " + exception);
               }
            });
      }
   } catch (Exception e) {
      e.printStackTrace();
   } finally {
      try {
            // 同步提交（确保最终提交成功）
            consumer.commitSync();
      } finally {
            consumer.close();
      }
   }
}
```

### 1. KafkaConsumer#poll
拉取消息顶层方法，do-while循环拉取消息，时间不超过入参的超时时间，拉取到立即返回
```java
do {
   this.client.maybeTriggerWakeup();
   if (includeMetadataInTimeout) {
      //检查消费者组状态（重平衡、心跳）
      this.updateAssignmentMetadataIfNeeded(timer, false);
   }
   ...
   // 拉取消息
   Fetch<K, V> fetch = this.pollForFetches(timer);
   if (!fetch.isEmpty()) {
      ...
      // 如果消费者配置了拦截器 interceptor.classes，则消息拉取后会对消息逐个应用拦截器处理
      // 拦截器可修改消息内容（如解密、格式转换）或元数据（如调整偏移量），但复杂逻辑可能增加poll()延迟
      ConsumerRecords var4 = this.interceptors.onConsume(new ConsumerRecords(fetch.records()));
      return var4;
   }
} while(timer.notExpired());
```

#### 1.1 KafkaConsumer#updateAssignmentMetadataIfNeeded
检查消费者组状态（重平衡、心跳）
##### 1.1.1 ConsumerCoordinator#poll
处理协调者相关信息
```java
// 更新心跳线程计时器，如果发现超过 heartbeat.interval.ms 没有发送心跳，
// 则唤醒可能阻塞在poll()方法的主线程，触发心跳请求的发送
this.pollHeartbeat(timer.currentTimeMs());
if (this.coordinatorUnknownAndUnreadySync(timer)) {
      return false;
}
// 检查是否需要重平衡
if (this.rejoinNeededOrPending()) {
   ...
   // 检查消费者组状态，若未初始化协调器或未加入组，则启动心跳线程
   if (!this.ensureActiveGroup(waitForJoinGroup ? timer : this.time.timer(0L))) {
      timer.update(this.time.milliseconds());
      return false;
   }
}
...
// 如果设置了 enable.auto.commit = true，
// 则更新异步提交计时器，如果发现超过 auto.commit.interval.ms 没有进行异步提交，
// 则进行一次异步提交
this.maybeAutoCommitOffsetsAsync(timer.currentTimeMs());
```

###### 1.1.1.1 AbstractCoordinator#ensureActiveGroup
检查消费者组状态，若未初始化协调器或未加入组，则启动心跳线程
```java
if (!this.ensureCoordinatorReady(timer)) {
   return false;
} else {
   // 若未初始化心跳线程，则启动心跳线程
   this.startHeartbeatThreadIfNeeded();
   return this.joinGroupIfNeeded(timer);
}
```

#### 1.2 KafkaConsumer#pollForFetches
从本地缓存或者Broker拉取消息，更新本地位移
```java
// 计算本次拉取的超时时间（受心跳间隔和剩余时间限制）
long pollTimeout = Math.min(coordinator.timeToNextPoll(timer.currentTimeMs()), timer.remainingMs());
// 检查本地缓存，如果有直接返回缓存消息
// Fetcher.completedFetches维护，存储已拉取但未处理的FetchResponse数据
Fetch<K, V> fetch = this.fetcher.collectFetch();
if (!fetch.isEmpty()) {
   return fetch;
} else {
   // 异步发送FetchRequest
   this.sendFetches();
   ...
   // 触发实际网络I/O
   this.client.poll(pollTimer, () -> {
         return !this.fetcher.hasAvailableFetches();
   });
   timer.update(pollTimer.currentTimeMs());
   // 返回结果，并更新消费位移
   return this.fetcher.collectFetch();
}
```
##### 1.2.1 AbstractFetch#collectFetch
从本地缓存返回结果，并更新消费位移
```java
List<ConsumerRecord<K, V>> fetchRecords(int maxRecords) {
   ...
   for(int i = 0; i < maxRecords; ++i) {
   ...
   // 下一拉取拉取位移+1
   this.nextFetchOffset = this.lastRecord.offset() + 1L;
   ...
   }
   ...
}
```

### 2. KafkaConsumer#commitAsync
异步提交消息位移的核心方法​，不阻塞消费者线程，通过回调函数处理提交结果，适合对吞吐量要求较高的场景
```java
// 入口方法，入参为所有分区的消费位移、回调函数
this.commitAsync(this.subscriptions.allConsumed(), callback);
```
#### 2.1 SubscriptionState#allConsumed
```java
Map<TopicPartition, OffsetAndMetadata> allConsumed = new HashMap();
this.assignment.forEach((topicPartition, partitionState) -> {
   if (partitionState.hasValidPosition()) {
      // 下一条待消费消息的位移，
      // Leader的版本号（在Leader切换时用到，避免旧位移提交），
      // 留空的metadata
      allConsumed.put(topicPartition, new OffsetAndMetadata(partitionState.position.offset, partitionState.position.offsetEpoch, ""));
   }
});
return allConsumed;
```

#### 2.2 ConsumerCoordinator#commitOffsetsAsync
```java
...
if (offsets.isEmpty()) {
   // 直接触发空提交（无实际效果）
   future = this.doCommitOffsetsAsync(offsets, callback);
} else if (!this.coordinatorUnknownAndUnreadyAsync()) {
   // 消费者组协调器已知且就绪，直接提交
   future = this.doCommitOffsetsAsync(offsets, callback);
} else {
   // 消费者组协调器未知或未就绪，需要找协调者后再提交
   this.pendingAsyncCommits.incrementAndGet();
   this.lookupCoordinator().addListener(new RequestFutureListener<Void>() {
         public void onSuccess(Void value) {
            ...
            // 将请求放入请求队列，异步发送
            ConsumerCoordinator.this.doCommitOffsetsAsync(offsets, callback);
            // 立即触发实际网络请求
            ConsumerCoordinator.this.client.pollNoWakeup();
         }

         public void onFailure(RuntimeException e) {
            ...
         }
   });
}
// 立即触发实际网络请求
this.client.pollNoWakeup();
return future;
```

### 3. KafkaConsumer#commitSync()
同步提交消息位移​​的核心方法，与异步提交（commitAsync）相比，通过阻塞调用线程确保提交结果的强一致性
```java
// 入口方法，入参为所有分区的消费位移、同步提交超时时间
this.commitSync(this.subscriptions.allConsumed(), timeout);
```

#### 3.1 ConsumerCoordinator#commitOffsetsSync

```java
if (offsets.isEmpty()) {
   // 处理遗留的异步提交
   return this.invokePendingAsyncCommits(timer);
} else {
   // 协调器未就绪，会阻塞在此判断中，直到就绪或抛出异常
   while(!this.coordinatorUnknownAndUnreadySync(timer)) {
      // 构造请求并放入发送队列
      RequestFuture<Void> future = this.sendOffsetCommitRequest(offsets);
      // 阻塞调用线程，直到收到Broker响应或超时
      this.client.poll(future, timer);
      this.invokeCompletedOffsetCommitCallbacks();
      if (future.succeeded()) {
         // 如果有拦截器会触发拦截器处理链路
         if (this.interceptors != null) {
            this.interceptors.onCommit(offsets);
         }

         return true;
      }
      if (future.failed() && !future.isRetriable()) {
         throw future.exception();
      }
      // 失败超时 retry.backoff.ms 重试，默认100ms
      // 异步提交故意不设计重试等待机制​，为了避免位移提交覆盖问题，保证位移提交的单调递增
      timer.sleep(this.rebalanceConfig.retryBackoffMs);
      if (!timer.notExpired()) {
         return false;
      }
   }
   return false;
}
```

### 4. AbstractCoordinator.HeartbeatThread

心跳线程类，在消费者第一次poll时启动

#### 4.1 AbstractCoordinator.HeartbeatThread#run

```java
while(true) {
   synchronized(AbstractCoordinator.this) {
      if (this.closed) {
            return;
      }

      if (!this.enabled) {
            AbstractCoordinator.this.wait();
      } else if (!AbstractCoordinator.this.state.hasNotJoinedGroup() && !this.hasFailed()) {
            AbstractCoordinator.this.client.pollNoWakeup();
            long now = AbstractCoordinator.this.time.milliseconds();
            if (AbstractCoordinator.this.coordinatorUnknown()) {
               if (AbstractCoordinator.this.findCoordinatorFuture != null) {
                  AbstractCoordinator.this.clearFindCoordinatorFuture();
               } else {
                  AbstractCoordinator.this.lookupCoordinator();
               }

               AbstractCoordinator.this.wait(AbstractCoordinator.this.rebalanceConfig.retryBackoffMs);
            } else if (AbstractCoordinator.this.heartbeat.sessionTimeoutExpired(now)) {
               // 超过 session.timeout.ms 没有收到协调者回复心跳请求，认为协调者不可用 
               AbstractCoordinator.this.markCoordinatorUnknown("session timed out without receiving a heartbeat response");
            } else if (AbstractCoordinator.this.heartbeat.pollTimeoutExpired(now)) {
               // 消费者处理消息时间过长（和上一次poll()时间间隔超过 max.poll.interval.ms ）
               // 建议增大 max.poll.interval.ms 或者 减少单次拉取的消息数目 max.poll.records
               ...
            } else if (!AbstractCoordinator.this.heartbeat.shouldHeartbeat(now)) {
               // 如果心跳计时器还没有超时，则等待 retry.backoff.ms 后再重新判断是否发送心跳（避免循环过于频繁，占用cpu资源）
               AbstractCoordinator.this.wait(AbstractCoordinator.this.rebalanceConfig.retryBackoffMs);
            } else {
               AbstractCoordinator.this.heartbeat.sentHeartbeat(now);
               RequestFuture<Void> heartbeatFuture = AbstractCoordinator.this.sendHeartbeatRequest();
               heartbeatFuture.addListener(new RequestFutureListener<Void>() {
                  public void onSuccess(Void value) {
                        synchronized(AbstractCoordinator.this) {
                           // 刷新心跳计时器、会话计时器、拉取消息间隔计时器超时时间
                           AbstractCoordinator.this.heartbeat.receiveHeartbeat();
                        }
                  }
                  public void onFailure(RuntimeException e) {
                        ...
                  }
               });
            }
      } else {
         // 禁用本线程
         this.disable();
      }
   }
}
```
