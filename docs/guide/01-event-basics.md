# Chapter 01: イベントの基本

## 学習の目的

この章を完了すると、以下のことができるようになります：

- イベント駆動アーキテクチャにおける「イベント」の本質を理解する
- Rustの型システムを活用して、型安全なイベントを定義できる
- イベントに必要なメタデータを設計できる
- serdeを使用してイベントをシリアライズ/デシリアライズできる

## 背景知識

### なぜイベントが重要なのか

従来のシステム設計では、データの「現在の状態」のみを保存することが一般的でした。例えば、銀行口座の残高が10,000円であるという事実は保存しますが、どのような取引を経てその残高になったかは保存しません。

```
従来のアプローチ（状態のみ保存）:
┌─────────────────────────────────┐
│ Account                         │
│ ├── id: "ACC-001"              │
│ ├── balance: 10000             │  ← 現在の状態のみ
│ └── updated_at: "2024-01-15"   │
└─────────────────────────────────┘
```

しかし、この方法には問題があります：

- **監査が困難**: なぜ残高が10,000円なのか追跡できない
- **バグの原因特定が困難**: 不正な状態になった経緯がわからない
- **ビジネスインサイトの欠如**: 顧客の行動パターンを分析できない

イベント駆動アプローチでは、「何が起きたか」をすべて記録します：

```
イベント駆動アプローチ（履歴を保存）:
┌─────────────────────────────────┐
│ Events                          │
│ ├── AccountOpened(ACC-001)     │
│ ├── MoneyDeposited(5000)       │
│ ├── MoneyDeposited(8000)       │
│ └── MoneyWithdrawn(3000)       │  → 合計: 10000
└─────────────────────────────────┘
```

### イベントの定義

> **イベント（Event）とは、過去に発生した事実の不変な記録である。**

この定義には3つの重要な要素があります：

1. **過去に発生した**: イベントは常に過去形で表現される（`OrderPlaced`、`PaymentReceived`）
2. **事実**: イベントは実際に起きたことであり、取り消すことはできない
3. **不変**: 一度記録されたイベントは変更されない

### イベント vs コマンド vs クエリ

EDAを理解する上で、イベント、コマンド、クエリの違いを明確にすることが重要です：

| 種類 | 時制 | 意図 | 例 |
|------|------|------|-----|
| **コマンド** | 命令形 | 何かをしてほしい | `PlaceOrder`, `CancelSubscription` |
| **イベント** | 過去形 | 何かが起きた | `OrderPlaced`, `SubscriptionCancelled` |
| **クエリ** | 疑問形 | 情報を知りたい | `GetOrderStatus`, `ListProducts` |

```
┌─────────────┐     コマンド      ┌─────────────┐
│   Client    │ ───────────────→ │   System    │
└─────────────┘  PlaceOrder      └─────────────┘
                                        │
                                        │ 処理
                                        ↓
┌─────────────┐     イベント      ┌─────────────┐
│ Subscriber  │ ←─────────────── │   System    │
└─────────────┘  OrderPlaced     └─────────────┘
```

### イベントの種類

イベントは用途によって分類できます：

#### 1. ドメインイベント（Domain Event）

ビジネスドメインで発生した重要な出来事を表します。ドメインエキスパートが理解できる言葉で表現されます。

```rust
// ドメインイベントの例
OrderPlaced { order_id, customer_id, items, total_amount }
PaymentReceived { payment_id, order_id, amount, method }
ShipmentDispatched { shipment_id, order_id, carrier, tracking_number }
```

#### 2. 統合イベント（Integration Event）

異なるサービス間で共有されるイベントです。サービス境界を越えて伝播します。

```rust
// 統合イベントの例（外部サービスに公開）
CustomerCreated { customer_id, email, created_at }
InventoryUpdated { product_id, quantity, warehouse_id }
```

#### 3. システムイベント（System Event）

技術的な関心事を表すイベントです。監視やデバッグに使用されます。

```rust
// システムイベントの例
ServiceStarted { service_name, version, timestamp }
ErrorOccurred { error_code, message, stack_trace }
```

## 概念の説明

### イベントの構造

適切に設計されたイベントは、以下の要素を含みます：

