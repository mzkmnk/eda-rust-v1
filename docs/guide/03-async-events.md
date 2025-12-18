# Chapter 03: 非同期イベント処理

## 学習の目的

この章を完了すると、以下のことができるようになります：

- tokioランタイムの基本を理解し、非同期プログラムを作成できる
- `tokio::sync::mpsc`と`broadcast`チャネルを使い分けられる
- 複数のイベントストリームを並行処理できる
- 非同期コンテキストでのエラーハンドリングができる

## 背景知識

### なぜ非同期処理が必要なのか

Chapter 02では`std::sync::mpsc`を使用しましたが、これは同期的なチャネルです。実際のイベント駆動システムでは、以下の理由から非同期処理が必要になります：

1. **I/O待ち時間の有効活用**: ネットワークやディスクI/Oの待ち時間に他の処理を実行
2. **スケーラビリティ**: 少ないスレッドで多数の接続を処理
3. **リソース効率**: スレッドプールのオーバーヘッドを削減

```
同期処理:
┌─────────────────────────────────────────────────────────┐
│ Thread 1: [処理]──[I/O待ち]──────────[処理]            │
│ Thread 2: [処理]──[I/O待ち]──────────[処理]            │
│ Thread 3: [処理]──[I/O待ち]──────────[処理]            │
│                                                         │
│ → 3スレッド必要、I/O待ち時間は無駄                      │
└─────────────────────────────────────────────────────────┘

非同期処理:
┌─────────────────────────────────────────────────────────┐
│ Thread 1: [Task1]─[Task2]─[Task3]─[Task1]─[Task2]─... │
│                                                         │
│ → 1スレッドで複数タスクを効率的に処理                   │
└─────────────────────────────────────────────────────────┘
```

### tokioとは

tokioは、Rustの非同期ランタイムのデファクトスタンダードです。

**主な機能:**
- 非同期タスクのスケジューリング
- 非同期I/O（ネットワーク、ファイル）
- タイマーと遅延実行
- 同期プリミティブ（Mutex、RwLock、チャネル）

### async/awaitの基本

```rust
// 非同期関数の定義
async fn fetch_data() -> String {
    // 非同期処理
    "data".to_string()
}

// 非同期関数の呼び出し
async fn main() {
    let data = fetch_data().await;  // .awaitで完了を待つ
}
```

## 概念の説明

### tokioのチャネル

tokioは複数の非同期チャネルを提供しています：

| チャネル | 特徴 | 用途 |
|---------|------|------|
| `mpsc` | 複数送信者、単一受信者 | タスク間の一方向通信 |
| `broadcast` | 複数送信者、複数受信者 | イベントのブロードキャスト |
| `oneshot` | 単一送信、単一受信 | リクエスト/レスポンス |
| `watch` | 単一送信者、複数受信者（最新値のみ） | 状態の共有 |

### mpsc vs broadcast

```
mpsc (Multiple Producer, Single Consumer):
┌────────┐
│ Sender │──┐
└────────┘  │    ┌──────────┐
┌────────┐  ├───→│ Receiver │  1つのReceiverのみ
│ Sender │──┤    └──────────┘
└────────┘  │
┌────────┐  │
│ Sender │──┘
└────────┘

broadcast (Multiple Producer, Multiple Consumer):
┌────────┐       ┌──────────┐
│ Sender │──┬───→│Receiver 1│
└────────┘  │    └──────────┘
            ├───→│Receiver 2│  複数のReceiverが同じメッセージを受信
            │    └──────────┘
            └───→│Receiver 3│
                 └──────────┘
```

### バックプレッシャー

非同期チャネルでは、バッファサイズを指定してバックプレッシャーを制御できます：

```rust
// バッファサイズ32のチャネル
let (tx, rx) = mpsc::channel(32);

// バッファが満杯の場合、send().awaitはブロック
tx.send(event).await?;
```

## 実装タスク

### タスク1: tokioを使った基本的な非同期イベント処理

`tokio::sync::mpsc`を使用して、非同期でイベントを送受信するプログラムを作成してください。

**要件:**
- `#[tokio::main]`マクロを使用
- 非同期タスクでイベントを受信
- 複数のイベントを送信

