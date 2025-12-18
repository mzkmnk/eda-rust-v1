# Chapter 09: Sagaパターン

## 学習の目的

この章を完了すると、以下のことができるようになります：

- Sagaパターンの概念と分散トランザクションの課題を理解する
- Choreography方式とOrchestration方式の違いを理解し、使い分けられる
- 補償トランザクション（Compensating Transaction）を実装できる
- 障害時のリカバリー戦略を設計できる

## 背景知識

### 分散トランザクションの課題

マイクロサービスアーキテクチャでは、1つのビジネス操作が複数のサービスにまたがることがあります。従来のACIDトランザクションは単一データベース内でしか機能しないため、分散環境では別のアプローチが必要です。

```
┌─────────────────────────────────────────────────────────────────┐
│                    注文処理の例                                  │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ Order    │   │ Payment  │   │ Inventory│   │ Shipping │    │
│  │ Service  │   │ Service  │   │ Service  │   │ Service  │    │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘    │
│       │              │              │              │           │
│       │  1. 注文作成 │              │              │           │
│       ├─────────────→│  2. 決済    │              │           │
│       │              ├─────────────→│  3. 在庫確保 │           │
│       │              │              ├─────────────→│  4. 発送  │
│       │              │              │              │           │
│  問題: 3で失敗したら、1と2をどう取り消す？                      │
└─────────────────────────────────────────────────────────────────┘
```

### Sagaパターンとは

Sagaは、長時間実行されるトランザクションを、一連のローカルトランザクションに分割するパターンです。各ステップが失敗した場合、それまでのステップを「補償」するトランザクションを実行します。

```
┌─────────────────────────────────────────────────────────────────┐
│                      Saga Pattern                               │
│                                                                 │
│  正常フロー:                                                    │
│  T1 ──→ T2 ──→ T3 ──→ T4 ──→ 完了                             │
│                                                                 │
│  T3で失敗した場合:                                              │
│  T1 ──→ T2 ──→ T3(失敗) ──→ C2 ──→ C1 ──→ ロールバック完了    │
│                            ↑     ↑                             │
│                         補償トランザクション                     │
│                                                                 │
│  Ti = ローカルトランザクション                                   │
│  Ci = 補償トランザクション（Tiを取り消す）                       │
└─────────────────────────────────────────────────────────────────┘
```

### Choreography vs Orchestration

Sagaの実装には2つのアプローチがあります：

#### Choreography（振り付け）

各サービスが自律的にイベントを発行・購読し、次のステップをトリガーします。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Choreography                                 │
│                                                                 │
│  Order      Payment     Inventory    Shipping                   │
│    │           │            │           │                       │
│    │ OrderCreated          │           │                       │
│    ├──────────→│            │           │                       │
│    │           │ PaymentCompleted       │                       │
│    │           ├───────────→│           │                       │
│    │           │            │ StockReserved                     │
│    │           │            ├──────────→│                       │
│    │           │            │           │ ShipmentCreated       │
│    │           │            │           ├──→                    │
│                                                                 │
│  特徴: 中央制御なし、各サービスが独立                           │
└─────────────────────────────────────────────────────────────────┘
```

#### Orchestration（指揮）

中央のオーケストレーターがSagaの実行を制御します。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Orchestration                                │
│                                                                 │
│              ┌─────────────────┐                               │
│              │  Orchestrator   │                               │
│              │  (Saga Manager) │                               │
│              └────────┬────────┘                               │
│                       │                                         │
│         ┌─────────────┼─────────────┐                          │
│         │             │             │                          │
│         ↓             ↓             ↓                          │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐                      │
│    │ Payment │  │Inventory│  │Shipping │                      │
│    │ Service │  │ Service │  │ Service │                      │
│    └─────────┘  └─────────┘  └─────────┘                      │
│                                                                 │
│  特徴: 中央制御、フロー把握が容易                               │
└─────────────────────────────────────────────────────────────────┘
```

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **Choreography** | 疎結合、シンプル | フロー把握困難、循環依存リスク |
| **Orchestration** | フロー明確、テスト容易 | 単一障害点、オーケストレーターの複雑化 |

