# Chapter 02: チャネルとPub/Subパターン

## 学習の目的

この章を完了すると、以下のことができるようになります：

- Rustのチャネル（`std::sync::mpsc`）を使用してコンポーネント間でメッセージを送受信できる
- Publisher/Subscriberパターンの概念と利点を理解する
- 複数のSubscriberにイベントをブロードキャストする仕組みを実装できる

## 背景知識

### なぜチャネルが必要なのか

Chapter 01では、イベントの定義方法を学びました。しかし、イベントを定義しただけでは意味がありません。イベントを**発行**し、関心のあるコンポーネントに**配信**する仕組みが必要です。

```
┌─────────────┐                      ┌─────────────┐
│  Producer   │  イベントをどうやって  │  Consumer   │
│  (発行者)   │  ───────────────→   │  (購読者)   │
└─────────────┘      届ける？        └─────────────┘
```

直接的な関数呼び出しでは、以下の問題があります：

1. **密結合**: ProducerがConsumerを直接知っている必要がある
2. **同期的**: Consumerの処理が完了するまでProducerがブロックされる
3. **拡張性の欠如**: 新しいConsumerを追加するたびにProducerを修正する必要がある

チャネルを使用することで、これらの問題を解決できます。

### Rustのチャネル

Rustの標準ライブラリは`std::sync::mpsc`モジュールでチャネルを提供しています。

> **mpsc** = Multiple Producer, Single Consumer（複数の送信者、単一の受信者）

```rust
use std::sync::mpsc;

// チャネルを作成
let (tx, rx) = mpsc::channel();

// 送信
tx.send("Hello").unwrap();

// 受信
let msg = rx.recv().unwrap();
```

### Publisher/Subscriberパターン

Pub/Subパターンは、メッセージの送信者（Publisher）と受信者（Subscriber）を疎結合にするデザインパターンです。

```
┌─────────────┐
│ Publisher   │
└──────┬──────┘
       │ publish
       ▼
┌─────────────────────────────────────┐
│           Event Channel             │
│  (メッセージブローカー / イベントバス)  │
└──────┬──────────┬──────────┬────────┘
       │          │          │
       ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Subscriber│ │Subscriber│ │Subscriber│
│    A     │ │    B     │ │    C     │
└──────────┘ └──────────┘ └──────────┘
```

**特徴:**
- Publisherは誰がSubscribeしているか知らない
- Subscriberは誰がPublishしているか知らない
- 新しいSubscriberの追加が容易

### チャネルの種類

| 種類 | 特徴 | 用途 |
|------|------|------|
| **unbounded** | 容量制限なし、送信はブロックしない | メモリに余裕がある場合 |
| **bounded** | 容量制限あり、満杯時は送信がブロック | バックプレッシャー制御 |
| **oneshot** | 一度だけ送信可能 | リクエスト/レスポンス |

## 概念の説明

### 基本的なチャネル通信

```
┌──────────────────────────────────────────────────────────────┐
│                    Channel Communication                      │
│                                                              │
│  ┌────────┐    send()     ┌─────────┐    recv()   ┌────────┐│
│  │ Sender │ ────────────→ │ Channel │ ──────────→ │Receiver││
│  │  (tx)  │               │ (buffer)│             │  (rx)  ││
│  └────────┘               └─────────┘             └────────┘│
│                                                              │
│  tx.send(event)?          内部バッファ            rx.recv()? │
└──────────────────────────────────────────────────────────────┘
```

### ブロードキャスト（1対多）の実現

`mpsc`は「複数送信者、単一受信者」のため、1対多のブロードキャストには工夫が必要です。

**方法1: 複数のチャネルを管理**

```
┌──────────┐     ┌────────────────────────────────┐
│Publisher │     │        EventBus                │
│          │────→│  subscribers: Vec<Sender<E>>   │
└──────────┘     │                                │
                 │  ┌─────┐ ┌─────┐ ┌─────┐      │
                 │  │ tx1 │ │ tx2 │ │ tx3 │      │
                 │  └──┬──┘ └──┬──┘ └──┬──┘      │
                 └─────┼──────┼──────┼──────────┘
                       │      │      │
                       ▼      ▼      ▼
                    ┌────┐ ┌────┐ ┌────┐
                    │rx1 │ │rx2 │ │rx3 │
                    └────┘ └────┘ └────┘
                    Sub A  Sub B  Sub C
```

