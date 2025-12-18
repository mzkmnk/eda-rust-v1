# Chapter 05: イベントソーシング

## 学習の目的

この章を完了すると、以下のことができるようになります：

- イベントソーシングパターンの概念と利点を理解する
- 集約（Aggregate）をイベントから再構築できる
- コマンドを処理してイベントを生成できる
- スナップショットによるパフォーマンス最適化ができる

## 背景知識

### イベントソーシングとは

イベントソーシングは、アプリケーションの状態を「イベントの履歴」として保存するパターンです。現在の状態は、すべてのイベントを順番に適用することで再構築されます。

```
従来のアプローチ（状態を直接保存）:
┌─────────────────────────────────────────────────────────────────┐
│  Command: UpdateBalance(+5000)                                  │
│                    ↓                                            │
│  ┌─────────────────────────────────┐                           │
│  │ Account { balance: 10000 }      │  ← 状態を直接更新          │
│  │         ↓                       │                           │
│  │ Account { balance: 15000 }      │                           │
│  └─────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘

イベントソーシング（イベントを保存）:
┌─────────────────────────────────────────────────────────────────┐
│  Command: Deposit(5000)                                         │
│                    ↓                                            │
│  ┌─────────────────────────────────┐                           │
│  │ Events:                         │                           │
│  │   1. AccountOpened(0)           │                           │
│  │   2. MoneyDeposited(10000)      │                           │
│  │   3. MoneyDeposited(5000)  ← 新規追加                       │
│  └─────────────────────────────────┘                           │
│                    ↓                                            │
│  状態を再構築: 0 + 10000 + 5000 = 15000                         │
└─────────────────────────────────────────────────────────────────┘
```

### なぜイベントソーシングを使うのか

| メリット | 説明 |
|---------|------|
| **完全な監査証跡** | すべての変更履歴が残る |
| **時間旅行** | 任意の時点の状態を再現できる |
| **デバッグ容易性** | 問題の原因を追跡しやすい |
| **イベント駆動統合** | 他システムとの連携が容易 |
| **柔軟なプロジェクション** | 新しいビューを後から構築可能 |

| デメリット | 説明 |
|-----------|------|
| **複雑性** | 従来のCRUDより設計が複雑 |
| **学習コスト** | 新しい概念の習得が必要 |
| **イベント設計の重要性** | 一度保存したイベントは変更困難 |

### 集約（Aggregate）とは

集約は、ドメイン駆動設計（DDD）の概念で、一貫性を保つべきオブジェクトのまとまりです。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Order Aggregate                              │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Order (Aggregate Root)                                   │ │
│  │  ├── order_id: "ORD-001"                                 │ │
│  │  ├── status: Paid                                        │ │
│  │  ├── customer_id: "CUST-001"                             │ │
│  │  └── items: [OrderItem, OrderItem]                       │ │
│  │           ├── product_id, quantity, price                │ │
│  │           └── product_id, quantity, price                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ルール: 集約内の変更は、集約ルートを通じてのみ行う              │
└─────────────────────────────────────────────────────────────────┘
```

## 概念の説明

### イベントソーシングの流れ

```
┌─────────────────────────────────────────────────────────────────┐
│                 Event Sourcing Flow                             │
│                                                                 │
│  1. コマンド受信                                                │
│     ┌──────────────────┐                                       │
│     │ PlaceOrder       │                                       │
│     │ {customer, items}│                                       │
│     └────────┬─────────┘                                       │
│              ↓                                                  │
│  2. 集約をロード（イベントから再構築）                           │
│     ┌──────────────────┐                                       │
│     │ Event Store      │ → [Event1, Event2, ...] → Aggregate   │
│     └──────────────────┘                                       │
│              ↓                                                  │
│  3. ビジネスルールを検証                                        │
│     ┌──────────────────┐                                       │
│     │ Aggregate.handle │ → 検証OK or Error                     │
│     └──────────────────┘                                       │
│              ↓                                                  │
│  4. イベントを生成                                              │
│     ┌──────────────────┐                                       │
│     │ OrderPlaced      │                                       │
│     │ {order_id, ...}  │                                       │
│     └──────────────────┘                                       │
│              ↓                                                  │
│  5. イベントを保存                                              │
│     ┌──────────────────┐                                       │
│     │ Event Store      │ ← append(events)                      │
│     └──────────────────┘                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 集約の実装パターン