## 概念の説明

### 補償トランザクション

補償トランザクションは、実行済みのトランザクションの効果を「意味的に」取り消すものです。

```
┌─────────────────────────────────────────────────────────────────┐
│              Compensating Transactions                          │
│                                                                 │
│  トランザクション          補償トランザクション                  │
│  ─────────────────────────────────────────────────────────────  │
│  CreateOrder              CancelOrder                           │
│  ProcessPayment           RefundPayment                         │
│  ReserveStock             ReleaseStock                          │
│  CreateShipment           CancelShipment                        │
│                                                                 │
│  注意: 補償は「元に戻す」ではなく「打ち消す」                    │
│  例: 決済の補償は「返金」であり、決済記録の削除ではない          │
└─────────────────────────────────────────────────────────────────┘
```

### Sagaの状態管理

```rust
enum SagaState {
    Started,
    PaymentProcessed,
    StockReserved,
    ShipmentCreated,
    Completed,
    // 補償状態
    Compensating,
    PaymentRefunded,
    StockReleased,
    Compensated,
    // エラー状態
    Failed,
}
```

### 冪等性の重要性

Sagaでは、同じ操作が複数回実行される可能性があります。すべての操作は冪等（何度実行しても同じ結果）である必要があります。

```rust
// 冪等な実装例
async fn reserve_stock(order_id: &str, product_id: &str, quantity: u32) -> Result<(), Error> {
    // 既に予約済みかチェック
    if is_already_reserved(order_id, product_id).await? {
        return Ok(());  // 冪等: 既に予約済みなら何もしない
    }

    // 予約を実行
    do_reserve(order_id, product_id, quantity).await
}
```

## 実装タスク

### タスク1: Sagaステップの定義

注文処理Sagaの各ステップと補償トランザクションを定義してください。

**要件:**
- 4つのステップ: 注文作成、決済、在庫確保、発送
- 各ステップの補償トランザクションを定義
- 状態遷移を管理する構造体

### タスク2: Orchestrator方式の実装

中央のオーケストレーターを実装してください。

**要件:**
- Sagaの実行を制御
- 失敗時に補償トランザクションを逆順で実行
- 状態の永続化

### タスク3: エラーハンドリングとリトライ

堅牢なエラーハンドリングを実装してください。

**要件:**
- 一時的なエラーはリトライ
- 永続的なエラーは補償を開始
- タイムアウト処理

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
#[async_trait]
trait SagaStep {
    async fn execute(&self, context: &mut SagaContext) -> Result<(), SagaError>;
    async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError>;
    fn name(&self) -> &'static str;
}

struct CreateOrderStep;
struct ProcessPaymentStep;
struct ReserveStockStep;
struct CreateShipmentStep;
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
struct SagaOrchestrator {
    steps: Vec<Box<dyn SagaStep>>,
    state_store: Box<dyn SagaStateStore>,
}

impl SagaOrchestrator {
    async fn execute(&self, saga_id: &str, context: &mut SagaContext) -> Result<(), SagaError> {
        let mut completed_steps = Vec::new();

        for step in &self.steps {
            match step.execute(context).await {
                Ok(()) => completed_steps.push(step),
                Err(e) => {
                    // 補償を逆順で実行
                    for completed in completed_steps.iter().rev() {
                        completed.compensate(context).await?;
                    }
                    return Err(e);
                }
            }
        }

        Ok(())
    }
}
```

</details>

## 回答（コード例）

### 完全な実装

```rust
use async_trait::async_trait;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use std::time::Duration;
use thiserror::Error;
use uuid::Uuid;

// ============================================================
// エラー型
// ============================================================

#[derive(Error, Debug, Clone)]
pub enum SagaError {
    #[error("Step failed: {step} - {message}")]
    StepFailed { step: String, message: String },
    #[error("Compensation failed: {step} - {message}")]
    CompensationFailed { step: String, message: String },
    #[error("Timeout: {step}")]
    Timeout { step: String },
    #[error("Saga not found: {0}")]
    NotFound(String),
}

