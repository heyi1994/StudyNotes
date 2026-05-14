# MQ

## 1. 消息队列简介

消息队列（Message Queue，MQ）是一种异步通信机制，允许生产者将消息发送到队列，消费者从队列中拉取并处理消息。两端解耦，互不依赖彼此的运行状态。

### 核心价值

| 能力      | 说明                                 |
| --------- | ------------------------------------ |
| 异步解耦  | 生产者发完即走，消费者按自身节奏处理 |
| 削峰填谷  | 突发流量缓冲到队列，消费者匀速消费   |
| 广播/扇出 | 一条消息可被多个消费者独立处理       |
| 可靠传递  | 消息持久化，消费失败可重试           |
| 顺序保证  | 同一分区/队列内消息有序              |

### 典型使用场景

```
电商下单  →  [订单MQ]  →  库存服务
                        →  支付服务
                        →  通知服务
                        →  数据分析

用户上传  →  [任务MQ]  →  转码服务（异步处理，解耦耗时操作）

秒杀请求  →  [限流MQ]  →  匀速处理（削峰，保护数据库）
```

---

## 2. 核心概念

### 消息模型

```
点对点（Point-to-Point）
  Producer ──▶ Queue ──▶ Consumer
  一条消息只被一个消费者处理

发布订阅（Publish/Subscribe）
  Producer ──▶ Topic/Exchange ──▶ Consumer A
                               ──▶ Consumer B
                               ──▶ Consumer C
  一条消息广播给所有订阅者
```

### 关键术语

| 术语               | 说明                                                 |
| ------------------ | ---------------------------------------------------- |
| Producer（生产者） | 发送消息的一端                                       |
| Consumer（消费者） | 接收并处理消息的一端                                 |
| Broker             | 消息中间件服务器，负责存储和路由消息                 |
| Queue（队列）      | 存储消息的容器，点对点模型                           |
| Topic（主题）      | 发布订阅模型中的消息分类                             |
| Partition（分区）  | Topic 的物理分片，Kafka 核心概念，提供并行能力       |
| Offset（偏移量）   | 消息在 Partition 中的位置编号，Kafka 特有            |
| Consumer Group     | 一组消费者协作消费同一 Topic，组内每条消息只处理一次 |
| ACK（确认）        | 消费者通知 Broker 消息已成功处理                     |
| Dead Letter        | 处理失败的消息转移到死信队列                         |

### 消息投递语义

```
At Most Once  （最多一次）：消息可能丢失，但不会重复  → 性能最高
At Least Once （至少一次）：消息不会丢失，但可能重复  → 最常用
Exactly Once  （恰好一次）：不丢不重                 → 实现最复杂，性能最低
```

---

## 3. 主流 MQ 对比

| 维度       | RabbitMQ       | Kafka                | NATS                | Redis Streams      |
| ---------- | -------------- | -------------------- | ------------------- | ------------------ |
| 协议       | AMQP           | 自研二进制协议       | 自研文本/二进制协议 | RESP               |
| 吞吐量     | 万级/秒        | 百万级/秒            | 百万级/秒           | 十万级/秒          |
| 消息持久化 | 支持           | 原生持久化           | 可选（JetStream）   | 支持               |
| 消费模式   | Push           | Pull                 | Push/Pull           | Pull               |
| 消息回溯   | 不支持         | 支持（按 Offset）    | JetStream 支持      | 支持               |
| 路由能力   | 强（Exchange） | 弱                   | 主题通配符          | 弱                 |
| Rust 生态  | lapin          | rdkafka              | async-nats          | redis-rs           |
| 适用场景   | 复杂路由、RPC  | 日志、流处理、大数据 | 微服务、IoT         | 轻量级、已有 Redis |

---

## 4. RabbitMQ + Rust

### 4.1 环境准备

```toml
# Cargo.toml
[dependencies]
lapin       = "2"
tokio       = { version = "1", features = ["full"] }
tokio-amqp  = "2"
futures-lite = "2"
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
tracing     = "0.1"
tracing-subscriber = "0.3"
```

```bash
# 启动 RabbitMQ（Docker）
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management
# 管理界面：http://localhost:15672  guest/guest
```

### 4.2 基础生产者

```rust
use lapin::{
    options::*, types::FieldTable, BasicProperties, Connection, ConnectionProperties,
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct OrderEvent {
    order_id: u64,
    user_id:  u64,
    amount:   f64,
    status:   String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let conn = Connection::connect(
        "amqp://guest:guest@localhost:5672",
        ConnectionProperties::default(),
    )
    .await?;

    let channel = conn.create_channel().await?;

    // 声明持久化队列（durable = true，RabbitMQ 重启后队列不丢失）
    channel
        .queue_declare(
            "orders",
            QueueDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    let event = OrderEvent {
        order_id: 10001,
        user_id:  42,
        amount:   299.99,
        status:   "created".to_string(),
    };

    let payload = serde_json::to_vec(&event)?;

    channel
        .basic_publish(
            "",        // exchange（空串表示使用默认 exchange，直接投递到队列）
            "orders",  // routing_key（默认 exchange 下即队列名）
            BasicPublishOptions::default(),
            &payload,
            BasicProperties::default()
                .with_delivery_mode(2)  // 2 = 持久化消息（随队列落盘）
                .with_content_type("application/json".into()),
        )
        .await?
        .await?; // 第二个 await 等待 Broker 的 confirm（需开启 publisher confirm）

    println!("消息已发送: {:?}", event);
    Ok(())
}
```