```
┌─────────────────────────────────────────────────────────────────┐
│                         Event                                    │
├─────────────────────────────────────────────────────────────────┤
│  Metadata（メタデータ）                                          │
│  ├── event_id: UUID          # イベントの一意識別子              │
│  ├── event_type: String      # イベントの種類                   │
│  ├── timestamp: DateTime     # 発生日時                         │
│  ├── version: u32            # スキーマバージョン               │
│  ├── aggregate_id: UUID      # 関連する集約のID                 │
│  └── correlation_id: UUID    # 関連するリクエストのID           │
├─────────────────────────────────────────────────────────────────┤
│  Payload（ペイロード）                                           │
│  └── ドメイン固有のデータ                                        │
└─────────────────────────────────────────────────────────────────┘
```

### メタデータの役割

| フィールド | 役割 | 例 |
|-----------|------|-----|
| `event_id` | イベントを一意に識別。重複検出に使用 | `550e8400-e29b-41d4-a716-446655440000` |
| `event_type` | イベントの種類を識別。ルーティングに使用 | `"OrderPlaced"` |
| `timestamp` | イベント発生時刻。順序付けや監査に使用 | `2024-01-15T10:30:00Z` |
| `version` | スキーマバージョン。互換性管理に使用 | `1` |
| `aggregate_id` | 関連する集約（エンティティ）のID | `order-123` |
| `correlation_id` | 一連の処理を追跡するためのID | `req-456` |

### Rustでのイベント表現

Rustでイベントを表現する方法はいくつかあります：

#### 方法1: 構造体 + enum

最も一般的なアプローチです。各イベントを構造体で定義し、enumでまとめます。

```rust
// 個別のイベント構造体
struct OrderPlaced {
    order_id: String,
    customer_id: String,
    total_amount: u64,
}

struct OrderShipped {
    order_id: String,
    tracking_number: String,
}

// enumでまとめる
enum OrderEvent {
    Placed(OrderPlaced),
    Shipped(OrderShipped),
}
```

#### 方法2: トレイトベース

より柔軟なアプローチです。共通のインターフェースをトレイトで定義します。

```rust
trait Event {
    fn event_type(&self) -> &str;
    fn aggregate_id(&self) -> &str;
}
```

## 実装タスク

以下のタスクを順番に実装してください。各タスクは前のタスクの上に構築されます。

### タスク1: 基本的なイベント構造体を定義する

ECサイトの注文ドメインを題材に、以下のイベントを定義してください：

1. `OrderPlaced` - 注文が作成された
2. `OrderPaid` - 注文の支払いが完了した
3. `OrderShipped` - 注文が発送された
4. `OrderCancelled` - 注文がキャンセルされた

**要件:**
- 各イベントに適切なフィールドを含める
- イベント名は過去形にする
- `Debug`トレイトを実装する

### タスク2: イベントメタデータを追加する

タスク1で作成したイベントに、以下のメタデータを追加してください：

- `event_id`: UUID
- `timestamp`: 日時
- `aggregate_id`: 注文ID

**要件:**
- `uuid`クレートを使用する
- `chrono`クレートを使用する

### タスク3: シリアライズ/デシリアライズを実装する

イベントをJSONに変換できるようにしてください。

**要件:**
- `serde`と`serde_json`を使用する
- シリアライズとデシリアライズの両方をテストする

### タスク4: イベントトレイトを実装する

共通のインターフェースを持つ`Event`トレイトを定義し、各イベントに実装してください。

**要件:**
- `event_type()` - イベントの種類を返す
- `aggregate_id()` - 集約IDを返す
- `timestamp()` - タイムスタンプを返す

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
// 構造体の定義には #[derive] マクロを活用しましょう
#[derive(Debug)]
struct OrderPlaced {
    // フィールドを定義
}

// 注文に必要な情報を考えてみましょう：
// - 注文ID
// - 顧客ID
// - 注文した商品（商品ID、数量、価格）
// - 合計金額
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
// Cargo.tomlに追加
// [dependencies]
// uuid = { version = "1.11", features = ["v4"] }
// chrono = { version = "0.4", features = ["serde"] }

use uuid::Uuid;
use chrono::{DateTime, Utc};

// メタデータを含む構造体
struct EventMetadata {
    event_id: Uuid,
    timestamp: DateTime<Utc>,
    aggregate_id: String,
}

// イベントにメタデータを含める方法：
// 1. 各イベントにメタデータフィールドを追加
// 2. ラッパー構造体を作成
```

</details>

<details>
<summary>タスク3のヒント</summary>

```rust
// Cargo.tomlに追加
// [dependencies]
// serde = { version = "1.0", features = ["derive"] }
// serde_json = "1.0"

