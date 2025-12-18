# Chapter 07: AWS SQS連携

## 学習の目的

この章を完了すると、以下のことができるようになります：

- AWS SQSの基本概念と用途を理解する
- Rust AWS SDKを使用してSQSにメッセージを送受信できる
- Dead Letter Queue（DLQ）を設定して障害に対応できる
- メッセージの可視性タイムアウトと削除を適切に処理できる

## 背景知識

### AWS SQSとは

Amazon Simple Queue Service（SQS）は、フルマネージドなメッセージキューサービスです。

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS SQS                                  │
│                                                                 │
│  ┌──────────┐     ┌─────────────────────┐     ┌──────────┐    │
│  │ Producer │ ──→ │        Queue        │ ──→ │ Consumer │    │
│  └──────────┘     │  ┌───┬───┬───┬───┐  │     └──────────┘    │
│                   │  │ M │ M │ M │ M │  │                      │
│                   │  └───┴───┴───┴───┘  │                      │
│                   └─────────────────────┘                      │
│                                                                 │
│  特徴:                                                          │
│  - フルマネージド（インフラ管理不要）                            │
│  - 高可用性（複数AZに自動レプリケーション）                      │
│  - スケーラブル（無制限のスループット）                          │
└─────────────────────────────────────────────────────────────────┘
```

### SQSのキュータイプ

| タイプ | 特徴 | 用途 |
|--------|------|------|
| **Standard** | 最大スループット、少なくとも1回配信、順序保証なし | 高スループットが必要な場合 |
| **FIFO** | 厳密な順序保証、正確に1回配信、300 TPS | 順序が重要な場合 |

### メッセージのライフサイクル

```
┌─────────────────────────────────────────────────────────────────┐
│                 Message Lifecycle                               │
│                                                                 │
│  1. 送信                                                        │
│     Producer ──SendMessage──→ Queue                             │
│                                                                 │
│  2. 受信（可視性タイムアウト開始）                               │
│     Consumer ←──ReceiveMessage── Queue                          │
│     ┌─────────────────────────────────────────┐                │
│     │ Message: Invisible (処理中)              │                │
│     │ Visibility Timeout: 30秒（デフォルト）   │                │
│     └─────────────────────────────────────────┘                │
│                                                                 │
│  3a. 成功: 削除                                                 │
│      Consumer ──DeleteMessage──→ Queue                          │
│                                                                 │
│  3b. 失敗: タイムアウト後に再度可視化                           │
│      Message: Visible again (再処理可能)                        │
│                                                                 │
│  4. DLQ: 最大受信回数超過                                       │
│     Queue ──→ Dead Letter Queue                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Dead Letter Queue（DLQ）

処理に繰り返し失敗したメッセージを隔離するためのキューです。

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Main Queue                          Dead Letter Queue          │
│  ┌─────────────────┐                ┌─────────────────┐        │
│  │ maxReceiveCount │                │ 失敗メッセージ   │        │
│  │ = 3             │ ─────────────→ │ の調査・再処理   │        │
│  └─────────────────┘  3回失敗後     └─────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### Rust AWS SDKの構成

```rust
// 依存関係
// aws-config: AWS設定（認証情報、リージョン）
// aws-sdk-sqs: SQS操作

use aws_config::BehaviorVersion;
use aws_sdk_sqs::Client;

// クライアント初期化
let config = aws_config::defaults(BehaviorVersion::latest())
    .region("ap-northeast-1")
    .load()
    .await;
let client = Client::new(&config);
```

### LocalStackでのローカル開発

実際のAWSリソースを使用せずにローカルで開発・テストする場合は、LocalStackを使用します。

```bash
# LocalStackの起動
docker run -d --name localstack -p 4566:4566 -e SERVICES=sqs localstack/localstack

# SQSキューの作成
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name order-events

# キューURLの確認
aws --endpoint-url=http://localhost:4566 sqs list-queues
```

LocalStack使用時のRustコード：

```rust
let config = aws_config::defaults(BehaviorVersion::latest())
    .endpoint_url("http://localhost:4566")  // LocalStackエンドポイント
    .region("ap-northeast-1")
    .load()
    .await;
let client = Client::new(&config);

// キューURL（LocalStack）
let queue_url = "http://sqs.ap-northeast-1.localhost.localstack.cloud:4566/000000000000/order-events";
```