### イベントバスの設計

イベントバスは、Pub/Subパターンを実現する中央コンポーネントです。

```rust
// 概念的な設計
struct EventBus<E> {
    subscribers: Vec<Sender<E>>,
}

impl<E: Clone> EventBus<E> {
    fn publish(&self, event: E) {
        for subscriber in &self.subscribers {
            subscriber.send(event.clone()).ok();
        }
    }

    fn subscribe(&mut self) -> Receiver<E> {
        let (tx, rx) = channel();
        self.subscribers.push(tx);
        rx
    }
}
```

## 実装タスク

### タスク1: 基本的なチャネル通信を実装する

`std::sync::mpsc`を使用して、イベントを送受信する基本的なプログラムを作成してください。

**要件:**
- Chapter 01で作成した`OrderEvent`を送信する
- 別スレッドで受信して処理する
- 受信したイベントの内容を表示する

### タスク2: EventBusを実装する

複数のSubscriberにイベントをブロードキャストできる`EventBus`を実装してください。

**要件:**
- `subscribe()`メソッドで新しいSubscriberを登録
- `publish()`メソッドで全Subscriberにイベントを配信
- Subscriberが切断されても他のSubscriberに影響しない

### タスク3: 型安全なイベントフィルタリング

特定の種類のイベントのみを受信するフィルタリング機能を追加してください。

**要件:**
- `OrderPlaced`イベントのみを受信するSubscriber
- `OrderShipped`イベントのみを受信するSubscriber
- すべてのイベントを受信するSubscriber

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    // 受信スレッド
    let handle = thread::spawn(move || {
        while let Ok(event) = rx.recv() {
            println!("Received: {:?}", event);
        }
    });

    // イベント送信
    tx.send(/* イベント */).unwrap();

    // チャネルを閉じる（txをドロップ）
    drop(tx);

    handle.join().unwrap();
}
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
use std::sync::mpsc::{channel, Receiver, Sender};

struct EventBus<E> {
    subscribers: Vec<Sender<E>>,
}

impl<E: Clone> EventBus<E> {
    fn new() -> Self {
        Self { subscribers: vec![] }
    }

    fn subscribe(&mut self) -> Receiver<E> {
        let (tx, rx) = channel();
        self.subscribers.push(tx);
        rx
    }

    fn publish(&mut self, event: E) {
        // 切断されたSubscriberを除去しながら送信
        self.subscribers.retain(|tx| {
            tx.send(event.clone()).is_ok()
        });
    }
}
```

</details>

<details>
<summary>タスク3のヒント</summary>

```rust
// フィルタリングの方法1: enumのマッチング
fn filter_placed_events(rx: Receiver<OrderEvent>) -> impl Iterator<Item = OrderPlaced> {
    rx.into_iter().filter_map(|event| {
        match event {
            OrderEvent::Placed(e) => Some(e),
            _ => None,
        }
    })
}