### 4.3 基础消费者

```rust
use futures_lite::stream::StreamExt;
use lapin::{options::*, types::FieldTable, Connection, ConnectionProperties};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let conn = Connection::connect(
        "amqp://guest:guest@localhost:5672",
        ConnectionProperties::default(),
    )
    .await?;

    let channel = conn.create_channel().await?;

    // 每次只预取 1 条消息，处理完再取下一条（公平调度）
    channel
        .basic_qos(1, BasicQosOptions::default())
        .await?;

    let mut consumer = channel
        .basic_consume(
            "orders",
            "order_consumer_1",  // consumer tag，需唯一
            BasicConsumeOptions::default(),
            FieldTable::default(),
        )
        .await?;

    println!("等待消息中...");

    while let Some(delivery) = consumer.next().await {
        let delivery = delivery?;

        match serde_json::from_slice::<OrderEvent>(&delivery.data) {
            Ok(event) => {
                println!("收到订单: {:?}", event);

                // 处理成功，发送 ACK
                delivery
                    .ack(BasicAckOptions::default())
                    .await?;
            }
            Err(e) => {
                eprintln!("消息解析失败: {}", e);

                // requeue = false：不重新入队，转入死信队列
                delivery
                    .nack(BasicNackOptions { requeue: false, ..Default::default() })
                    .await?;
            }
        }
    }

    Ok(())
}
```

### 4.4 Exchange 路由模式

```rust
use lapin::{ExchangeKind, options::*, types::FieldTable};

// ---- Direct Exchange（精确匹配 routing key）----
channel.exchange_declare(
    "order_direct",
    ExchangeKind::Direct,
    ExchangeDeclareOptions { durable: true, ..Default::default() },
    FieldTable::default(),
).await?;

channel.queue_bind("orders_vip",    "order_direct", "vip",    QueueBindOptions::default(), FieldTable::default()).await?;
channel.queue_bind("orders_normal", "order_direct", "normal", QueueBindOptions::default(), FieldTable::default()).await?;

// 发送到 vip 队列
channel.basic_publish("order_direct", "vip", BasicPublishOptions::default(), &payload, BasicProperties::default()).await?;

// ---- Topic Exchange（通配符匹配）----
// * 匹配一个词，# 匹配零个或多个词
channel.exchange_declare("app_events", ExchangeKind::Topic, ExchangeDeclareOptions { durable: true, ..Default::default() }, FieldTable::default()).await?;

channel.queue_bind("order_queue",  "app_events", "order.#",       QueueBindOptions::default(), FieldTable::default()).await?;
channel.queue_bind("pay_queue",    "app_events", "order.pay.*",   QueueBindOptions::default(), FieldTable::default()).await?;
channel.queue_bind("notify_queue", "app_events", "#.notification",QueueBindOptions::default(), FieldTable::default()).await?;

// 发布：匹配 order_queue 和 pay_queue
channel.basic_publish("app_events", "order.pay.alipay", BasicPublishOptions::default(), &payload, BasicProperties::default()).await?;

// ---- Fanout Exchange（广播，忽略 routing key）----
channel.exchange_declare("broadcast", ExchangeKind::Fanout, ExchangeDeclareOptions { durable: true, ..Default::default() }, FieldTable::default()).await?;
// 所有绑定到 broadcast 的队列都会收到消息
channel.basic_publish("broadcast", "", BasicPublishOptions::default(), &payload, BasicProperties::default()).await?;
```

### 4.5 Publisher Confirm（发送确认）

```rust
// 开启 publisher confirm 模式，确保消息到达 Broker
channel.confirm_select(ConfirmSelectOptions::default()).await?;

let confirm = channel
    .basic_publish(
        "",
        "orders",
        BasicPublishOptions::default(),
        &payload,
        BasicProperties::default().with_delivery_mode(2),
    )
    .await?
    .await?;  // 等待 Broker 的 ack/nack

match confirm {
    lapin::publisher_confirm::Confirmation::Ack(_) => {
        println!("Broker 已确认接收消息");
    }
    lapin::publisher_confirm::Confirmation::Nack(_) => {
        eprintln!("Broker 拒绝了消息，需要重发");
    }
    _ => {}
}
```

---

## 5. Kafka + Rust

### 5.1 环境准备

```toml
# Cargo.toml
[dependencies]
rdkafka   = { version = "0.36", features = ["cmake-build", "tokio"] }
tokio     = { version = "1", features = ["full"] }
serde     = { version = "1", features = ["derive"] }
serde_json = "1"
```