// ============================================================
// Sagaコンテキスト
// ============================================================

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SagaContext {
    pub saga_id: String,
    pub order_id: String,
    pub customer_id: String,
    pub items: Vec<OrderItem>,
    pub total_amount: u64,
    // 各ステップの結果
    pub payment_id: Option<String>,
    pub reservation_id: Option<String>,
    pub shipment_id: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderItem {
    pub product_id: String,
    pub quantity: u32,
    pub unit_price: u64,
}

impl SagaContext {
    pub fn new(order_id: String, customer_id: String, items: Vec<OrderItem>) -> Self {
        let total_amount = items.iter().map(|i| i.unit_price * i.quantity as u64).sum();
        Self {
            saga_id: Uuid::new_v4().to_string(),
            order_id,
            customer_id,
            items,
            total_amount,
            payment_id: None,
            reservation_id: None,
            shipment_id: None,
        }
    }
}

// ============================================================
// Saga状態
// ============================================================

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum SagaState {
    Started,
    OrderCreated,
    PaymentProcessed,
    StockReserved,
    ShipmentCreated,
    Completed,
    Compensating,
    Compensated,
    Failed,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SagaRecord {
    pub saga_id: String,
    pub state: SagaState,
    pub context: SagaContext,
    pub current_step: usize,
    pub error: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// ============================================================
// Sagaステップトレイト
// ============================================================

#[async_trait]
pub trait SagaStep: Send + Sync {
    async fn execute(&self, context: &mut SagaContext) -> Result<(), SagaError>;
    async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError>;
    fn name(&self) -> &'static str;
}

// ============================================================
// 各ステップの実装
// ============================================================

pub struct CreateOrderStep;

#[async_trait]
impl SagaStep for CreateOrderStep {
    async fn execute(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        println!("[CreateOrder] Creating order: {}", context.order_id);
        // 実際の実装ではDBに保存
        tokio::time::sleep(Duration::from_millis(50)).await;
        println!("[CreateOrder] Order created successfully");
        Ok(())
    }

    async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        println!("[CreateOrder] Cancelling order: {}", context.order_id);
        tokio::time::sleep(Duration::from_millis(50)).await;
        println!("[CreateOrder] Order cancelled");
        Ok(())
    }

    fn name(&self) -> &'static str {
        "CreateOrder"
    }
}

pub struct ProcessPaymentStep;

#[async_trait]
impl SagaStep for ProcessPaymentStep {
    async fn execute(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        println!("[ProcessPayment] Processing payment: {} yen", context.total_amount);
        tokio::time::sleep(Duration::from_millis(100)).await;

        // 決済処理（シミュレーション）
        let payment_id = format!("PAY-{}", Uuid::new_v4());
        context.payment_id = Some(payment_id.clone());

        println!("[ProcessPayment] Payment successful: {}", payment_id);
        Ok(())
    }

    async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        if let Some(payment_id) = &context.payment_id {
            println!("[ProcessPayment] Refunding payment: {}", payment_id);
            tokio::time::sleep(Duration::from_millis(100)).await;
            println!("[ProcessPayment] Refund completed");
        }
        Ok(())
    }

    fn name(&self) -> &'static str {
        "ProcessPayment"
    }
}

pub struct ReserveStockStep {
    pub should_fail: bool,  // テスト用
}

#[async_trait]
impl SagaStep for ReserveStockStep {
    async fn execute(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        println!("[ReserveStock] Reserving stock for {} items", context.items.len());
        tokio::time::sleep(Duration::from_millis(50)).await;

        // テスト用: 失敗をシミュレート
        if self.should_fail {
            return Err(SagaError::StepFailed {
                step: self.name().to_string(),
                message: "Insufficient stock".to_string(),
            });
        }

        let reservation_id = format!("RES-{}", Uuid::new_v4());
        context.reservation_id = Some(reservation_id.clone());

        println!("[ReserveStock] Stock reserved: {}", reservation_id);
        Ok(())
    }