```rust
// 集約の基本構造
struct OrderAggregate {
    // 状態
    id: Option<String>,
    status: OrderStatus,
    items: Vec<OrderItem>,
    version: u64,

    // 未保存のイベント
    pending_events: Vec<OrderEvent>,
}

impl OrderAggregate {
    // イベントから状態を再構築
    fn apply(&mut self, event: &OrderEvent) {
        match event {
            OrderEvent::Placed(e) => {
                self.id = Some(e.order_id.clone());
                self.status = OrderStatus::Placed;
                self.items = e.items.clone();
            }
            OrderEvent::Paid(_) => {
                self.status = OrderStatus::Paid;
            }
            // ...
        }
        self.version += 1;
    }

    // コマンドを処理してイベントを生成
    fn handle(&mut self, command: OrderCommand) -> Result<(), Error> {
        match command {
            OrderCommand::Place { .. } => {
                // ビジネスルールを検証
                // イベントを生成
                let event = OrderEvent::Placed { .. };
                self.apply(&event);
                self.pending_events.push(event);
                Ok(())
            }
            // ...
        }
    }
}
```

### スナップショット

大量のイベントがある場合、毎回すべてのイベントを再生するのは非効率です。スナップショットを使用して最適化します。

```
スナップショットなし:
[E1] → [E2] → [E3] → ... → [E1000] → 現在の状態
                                      ↑
                              1000イベントを再生

スナップショットあり:
[E1] → ... → [E500] → [Snapshot@v500] → [E501] → ... → [E1000] → 現在の状態
                            ↑                                      ↑
                      スナップショットから開始              500イベントのみ再生
```

## 実装タスク

### タスク1: 基本的な集約を実装

注文（Order）集約を実装してください。

**要件:**
- `OrderAggregate`構造体を定義
- `apply()`メソッドでイベントを適用
- `from_events()`で履歴から再構築

### タスク2: コマンドハンドリングを実装

コマンドを受け取り、ビジネスルールを検証してイベントを生成してください。

**要件:**
- `PlaceOrder`コマンド: 新規注文を作成
- `PayOrder`コマンド: 注文の支払い（Placed状態のみ可）
- `ShipOrder`コマンド: 注文の発送（Paid状態のみ可）
- `CancelOrder`コマンド: 注文のキャンセル（Shipped以外で可）

### タスク3: リポジトリパターンを実装

イベントストアと連携して集約を永続化するリポジトリを実装してください。

**要件:**
- `load()`で集約をロード
- `save()`で未保存イベントを保存
- 楽観的ロックによる並行制御

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
#[derive(Debug, Clone, Default)]
struct OrderAggregate {
    id: Option<String>,
    status: OrderStatus,
    customer_id: Option<String>,
    items: Vec<OrderItem>,
    total_amount: u64,
    version: u64,
    pending_events: Vec<OrderEvent>,
}

impl OrderAggregate {
    fn apply(&mut self, event: &OrderEvent) {
        match event {
            OrderEvent::Placed(e) => { /* 状態を更新 */ }
            OrderEvent::Paid(e) => { /* 状態を更新 */ }
            // ...
        }
        self.version += 1;
    }