use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct OrderPlaced {
    // フィールド
}

// シリアライズ
let json = serde_json::to_string(&event)?;

// デシリアライズ
let event: OrderPlaced = serde_json::from_str(&json)?;
```

</details>

<details>
<summary>タスク4のヒント</summary>

```rust
// トレイトの定義
trait Event {
    fn event_type(&self) -> &'static str;
    fn aggregate_id(&self) -> &str;
    fn timestamp(&self) -> DateTime<Utc>;
}

// 実装例
impl Event for OrderPlaced {
    fn event_type(&self) -> &'static str {
        "OrderPlaced"
    }
    // ...
}
```

</details>



## 回答（コード例）

### 完全な実装

まず、`Cargo.toml`に必要な依存関係を追加します：

```toml
[package]
name = "eda-rust-v1"
version = "0.1.0"
edition = "2021"

[dependencies]
uuid = { version = "1.11", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

次に、イベントの実装です：

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

// ============================================================
// イベントメタデータ
// ============================================================

/// すべてのイベントに共通するメタデータ
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventMetadata {
    /// イベントの一意識別子
    pub event_id: Uuid,
    /// イベント発生時刻
    pub timestamp: DateTime<Utc>,
    /// 関連する集約（この場合は注文）のID
    pub aggregate_id: String,
    /// スキーマバージョン（将来の互換性のため）
    pub version: u32,
}

impl EventMetadata {
    pub fn new(aggregate_id: impl Into<String>) -> Self {
        Self {
            event_id: Uuid::new_v4(),
            timestamp: Utc::now(),
            aggregate_id: aggregate_id.into(),
            version: 1,
        }
    }
}

// ============================================================
// 注文アイテム（値オブジェクト）
// ============================================================

/// 注文に含まれる商品
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderItem {
    pub product_id: String,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price: u64,
}

impl OrderItem {
    pub fn subtotal(&self) -> u64 {
        self.unit_price * self.quantity as u64
    }
}

// ============================================================
// ドメインイベント
// ============================================================

/// 注文が作成されたイベント
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    pub metadata: EventMetadata,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
    pub total_amount: u64,
    pub shipping_address: String,
}

/// 注文の支払いが完了したイベント
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPaid {
    pub metadata: EventMetadata,
    pub payment_id: String,
    pub payment_method: String,
    pub amount: u64,
}

/// 注文が発送されたイベント
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderShipped {
    pub metadata: EventMetadata,
    pub shipment_id: String,
    pub carrier: String,
    pub tracking_number: String,
}

/// 注文がキャンセルされたイベント
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderCancelled {
    pub metadata: EventMetadata,
    pub reason: String,
    pub cancelled_by: String,
}

// ============================================================
// イベントトレイト
// ============================================================

/// すべてのイベントが実装すべきトレイト
pub trait Event {
    /// イベントの種類を返す
    fn event_type(&self) -> &'static str;

    /// 関連する集約のIDを返す
    fn aggregate_id(&self) -> &str;

    /// イベント発生時刻を返す
    fn timestamp(&self) -> DateTime<Utc>;

    /// イベントIDを返す
    fn event_id(&self) -> Uuid;
}

// 各イベントにトレイトを実装
impl Event for OrderPlaced {
    fn event_type(&self) -> &'static str {
        "OrderPlaced"
    }

    fn aggregate_id(&self) -> &str {
        &self.metadata.aggregate_id
    }

    fn timestamp(&self) -> DateTime<Utc> {
        self.metadata.timestamp
    }

    fn event_id(&self) -> Uuid {
        self.metadata.event_id
    }
}

impl Event for OrderPaid {
    fn event_type(&self) -> &'static str {
        "OrderPaid"
    }

    fn aggregate_id(&self) -> &str {
        &self.metadata.aggregate_id
    }

    fn timestamp(&self) -> DateTime<Utc> {
        self.metadata.timestamp
    }

    fn event_id(&self) -> Uuid {
        self.metadata.event_id
    }
}

impl Event for OrderShipped {
    fn event_type(&self) -> &'static str {
        "OrderShipped"
    }

    fn aggregate_id(&self) -> &str {
        &self.metadata.aggregate_id
    }

    fn timestamp(&self) -> DateTime<Utc> {
        self.metadata.timestamp
    }

    fn event_id(&self) -> Uuid {
        self.metadata.event_id
    }
}