    async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        if let Some(reservation_id) = &context.reservation_id {
            println!("[ReserveStock] Releasing stock: {}", reservation_id);
            tokio::time::sleep(Duration::from_millis(50)).await;
            println!("[ReserveStock] Stock released");
        }
        Ok(())
    }

    fn name(&self) -> &'static str {
        "ReserveStock"
    }
}

pub struct CreateShipmentStep;

#[async_trait]
impl SagaStep for CreateShipmentStep {
    async fn execute(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        println!("[CreateShipment] Creating shipment for order: {}", context.order_id);
        tokio::time::sleep(Duration::from_millis(50)).await;

        let shipment_id = format!("SHIP-{}", Uuid::new_v4());
        context.shipment_id = Some(shipment_id.clone());

        println!("[CreateShipment] Shipment created: {}", shipment_id);
        Ok(())
    }

    async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError> {
        if let Some(shipment_id) = &context.shipment_id {
            println!("[CreateShipment] Cancelling shipment: {}", shipment_id);
            tokio::time::sleep(Duration::from_millis(50)).await;
            println!("[CreateShipment] Shipment cancelled");
        }
        Ok(())
    }

    fn name(&self) -> &'static str {
        "CreateShipment"
    }
}
```



// ============================================================
// Saga状態ストア
// ============================================================

pub trait SagaStateStore: Send + Sync {
    fn save(&self, record: &SagaRecord);
    fn load(&self, saga_id: &str) -> Option<SagaRecord>;
    fn update_state(&self, saga_id: &str, state: SagaState);
}

#[derive(Default)]
pub struct InMemorySagaStateStore {
    records: RwLock<HashMap<String, SagaRecord>>,
}

impl SagaStateStore for InMemorySagaStateStore {
    fn save(&self, record: &SagaRecord) {
        self.records.write().unwrap().insert(record.saga_id.clone(), record.clone());
    }

    fn load(&self, saga_id: &str) -> Option<SagaRecord> {
        self.records.read().unwrap().get(saga_id).cloned()
    }

    fn update_state(&self, saga_id: &str, state: SagaState) {
        if let Some(record) = self.records.write().unwrap().get_mut(saga_id) {
            record.state = state;
            record.updated_at = Utc::now();
        }
    }
}

// ============================================================
// Sagaオーケストレーター
// ============================================================

pub struct SagaOrchestrator {
    steps: Vec<Arc<dyn SagaStep>>,
    state_store: Arc<dyn SagaStateStore>,
}

impl SagaOrchestrator {
    pub fn new(state_store: Arc<dyn SagaStateStore>) -> Self {
        Self {
            steps: Vec::new(),
            state_store,
        }
    }

    pub fn add_step(mut self, step: Arc<dyn SagaStep>) -> Self {
        self.steps.push(step);
        self
    }

    pub async fn execute(&self, mut context: SagaContext) -> Result<SagaContext, SagaError> {
        let saga_id = context.saga_id.clone();

        // 初期状態を保存
        let record = SagaRecord {
            saga_id: saga_id.clone(),
            state: SagaState::Started,
            context: context.clone(),
            current_step: 0,
            error: None,
            created_at: Utc::now(),
            updated_at: Utc::now(),
        };
        self.state_store.save(&record);

        println!("\n=== Saga Started: {} ===\n", saga_id);

        let mut completed_steps: Vec<Arc<dyn SagaStep>> = Vec::new();

        for (i, step) in self.steps.iter().enumerate() {
            println!("--- Step {}: {} ---", i + 1, step.name());

            match step.execute(&mut context).await {
                Ok(()) => {
                    completed_steps.push(step.clone());
                    self.state_store.update_state(&saga_id, self.state_for_step(i));
                }
                Err(e) => {
                    println!("\n!!! Step {} failed: {} !!!\n", step.name(), e);
                    self.state_store.update_state(&saga_id, SagaState::Compensating);

                    // 補償を逆順で実行
                    self.compensate(&mut context, &completed_steps).await?;

                    self.state_store.update_state(&saga_id, SagaState::Compensated);
                    return Err(e);
                }
            }
        }

        self.state_store.update_state(&saga_id, SagaState::Completed);
        println!("\n=== Saga Completed: {} ===\n", saga_id);

        Ok(context)
    }

    async fn compensate(
        &self,
        context: &mut SagaContext,
        completed_steps: &[Arc<dyn SagaStep>],
    ) -> Result<(), SagaError> {
        println!("--- Starting Compensation ---\n");

        for step in completed_steps.iter().rev() {
            println!("Compensating: {}", step.name());
            if let Err(e) = step.compensate(context).await {
                println!("Compensation failed for {}: {}", step.name(), e);
                // 補償の失敗は記録するが、続行する
            }
        }

        println!("\n--- Compensation Complete ---");
        Ok(())
    }

    fn state_for_step(&self, step_index: usize) -> SagaState {
        match step_index {
            0 => SagaState::OrderCreated,
            1 => SagaState::PaymentProcessed,
            2 => SagaState::StockReserved,
            3 => SagaState::ShipmentCreated,
            _ => SagaState::Completed,
        }
    }
}

// ============================================================
// 使用例
// ============================================================

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== Saga Pattern Demo ===");

    let state_store = Arc::new(InMemorySagaStateStore::default());

    // --- 成功ケース ---
    println!("\n########## SUCCESS CASE ##########");

    let orchestrator = SagaOrchestrator::new(state_store.clone())
        .add_step(Arc::new(CreateOrderStep))
        .add_step(Arc::new(ProcessPaymentStep))
        .add_step(Arc::new(ReserveStockStep { should_fail: false }))
        .add_step(Arc::new(CreateShipmentStep));

    let context = SagaContext::new(
        "ORD-001".to_string(),
        "CUST-001".to_string(),
        vec![
            OrderItem { product_id: "PROD-001".to_string(), quantity: 2, unit_price: 1000 },
            OrderItem { product_id: "PROD-002".to_string(), quantity: 1, unit_price: 3000 },
        ],
    );

    match orchestrator.execute(context).await {
        Ok(ctx) => {
            println!("Saga succeeded!");
            println!("  Payment ID: {:?}", ctx.payment_id);
            println!("  Reservation ID: {:?}", ctx.reservation_id);
            println!("  Shipment ID: {:?}", ctx.shipment_id);
        }
        Err(e) => println!("Saga failed: {}", e),
    }

    // --- 失敗ケース（補償実行） ---
    println!("\n########## FAILURE CASE (with compensation) ##########");

    let orchestrator_fail = SagaOrchestrator::new(state_store.clone())
        .add_step(Arc::new(CreateOrderStep))
        .add_step(Arc::new(ProcessPaymentStep))
        .add_step(Arc::new(ReserveStockStep { should_fail: true }))  // 失敗させる
        .add_step(Arc::new(CreateShipmentStep));

    let context_fail = SagaContext::new(
        "ORD-002".to_string(),
        "CUST-002".to_string(),
        vec![OrderItem { product_id: "PROD-003".to_string(), quantity: 100, unit_price: 500 }],
    );

    match orchestrator_fail.execute(context_fail).await {
        Ok(_) => println!("Unexpected success"),
        Err(e) => println!("\nSaga failed as expected: {}", e),
    }

    println!("\n=== Demo Complete ===");
    Ok(())
}
```