// フィルタリングの方法2: 専用のSubscriber型
struct FilteredSubscriber<E, F> {
    receiver: Receiver<E>,
    filter: F,
}
```

</details>

## 回答（コード例）

### 完全な実装

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::sync::mpsc::{channel, Receiver, Sender};
use std::thread;
use std::time::Duration;
use uuid::Uuid;

// ============================================================
// イベント定義（Chapter 01から）
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
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPaid {
    pub metadata: EventMetadata,
    pub payment_id: String,
    pub amount: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderShipped {
    pub metadata: EventMetadata,
    pub tracking_number: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum OrderEvent {
    Placed(OrderPlaced),
    Paid(OrderPaid),
    Shipped(OrderShipped),
}

// ============================================================
// EventBus実装
// ============================================================

/// イベントを複数のSubscriberにブロードキャストするEventBus
pub struct EventBus<E> {
    subscribers: Vec<Sender<E>>,
}

impl<E: Clone> EventBus<E> {
    pub fn new() -> Self {
        Self {
            subscribers: Vec::new(),
        }
    }

    /// 新しいSubscriberを登録し、Receiverを返す
    pub fn subscribe(&mut self) -> Receiver<E> {
        let (tx, rx) = channel();
        self.subscribers.push(tx);
        rx
    }

    /// 全Subscriberにイベントを配信
    pub fn publish(&mut self, event: E) {
        // 切断されたSubscriberを自動的に除去
        self.subscribers.retain(|tx| tx.send(event.clone()).is_ok());
    }

    /// 現在のSubscriber数を返す
    pub fn subscriber_count(&self) -> usize {
        self.subscribers.len()
    }
}

impl<E: Clone> Default for EventBus<E> {
    fn default() -> Self {
        Self::new()
    }
}

// ============================================================
// フィルタリング機能
// ============================================================

/// 特定のイベントタイプのみをフィルタリングするユーティリティ
pub struct EventFilter;

impl EventFilter {
    /// OrderPlacedイベントのみを抽出
    pub fn placed_events(
        rx: Receiver<OrderEvent>,
    ) -> impl Iterator<Item = OrderPlaced> {
        rx.into_iter().filter_map(|event| match event {
            OrderEvent::Placed(e) => Some(e),
            _ => None,
        })
    }

    /// OrderShippedイベントのみを抽出
    pub fn shipped_events(
        rx: Receiver<OrderEvent>,
    ) -> impl Iterator<Item = OrderShipped> {
        rx.into_iter().filter_map(|event| match event {
            OrderEvent::Shipped(e) => Some(e),
            _ => None,
        })
    }
}

// ============================================================
// 使用例
// ============================================================

fn main() {
    println!("=== EventBus Demo ===\n");

    let mut event_bus: EventBus<OrderEvent> = EventBus::new();

    // Subscriber 1: すべてのイベントを処理
    let rx_all = event_bus.subscribe();
    let handle_all = thread::spawn(move || {
        println!("[All Events Subscriber] Started");
        for event in rx_all {
            match &event {
                OrderEvent::Placed(e) => {
                    println!("[All] Order placed: {}", e.metadata.aggregate_id);
                }
                OrderEvent::Paid(e) => {
                    println!("[All] Order paid: {}", e.metadata.aggregate_id);
                }
                OrderEvent::Shipped(e) => {
                    println!("[All] Order shipped: {}", e.metadata.aggregate_id);
                }
            }
        }
        println!("[All Events Subscriber] Finished");
    });

    // Subscriber 2: OrderPlacedのみを処理
    let rx_placed = event_bus.subscribe();
    let handle_placed = thread::spawn(move || {
        println!("[Placed Events Subscriber] Started");
        for event in EventFilter::placed_events(rx_placed) {
            println!(
                "[Placed Only] New order! Customer: {}, Amount: {}",
                event.customer_id, event.total_amount
            );
        }
        println!("[Placed Events Subscriber] Finished");
    });

    // Subscriber 3: OrderShippedのみを処理
    let rx_shipped = event_bus.subscribe();
    let handle_shipped = thread::spawn(move || {
        println!("[Shipped Events Subscriber] Started");
        for event in EventFilter::shipped_events(rx_shipped) {
            println!(
                "[Shipped Only] Order shipped! Tracking: {}",
                event.tracking_number
            );
        }
        println!("[Shipped Events Subscriber] Finished");
    });

    // イベントを発行
    thread::sleep(Duration::from_millis(100));

    let order_id = "ORD-001".to_string();

    // 注文作成
    event_bus.publish(OrderEvent::Placed(OrderPlaced {
        metadata: EventMetadata::new(&order_id),
        customer_id: "CUST-001".to_string(),
        total_amount: 15000,
    }));

    // 支払い完了
    event_bus.publish(OrderEvent::Paid(OrderPaid {
        metadata: EventMetadata::new(&order_id),
        payment_id: "PAY-001".to_string(),
        amount: 15000,
    }));

    // 発送完了
    event_bus.publish(OrderEvent::Shipped(OrderShipped {
        metadata: EventMetadata::new(&order_id),
        tracking_number: "TRACK-123456".to_string(),
    }));

    // EventBusをドロップしてチャネルを閉じる
    drop(event_bus);

    // 全スレッドの完了を待機
    handle_all.join().unwrap();
    handle_placed.join().unwrap();
    handle_shipped.join().unwrap();

    println!("\n=== Demo Complete ===");
}
```