impl Event for OrderCancelled {
    fn event_type(&self) -> &'static str {
        "OrderCancelled"
    }

    fn aggregate_id(&self) -> &str {
        &self.metadata.aggregate_id
    }

    fn timestamp(&self) -> DateTime<Utc> {
        self.metadata.timestamp
    }

    fn event_id(&self) -> Uuid {
        self.metadata.event_id
    }
}

// ============================================================
// イベントのenum（型安全なディスパッチ用）
// ============================================================

/// 注文ドメインのすべてのイベントを表すenum
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum OrderEvent {
    Placed(OrderPlaced),
    Paid(OrderPaid),
    Shipped(OrderShipped),
    Cancelled(OrderCancelled),
}

impl Event for OrderEvent {
    fn event_type(&self) -> &'static str {
        match self {
            OrderEvent::Placed(_) => "OrderPlaced",
            OrderEvent::Paid(_) => "OrderPaid",
            OrderEvent::Shipped(_) => "OrderShipped",
            OrderEvent::Cancelled(_) => "OrderCancelled",
        }
    }

    fn aggregate_id(&self) -> &str {
        match self {
            OrderEvent::Placed(e) => e.aggregate_id(),
            OrderEvent::Paid(e) => e.aggregate_id(),
            OrderEvent::Shipped(e) => e.aggregate_id(),
            OrderEvent::Cancelled(e) => e.aggregate_id(),
        }
    }

    fn timestamp(&self) -> DateTime<Utc> {
        match self {
            OrderEvent::Placed(e) => e.timestamp(),
            OrderEvent::Paid(e) => e.timestamp(),
            OrderEvent::Shipped(e) => e.timestamp(),
            OrderEvent::Cancelled(e) => e.timestamp(),
        }
    }

    fn event_id(&self) -> Uuid {
        match self {
            OrderEvent::Placed(e) => e.event_id(),
            OrderEvent::Paid(e) => e.event_id(),
            OrderEvent::Shipped(e) => e.event_id(),
            OrderEvent::Cancelled(e) => e.event_id(),
        }
    }
}
```

### 使用例

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 注文IDを生成
    let order_id = format!("ORD-{}", Uuid::new_v4());

    // 注文作成イベント
    let order_placed = OrderPlaced {
        metadata: EventMetadata::new(&order_id),
        customer_id: "CUST-001".to_string(),
        items: vec![
            OrderItem {
                product_id: "PROD-001".to_string(),
                product_name: "Rust Programming Book".to_string(),
                quantity: 1,
                unit_price: 4500,
            },
            OrderItem {
                product_id: "PROD-002".to_string(),
                product_name: "Mechanical Keyboard".to_string(),
                quantity: 1,
                unit_price: 15000,
            },
        ],
        total_amount: 19500,
        shipping_address: "東京都渋谷区...".to_string(),
    };

    // JSONにシリアライズ
    let json = serde_json::to_string_pretty(&order_placed)?;
    println!("Serialized Event:\n{}\n", json);

    // JSONからデシリアライズ
    let deserialized: OrderPlaced = serde_json::from_str(&json)?;
    println!("Event Type: {}", deserialized.event_type());
    println!("Aggregate ID: {}", deserialized.aggregate_id());
    println!("Timestamp: {}", deserialized.timestamp());

    // enumを使用した例
    let event = OrderEvent::Placed(order_placed);
    let enum_json = serde_json::to_string_pretty(&event)?;
    println!("\nEnum Serialized:\n{}", enum_json);

    Ok(())
}
```

### 出力例

```json
Serialized Event:
{
  "metadata": {
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-15T10:30:00Z",
    "aggregate_id": "ORD-123e4567-e89b-12d3-a456-426614174000",
    "version": 1
  },
  "customer_id": "CUST-001",
  "items": [
    {
      "product_id": "PROD-001",
      "product_name": "Rust Programming Book",
      "quantity": 1,
      "unit_price": 4500
    }
  ],
  "total_amount": 19500,
  "shipping_address": "東京都渋谷区..."
}
```

### コード解説

#### なぜ`EventMetadata`を分離するのか

メタデータを別構造体にすることで：

1. **再利用性**: すべてのイベントで同じメタデータ構造を使用
2. **一貫性**: メタデータの生成ロジックを一箇所に集約
3. **拡張性**: 新しいメタデータフィールドの追加が容易