### 出力例

```
=== Saga Pattern Demo ===

########## SUCCESS CASE ##########

=== Saga Started: abc123... ===

--- Step 1: CreateOrder ---
[CreateOrder] Creating order: ORD-001
[CreateOrder] Order created successfully
--- Step 2: ProcessPayment ---
[ProcessPayment] Processing payment: 5000 yen
[ProcessPayment] Payment successful: PAY-xyz...
--- Step 3: ReserveStock ---
[ReserveStock] Reserving stock for 2 items
[ReserveStock] Stock reserved: RES-xyz...
--- Step 4: CreateShipment ---
[CreateShipment] Creating shipment for order: ORD-001
[CreateShipment] Shipment created: SHIP-xyz...

=== Saga Completed: abc123... ===

Saga succeeded!
  Payment ID: Some("PAY-xyz...")
  Reservation ID: Some("RES-xyz...")
  Shipment ID: Some("SHIP-xyz...")

########## FAILURE CASE (with compensation) ##########

=== Saga Started: def456... ===

--- Step 1: CreateOrder ---
[CreateOrder] Creating order: ORD-002
[CreateOrder] Order created successfully
--- Step 2: ProcessPayment ---
[ProcessPayment] Processing payment: 50000 yen
[ProcessPayment] Payment successful: PAY-abc...

--- Step 3: ReserveStock ---
[ReserveStock] Reserving stock for 1 items

!!! Step ReserveStock failed: Step failed: ReserveStock - Insufficient stock !!!

--- Starting Compensation ---

Compensating: ProcessPayment
[ProcessPayment] Refunding payment: PAY-abc...
[ProcessPayment] Refund completed
Compensating: CreateOrder
[CreateOrder] Cancelling order: ORD-002
[CreateOrder] Order cancelled

--- Compensation Complete ---

Saga failed as expected: Step failed: ReserveStock - Insufficient stock
```