### 出力例

```
=== EventBus Demo ===

[All Events Subscriber] Started
[Placed Events Subscriber] Started
[Shipped Events Subscriber] Started
[All] Order placed: ORD-001
[Placed Only] New order! Customer: CUST-001, Amount: 15000
[All] Order paid: ORD-001
[All] Order shipped: ORD-001
[Shipped Only] Order shipped! Tracking: TRACK-123456
[All Events Subscriber] Finished
[Placed Events Subscriber] Finished
[Shipped Events Subscriber] Finished

=== Demo Complete ===
```

### コード解説

#### `retain`を使った切断検出

```rust
self.subscribers.retain(|tx| tx.send(event.clone()).is_ok());
```

`send()`が`Err`を返す場合、対応するReceiverがドロップされています。`retain`を使うことで、切断されたSubscriberを自動的にリストから除去します。

#### フィルタリングの実装

```rust
rx.into_iter().filter_map(|event| match event {
    OrderEvent::Placed(e) => Some(e),
    _ => None,
})
```

`filter_map`を使用することで、フィルタリングと型変換を同時に行えます。

## 発展課題

### 課題1: 優先度付きイベント配信

イベントに優先度を付け、高優先度のイベントを先に処理する仕組みを実装してください。

### 課題2: イベントの永続化

EventBusを通過するすべてのイベントをログファイルに記録する機能を追加してください。

### 課題3: バックプレッシャー制御

Subscriberの処理が遅い場合に、Publisherの送信速度を制限する仕組みを実装してください。

**ヒント**: `sync_channel`を使用してboundedチャネルを作成

## よくある間違い

### ❌ 間違い1: Senderをドロップし忘れる

```rust
// 悪い例: txがドロップされないため、rx.recv()が永遠にブロック
let (tx, rx) = channel();
thread::spawn(move || {
    for msg in rx {  // 永遠に待ち続ける
        println!("{}", msg);
    }
});
tx.send("hello").unwrap();
// txがスコープ内に残っている...
```

```rust
// 良い例: 明示的にドロップ
drop(tx);
```

### ❌ 間違い2: Cloneを実装していないイベント

```rust
// 悪い例: Cloneがないとブロードキャストできない
struct Event {
    data: Vec<u8>,  // Cloneは自動実装されるが...
    file: std::fs::File,  // FileはCloneを実装していない！
}
```

### ❌ 間違い3: 受信エラーを無視する

```rust
// 悪い例
let msg = rx.recv().unwrap();  // チャネルが閉じるとパニック

// 良い例
match rx.recv() {
    Ok(msg) => println!("{:?}", msg),
    Err(_) => println!("Channel closed"),
}
```

## まとめ

この章では、Rustのチャネルを使用したPub/Subパターンを学びました。

### 学んだこと

1. **チャネルの基本**: `std::sync::mpsc`による送受信
2. **Pub/Subパターン**: Publisherと Subscriberの疎結合
3. **EventBus**: 複数Subscriberへのブロードキャスト
4. **フィルタリング**: 特定のイベントのみを受信する方法
5. **エラーハンドリング**: 切断されたSubscriberの処理

### 次の章への準備

この章では同期的なチャネルを使用しました。次の章では、`tokio`を使用した非同期イベント処理を学びます。非同期処理により、より効率的で スケーラブルなイベント駆動システムを構築できます。

## 参考文献

- [std::sync::mpsc - Rust Documentation](https://doc.rust-lang.org/std/sync/mpsc/)
- [The Rust Programming Language - Message Passing](https://doc.rust-lang.org/book/ch16-02-message-passing.html)
- [Enterprise Integration Patterns - Publish-Subscribe Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html)

---

[← 前の章: イベントの基本](./01-event-basics.md) | [次の章: 非同期イベント処理 →](./03-async-events.md)