```bash
# 启动 Kafka（Docker Compose）
# docker-compose.yml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on: [zookeeper]
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

### 5.2 生产者

```rust
use rdkafka::{
    config::ClientConfig,
    producer::{FutureProducer, FutureRecord},
};
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let producer: FutureProducer = ClientConfig::new()
        .set("bootstrap.servers", "localhost:9092")
        // 所有副本确认后才算成功（最强可靠性）
        .set("acks", "all")
        // 批量发送等待时间（ms），增大可提升吞吐量
        .set("linger.ms", "5")
        // 批次大小（bytes）
        .set("batch.size", "65536")
        // 启用幂等生产者（防止重试导致重复消息）
        .set("enable.idempotence", "true")
        .create()?;

    for i in 0..10 {
        let key   = format!("user_{}", i % 3);          // 相同 key 路由到同一分区，保证顺序
        let value = serde_json::json!({
            "event_id": i,
            "user_id":  i % 3,
            "action":   "purchase",
        })
        .to_string();

        let record = FutureRecord::to("user_events")
            .key(&key)
            .payload(&value);

        match producer.send(record, Duration::from_secs(5)).await {
            Ok((partition, offset)) => {
                println!("消息发送成功 → partition={partition}, offset={offset}");
            }
            Err((err, _msg)) => {
                eprintln!("发送失败: {}", err);
            }
        }
    }

    Ok(())
}
```

### 5.3 消费者

```rust
use rdkafka::{
    config::ClientConfig,
    consumer::{CommitMode, Consumer, StreamConsumer},
    message::Message,
};
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let consumer: StreamConsumer = ClientConfig::new()
        .set("bootstrap.servers", "localhost:9092")
        .set("group.id", "order_service_group")
        // latest：从最新消息开始消费（新消费者组首次启动）
        // earliest：从最早消息开始（适合回放）
        .set("auto.offset.reset", "earliest")
        // 关闭自动提交，改为手动提交（确保 at-least-once）
        .set("enable.auto.commit", "false")
        .create()?;

    consumer.subscribe(&["user_events"])?;

    println!("开始消费...");

    let mut stream = consumer.stream();

    while let Some(result) = stream.next().await {
        match result {
            Ok(msg) => {
                let partition = msg.partition();
                let offset    = msg.offset();
                let key       = msg.key_view::<str>().unwrap_or(Ok("")).unwrap_or("");
                let payload   = msg.payload_view::<str>().unwrap_or(Ok("")).unwrap_or("");

                println!(
                    "[partition={partition} offset={offset}] key={key} payload={payload}"
                );

                // 业务处理
                if let Err(e) = process_message(payload).await {
                    eprintln!("处理失败: {}", e);
                    // 失败时不提交 offset，下次重新消费（at-least-once）
                    continue;
                }

                // 手动提交 offset（同步提交，确保不丢失进度）
                consumer.commit_message(&msg, CommitMode::Sync)?;
            }
            Err(e) => eprintln!("消费错误: {}", e),
        }
    }

    Ok(())
}

async fn process_message(payload: &str) -> Result<(), Box<dyn std::error::Error>> {
    println!("处理消息: {}", payload);
    // 实际业务逻辑...
    Ok(())
}
```

### 5.4 消费者组与分区分配

```
Topic: user_events  (3个分区)
  Partition 0 ──▶ Consumer A（group: order_service）
  Partition 1 ──▶ Consumer B（group: order_service）
  Partition 2 ──▶ Consumer C（group: order_service）

规则：
  - 同一 group 内，每个分区只被一个消费者处理
  - group 内消费者数 > 分区数时，多余的消费者闲置
  - 消费者退出/加入时，触发 Rebalance，重新分配分区
```

### 5.5 指定 Offset 消费（消息回放）

```rust
use rdkafka::TopicPartitionList;
use rdkafka::Offset;

// 从指定 offset 开始消费
let mut tpl = TopicPartitionList::new();
tpl.add_partition_offset("user_events", 0, Offset::Offset(100))?;
tpl.add_partition_offset("user_events", 1, Offset::Offset(200))?;
tpl.add_partition_offset("user_events", 2, Offset::Beginning)?; // 从头开始

consumer.assign(&tpl)?;
```

### 5.6 事务性生产者（Exactly Once）

```rust
use rdkafka::producer::BaseProducer;

let producer: BaseProducer = ClientConfig::new()
    .set("bootstrap.servers", "localhost:9092")
    .set("transactional.id", "my_transactional_producer")  // 必须设置
    .set("enable.idempotence", "true")
    .create()?;

producer.init_transactions(Duration::from_secs(10))?;

// 开启事务
producer.begin_transaction()?;

// 在事务内发送多条消息
producer.send(BaseRecord::to("topic_a").key("k1").payload("v1"))?;
producer.send(BaseRecord::to("topic_b").key("k2").payload("v2"))?;

// 提交或中止
producer.commit_transaction(Duration::from_secs(10))?;
// producer.abort_transaction(Duration::from_secs(10))?;
```

---

## 6. NATS + Rust

### 6.1 环境准备

```toml
[dependencies]
async-nats = "0.35"
tokio      = { version = "1", features = ["full"] }
```

```bash
# 启动 NATS Server
docker run -d --name nats -p 4222:4222 nats:latest