> 💡 **Tip**: 環境変数で切り替えると便利です
> ```bash
> # LocalStack使用時
> export AWS_ENDPOINT_URL=http://localhost:4566
> export SQS_QUEUE_URL=http://sqs.ap-northeast-1.localhost.localstack.cloud:4566/000000000000/order-events
> ```

### 主要な操作

| 操作 | 説明 |
|------|------|
| `send_message` | メッセージを送信 |
| `receive_message` | メッセージを受信 |
| `delete_message` | メッセージを削除 |
| `change_message_visibility` | 可視性タイムアウトを変更 |

## 実装タスク

### タスク1: メッセージの送信

SQSキューにイベントメッセージを送信する機能を実装してください。

**要件:**
- イベントをJSONにシリアライズして送信
- メッセージ属性にイベントタイプを設定
- 送信結果（MessageId）をログ出力

### タスク2: メッセージの受信と処理

SQSキューからメッセージを受信して処理する機能を実装してください。

**要件:**
- ロングポーリングで効率的に受信
- メッセージをデシリアライズしてイベントとして処理
- 処理成功後にメッセージを削除

### タスク3: エラーハンドリングとDLQ

処理失敗時の適切なエラーハンドリングを実装してください。

**要件:**
- 一時的なエラーは可視性タイムアウトを延長してリトライ
- 永続的なエラーはDLQに送信
- エラーログの出力

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
async fn send_event(
    client: &Client,
    queue_url: &str,
    event: &OrderEvent,
) -> Result<String, Error> {
    let body = serde_json::to_string(event)?;

    let result = client
        .send_message()
        .queue_url(queue_url)
        .message_body(body)
        .message_attribute(
            "EventType",
            MessageAttributeValue::builder()
                .data_type("String")
                .string_value(event.event_type())
                .build()?,
        )
        .send()
        .await?;

    Ok(result.message_id().unwrap_or_default().to_string())
}
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
async fn receive_and_process(
    client: &Client,
    queue_url: &str,
) -> Result<(), Error> {
    let result = client
        .receive_message()
        .queue_url(queue_url)
        .max_number_of_messages(10)
        .wait_time_seconds(20)  // ロングポーリング
        .send()
        .await?;

    for message in result.messages() {
        let body = message.body().unwrap_or_default();
        let event: OrderEvent = serde_json::from_str(body)?;

        // 処理
        process_event(&event).await?;

        // 削除
        client
            .delete_message()
            .queue_url(queue_url)
            .receipt_handle(message.receipt_handle().unwrap())
            .send()
            .await?;
    }

    Ok(())
}
```

</details>

## 回答（コード例）

### Cargo.toml

```toml
[dependencies]
tokio = { version = "1.48", features = ["full"] }
aws-config = { version = "1.6", features = ["behavior-version-latest"] }
aws-sdk-sqs = "1.84"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1.11", features = ["v4", "serde"] }
thiserror = "2.0"
```

### 完全な実装

```rust
use aws_config::BehaviorVersion;
use aws_sdk_sqs::{
    types::{MessageAttributeValue, QueueAttributeName},
    Client,
};
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::time::Duration;
use thiserror::Error;
use uuid::Uuid;

// ============================================================
// エラー型
// ============================================================

#[derive(Error, Debug)]
pub enum SqsError {
    #[error("AWS SDK error: {0}")]
    AwsSdk(String),
    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),
    #[error("Processing error: {0}")]
    Processing(String),
}