## 発展課題

### 課題1: Choreography方式の実装

イベント駆動のChoreography方式でSagaを実装してください。

### 課題2: タイムアウトとリトライ

各ステップにタイムアウトと指数バックオフリトライを追加してください。

### 課題3: Saga状態の永続化

Sagaの状態をデータベースに永続化し、システム再起動後も継続できるようにしてください。

## よくある間違い

### ❌ 間違い1: 補償トランザクションを冪等にしない

```rust
// 悪い例: 冪等でない補償
async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError> {
    // 2回呼ばれると2回返金してしまう
    refund(context.payment_id, context.total_amount).await
}

// 良い例: 冪等な補償
async fn compensate(&self, context: &mut SagaContext) -> Result<(), SagaError> {
    if is_already_refunded(context.payment_id).await? {
        return Ok(());  // 既に返金済み
    }
    refund(context.payment_id, context.total_amount).await
}
```

### ❌ 間違い2: 補償の失敗で全体を停止

```rust
// 悪い例: 補償失敗で停止
for step in completed_steps.iter().rev() {
    step.compensate(context).await?;  // 失敗したら終了
}

// 良い例: 補償失敗を記録して続行
for step in completed_steps.iter().rev() {
    if let Err(e) = step.compensate(context).await {
        log::error!("Compensation failed: {}", e);
        // 続行して他の補償も実行
    }
}
```

### ❌ 間違い3: 状態を永続化しない

```rust
// 悪い例: メモリのみで状態管理
// システム再起動で状態が失われる

// 良い例: 各ステップ後に状態を永続化
step.execute(context).await?;
state_store.save(&saga_record).await?;  // 永続化
```

## まとめ

この章では、Sagaパターンによる分散トランザクション管理を学びました。

### 学んだこと

1. **Sagaパターンの概念**: 分散トランザクションの課題と解決策
2. **Choreography vs Orchestration**: 2つの実装アプローチ
3. **補償トランザクション**: 失敗時のロールバック
4. **冪等性**: 重複実行への対応
5. **状態管理**: Sagaの進行状況の追跡

### 学習の完了

おめでとうございます！これでEvent Driven Architecture学習ガイドのすべての章を完了しました。

学んだ内容を振り返ると：

1. **基礎編**: イベントの定義、チャネル通信、非同期処理
2. **応用編**: イベントストア、イベントソーシング、CQRS
3. **実践編**: AWS SQS、EventBridge、Sagaパターン

これらの知識を組み合わせることで、スケーラブルで堅牢なイベント駆動システムを構築できます。

## 参考文献

- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/data/saga.html)
- [Saga Pattern - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)
- [Compensating Transaction Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction)
- [Designing Data-Intensive Applications - Martin Kleppmann](https://dataintensive.net/)

---

[← 前の章: AWS EventBridge連携](./08-aws-eventbridge.md) | [Overview に戻る](./00-overview.md)