# 启动带 JetStream（持久化）的 NATS
docker run -d --name nats -p 4222:4222 nats:latest --jetstream
```

### 6.2 Core NATS（轻量级 Pub/Sub）

```rust
use async_nats;
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::connect("nats://localhost:4222").await?;

    // 订阅者
    let client_sub = client.clone();
    tokio::spawn(async move {
        let mut subscriber = client_sub
            .subscribe("events.orders.>")  // > 匹配多层级
            .await
            .unwrap();

        while let Some(msg) = subscriber.next().await {
            println!(
                "收到消息 [subject={}]: {}",
                msg.subject,
                std::str::from_utf8(&msg.payload).unwrap_or("")
            );

            // 如果是 Request-Reply 模式，返回响应
            if let Some(reply) = msg.reply {
                client_sub.publish(reply, "已处理".into()).await.unwrap();
            }
        }
    });

    // 发布消息
    client.publish("events.orders.created", r#"{"order_id":1001}"#.into()).await?;
    client.publish("events.orders.paid",    r#"{"order_id":1001}"#.into()).await?;

    // Request-Reply（同步请求）
    let response = client
        .request("events.orders.query", r#"{"order_id":1001}"#.into())
        .await?;
    println!("响应: {}", std::str::from_utf8(&response.payload).unwrap_or(""));

    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    Ok(())
}
```

### 6.3 JetStream（持久化消息流）

```rust
use async_nats::jetstream;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::connect("nats://localhost:4222").await?;
    let jetstream = jetstream::new(client);

    // 创建 Stream（持久化存储）
    let stream = jetstream
        .get_or_create_stream(jetstream::stream::Config {
            name:       "ORDERS".to_string(),
            subjects:   vec!["orders.>".to_string()],
            max_age:    std::time::Duration::from_secs(86400 * 7), // 保留 7 天
            storage:    jetstream::stream::StorageType::File,
            ..Default::default()
        })
        .await?;

    // 发布持久化消息
    let ack = jetstream
        .publish("orders.created", r#"{"order_id":2001}"#.into())
        .await?
        .await?;
    println!("消息序号: {}", ack.sequence);

    // 创建消费者（Pull 模式）
    let consumer = stream
        .get_or_create_consumer(
            "order_processor",
            jetstream::consumer::pull::Config {
                durable_name: Some("order_processor".to_string()),
                ack_policy:   jetstream::consumer::AckPolicy::Explicit,
                ..Default::default()
            },
        )
        .await?;

    // 拉取并处理消息
    let mut messages = consumer.fetch().max_messages(10).messages().await?;
    while let Some(msg) = messages.next().await {
        let msg = msg?;
        println!("处理: {}", std::str::from_utf8(&msg.payload).unwrap_or(""));
        msg.ack().await?;
    }

    Ok(())
}
```

---

## 7. Redis Pub/Sub + Rust

### 7.1 环境准备

```toml
[dependencies]
redis = { version = "0.25", features = ["tokio-comp", "streams"] }
tokio = { version = "1", features = ["full"] }
```

### 7.2 Pub/Sub（非持久化）

```rust
use redis::AsyncCommands;

// 发布者
#[tokio::main]
async fn publisher() -> redis::RedisResult<()> {
    let client = redis::Client::open("redis://127.0.0.1/")?;
    let mut conn = client.get_async_connection().await?;

    conn.publish("notifications", r#"{"type":"new_order","id":1001}"#).await?;
    println!("消息已发布");
    Ok(())
}

// 订阅者
#[tokio::main]
async fn subscriber() -> redis::RedisResult<()> {
    let client = redis::Client::open("redis://127.0.0.1/")?;
    let mut pubsub = client.get_async_connection().await?.into_pubsub();

    // 精确订阅
    pubsub.subscribe("notifications").await?;
    // 模式订阅（支持 glob）
    pubsub.psubscribe("order.*").await?;

    let mut stream = pubsub.on_message();
    loop {
        let msg = stream.next().await.unwrap();  // futures::StreamExt
        let channel: String  = msg.get_channel_name().to_string();
        let payload: String  = msg.get_payload()?;
        println!("[{}] {}", channel, payload);
    }
}
```

### 7.3 Redis Streams（持久化，类 Kafka）

```rust
use redis::{streams::*, AsyncCommands};

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    let client = redis::Client::open("redis://127.0.0.1/")?;
    let mut conn = client.get_async_connection().await?;

    // 写入 Stream（* 表示自动生成 ID）
    let id: String = conn
        .xadd(
            "orders",
            "*",
            &[("order_id", "3001"), ("status", "created"), ("amount", "199.9")],
        )
        .await?;
    println!("写入消息 ID: {}", id);

    // 创建消费者组
    let _: redis::RedisResult<()> = conn
        .xgroup_create_mkstream("orders", "order_processors", "$")
        .await;

    // 消费者组读取（> 表示读取未投递给本组的新消息）
    let results: StreamReadReply = conn
        .xread_options(
            &["orders"],
            &[">"],
            &StreamReadOptions::default()
                .group("order_processors", "consumer_1")
                .count(10)
                .block(2000),  // 阻塞等待 2000ms
        )
        .await?;

    for key in &results.keys {
        for entry in &key.ids {
            println!("ID={} data={:?}", entry.id, entry.map);

            // 处理成功后确认（ACK）
            let _: () = conn.xack("orders", "order_processors", &[&entry.id]).await?;
        }
    }

    // 查看待确认消息（消费了但未 ACK）
    let pending: StreamPendingReply = conn
        .xpending("orders", "order_processors")
        .await?;
    println!("待确认消息数: {:?}", pending);

    Ok(())
}
```

---

## 8. 消息可靠性保障

### 生产者侧

```
可靠性等级（以 Kafka 为例）：

acks=0  ──▶ Fire and forget，不等确认（最快，可能丢失）
acks=1  ──▶ Leader 写入即确认（可能丢，Leader 宕机时）
acks=-1 ──▶ 所有 ISR 副本写入才确认（最可靠，推荐生产环境）
```

```rust
// Kafka 生产者可靠性配置
let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "localhost:9092")
    .set("acks", "all")               // 等待所有副本
    .set("retries", "3")              // 失败自动重试
    .set("retry.backoff.ms", "300")   // 重试间隔
    .set("enable.idempotence", "true")// 幂等，防止重试导致重复
    .set("max.in.flight.requests.per.connection", "5") // 幂等模式下最大并发请求
    .create()?;
```

### 消费者侧

```rust
// 手动 ACK 模式（at-least-once）
// 处理完成后再提交 offset/ack，保证不丢失

// 关键：不能在处理失败时提交 offset
async fn consume_loop(consumer: &StreamConsumer) {
    let mut stream = consumer.stream();
    while let Some(Ok(msg)) = stream.next().await {
        // 1. 处理消息
        if let Err(e) = handle(&msg).await {
            eprintln!("处理失败，不提交 offset: {}", e);
            continue; // offset 不提交，下次重消费
        }

        // 2. 处理成功才提交
        consumer.commit_message(&msg, CommitMode::Sync).unwrap();
    }
}
```

### 消息持久化

| 措施         | RabbitMQ                    | Kafka                     |
| ------------ | --------------------------- | ------------------------- |
| 队列持久化   | `durable: true`             | Topic 默认持久            |
| 消息持久化   | `delivery_mode: 2`          | 消息默认持久              |
| 副本冗余     | Mirror Queue / Quorum Queue | `replication.factor >= 3` |
| 数据目录备份 | `/var/lib/rabbitmq`         | `log.dirs` 配置的目录     |

---

## 9. 消息顺序与幂等性

### 顺序保证

```
Kafka 保证：同一 Partition 内消息严格有序
策略：相同业务实体使用相同的 key，路由到同一 partition

RabbitMQ 保证：单个队列内消息有序
策略：同一业务线使用独立队列，或限制单消费者
```

```rust
// Kafka：使用 user_id 作为 key，确保同一用户的事件有序
let record = FutureRecord::to("user_events")
    .key(&format!("user_{}", user_id))  // 相同 user_id → 相同 partition
    .payload(&event_json);
```

### 幂等性设计

消费者可能因网络抖动收到重复消息，需在业务层实现幂等：

```rust
use std::collections::HashSet;
use tokio::sync::Mutex;

struct IdempotentConsumer {
    processed_ids: Mutex<HashSet<String>>,
}

impl IdempotentConsumer {
    async fn process(&self, msg_id: &str, payload: &str) -> Result<(), String> {
        let mut ids = self.processed_ids.lock().await;

        if ids.contains(msg_id) {
            println!("重复消息，跳过: {}", msg_id);
            return Ok(());
        }

        // 实际业务处理
        println!("处理消息 {}: {}", msg_id, payload);
        // do_business_logic(payload).await?;

        ids.insert(msg_id.to_string());
        Ok(())
    }
}

// 生产场景：processed_ids 应持久化到 Redis 或数据库
// Redis SETNX / SET NX EX 是常用的幂等 key 存储方式
```

---

## 10. 死信队列与重试机制

### RabbitMQ 死信队列（DLQ）

```rust
use lapin::{options::*, types::{AMQPValue, FieldTable}, ExchangeKind};

// 1. 先创建死信 Exchange 和队列
channel.exchange_declare("dlx_exchange", ExchangeKind::Direct,
    ExchangeDeclareOptions { durable: true, ..Default::default() },
    FieldTable::default()).await?;

channel.queue_declare("dead_letters",
    QueueDeclareOptions { durable: true, ..Default::default() },
    FieldTable::default()).await?;

channel.queue_bind("dead_letters", "dlx_exchange", "dlx_key",
    QueueBindOptions::default(), FieldTable::default()).await?;

// 2. 创建业务队列时，指定死信路由
let mut args = FieldTable::default();
args.insert("x-dead-letter-exchange".into(),  AMQPValue::LongString("dlx_exchange".into()));
args.insert("x-dead-letter-routing-key".into(), AMQPValue::LongString("dlx_key".into()));
args.insert("x-message-ttl".into(), AMQPValue::LongInt(30000)); // 30秒未消费则死信

channel.queue_declare("orders",
    QueueDeclareOptions { durable: true, ..Default::default() },
    args).await?;

// 消费者：nack + requeue=false → 消息自动进入死信队列
delivery.nack(BasicNackOptions { requeue: false, ..Default::default() }).await?;
```

### 带退避重试的消费者（Rust）

```rust
use std::time::Duration;
use tokio::time::sleep;

struct RetryConfig {
    max_retries:    u32,
    initial_delay:  Duration,
    max_delay:      Duration,
}

async fn consume_with_retry<F, Fut>(
    payload: &[u8],
    handler: F,
    config: &RetryConfig,
) -> Result<(), String>
where
    F: Fn(Vec<u8>) -> Fut,
    Fut: std::future::Future<Output = Result<(), String>>,
{
    let mut attempt = 0;
    let mut delay = config.initial_delay;

    loop {
        match handler(payload.to_vec()).await {
            Ok(_) => return Ok(()),
            Err(e) if attempt < config.max_retries => {
                attempt += 1;
                eprintln!("第 {} 次失败: {}，{}ms 后重试", attempt, e, delay.as_millis());
                sleep(delay).await;

                // 指数退避：每次翻倍，不超过 max_delay
                delay = (delay * 2).min(config.max_delay);
            }
            Err(e) => {
                eprintln!("已达最大重试次数，放弃: {}", e);
                return Err(e);
            }
        }
    }
}

// 使用示例
let config = RetryConfig {
    max_retries:   5,
    initial_delay: Duration::from_millis(500),
    max_delay:     Duration::from_secs(30),
};

let result = consume_with_retry(
    &delivery.data,
    |data| async move {
        let payload = String::from_utf8(data).map_err(|e| e.to_string())?;
        process_order(&payload).await
    },
    &config,
).await;
```

---

## 11. 背压与流量控制

### 核心问题

```
没有背压时：
  Producer (10万/s) ──▶ Queue ──▶ Consumer (1万/s)
                          ↓
                    队列无限积压
                    Broker 内存/磁盘耗尽 → OOM 宕机

有背压时：
  Producer ──▶ [有界缓冲] ──▶ Consumer
                   ↑
              满了就阻塞生产者，或主动拒绝新消息
              下游压力传递给上游 → 整个链路自动调速
```

### 背压的三种策略

| 策略     | 行为                             | 适用场景                     |
| -------- | -------------------------------- | ---------------------------- |
| 阻塞     | 缓冲区满时生产者挂起等待         | 不能丢消息、生产者可阻塞     |
| 丢弃     | 超出阈值直接丢弃最新/最旧消息    | 允许丢失（如监控采样、日志） |
| 有界缓冲 | 固定大小队列，满则阻塞或返回错误 | 通用场景，结合监控告警       |

---

### 11.1 MQ 层面的背压配置

```rust
// RabbitMQ：prefetch_count 限制单个消费者的飞行中消息数
// 未 ACK 的消息达到上限后，Broker 停止推送新消息给该消费者
channel.basic_qos(10, BasicQosOptions::default()).await?;

// Kafka Consumer：控制每次拉取的消息量（Pull 模式天然支持背压）
ClientConfig::new()
    .set("max.poll.records",           "100")      // 每次最多拉 100 条
    .set("fetch.max.bytes",       "52428800")      // 单次 fetch 最大 50MB
    .set("max.partition.fetch.bytes", "1048576");  // 单分区 fetch 最大 1MB
```

---

### 11.2 令牌桶（Token Bucket）— 平滑限速

令牌桶按固定速率补充令牌，发消息前必须先拿到令牌。桶满后多余令牌丢弃，允许短暂突发（桶内积累的令牌）。

```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use tokio::time::{Duration, Instant, sleep};

struct TokenBucket {
    tokens:      f64,
    max_tokens:  f64,
    refill_rate: f64,    // 每秒补充的令牌数
    last_refill: Instant,
}

impl TokenBucket {
    fn new(rate: f64, capacity: f64) -> Self {
        Self {
            tokens:      capacity,
            max_tokens:  capacity,
            refill_rate: rate,
            last_refill: Instant::now(),
        }
    }

    async fn acquire(&mut self) {
        loop {
            let now     = Instant::now();
            let elapsed = (now - self.last_refill).as_secs_f64();
            // 按时间比例补充令牌，不超过桶容量
            self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.max_tokens);
            self.last_refill = now;

            if self.tokens >= 1.0 {
                self.tokens -= 1.0;
                return; // 拿到令牌，可以发送
            }

            // 令牌不足，计算最短等待时间后挂起
            let wait = Duration::from_secs_f64((1.0 - self.tokens) / self.refill_rate);
            sleep(wait).await;
        }
    }

    // 非阻塞版本：拿不到令牌返回 false
    fn try_acquire(&mut self) -> bool {
        let now     = Instant::now();
        let elapsed = (now - self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.max_tokens);
        self.last_refill = now;

        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            true
        } else {
            false
        }
    }
}

