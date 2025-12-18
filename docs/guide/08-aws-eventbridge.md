# Chapter 08: AWS EventBridge連携

## 学習の目的

この章を完了すると、以下のことができるようになります：

- AWS EventBridgeの基本概念とSQSとの違いを理解する
- Rust AWS SDKを使用してEventBridgeにイベントを発行できる
- イベントパターンによるルーティングを設計できる
- 複数のターゲットにイベントを配信できる

## 背景知識

### AWS EventBridgeとは

Amazon EventBridgeは、サーバーレスイベントバスサービスです。イベントをルールに基づいて複数のターゲットにルーティングできます。

```
┌─────────────────────────────────────────────────────────────────┐
│                     AWS EventBridge                             │
│                                                                 │
│  ┌──────────┐     ┌─────────────────────────────────────────┐  │
│  │ Producer │ ──→ │              Event Bus                  │  │
│  └──────────┘     │  ┌─────────────────────────────────┐    │  │
│                   │  │           Rules                  │    │  │
│  ┌──────────┐     │  │  ┌───────┐  ┌───────┐          │    │  │
│  │ Producer │ ──→ │  │  │Rule 1 │  │Rule 2 │  ...     │    │  │
│  └──────────┘     │  │  └───┬───┘  └───┬───┘          │    │  │
│                   │  └──────┼──────────┼──────────────┘    │  │
│                   └─────────┼──────────┼───────────────────┘  │
│                             │          │                       │
│                             ↓          ↓                       │
│                        ┌────────┐ ┌────────┐                  │
│                        │ Lambda │ │  SQS   │                  │
│                        └────────┘ └────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

### SQS vs EventBridge

| 特徴 | SQS | EventBridge |
|------|-----|-------------|
| **モデル** | Point-to-Point | Pub/Sub |
| **ルーティング** | なし | ルールベース |
| **ターゲット** | 単一コンシューマー | 複数ターゲット |
| **フィルタリング** | 受信側で実装 | ルールで定義 |
| **用途** | ワーカーキュー | イベント配信 |

### EventBridgeの主要概念

| 概念 | 説明 |
|------|------|
| **Event Bus** | イベントを受け取るパイプライン |
| **Rule** | イベントをフィルタリングしてターゲットに送る |
| **Event Pattern** | ルールのフィルタ条件（JSON形式） |
| **Target** | イベントの送信先（Lambda、SQS、SNS等） |

### イベントの構造

```json
{
  "version": "0",
  "id": "12345678-1234-1234-1234-123456789012",
  "detail-type": "OrderPlaced",
  "source": "com.example.orders",
  "account": "123456789012",
  "time": "2024-01-15T10:30:00Z",
  "region": "ap-northeast-1",
  "resources": [],
  "detail": {
    "order_id": "ORD-001",
    "customer_id": "CUST-001",
    "total_amount": 15000
  }
}
```

## 概念の説明

### イベントパターンによるフィルタリング

```
┌─────────────────────────────────────────────────────────────────┐
│                    Event Pattern Matching                       │
│                                                                 │
│  Event:                          Pattern:                       │
│  {                               {                              │
│    "source": "orders",             "source": ["orders"],        │
│    "detail-type": "OrderPlaced",   "detail-type": ["OrderPlaced"]│
│    "detail": {                   }                              │
│      "total_amount": 15000       → Match!                       │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  Pattern (金額フィルタ):                                        │
│  {                                                              │
│    "source": ["orders"],                                        │
│    "detail": {                                                  │
│      "total_amount": [{"numeric": [">=", 10000]}]              │
│    }                                                            │
│  }                                                              │
│  → Match! (15000 >= 10000)                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 複数ターゲットへの配信

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  OrderPlaced Event                                              │
│        │                                                        │
│        ↓                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Event Bus                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        ├──→ Rule: "all-orders" ──→ CloudWatch Logs (監査)      │
│        │                                                        │
│        ├──→ Rule: "high-value" ──→ SNS (通知)                  │
│        │    (amount >= 10000)                                   │
│        │                                                        │
│        └──→ Rule: "fulfillment" ──→ SQS (処理キュー)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 実装タスク

### タスク1: EventBridgeへのイベント発行

EventBridgeにドメインイベントを発行する機能を実装してください。

**要件:**
- `put_events`でイベントを送信
- `source`と`detail-type`を適切に設定
- イベントデータを`detail`に含める

### タスク2: バッチイベント発行

複数のイベントを一度に発行する機能を実装してください。

**要件:**
- 最大10件のイベントをバッチ送信
- 部分的な失敗を処理
- 失敗したイベントをリトライ

### タスク3: イベントパターンの設計

様々なユースケースに対応するイベントパターンを設計してください。

**要件:**
- 特定のイベントタイプのみマッチ
- 金額による条件フィルタ
- 複数条件の組み合わせ

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
use aws_sdk_eventbridge::{Client, types::PutEventsRequestEntry};

async fn put_event(
    client: &Client,
    event_bus_name: &str,
    event: &OrderEvent,
) -> Result<(), Error> {
    let detail = serde_json::to_string(event)?;

    let entry = PutEventsRequestEntry::builder()
        .event_bus_name(event_bus_name)
        .source("com.example.orders")
        .detail_type(event.event_type())
        .detail(detail)
        .build();

    client
        .put_events()
        .entries(entry)
        .send()
        .await?;

    Ok(())
}
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
async fn put_events_batch(
    client: &Client,
    event_bus_name: &str,
    events: Vec<OrderEvent>,
) -> Result<Vec<OrderEvent>, Error> {
    let entries: Vec<_> = events.iter()
        .map(|e| /* エントリを作成 */)
        .collect();

    let result = client
        .put_events()
        .set_entries(Some(entries))
        .send()
        .await?;

    // 失敗したイベントを収集
    let failed: Vec<_> = result.entries()
        .iter()
        .zip(events.iter())
        .filter(|(entry, _)| entry.error_code().is_some())
        .map(|(_, event)| event.clone())
        .collect();

    Ok(failed)
}
```

</details>

## 回答（コード例）

### Cargo.toml

```toml
[dependencies]
tokio = { version = "1.48", features = ["full"] }
aws-config = { version = "1.6", features = ["behavior-version-latest"] }
aws-sdk-eventbridge = "1.80"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1.11", features = ["v4", "serde"] }
thiserror = "2.0"
```

### 完全な実装

```rust
use aws_config::BehaviorVersion;
use aws_sdk_eventbridge::{types::PutEventsRequestEntry, Client};
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use thiserror::Error;
use uuid::Uuid;

// ============================================================
// エラー型
// ============================================================

#[derive(Error, Debug)]
pub enum EventBridgeError {
    #[error("AWS SDK error: {0}")]
    AwsSdk(String),
    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),
    #[error("Partial failure: {0} events failed")]
    PartialFailure(usize),
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

impl EventMetadata {
    pub fn new(aggregate_id: impl Into<String>) -> Self {
        Self {
            event_id: Uuid::new_v4(),
            timestamp: Utc::now(),
            aggregate_id: aggregate_id.into(),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    pub metadata: EventMetadata,
    pub customer_id: String,
    pub total_amount: u64,
    pub items: Vec<OrderItem>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderItem {
    pub product_id: String,
    pub quantity: u32,
    pub unit_price: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderShipped {
    pub metadata: EventMetadata,
    pub tracking_number: String,
    pub carrier: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderCancelled {
    pub metadata: EventMetadata,
    pub reason: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum OrderEvent {
    Placed(OrderPlaced),
    Shipped(OrderShipped),
    Cancelled(OrderCancelled),
}

impl OrderEvent {
    pub fn event_type(&self) -> &'static str {
        match self {
            OrderEvent::Placed(_) => "OrderPlaced",
            OrderEvent::Shipped(_) => "OrderShipped",
            OrderEvent::Cancelled(_) => "OrderCancelled",
        }
    }

    pub fn aggregate_id(&self) -> &str {
        match self {
            OrderEvent::Placed(e) => &e.metadata.aggregate_id,
            OrderEvent::Shipped(e) => &e.metadata.aggregate_id,
            OrderEvent::Cancelled(e) => &e.metadata.aggregate_id,
        }
    }
}

// ============================================================
// EventBridgeパブリッシャー
// ============================================================

pub struct EventBridgePublisher {
    client: Client,
    event_bus_name: String,
    source: String,
}

impl EventBridgePublisher {
    pub fn new(client: Client, event_bus_name: String, source: String) -> Self {
        Self {
            client,
            event_bus_name,
            source,
        }
    }

    /// 単一イベントを発行
    pub async fn publish(&self, event: &OrderEvent) -> Result<String, EventBridgeError> {
        let detail = serde_json::to_string(event)?;

        let entry = PutEventsRequestEntry::builder()
            .event_bus_name(&self.event_bus_name)
            .source(&self.source)
            .detail_type(event.event_type())
            .detail(detail)
            .resources(format!("order/{}", event.aggregate_id()))
            .build();

        let result = self
            .client
            .put_events()
            .entries(entry)
            .send()
            .await
            .map_err(|e| EventBridgeError::AwsSdk(e.to_string()))?;

        if result.failed_entry_count() > 0 {
            let error = result
                .entries()
                .first()
                .and_then(|e| e.error_message())
                .unwrap_or("Unknown error");
            return Err(EventBridgeError::AwsSdk(error.to_string()));
        }

        let event_id = result
            .entries()
            .first()
            .and_then(|e| e.event_id())
            .unwrap_or_default()
            .to_string();

        println!("[EventBridge] Published: {} ({})", event.event_type(), event_id);

        Ok(event_id)
    }

    /// バッチでイベントを発行（最大10件）
    pub async fn publish_batch(
        &self,
        events: &[OrderEvent],
    ) -> Result<BatchResult, EventBridgeError> {
        if events.is_empty() {
            return Ok(BatchResult::default());
        }

        let entries: Result<Vec<_>, _> = events
            .iter()
            .map(|event| {
                let detail = serde_json::to_string(event)?;
                Ok(PutEventsRequestEntry::builder()
                    .event_bus_name(&self.event_bus_name)
                    .source(&self.source)
                    .detail_type(event.event_type())
                    .detail(detail)
                    .build())
            })
            .collect();

        let entries = entries?;

        let result = self
            .client
            .put_events()
            .set_entries(Some(entries))
            .send()
            .await
            .map_err(|e| EventBridgeError::AwsSdk(e.to_string()))?;

        let mut batch_result = BatchResult::default();

        for (i, entry) in result.entries().iter().enumerate() {
            if entry.error_code().is_some() {
                batch_result.failed.push(FailedEntry {
                    index: i,
                    error_code: entry.error_code().unwrap_or_default().to_string(),
                    error_message: entry.error_message().unwrap_or_default().to_string(),
                });
            } else {
                batch_result.successful += 1;
            }
        }

        println!(
            "[EventBridge] Batch result: {} successful, {} failed",
            batch_result.successful,
            batch_result.failed.len()
        );

        Ok(batch_result)
    }

    /// リトライ付きバッチ発行
    pub async fn publish_batch_with_retry(
        &self,
        events: &[OrderEvent],
        max_retries: usize,
    ) -> Result<(), EventBridgeError> {
        let mut remaining: Vec<_> = events.to_vec();
        let mut retries = 0;

        while !remaining.is_empty() && retries < max_retries {
            let result = self.publish_batch(&remaining).await?;

            if result.failed.is_empty() {
                return Ok(());
            }

            // 失敗したイベントのみ残す
            remaining = result
                .failed
                .iter()
                .filter_map(|f| remaining.get(f.index).cloned())
                .collect();

            retries += 1;
            println!("[EventBridge] Retry {}/{}: {} events remaining", retries, max_retries, remaining.len());

            tokio::time::sleep(std::time::Duration::from_millis(100 * (1 << retries))).await;
        }

        if !remaining.is_empty() {
            return Err(EventBridgeError::PartialFailure(remaining.len()));
        }

        Ok(())
    }
}

#[derive(Debug, Default)]
pub struct BatchResult {
    pub successful: usize,
    pub failed: Vec<FailedEntry>,
}

#[derive(Debug)]
pub struct FailedEntry {
    pub index: usize,
    pub error_code: String,
    pub error_message: String,
}

// ============================================================
// イベントパターン（ルール設定用）
// ============================================================

pub mod patterns {
    use serde_json::json;

    /// すべての注文イベントにマッチ
    pub fn all_orders() -> serde_json::Value {
        json!({
            "source": ["com.example.orders"]
        })
    }

    /// 特定のイベントタイプにマッチ
    pub fn by_event_type(event_type: &str) -> serde_json::Value {
        json!({
            "source": ["com.example.orders"],
            "detail-type": [event_type]
        })
    }

    /// 高額注文にマッチ（10,000円以上）
    pub fn high_value_orders(min_amount: u64) -> serde_json::Value {
        json!({
            "source": ["com.example.orders"],
            "detail-type": ["OrderPlaced"],
            "detail": {
                "total_amount": [{"numeric": [">=", min_amount]}]
            }
        })
    }

    /// 特定の顧客の注文にマッチ
    pub fn by_customer(customer_id: &str) -> serde_json::Value {
        json!({
            "source": ["com.example.orders"],
            "detail": {
                "customer_id": [customer_id]
            }
        })
    }
}

// ============================================================
// 使用例
// ============================================================

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== AWS EventBridge Demo ===\n");

    // AWS設定をロード
    let config = aws_config::defaults(BehaviorVersion::latest())
        .region("ap-northeast-1")
        .load()
        .await;

    let client = Client::new(&config);

    // 注意: 実際のイベントバス名に置き換えてください
    let event_bus_name = std::env::var("EVENT_BUS_NAME")
        .unwrap_or_else(|_| "default".to_string());

    let publisher = EventBridgePublisher::new(
        client,
        event_bus_name,
        "com.example.orders".to_string(),
    );

    // --- 単一イベント発行 ---
    println!("--- Publishing Single Event ---\n");

    let event = OrderEvent::Placed(OrderPlaced {
        metadata: EventMetadata::new("ORD-001"),
        customer_id: "CUST-001".to_string(),
        total_amount: 15000,
        items: vec![OrderItem {
            product_id: "PROD-001".to_string(),
            quantity: 2,
            unit_price: 7500,
        }],
    });

    publisher.publish(&event).await?;

    // --- バッチイベント発行 ---
    println!("\n--- Publishing Batch Events ---\n");

    let events = vec![
        OrderEvent::Placed(OrderPlaced {
            metadata: EventMetadata::new("ORD-002"),
            customer_id: "CUST-002".to_string(),
            total_amount: 5000,
            items: vec![],
        }),
        OrderEvent::Shipped(OrderShipped {
            metadata: EventMetadata::new("ORD-001"),
            tracking_number: "TRACK-123".to_string(),
            carrier: "Yamato".to_string(),
        }),
        OrderEvent::Cancelled(OrderCancelled {
            metadata: EventMetadata::new("ORD-002"),
            reason: "Customer request".to_string(),
        }),
    ];

    publisher.publish_batch(&events).await?;

    // --- イベントパターンの例 ---
    println!("\n--- Event Patterns ---\n");

    println!("All orders pattern:");
    println!("{}\n", serde_json::to_string_pretty(&patterns::all_orders())?);

    println!("High value orders pattern (>= 10000):");
    println!("{}\n", serde_json::to_string_pretty(&patterns::high_value_orders(10000))?);

    println!("By customer pattern:");
    println!("{}\n", serde_json::to_string_pretty(&patterns::by_customer("CUST-001"))?);

    println!("=== Demo Complete ===");
    Ok(())
}
```

## 発展課題

### 課題1: アーカイブとリプレイ

EventBridgeのアーカイブ機能を使用して、過去のイベントをリプレイする仕組みを実装してください。

### 課題2: スキーマレジストリ

EventBridge Schema Registryを使用して、イベントスキーマを管理してください。

### 課題3: クロスアカウント配信

異なるAWSアカウントにイベントを配信する設定を行ってください。

## よくある間違い

### ❌ 間違い1: detailをJSON文字列にしない

```rust
// 悪い例: detailにオブジェクトを直接渡す
.detail(event)  // コンパイルエラー

// 良い例: JSON文字列に変換
.detail(serde_json::to_string(&event)?)
```

### ❌ 間違い2: バッチサイズの制限を超える

```rust
// 悪い例: 10件を超えるバッチ
let events: Vec<_> = (0..20).map(|_| create_event()).collect();
publisher.publish_batch(&events).await?;  // エラー

// 良い例: チャンクに分割
for chunk in events.chunks(10) {
    publisher.publish_batch(chunk).await?;
}
```

### ❌ 間違い3: 部分的な失敗を無視

```rust
// 悪い例: failed_entry_countをチェックしない
let result = client.put_events().entries(entries).send().await?;
// 一部失敗していても気づかない

// 良い例: 失敗をチェック
if result.failed_entry_count() > 0 {
    // 失敗したエントリを処理
}
```

## まとめ

この章では、AWS EventBridgeを使用したイベントルーティングを学びました。

### 学んだこと

1. **EventBridgeの基本**: イベントバス、ルール、ターゲット
2. **SQSとの違い**: Pub/Subモデル、ルールベースルーティング
3. **イベント発行**: 単一およびバッチ発行
4. **イベントパターン**: フィルタリング条件の設計

### 次の章への準備

次の章では、Sagaパターンを学びます。分散トランザクションを管理し、障害時の補償処理を実装します。

## 参考文献

- [AWS EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)
- [AWS SDK for Rust - EventBridge](https://docs.rs/aws-sdk-eventbridge/)
- [EventBridge Event Patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)

---

[← 前の章: AWS SQS連携](./07-aws-sqs.md) | [次の章: Sagaパターン →](./09-saga-pattern.md)