// ============================================================
// イベント定義
// ============================================================

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventMetadata {
    pub event_id: Uuid,
    pub timestamp: DateTime<Utc>,
    pub aggregate_id: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    pub metadata: EventMetadata,
    pub customer_id: String,
    pub total_amount: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderShipped {
    pub metadata: EventMetadata,
    pub tracking_number: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum OrderEvent {
    Placed(OrderPlaced),
    Shipped(OrderShipped),
}

impl OrderEvent {
    pub fn event_type(&self) -> &'static str {
        match self {
            OrderEvent::Placed(_) => "OrderPlaced",
            OrderEvent::Shipped(_) => "OrderShipped",
        }
    }
}

// ============================================================
// SQSプロデューサー
// ============================================================

pub struct SqsProducer {
    client: Client,
    queue_url: String,
}

impl SqsProducer {
    pub fn new(client: Client, queue_url: String) -> Self {
        Self { client, queue_url }
    }

    pub async fn send(&self, event: &OrderEvent) -> Result<String, SqsError> {
        let body = serde_json::to_string(event)?;

        let attr = MessageAttributeValue::builder()
            .data_type("String")
            .string_value(event.event_type())
            .build()
            .map_err(|e| SqsError::AwsSdk(e.to_string()))?;

        let result = self.client
            .send_message()
            .queue_url(&self.queue_url)
            .message_body(body)
            .message_attribute("EventType", attr)
            .send()
            .await
            .map_err(|e| SqsError::AwsSdk(e.to_string()))?;

        let message_id = result.message_id().unwrap_or_default().to_string();
        println!("[Producer] Sent message: {}", message_id);

        Ok(message_id)
    }
}

// ============================================================
// SQSコンシューマー
// ============================================================

pub struct SqsConsumer {
    client: Client,
    queue_url: String,
    visibility_timeout: i32,
}

impl SqsConsumer {
    pub fn new(client: Client, queue_url: String) -> Self {
        Self {
            client,
            queue_url,
            visibility_timeout: 30,
        }
    }

    pub async fn poll_and_process<F, Fut>(&self, handler: F) -> Result<usize, SqsError>
    where
        F: Fn(OrderEvent) -> Fut,
        Fut: std::future::Future<Output = Result<(), SqsError>>,
    {
        let result = self.client
            .receive_message()
            .queue_url(&self.queue_url)
            .max_number_of_messages(10)
            .wait_time_seconds(20)
            .visibility_timeout(self.visibility_timeout)
            .message_attribute_names("All")
            .send()
            .await
            .map_err(|e| SqsError::AwsSdk(e.to_string()))?;

        let messages = result.messages();
        let count = messages.len();

        for message in messages {
            let body = message.body().unwrap_or_default();
            let receipt_handle = message.receipt_handle().unwrap_or_default();

            println!("[Consumer] Received message: {:?}", message.message_id());

            match serde_json::from_str::<OrderEvent>(body) {
                Ok(event) => {
                    match handler(event).await {
                        Ok(()) => {
                            self.delete_message(receipt_handle).await?;
                            println!("[Consumer] Message processed and deleted");
                        }
                        Err(e) => {
                            println!("[Consumer] Processing failed: {}", e);
                            // メッセージは削除せず、可視性タイムアウト後に再処理
                        }
                    }
                }
                Err(e) => {
                    println!("[Consumer] Failed to deserialize: {}", e);
                    // 不正なメッセージは削除（DLQに移動させる場合はここで処理）
                }
            }
        }

        Ok(count)
    }

    async fn delete_message(&self, receipt_handle: &str) -> Result<(), SqsError> {
        self.client
            .delete_message()
            .queue_url(&self.queue_url)
            .receipt_handle(receipt_handle)
            .send()
            .await
            .map_err(|e| SqsError::AwsSdk(e.to_string()))?;

        Ok(())
    }

    pub async fn extend_visibility(
        &self,
        receipt_handle: &str,
        timeout: i32,
    ) -> Result<(), SqsError> {
        self.client
            .change_message_visibility()
            .queue_url(&self.queue_url)
            .receipt_handle(receipt_handle)
            .visibility_timeout(timeout)
            .send()
            .await
            .map_err(|e| SqsError::AwsSdk(e.to_string()))?;

        Ok(())
    }
}

// ============================================================
// イベントハンドラー
// ============================================================

async fn handle_event(event: OrderEvent) -> Result<(), SqsError> {
    match &event {
        OrderEvent::Placed(e) => {
            println!(
                "  Processing OrderPlaced: order={}, customer={}, amount={}",
                e.metadata.aggregate_id, e.customer_id, e.total_amount
            );
            // 実際の処理をここに実装
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
        OrderEvent::Shipped(e) => {
            println!(
                "  Processing OrderShipped: order={}, tracking={}",
                e.metadata.aggregate_id, e.tracking_number
            );
        }
    }
    Ok(())
}

// ============================================================
// メイン関数
// ============================================================

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== AWS SQS Demo ===\n");

    // AWS設定をロード
    let config = aws_config::defaults(BehaviorVersion::latest())
        .region("ap-northeast-1")
        .load()
        .await;

    let client = Client::new(&config);

    // 注意: 実際のキューURLに置き換えてください
    let queue_url = std::env::var("SQS_QUEUE_URL")
        .unwrap_or_else(|_| "https://sqs.ap-northeast-1.amazonaws.com/123456789012/my-queue".to_string());

    let producer = SqsProducer::new(client.clone(), queue_url.clone());
    let consumer = SqsConsumer::new(client, queue_url);

    // --- メッセージ送信 ---
    println!("--- Sending Messages ---\n");

    let events = vec![
        OrderEvent::Placed(OrderPlaced {
            metadata: EventMetadata {
                event_id: Uuid::new_v4(),
                timestamp: Utc::now(),
                aggregate_id: "ORD-001".to_string(),
            },
            customer_id: "CUST-001".to_string(),
            total_amount: 15000,
        }),
        OrderEvent::Shipped(OrderShipped {
            metadata: EventMetadata {
                event_id: Uuid::new_v4(),
                timestamp: Utc::now(),
                aggregate_id: "ORD-001".to_string(),
            },
            tracking_number: "TRACK-123".to_string(),
        }),
    ];

    for event in &events {
        producer.send(event).await?;
    }

    // --- メッセージ受信 ---
    println!("\n--- Receiving Messages ---\n");

    let processed = consumer.poll_and_process(handle_event).await?;
    println!("\nProcessed {} messages", processed);

    println!("\n=== Demo Complete ===");
    Ok(())
}
```

## 発展課題

### 課題1: バッチ送信

複数のメッセージを一度に送信する`send_message_batch`を実装してください。

### 課題2: FIFOキュー対応

FIFOキューを使用して、メッセージの順序を保証する実装を行ってください。

### 課題3: メトリクス収集

処理時間、成功/失敗数などのメトリクスを収集する仕組みを追加してください。

## よくある間違い

### ❌ 間違い1: メッセージを削除し忘れる

```rust
// 悪い例: 処理後に削除しない
async fn process(message: &Message) {
    handle_event(message).await;
    // delete_message を呼び忘れ → 可視性タイムアウト後に再処理される
}

// 良い例
async fn process(message: &Message) {
    handle_event(message).await?;
    client.delete_message(...).await?;
}
```

### ❌ 間違い2: 可視性タイムアウトが短すぎる

```rust
// 悪い例: 処理時間より短いタイムアウト
.visibility_timeout(5)  // 5秒
// 処理に10秒かかる → 処理中に再度可視化され、重複処理

// 良い例: 処理時間の2-3倍を設定
.visibility_timeout(30)
```

### ❌ 間違い3: ロングポーリングを使わない

```rust
// 悪い例: ショートポーリング（コスト増、遅延）
.wait_time_seconds(0)

// 良い例: ロングポーリング
.wait_time_seconds(20)  // 最大20秒
```

## まとめ

この章では、AWS SQSを使用したメッセージキューイングを学びました。

### 学んだこと

1. **SQSの基本**: キュータイプ、メッセージライフサイクル
2. **Rust AWS SDK**: クライアント初期化、メッセージ操作
3. **プロデューサー/コンシューマー**: 送受信の実装
4. **エラーハンドリング**: DLQ、可視性タイムアウト

### 次の章への準備

次の章では、AWS EventBridgeを使用したイベントルーティングを学びます。より複雑なイベント駆動アーキテクチャを構築します。

## 参考文献

- [AWS SQS Developer Guide](https://docs.aws.amazon.com/sqs/latest/dg/welcome.html)
- [AWS SDK for Rust - SQS](https://docs.rs/aws-sdk-sqs/)
- [SQS Best Practices](https://docs.aws.amazon.com/sqs/latest/dg/sqs-best-practices.html)

---

[← 前の章: CQRSパターン](./06-cqrs.md) | [次の章: AWS EventBridge連携 →](./08-aws-eventbridge.md)