    fn from_events(events: impl IntoIterator<Item = OrderEvent>) -> Self {
        let mut aggregate = Self::default();
        for event in events {
            aggregate.apply(&event);
        }
        aggregate
    }
}
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
impl OrderAggregate {
    fn handle(&mut self, command: OrderCommand) -> Result<Vec<OrderEvent>, OrderError> {
        let events = match command {
            OrderCommand::Place { order_id, customer_id, items } => {
                if self.id.is_some() {
                    return Err(OrderError::AlreadyExists);
                }
                vec![OrderEvent::Placed(OrderPlaced { ... })]
            }
            OrderCommand::Pay { payment_id } => {
                if self.status != OrderStatus::Placed {
                    return Err(OrderError::InvalidStatus);
                }
                vec![OrderEvent::Paid(OrderPaid { ... })]
            }
            // ...
        };

        for event in &events {
            self.apply(event);
        }
        self.pending_events.extend(events.clone());
        Ok(events)
    }
}
```

</details>

## 回答（コード例）

### 完全な実装

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use uuid::Uuid;

// ============================================================
// ドメインモデル
// ============================================================

#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub enum OrderStatus {
    #[default]
    None,
    Placed,
    Paid,
    Shipped,
    Cancelled,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderItem {
    pub product_id: String,
    pub quantity: u32,
    pub unit_price: u64,
}

// ============================================================
// イベント
// ============================================================

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    pub order_id: String,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
    pub total_amount: u64,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPaid {
    pub order_id: String,
    pub payment_id: String,
    pub amount: u64,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderShipped {
    pub order_id: String,
    pub tracking_number: String,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderCancelled {
    pub order_id: String,
    pub reason: String,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum OrderEvent {
    Placed(OrderPlaced),
    Paid(OrderPaid),
    Shipped(OrderShipped),
    Cancelled(OrderCancelled),
}

// ============================================================
// コマンド
// ============================================================

#[derive(Debug)]
pub enum OrderCommand {
    Place {
        order_id: String,
        customer_id: String,
        items: Vec<OrderItem>,
    },
    Pay {
        payment_id: String,
        amount: u64,
    },
    Ship {
        tracking_number: String,
    },
    Cancel {
        reason: String,
    },
}

// ============================================================
// エラー
// ============================================================

#[derive(Debug, Clone)]
pub enum OrderError {
    AlreadyExists,
    NotFound,
    InvalidStatus { current: OrderStatus, expected: Vec<OrderStatus> },
    InsufficientPayment { expected: u64, actual: u64 },
}

// ============================================================
// 集約
// ============================================================

#[derive(Debug, Clone, Default)]
pub struct OrderAggregate {
    pub id: Option<String>,
    pub status: OrderStatus,
    pub customer_id: Option<String>,
    pub items: Vec<OrderItem>,
    pub total_amount: u64,
    pub version: u64,
    pending_events: Vec<OrderEvent>,
}

impl OrderAggregate {
    pub fn new() -> Self {
        Self::default()
    }

    /// イベント履歴から集約を再構築
    pub fn from_events(events: impl IntoIterator<Item = OrderEvent>) -> Self {
        let mut aggregate = Self::new();
        for event in events {
            aggregate.apply(&event);
        }
        aggregate
    }

    /// イベントを適用して状態を更新
    pub fn apply(&mut self, event: &OrderEvent) {
        match event {
            OrderEvent::Placed(e) => {
                self.id = Some(e.order_id.clone());
                self.customer_id = Some(e.customer_id.clone());
                self.items = e.items.clone();
                self.total_amount = e.total_amount;
                self.status = OrderStatus::Placed;
            }
            OrderEvent::Paid(_) => {
                self.status = OrderStatus::Paid;
            }
            OrderEvent::Shipped(_) => {
                self.status = OrderStatus::Shipped;
            }
            OrderEvent::Cancelled(_) => {
                self.status = OrderStatus::Cancelled;
            }
        }
        self.version += 1;
    }

    /// コマンドを処理してイベントを生成
    pub fn handle(&mut self, command: OrderCommand) -> Result<Vec<OrderEvent>, OrderError> {
        let events = match command {
            OrderCommand::Place { order_id, customer_id, items } => {
                if self.id.is_some() {
                    return Err(OrderError::AlreadyExists);
                }
                let total = items.iter().map(|i| i.unit_price * i.quantity as u64).sum();
                vec![OrderEvent::Placed(OrderPlaced {
                    order_id,
                    customer_id,
                    items,
                    total_amount: total,
                    timestamp: Utc::now(),
                })]
            }
            OrderCommand::Pay { payment_id, amount } => {
                if self.status != OrderStatus::Placed {
                    return Err(OrderError::InvalidStatus {
                        current: self.status.clone(),
                        expected: vec![OrderStatus::Placed],
                    });
                }
                if amount < self.total_amount {
                    return Err(OrderError::InsufficientPayment {
                        expected: self.total_amount,
                        actual: amount,
                    });
                }
                vec![OrderEvent::Paid(OrderPaid {
                    order_id: self.id.clone().unwrap(),
                    payment_id,
                    amount,
                    timestamp: Utc::now(),
                })]
            }
            OrderCommand::Ship { tracking_number } => {
                if self.status != OrderStatus::Paid {
                    return Err(OrderError::InvalidStatus {
                        current: self.status.clone(),
                        expected: vec![OrderStatus::Paid],
                    });
                }
                vec![OrderEvent::Shipped(OrderShipped {
                    order_id: self.id.clone().unwrap(),
                    tracking_number,
                    timestamp: Utc::now(),
                })]
            }
            OrderCommand::Cancel { reason } => {
                if self.status == OrderStatus::Shipped || self.status == OrderStatus::None {
                    return Err(OrderError::InvalidStatus {
                        current: self.status.clone(),
                        expected: vec![OrderStatus::Placed, OrderStatus::Paid],
                    });
                }
                vec![OrderEvent::Cancelled(OrderCancelled {
                    order_id: self.id.clone().unwrap(),
                    reason,
                    timestamp: Utc::now(),
                })]
            }
        };

        // イベントを適用
        for event in &events {
            self.apply(event);
        }
        self.pending_events.extend(events.clone());

        Ok(events)
    }

    /// 未保存のイベントを取得してクリア
    pub fn take_pending_events(&mut self) -> Vec<OrderEvent> {
        std::mem::take(&mut self.pending_events)
    }
}

// ============================================================
// リポジトリ
// ============================================================

pub struct OrderRepository {
    events: Arc<RwLock<HashMap<String, Vec<OrderEvent>>>>,
}

impl OrderRepository {
    pub fn new() -> Self {
        Self {
            events: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    pub fn load(&self, order_id: &str) -> Option<OrderAggregate> {
        let events = self.events.read().unwrap();
        events.get(order_id).map(|e| OrderAggregate::from_events(e.clone()))
    }

    pub fn save(&self, aggregate: &mut OrderAggregate) -> Result<(), OrderError> {
        let pending = aggregate.take_pending_events();
        if pending.is_empty() {
            return Ok(());
        }

        let order_id = aggregate.id.clone().ok_or(OrderError::NotFound)?;
        let mut events = self.events.write().unwrap();
        events.entry(order_id).or_default().extend(pending);
        Ok(())
    }
}

impl Default for OrderRepository {
    fn default() -> Self {
        Self::new()
    }
}

// ============================================================
// 使用例
// ============================================================

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== Event Sourcing Demo ===\n");

    let repository = OrderRepository::new();

    // --- 注文を作成 ---
    println!("--- Creating Order ---");
    let order_id = format!("ORD-{}", Uuid::new_v4());

    let mut order = OrderAggregate::new();
    order.handle(OrderCommand::Place {
        order_id: order_id.clone(),
        customer_id: "CUST-001".to_string(),
        items: vec![
            OrderItem { product_id: "PROD-001".to_string(), quantity: 2, unit_price: 1000 },
            OrderItem { product_id: "PROD-002".to_string(), quantity: 1, unit_price: 3000 },
        ],
    })?;
    repository.save(&mut order)?;

    println!("Order created: {}", order_id);
    println!("Status: {:?}, Total: {}", order.status, order.total_amount);

    // --- 支払い ---
    println!("\n--- Processing Payment ---");
    let mut order = repository.load(&order_id).unwrap();
    order.handle(OrderCommand::Pay {
        payment_id: "PAY-001".to_string(),
        amount: 5000,
    })?;
    repository.save(&mut order)?;
    println!("Status: {:?}", order.status);

    // --- 発送 ---
    println!("\n--- Shipping Order ---");
    let mut order = repository.load(&order_id).unwrap();
    order.handle(OrderCommand::Ship {
        tracking_number: "TRACK-123".to_string(),
    })?;
    repository.save(&mut order)?;
    println!("Status: {:?}", order.status);

    // --- イベント履歴から再構築 ---
    println!("\n--- Reconstructing from Events ---");
    let reconstructed = repository.load(&order_id).unwrap();
    println!("Reconstructed order:");
    println!("  ID: {:?}", reconstructed.id);
    println!("  Status: {:?}", reconstructed.status);
    println!("  Version: {}", reconstructed.version);

    // --- 不正な操作のテスト ---
    println!("\n--- Testing Invalid Operations ---");
    let mut order = repository.load(&order_id).unwrap();
    match order.handle(OrderCommand::Cancel { reason: "Test".to_string() }) {
        Ok(_) => println!("Unexpected success"),
        Err(e) => println!("Expected error: {:?}", e),
    }

    println!("\n=== Demo Complete ===");
    Ok(())
}
```