// 每秒最多发 500 条，桶容量 1000（允许短暂突发）
let bucket = Arc::new(Mutex::new(TokenBucket::new(500.0, 1000.0)));

for i in 0..10_000u64 {
    bucket.lock().await.acquire().await; // 阻塞直到拿到令牌
    println!("发送第 {} 条", i);
    // producer.send(...).await?;
}
```

---

### 11.3 漏桶（Leaky Bucket）— 严格匀速

漏桶以恒定速率处理消息，不允许突发。用 `tokio::sync::mpsc` 的有界通道天然实现：

```rust
use tokio::sync::mpsc;
use tokio::time::{interval, Duration};

// 漏桶：有界 channel 作为桶，固定间隔消费（匀速漏出）
async fn leaky_bucket_demo() {
    // 桶容量 = 100，满了生产者会被阻塞（背压传递）
    let (tx, mut rx) = mpsc::channel::<String>(100);

    // 消费者：每 2ms 处理一条（500条/秒，严格匀速）
    tokio::spawn(async move {
        let mut ticker = interval(Duration::from_millis(2));
        while let Some(msg) = rx.recv().await {
            ticker.tick().await; // 等到下一个 tick 才处理
            println!("处理: {}", msg);
        }
    });

    // 生产者：发送速度超过 500/s 时自动被阻塞（背压）
    for i in 0..10_000 {
        // channel 满时，send 挂起 → 生产者自动减速
        tx.send(format!("msg_{}", i)).await.unwrap();
    }
}
```

---

### 11.4 Tokio 有界信号量 — 控制并发消费数

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;
use futures::StreamExt;

async fn bounded_concurrent_consumer(consumer: StreamConsumer) {
    // 最多同时处理 50 条消息，超出则等待
    let semaphore = Arc::new(Semaphore::new(50));
    let mut stream = consumer.stream();

    while let Some(Ok(msg)) = stream.next().await {
        let sem  = semaphore.clone();
        let data = msg.payload().unwrap_or(&[]).to_vec();

        tokio::spawn(async move {
            // 获取许可，超过 50 个并发时阻塞在这里（背压）
            let _permit = sem.acquire().await.unwrap();

            if let Err(e) = process(&data).await {
                eprintln!("处理失败: {}", e);
            }
            // _permit drop → 自动归还许可，允许下一条进入
        });
    }
}
```

