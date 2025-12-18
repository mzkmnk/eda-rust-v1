# Chapter 06: CQRSパターン

## 学習の目的

この章を完了すると、以下のことができるようになります：

- CQRSパターンの概念と適用場面を理解する
- コマンドサイドとクエリサイドを分離して実装できる
- Read Modelを構築・更新できる
- イベントハンドラーによるプロジェクションを実装できる

## 背景知識

### CQRSとは

CQRS（Command Query Responsibility Segregation）は、データの書き込み（コマンド）と読み取り（クエリ）を分離するアーキテクチャパターンです。

```
従来のアーキテクチャ:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ┌─────────┐     ┌─────────────┐     ┌─────────────┐          │
│  │ Client  │ ──→ │   Service   │ ──→ │  Database   │          │
│  └─────────┘     │ (CRUD)      │     │ (単一モデル) │          │
│                  └─────────────┘     └─────────────┘          │
│                                                                 │
│  問題: 読み取りと書き込みで異なる最適化が必要な場合に対応困難     │
└─────────────────────────────────────────────────────────────────┘

CQRS:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ┌─────────┐     ┌─────────────┐     ┌─────────────┐          │
│  │ Client  │ ──→ │  Command    │ ──→ │ Write Model │          │
│  │ (Write) │     │  Handler    │     │ (Event Store)│          │
│  └─────────┘     └─────────────┘     └──────┬──────┘          │
│                                             │ Events           │
│                                             ↓                  │
│  ┌─────────┐     ┌─────────────┐     ┌─────────────┐          │
│  │ Client  │ ←── │   Query     │ ←── │ Read Model  │          │
│  │ (Read)  │     │   Handler   │     │ (Projection)│          │
│  └─────────┘     └─────────────┘     └─────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### なぜCQRSを使うのか

| メリット | 説明 |
|---------|------|
| **独立したスケーリング** | 読み取りと書き込みを別々にスケール |
| **最適化されたモデル** | 各用途に最適なデータ構造を使用 |
| **パフォーマンス** | 読み取りは非正規化されたビューから高速に取得 |
| **柔軟性** | 新しいビューを後から追加可能 |

| デメリット | 説明 |
|-----------|------|
| **複雑性** | 2つのモデルを管理する必要がある |
| **結果整合性** | Read Modelの更新に遅延がある |
| **同期の課題** | Write ModelとRead Modelの整合性維持 |

### いつCQRSを使うべきか

**適している場合:**
- 読み取りと書き込みの負荷が大きく異なる
- 複雑なクエリが必要
- イベントソーシングと組み合わせる場合

**適していない場合:**
- シンプルなCRUDアプリケーション
- 読み取りと書き込みの要件が似ている
- 強い整合性が必要

## 概念の説明

### コマンドサイド

コマンドサイドは、ビジネスロジックとデータの整合性を担当します。

```rust
// コマンド
enum OrderCommand {
    Place { customer_id: String, items: Vec<Item> },
    Pay { payment_id: String },
    Ship { tracking_number: String },
}

// コマンドハンドラー
impl CommandHandler {
    fn handle(&self, cmd: OrderCommand) -> Result<(), Error> {
        // 1. 集約をロード
        // 2. コマンドを処理
        // 3. イベントを保存
    }
}
```

### クエリサイド

クエリサイドは、効率的なデータ取得を担当します。

```rust
// クエリ
enum OrderQuery {
    GetById { order_id: String },
    ListByCustomer { customer_id: String },
    GetStatistics { from: DateTime, to: DateTime },
}