## 発展課題

### 課題1: スナップショット機能の実装

一定数のイベントごとにスナップショットを保存し、再構築を高速化してください。

### 課題2: イベントのアップキャスト

古いバージョンのイベントを新しいバージョンに変換する仕組みを実装してください。

### 課題3: 複数集約の整合性

複数の集約にまたがる操作の整合性を保証する仕組みを検討してください。

## よくある間違い

### ❌ 間違い1: 集約内でイベントを保存する

```rust
// 悪い例: 集約がリポジトリに依存
impl OrderAggregate {
    fn handle(&mut self, cmd: OrderCommand, repo: &Repository) {
        let events = /* ... */;
        repo.save(events);  // 集約がインフラに依存！
    }
}

// 良い例: 集約はイベントを返すだけ
impl OrderAggregate {
    fn handle(&mut self, cmd: OrderCommand) -> Result<Vec<Event>, Error> {
        // イベントを生成して返す
    }
}
// 保存はアプリケーション層で行う
```

### ❌ 間違い2: apply()でビジネスルールを検証

```rust
// 悪い例: apply()で検証
fn apply(&mut self, event: &OrderEvent) {
    if self.status != OrderStatus::Placed {
        panic!("Invalid status!");  // apply()は常に成功すべき
    }
}

// 良い例: handle()で検証、apply()は状態更新のみ
fn handle(&mut self, cmd: OrderCommand) -> Result<Vec<Event>, Error> {
    if self.status != OrderStatus::Placed {
        return Err(Error::InvalidStatus);
    }
    // ...
}

fn apply(&mut self, event: &OrderEvent) {
    // 状態更新のみ、検証なし
}
```