---

### 11.5 自适应背压 — 动态调整消费速率

根据队列积压深度（Consumer Lag）动态调整拉取速率：

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use tokio::time::{sleep, Duration};

struct AdaptiveConsumer {
    lag_gauge:    Arc<AtomicU64>,  // 当前积压量
    base_batch:   usize,           // 正常批次大小
    min_batch:    usize,           // 积压过大时的最小批次
    max_batch:    usize,           // 积压正常时的最大批次
}

impl AdaptiveConsumer {
    fn batch_size(&self) -> usize {
        let lag = self.lag_gauge.load(Ordering::Relaxed);

        match lag {
            0..=1_000      => self.max_batch,  // 积压小：大批次，提升吞吐
            1_001..=10_000 => self.base_batch, // 积压中：正常速率
            _              => self.min_batch,  // 积压大：小批次，让系统喘息
        }
    }

    async fn sleep_between_batches(&self) {
        let lag = self.lag_gauge.load(Ordering::Relaxed);

        // 积压越严重，消费者主动降速（给其他服务让资源）
        let delay = match lag {
            0..=1_000      => Duration::ZERO,
            1_001..=10_000 => Duration::from_millis(10),
            _              => Duration::from_millis(100),
        };

        if !delay.is_zero() {
            sleep(delay).await;
        }
    }
}
```

---

### 11.6 背压监控指标

| 指标               | 来源                      | 含义                              | 告警阈值           |
| ------------------ | ------------------------- | --------------------------------- | ------------------ |
| Consumer Lag       | Kafka / RabbitMQ 管理 API | 消费者落后生产者的消息数          | > 10 万持续 5 分钟 |
| Queue Depth        | RabbitMQ / Redis          | 队列中未消费消息数                | > 配置上限 80%     |
| Channel Buffer %   | Prometheus / 自定义指标   | mpsc channel 占用率               | > 70%              |
| Processing Latency | 业务埋点                  | 单条消息处理耗时 P99              | > 基线 3 倍        |
| Error Rate         | 业务埋点                  | 处理失败率（触发重试/死信的比例） | > 1%               |

```rust
// 在消费循环中暴露背压指标（配合 metrics crate）
use metrics::{counter, gauge, histogram};