#### なぜ`#[serde(tag = "type")]`を使うのか

```rust
#[serde(tag = "type")]
pub enum OrderEvent { ... }
```

この属性により、JSONに`type`フィールドが追加され、デシリアライズ時にどのバリアントかを判別できます：

```json
{
  "type": "Placed",
  "metadata": { ... },
  "customer_id": "..."
}
```

## 発展課題

### 課題1: イベントのバージョニング

イベントスキーマは時間とともに変化します。古いバージョンのイベントを新しいコードで読み取れるようにする仕組みを実装してください。

**ヒント**: `version`フィールドを使用し、デシリアライズ時にマイグレーションを行う

### 課題2: イベントのバリデーション

イベント作成時にビジネスルールを検証する仕組みを追加してください。

例：
- `total_amount`は`items`の合計と一致すること
- `items`は空でないこと

### 課題3: 相関IDとcausation IDの追加

分散システムでのトレーシングのために、以下のフィールドを追加してください：

- `correlation_id`: 一連のリクエストを追跡するID
- `causation_id`: このイベントを引き起こしたイベントのID

## よくある間違い

### ❌ 間違い1: イベント名を現在形にする

```rust
// 悪い例
struct OrderPlace { ... }  // 現在形
struct PlaceOrder { ... }  // 命令形（これはコマンド）

// 良い例
struct OrderPlaced { ... }  // 過去形
```

イベントは「すでに起きたこと」を表すため、必ず過去形を使用します。

### ❌ 間違い2: イベントにビジネスロジックを入れる

```rust
// 悪い例
impl OrderPlaced {
    pub fn apply_discount(&mut self, rate: f64) {
        self.total_amount = (self.total_amount as f64 * (1.0 - rate)) as u64;
    }
}

// 良い例
// イベントは不変。ビジネスロジックは別の場所（集約やサービス）に置く
```

イベントは事実の記録であり、変更されるべきではありません。

### ❌ 間違い3: 大きすぎるイベント

```rust
// 悪い例: 関係ないデータまで含めている
struct OrderPlaced {
    order_id: String,
    customer_id: String,
    customer_email: String,      // 顧客情報は別イベントで
    customer_address: String,    // 顧客情報は別イベントで
    product_details: Vec<...>,   // 商品の詳細情報は不要
    // ...
}

// 良い例: 必要最小限のデータ
struct OrderPlaced {
    order_id: String,
    customer_id: String,         // IDのみ
    items: Vec<OrderItem>,       // 注文時点の情報のみ
    total_amount: u64,
}
```

イベントには、そのイベントを理解するために必要な最小限の情報のみを含めます。

### ❌ 間違い4: 可変なフィールドを使用する

```rust
// 悪い例
struct OrderPlaced {
    pub items: Vec<OrderItem>,  // 外部から変更可能
}

// 良い例: 不変性を保証
struct OrderPlaced {
    items: Vec<OrderItem>,  // privateにする
}

impl OrderPlaced {
    pub fn items(&self) -> &[OrderItem] {
        &self.items
    }
}
```

## まとめ

この章では、イベント駆動アーキテクチャの基礎となる「イベント」について学びました。

### 学んだこと

1. **イベントの定義**: 過去に発生した事実の不変な記録
2. **イベントの種類**: ドメインイベント、統合イベント、システムイベント
3. **イベント vs コマンド**: イベントは過去形、コマンドは命令形
4. **Rustでの実装**: 構造体、enum、トレイトを組み合わせた型安全な設計
5. **メタデータの重要性**: event_id、timestamp、aggregate_idなど
6. **シリアライズ**: serdeを使用したJSON変換

### 次の章への準備

次の章では、これらのイベントを複数のコンポーネント間で伝達する方法を学びます。Rustのチャネルを使用したPublisher/Subscriberパターンを実装します。

## 参考文献

- [Domain-Driven Design Reference - Eric Evans](https://www.domainlanguage.com/ddd/reference/)
- [Event Storming - Alberto Brandolini](https://www.eventstorming.com/)
- [Serde Documentation](https://serde.rs/)
- [UUID Crate Documentation](https://docs.rs/uuid/)
- [Chrono Crate Documentation](https://docs.rs/chrono/)

---

[← 前の章: Overview](./00-overview.md) | [次の章: チャネルとPub/Sub →](./02-channel-pubsub.md)