### ❌ 間違い3: イベントに現在時刻以外のタイムスタンプを使用

```rust
// 悪い例: 外部から渡されたタイムスタンプ
fn handle(&mut self, cmd: PlaceOrder) -> Vec<Event> {
    vec![OrderPlaced { timestamp: cmd.timestamp }]  // 改ざん可能
}

// 良い例: イベント生成時に現在時刻を使用
fn handle(&mut self, cmd: PlaceOrder) -> Vec<Event> {
    vec![OrderPlaced { timestamp: Utc::now() }]
}
```

## まとめ

この章では、イベントソーシングパターンを学びました。

### 学んだこと

1. **イベントソーシングの概念**: 状態ではなくイベントを保存
2. **集約の実装**: apply()とhandle()の分離
3. **コマンドハンドリング**: ビジネスルールの検証とイベント生成
4. **リポジトリパターン**: 集約の永続化と再構築

### 次の章への準備

次の章では、CQRSパターンを学びます。コマンド（書き込み）とクエリ（読み取り）を分離することで、それぞれを最適化できます。

## 参考文献

- [Martin Fowler - Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Greg Young - CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- [Microsoft - Event Sourcing Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)

---

[← 前の章: イベントストア](./04-event-store.md) | [次の章: CQRSパターン →](./06-cqrs.md)