let start  = Instant::now();
let result = process(&data).await;
let elapsed = start.elapsed();

histogram!("message.processing.duration_ms", elapsed.as_millis() as f64);

match result {
    Ok(_)  => counter!("message.processed.success", 1),
    Err(_) => counter!("message.processed.error",   1),
}

// 上报 channel 使用率
gauge!("consumer.channel.usage", tx.capacity() as f64 / MAX_CAPACITY as f64);
```

---

## 12. 分布式事务与消息

### 本地消息表（Transactional Outbox）

避免"写数据库成功但发消息失败"的问题：

```sql
-- 在业务数据库中创建消息发件箱表
CREATE TABLE outbox_messages (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    topic       VARCHAR(100) NOT NULL,
    payload     JSON NOT NULL,
    status      ENUM('pending', 'sent', 'failed') DEFAULT 'pending',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    sent_at     DATETIME,
    INDEX idx_status (status, created_at)
);
```

```rust
// 在同一个数据库事务中：写业务数据 + 写 outbox
async fn create_order_with_outbox(
    db: &mut Transaction<'_, MySql>,
    order: &Order,
) -> Result<(), sqlx::Error> {
    // 1. 写业务数据
    sqlx::query!("INSERT INTO orders (user_id, amount) VALUES (?, ?)",
        order.user_id, order.amount)
        .execute(&mut *db)
        .await?;

    // 2. 写 outbox（同一事务，原子性保证）
    let payload = serde_json::to_string(order).unwrap();
    sqlx::query!("INSERT INTO outbox_messages (topic, payload) VALUES (?, ?)",
        "orders.created", payload)
        .execute(&mut *db)
        .await?;

    Ok(())
}