### タスク2: broadcastチャネルでPub/Subを実装

`tokio::sync::broadcast`を使用して、複数のSubscriberにイベントをブロードキャストしてください。

**要件:**
- 3つのSubscriberを作成
- 各Subscriberは異なる処理を行う
- すべてのSubscriberが同じイベントを受信

### タスク3: 複数イベントストリームの並行処理

`tokio::select!`を使用して、複数のイベントソースを同時に監視してください。

**要件:**
- 注文イベントと在庫イベントの2つのストリーム
- どちらかのイベントが来たら処理
- タイムアウト処理を追加

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);

    // 受信タスク
    let handle = tokio::spawn(async move {
        while let Some(event) = rx.recv().await {
            println!("Received: {:?}", event);
        }
    });

    // 送信
    tx.send(event).await.unwrap();

    // チャネルを閉じる
    drop(tx);

    handle.await.unwrap();
}
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel(16);

// Subscriberを作成
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

// 受信
tokio::spawn(async move {
    while let Ok(event) = rx1.recv().await {
        // 処理
    }
});
```

</details>

<details>
<summary>タスク3のヒント</summary>

```rust
use tokio::select;
use tokio::time::{timeout, Duration};

loop {
    select! {
        Some(order) = order_rx.recv() => {
            println!("Order event: {:?}", order);
        }
        Some(inventory) = inventory_rx.recv() => {
            println!("Inventory event: {:?}", inventory);
        }
        _ = tokio::time::sleep(Duration::from_secs(5)) => {
            println!("Timeout - no events");
            break;
        }
    }
}
```

</details>

## 回答（コード例）

### Cargo.toml

```toml
[dependencies]
tokio = { version = "1.48", features = ["full"] }
uuid = { version = "1.11", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

### 完全な実装

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::time::Duration;
use tokio::sync::{broadcast, mpsc};
use tokio::time::sleep;
use uuid::Uuid;

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

#[derive(Debug, Clone)]
pub struct OrderPlaced {
    pub metadata: EventMetadata,
    pub customer_id: String,
    pub total_amount: u64,
}

#[derive(Debug, Clone)]
pub struct InventoryUpdated {
    pub metadata: EventMetadata,
    pub product_id: String,
    pub quantity_change: i32,
}

#[derive(Debug, Clone)]
pub enum DomainEvent {
    Order(OrderPlaced),
    Inventory(InventoryUpdated),
}

// ============================================================
// 非同期EventBus（broadcastベース）
// ============================================================

pub struct AsyncEventBus {
    sender: broadcast::Sender<DomainEvent>,
}

impl AsyncEventBus {
    pub fn new(capacity: usize) -> Self {
        let (sender, _) = broadcast::channel(capacity);
        Self { sender }
    }

    pub fn subscribe(&self) -> broadcast::Receiver<DomainEvent> {
        self.sender.subscribe()
    }

    pub fn publish(&self, event: DomainEvent) -> Result<usize, broadcast::error::SendError<DomainEvent>> {
        self.sender.send(event)
    }
}

// ============================================================
// イベントハンドラー
// ============================================================

/// 注文処理ハンドラー
async fn order_handler(mut rx: broadcast::Receiver<DomainEvent>) {
    println!("[OrderHandler] Started");
    loop {
        match rx.recv().await {
            Ok(DomainEvent::Order(order)) => {
                println!(
                    "[OrderHandler] Processing order: {} (Amount: {})",
                    order.metadata.aggregate_id, order.total_amount
                );
                // 注文処理のシミュレーション
                sleep(Duration::from_millis(100)).await;
            }
            Ok(_) => {} // 他のイベントは無視
            Err(broadcast::error::RecvError::Closed) => {
                println!("[OrderHandler] Channel closed");
                break;
            }
            Err(broadcast::error::RecvError::Lagged(n)) => {
                println!("[OrderHandler] Lagged behind by {} messages", n);
            }
        }
    }
}

/// 在庫管理ハンドラー
async fn inventory_handler(mut rx: broadcast::Receiver<DomainEvent>) {
    println!("[InventoryHandler] Started");
    loop {
        match rx.recv().await {
            Ok(DomainEvent::Inventory(inv)) => {
                println!(
                    "[InventoryHandler] Updating inventory: {} (Change: {:+})",
                    inv.product_id, inv.quantity_change
                );
            }
            Ok(_) => {}
            Err(broadcast::error::RecvError::Closed) => {
                println!("[InventoryHandler] Channel closed");
                break;
            }
            Err(broadcast::error::RecvError::Lagged(n)) => {
                println!("[InventoryHandler] Lagged behind by {} messages", n);
            }
        }
    }
}

/// 監査ログハンドラー（すべてのイベントを記録）
async fn audit_handler(mut rx: broadcast::Receiver<DomainEvent>) {
    println!("[AuditHandler] Started");
    loop {
        match rx.recv().await {
            Ok(event) => {
                let event_type = match &event {
                    DomainEvent::Order(_) => "Order",
                    DomainEvent::Inventory(_) => "Inventory",
                };
                println!("[AuditHandler] Logged event: {}", event_type);
            }
            Err(broadcast::error::RecvError::Closed) => {
                println!("[AuditHandler] Channel closed");
                break;
            }
            Err(broadcast::error::RecvError::Lagged(n)) => {
                println!("[AuditHandler] Lagged behind by {} messages", n);
            }
        }
    }
}

// ============================================================
// select!を使った複数ストリームの処理
// ============================================================

async fn multi_stream_processor(
    mut order_rx: mpsc::Receiver<OrderPlaced>,
    mut inventory_rx: mpsc::Receiver<InventoryUpdated>,
) {
    println!("[MultiStreamProcessor] Started");

    loop {
        tokio::select! {
            Some(order) = order_rx.recv() => {
                println!(
                    "[MultiStream] Order received: {}",
                    order.metadata.aggregate_id
                );
            }
            Some(inv) = inventory_rx.recv() => {
                println!(
                    "[MultiStream] Inventory update: {}",
                    inv.product_id
                );
            }
            _ = sleep(Duration::from_secs(2)) => {
                println!("[MultiStream] Timeout - no events for 2 seconds");
                break;
            }
        }
    }

    println!("[MultiStreamProcessor] Finished");
}

// ============================================================
// メイン関数
// ============================================================

#[tokio::main]
async fn main() {
    println!("=== Async Event Processing Demo ===\n");

    // --- Part 1: broadcastを使ったPub/Sub ---
    println!("--- Part 1: Broadcast Pub/Sub ---\n");

    let event_bus = AsyncEventBus::new(16);

    // ハンドラーを起動
    let order_handle = tokio::spawn(order_handler(event_bus.subscribe()));
    let inventory_handle = tokio::spawn(inventory_handler(event_bus.subscribe()));
    let audit_handle = tokio::spawn(audit_handler(event_bus.subscribe()));

    // イベントを発行
    sleep(Duration::from_millis(50)).await;

    event_bus
        .publish(DomainEvent::Order(OrderPlaced {
            metadata: EventMetadata::new("ORD-001"),
            customer_id: "CUST-001".to_string(),
            total_amount: 15000,
        }))
        .ok();

    event_bus
        .publish(DomainEvent::Inventory(InventoryUpdated {
            metadata: EventMetadata::new("INV-001"),
            product_id: "PROD-001".to_string(),
            quantity_change: -1,
        }))
        .ok();

    event_bus
        .publish(DomainEvent::Order(OrderPlaced {
            metadata: EventMetadata::new("ORD-002"),
            customer_id: "CUST-002".to_string(),
            total_amount: 8500,
        }))
        .ok();

    // 処理完了を待つ
    sleep(Duration::from_millis(500)).await;
    drop(event_bus);

    let _ = tokio::join!(order_handle, inventory_handle, audit_handle);

    // --- Part 2: select!を使った複数ストリーム処理 ---
    println!("\n--- Part 2: Multi-Stream Processing ---\n");

    let (order_tx, order_rx) = mpsc::channel(32);
    let (inventory_tx, inventory_rx) = mpsc::channel(32);

    let processor_handle = tokio::spawn(multi_stream_processor(order_rx, inventory_rx));

    // イベントを交互に送信
    order_tx
        .send(OrderPlaced {
            metadata: EventMetadata::new("ORD-003"),
            customer_id: "CUST-003".to_string(),
            total_amount: 5000,
        })
        .await
        .ok();

    inventory_tx
        .send(InventoryUpdated {
            metadata: EventMetadata::new("INV-002"),
            product_id: "PROD-002".to_string(),
            quantity_change: 10,
        })
        .await
        .ok();

    order_tx
        .send(OrderPlaced {
            metadata: EventMetadata::new("ORD-004"),
            customer_id: "CUST-004".to_string(),
            total_amount: 12000,
        })
        .await
        .ok();

    // タイムアウトを待つ
    processor_handle.await.unwrap();

    println!("\n=== Demo Complete ===");
}
```

### コード解説

#### broadcast vs mpsc の使い分け

- **broadcast**: 同じイベントを複数のハンドラーで処理したい場合
- **mpsc**: イベントを1つのハンドラーでのみ処理したい場合（ワーカーキュー）

#### Laggedエラーの処理

```rust
Err(broadcast::error::RecvError::Lagged(n)) => {
    println!("Lagged behind by {} messages", n);
}
```

broadcastチャネルはバッファサイズが固定です。Subscriberの処理が遅いと、古いメッセージが上書きされ`Lagged`エラーが発生します。

#### select!マクロ

```rust
tokio::select! {
    Some(order) = order_rx.recv() => { /* 処理 */ }
    Some(inv) = inventory_rx.recv() => { /* 処理 */ }
    _ = sleep(Duration::from_secs(2)) => { /* タイムアウト */ }
}
```

複数の非同期操作を同時に待ち、最初に完了したものを処理します。

## 発展課題

### 課題1: グレースフルシャットダウン

`tokio::signal`を使用して、Ctrl+Cでプログラムを安全に終了する仕組みを実装してください。

### 課題2: イベントの順序保証

同じaggregate_idを持つイベントが順序通りに処理されることを保証する仕組みを実装してください。

### 課題3: リトライ機能

イベント処理が失敗した場合に、指数バックオフでリトライする機能を追加してください。

## よくある間違い

### ❌ 間違い1: awaitを忘れる

```rust
// 悪い例: awaitがないため、Futureが実行されない
async fn process() {
    fetch_data();  // 何も起きない！
}

// 良い例
async fn process() {
    fetch_data().await;
}
```

### ❌ 間違い2: ブロッキング処理を非同期タスク内で実行

```rust
// 悪い例: std::thread::sleepはブロッキング
async fn bad_handler() {
    std::thread::sleep(Duration::from_secs(1));  // ランタイム全体をブロック！
}

// 良い例: tokio::time::sleepを使用
async fn good_handler() {
    tokio::time::sleep(Duration::from_secs(1)).await;
}
```

### ❌ 間違い3: broadcastのLaggedエラーを無視

```rust
// 悪い例
while let Ok(event) = rx.recv().await {
    // Laggedエラーでループが終了してしまう
}

// 良い例
loop {
    match rx.recv().await {
        Ok(event) => { /* 処理 */ }
        Err(RecvError::Lagged(n)) => {
            log::warn!("Missed {} events", n);
            continue;
        }
        Err(RecvError::Closed) => break,
    }
}
```

## まとめ

この章では、tokioを使用した非同期イベント処理を学びました。

### 学んだこと

1. **tokioの基本**: ランタイム、async/await
2. **非同期チャネル**: mpsc、broadcast、それぞれの特徴
3. **select!マクロ**: 複数ストリームの並行処理
4. **エラーハンドリング**: Lagged、Closedエラーの処理
5. **バックプレッシャー**: バッファサイズによる制御

### 次の章への準備

次の章では、イベントを永続化するイベントストアを実装します。これにより、システム再起動後もイベント履歴を保持できるようになります。

## 参考文献

- [Tokio Tutorial](https://tokio.rs/tokio/tutorial)
- [Tokio API Documentation](https://docs.rs/tokio/)
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)

---

[← 前の章: チャネルとPub/Sub](./02-channel-pubsub.md) | [次の章: イベントストア →](./04-event-store.md)