// Read Model
struct OrderReadModel {
    order_id: String,
    customer_name: String,  // 非正規化
    status: String,
    total_amount: u64,
    item_count: usize,
}
```

### プロジェクション

プロジェクションは、イベントからRead Modelを構築するプロセスです。

```
┌─────────────────────────────────────────────────────────────────┐
│                      Projection Flow                            │
│                                                                 │
│  Event Store                                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ [OrderPlaced] [OrderPaid] [OrderShipped] [OrderPlaced]   │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                     │
│                           ↓                                     │
│  Projector                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ match event {                                            │  │
│  │   OrderPlaced => insert_order(...)                       │  │
│  │   OrderPaid => update_status(...)                        │  │
│  │   OrderShipped => update_status(...)                     │  │
│  │ }                                                        │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                     │
│                           ↓                                     │
│  Read Model (非正規化されたビュー)                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ orders_view: [{id, customer_name, status, total}, ...]   │  │
│  │ customer_orders: {customer_id -> [order_ids]}            │  │
│  │ daily_stats: {date -> {count, revenue}}                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 実装タスク

### タスク1: Read Modelを定義

注文の一覧表示に最適化されたRead Modelを定義してください。

**要件:**
- 注文ID、顧客名、ステータス、合計金額、商品数
- 顧客IDで検索可能
- ステータスでフィルタリング可能

### タスク2: プロジェクションを実装

イベントからRead Modelを構築するプロジェクションを実装してください。

**要件:**
- `OrderPlaced`で新規レコード作成
- `OrderPaid`、`OrderShipped`でステータス更新
- `OrderCancelled`でステータス更新

### タスク3: クエリハンドラーを実装

Read Modelに対するクエリを処理するハンドラーを実装してください。