// 独立的 Relay 进程：轮询 outbox，发送到 MQ
async fn outbox_relay(db: &Pool<MySql>, producer: &FutureProducer) {
    loop {
        let rows = sqlx::query!(
            "SELECT id, topic, payload FROM outbox_messages WHERE status = 'pending' LIMIT 100"
        )
        .fetch_all(db)
        .await
        .unwrap();

        for row in rows {
            let record = FutureRecord::to(&row.topic).payload(&row.payload);
            match producer.send(record, Duration::from_secs(5)).await {
                Ok(_) => {
                    sqlx::query!("UPDATE outbox_messages SET status='sent', sent_at=NOW() WHERE id=?", row.id)
                        .execute(db).await.unwrap();
                }
                Err(_) => {
                    sqlx::query!("UPDATE outbox_messages SET status='failed' WHERE id=?", row.id)
                        .execute(db).await.unwrap();
                }
            }
        }

        tokio::time::sleep(Duration::from_secs(1)).await;
    }
}
```

---

## 13. 性能调优

### Kafka 生产者调优

```rust
let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "localhost:9092")
    // 批量发送：积累到 64KB 或等待 5ms，批量写入
    .set("batch.size",  "65536")
    .set("linger.ms",   "5")
    // 压缩（snappy/lz4/zstd，减少网络传输）
    .set("compression.type", "snappy")
    // 发送缓冲区大小
    .set("buffer.memory", "67108864")  // 64MB
    // 最大请求大小
    .set("max.request.size", "10485760")
    .create()?;
```

### Kafka 消费者调优

```rust
let consumer: StreamConsumer = ClientConfig::new()
    .set("bootstrap.servers", "localhost:9092")
    .set("group.id", "my_group")
    // 单次 fetch 最小字节数（等够了再返回，减少请求次数）
    .set("fetch.min.bytes",        "1024")
    // 等待 fetch.min.bytes 的最长时间
    .set("fetch.wait.max.ms",      "100")
    // 单次 fetch 最大字节数
    .set("fetch.max.bytes",        "52428800")
    // 批量处理消息数
    .set("max.poll.records",       "500")
    // 自动提交间隔
    .set("auto.commit.interval.ms","1000")
    .create()?;
```

### RabbitMQ 调优

```rust
// 合理的 prefetch_count
// 太小：消费者频繁空闲，吞吐量低
// 太大：消费者崩溃时大量消息重新投递
channel.basic_qos(20, BasicQosOptions::default()).await?;

// 批量 ACK（每处理 N 条再 ACK 一次）
// BasicAckOptions { multiple: true } 会确认所有 <= 当前 delivery_tag 的消息
delivery.ack(BasicAckOptions { multiple: true }).await?;
```

### 并发消费（Tokio 多任务）

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

// 控制并发处理数量
let semaphore = Arc::new(Semaphore::new(50)); // 最多 50 个并发

while let Some(Ok(msg)) = stream.next().await {
    let sem   = semaphore.clone();
    let data  = msg.payload().unwrap_or(&[]).to_vec();

    tokio::spawn(async move {
        let _permit = sem.acquire().await.unwrap(); // 获取令牌
        if let Err(e) = process(&data).await {
            eprintln!("处理失败: {}", e);
        }
        // _permit drop 时自动释放令牌
    });
}
```

---

## 14. 生产实践与最佳实践

### 连接管理

```rust
// 使用连接池，避免频繁创建/销毁连接
// Kafka rdkafka 内部已管理连接池

// RabbitMQ：单连接多 Channel（Channel 是轻量级）
let conn    = Connection::connect(url, ConnectionProperties::default()).await?;
let channel = conn.create_channel().await?; // 每个业务线用独立 channel
```

### 消息结构规范

```rust
#[derive(Serialize, Deserialize)]
struct Message<T> {
    // 元数据
    id:         String,           // UUID，用于幂等去重
    version:    u32,              // 消息版本，便于 schema 演进
    event_type: String,           // "order.created"
    source:     String,           // 来源服务
    timestamp:  i64,              // Unix 毫秒时间戳
    trace_id:   Option<String>,   // 分布式追踪 ID

    // 业务数据
    data: T,
}
```

### 监控指标

| 指标            | 意义                     | 告警阈值     |
| --------------- | ------------------------ | ------------ |
| Consumer Lag    | 消费者落后生产者的消息数 | > 10万       |
| Message Rate    | 每秒消息数               | 超过容量 80% |
| Error Rate      | 处理失败率               | > 1%         |
| Processing Time | 单条消息处理耗时         | > P99 基线   |
| Queue Depth     | 队列中积压消息数         | > 告警阈值   |

### 常见生产问题排查

```
问题 1：消息积压
原因：生产速度 > 消费速度
排查：查看 Consumer Lag 指标
解决：水平扩展消费者（增加实例）、优化处理逻辑、降低生产速率

问题 2：消息重复消费
原因：消费者重启、Rebalance、网络超时后重试
解决：业务层实现幂等（去重表、Redis SETNX）

问题 3：消息丢失
原因：acks 配置不当、消费者未处理完就 ACK、Broker 宕机
解决：acks=all、手动 ACK、副本数 >= 3

问题 4：消费者 Rebalance 频繁
原因：消费者心跳超时、处理时间过长
解决：增大 session.timeout.ms、减少单批消息量

问题 5：顺序消乱
原因：多 partition、多线程并发处理
解决：同业务用相同 key 固定到同一 partition，单线程消费该 partition
```

---

## 快速选型参考

```
需要复杂路由（按条件转发）            → RabbitMQ（Exchange）
需要高吞吐日志/流处理                 → Kafka
需要消息回放/时间旅行                 → Kafka（Offset 机制）
微服务间轻量通信                      → NATS
已有 Redis，规模不大                  → Redis Streams
需要 Exactly Once 事务保证            → Kafka（事务生产者）
需要 RPC 请求-响应模式                → NATS / RabbitMQ
```

---