**要件:**
- IDで注文を取得
- 顧客の注文一覧を取得
- ステータス別の注文数を取得

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
#[derive(Debug, Clone)]
struct OrderView {
    order_id: String,
    customer_id: String,
    customer_name: String,
    status: String,
    total_amount: u64,
    item_count: usize,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

struct OrderReadModelStore {
    orders: HashMap<String, OrderView>,
    by_customer: HashMap<String, Vec<String>>,
    by_status: HashMap<String, Vec<String>>,
}
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
impl OrderProjection {
    fn apply(&mut self, event: &OrderEvent) {
        match event {
            OrderEvent::Placed(e) => {
                let view = OrderView {
                    order_id: e.order_id.clone(),
                    status: "Placed".to_string(),
                    // ...
                };
                self.store.orders.insert(e.order_id.clone(), view);
                self.store.by_customer
                    .entry(e.customer_id.clone())
                    .or_default()
                    .push(e.order_id.clone());
            }
            // ...
        }
    }
}
```

</details>

## 回答（コード例）

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

// ============================================================
// イベント（Chapter 05から）
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
pub struct OrderItem {
    pub product_id: String,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPaid {
    pub order_id: String,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderShipped {
    pub order_id: String,
    pub timestamp: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderCancelled {
    pub order_id: String,
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
// Read Model
// ============================================================

#[derive(Debug, Clone, Serialize)]
pub struct OrderView {
    pub order_id: String,
    pub customer_id: String,
    pub status: String,
    pub total_amount: u64,
    pub item_count: usize,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Default)]
struct ReadModelStore {
    orders: HashMap<String, OrderView>,
    by_customer: HashMap<String, Vec<String>>,
    by_status: HashMap<String, Vec<String>>,
}

// ============================================================
// プロジェクション
// ============================================================

pub struct OrderProjection {
    store: Arc<RwLock<ReadModelStore>>,
}

impl OrderProjection {
    pub fn new() -> Self {
        Self {
            store: Arc::new(RwLock::new(ReadModelStore::default())),
        }
    }

    pub fn apply(&self, event: &OrderEvent) {
        let mut store = self.store.write().unwrap();

        match event {
            OrderEvent::Placed(e) => {
                let view = OrderView {
                    order_id: e.order_id.clone(),
                    customer_id: e.customer_id.clone(),
                    status: "Placed".to_string(),
                    total_amount: e.total_amount,
                    item_count: e.items.len(),
                    created_at: e.timestamp,
                    updated_at: e.timestamp,
                };

                // インデックス更新
                store.by_customer
                    .entry(e.customer_id.clone())
                    .or_default()
                    .push(e.order_id.clone());
                store.by_status
                    .entry("Placed".to_string())
                    .or_default()
                    .push(e.order_id.clone());

                store.orders.insert(e.order_id.clone(), view);
            }
            OrderEvent::Paid(e) => {
                if let Some(view) = store.orders.get_mut(&e.order_id) {
                    Self::update_status(&mut store.by_status, &e.order_id, &view.status, "Paid");
                    view.status = "Paid".to_string();
                    view.updated_at = e.timestamp;
                }
            }
            OrderEvent::Shipped(e) => {
                if let Some(view) = store.orders.get_mut(&e.order_id) {
                    Self::update_status(&mut store.by_status, &e.order_id, &view.status, "Shipped");
                    view.status = "Shipped".to_string();
                    view.updated_at = e.timestamp;
                }
            }
            OrderEvent::Cancelled(e) => {
                if let Some(view) = store.orders.get_mut(&e.order_id) {
                    Self::update_status(&mut store.by_status, &e.order_id, &view.status, "Cancelled");
                    view.status = "Cancelled".to_string();
                    view.updated_at = e.timestamp;
                }
            }
        }
    }

    fn update_status(
        by_status: &mut HashMap<String, Vec<String>>,
        order_id: &str,
        old_status: &str,
        new_status: &str,
    ) {
        if let Some(ids) = by_status.get_mut(old_status) {
            ids.retain(|id| id != order_id);
        }
        by_status.entry(new_status.to_string()).or_default().push(order_id.to_string());
    }

    pub fn rebuild(&self, events: impl IntoIterator<Item = OrderEvent>) {
        for event in events {
            self.apply(&event);
        }
    }
}

impl Default for OrderProjection {
    fn default() -> Self {
        Self::new()
    }
}

// ============================================================
// クエリハンドラー
// ============================================================

pub struct OrderQueryHandler {
    projection: Arc<OrderProjection>,
}

impl OrderQueryHandler {
    pub fn new(projection: Arc<OrderProjection>) -> Self {
        Self { projection }
    }

    pub fn get_by_id(&self, order_id: &str) -> Option<OrderView> {
        self.projection.store.read().unwrap().orders.get(order_id).cloned()
    }

    pub fn list_by_customer(&self, customer_id: &str) -> Vec<OrderView> {
        let store = self.projection.store.read().unwrap();
        store.by_customer
            .get(customer_id)
            .map(|ids| {
                ids.iter()
                    .filter_map(|id| store.orders.get(id).cloned())
                    .collect()
            })
            .unwrap_or_default()
    }

    pub fn list_by_status(&self, status: &str) -> Vec<OrderView> {
        let store = self.projection.store.read().unwrap();
        store.by_status
            .get(status)
            .map(|ids| {
                ids.iter()
                    .filter_map(|id| store.orders.get(id).cloned())
                    .collect()
            })
            .unwrap_or_default()
    }

    pub fn get_statistics(&self) -> OrderStatistics {
        let store = self.projection.store.read().unwrap();
        OrderStatistics {
            total_orders: store.orders.len(),
            by_status: store.by_status.iter().map(|(k, v)| (k.clone(), v.len())).collect(),
            total_revenue: store.orders.values()
                .filter(|o| o.status != "Cancelled")
                .map(|o| o.total_amount)
                .sum(),
        }
    }
}

#[derive(Debug, Serialize)]
pub struct OrderStatistics {
    pub total_orders: usize,
    pub by_status: HashMap<String, usize>,
    pub total_revenue: u64,
}

// ============================================================
// 使用例
// ============================================================

fn main() {
    println!("=== CQRS Demo ===\n");

    let projection = Arc::new(OrderProjection::new());
    let query_handler = OrderQueryHandler::new(projection.clone());

    // イベントを適用
    let events = vec![
        OrderEvent::Placed(OrderPlaced {
            order_id: "ORD-001".to_string(),
            customer_id: "CUST-001".to_string(),
            items: vec![OrderItem {
                product_id: "P1".to_string(),
                product_name: "Product 1".to_string(),
                quantity: 2,
                unit_price: 1000,
            }],
            total_amount: 2000,
            timestamp: Utc::now(),
        }),
        OrderEvent::Placed(OrderPlaced {
            order_id: "ORD-002".to_string(),
            customer_id: "CUST-001".to_string(),
            items: vec![OrderItem {
                product_id: "P2".to_string(),
                product_name: "Product 2".to_string(),
                quantity: 1,
                unit_price: 5000,
            }],
            total_amount: 5000,
            timestamp: Utc::now(),
        }),
        OrderEvent::Paid(OrderPaid {
            order_id: "ORD-001".to_string(),
            timestamp: Utc::now(),
        }),
        OrderEvent::Shipped(OrderShipped {
            order_id: "ORD-001".to_string(),
            timestamp: Utc::now(),
        }),
    ];

    projection.rebuild(events);

    // クエリ実行
    println!("--- Query: Get by ID ---");
    if let Some(order) = query_handler.get_by_id("ORD-001") {
        println!("Order: {} - Status: {}", order.order_id, order.status);
    }

    println!("\n--- Query: List by Customer ---");
    for order in query_handler.list_by_customer("CUST-001") {
        println!("  {} - {} - ¥{}", order.order_id, order.status, order.total_amount);
    }

    println!("\n--- Query: Statistics ---");
    let stats = query_handler.get_statistics();
    println!("Total orders: {}", stats.total_orders);
    println!("By status: {:?}", stats.by_status);
    println!("Total revenue: ¥{}", stats.total_revenue);

    println!("\n=== Demo Complete ===");
}
```

## 発展課題

### 課題1: 非同期プロジェクション

イベントの発行とプロジェクションの更新を非同期で行う仕組みを実装してください。

### 課題2: 複数のRead Model

同じイベントから異なる目的のRead Model（例：管理者用、顧客用）を構築してください。

### 課題3: プロジェクションのリビルド

Read Modelを最初から再構築する機能を実装してください。

## よくある間違い

### ❌ 間違い1: Read Modelから書き込みを行う

```rust
// 悪い例
fn update_order_status(&mut self, order_id: &str, status: &str) {
    self.read_model.orders.get_mut(order_id).unwrap().status = status.to_string();
}

// 良い例: コマンドを発行し、イベント経由で更新
fn update_order_status(&self, order_id: &str, status: &str) {
    self.command_handler.handle(UpdateStatusCommand { order_id, status });
    // イベントがプロジェクションに伝播してRead Modelが更新される
}
```

### ❌ 間違い2: 結果整合性を考慮しない

```rust
// 悪い例: コマンド直後にRead Modelを参照
command_handler.handle(PlaceOrderCommand { ... });
let order = query_handler.get_by_id(order_id);  // まだ反映されていない可能性

// 良い例: 結果整合性を考慮したUI設計
// - 楽観的UI更新
// - ポーリングまたはWebSocket
// - コマンドの結果を直接返す
```

## まとめ

この章では、CQRSパターンを学びました。

### 学んだこと

1. **CQRSの概念**: コマンドとクエリの分離
2. **Read Model**: クエリに最適化されたビュー
3. **プロジェクション**: イベントからRead Modelを構築
4. **クエリハンドラー**: 効率的なデータ取得

### 次の章への準備

次の章では、AWS SQSを使用したメッセージキューイングを学びます。クラウドサービスと連携した実践的なイベント駆動システムを構築します。

## 参考文献

- [Martin Fowler - CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft - CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Greg Young - CQRS and Event Sourcing](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)

---

[← 前の章: イベントソーシング](./05-event-sourcing.md) | [次の章: AWS SQS連携 →](./07-aws-sqs.md)
