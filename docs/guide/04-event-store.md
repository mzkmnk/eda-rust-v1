# Chapter 04: イベントストア

## 学習の目的

この章を完了すると、以下のことができるようになります：

- イベントストアの役割と設計原則を理解する
- インメモリイベントストアを実装できる
- イベントの追記、検索、フィルタリングができる
- 楽観的ロックによる並行制御を実装できる

## 背景知識

### イベントストアとは

イベントストアは、イベントを永続化するための専用データストアです。従来のデータベースが「現在の状態」を保存するのに対し、イベントストアは「すべての変更履歴」を保存します。

```
従来のデータベース:
┌─────────────────────────────────────┐
│ accounts                            │
│ ┌─────┬─────────┬─────────────────┐ │
│ │ id  │ balance │ updated_at      │ │
│ ├─────┼─────────┼─────────────────┤ │
│ │ 001 │ 10000   │ 2024-01-15      │ │  ← 現在の状態のみ
│ └─────┴─────────┴─────────────────┘ │
└─────────────────────────────────────┘

イベントストア:
┌─────────────────────────────────────────────────────────┐
│ events                                                   │
│ ┌─────┬────────────────┬────────┬─────────────────────┐ │
│ │ seq │ aggregate_id   │ type   │ data                │ │
│ ├─────┼────────────────┼────────┼─────────────────────┤ │
│ │ 1   │ ACC-001        │ Opened │ {initial: 0}        │ │
│ │ 2   │ ACC-001        │ Deposit│ {amount: 5000}      │ │
│ │ 3   │ ACC-001        │ Deposit│ {amount: 8000}      │ │
│ │ 4   │ ACC-001        │ Withdraw│{amount: 3000}      │ │  ← 全履歴
│ └─────┴────────────────┴────────┴─────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### イベントストアの特性

| 特性 | 説明 |
|------|------|
| **追記のみ** | イベントは追加のみ。更新・削除は行わない |
| **不変性** | 保存されたイベントは変更されない |
| **時系列順序** | イベントは発生順に保存される |
| **集約単位** | 同じ集約のイベントをまとめて取得できる |

### なぜイベントストアが必要か

1. **完全な監査証跡**: すべての変更履歴を追跡
2. **時間旅行**: 任意の時点の状態を再構築
3. **デバッグ**: 問題発生時の原因特定が容易
4. **イベントリプレイ**: 新しいプロジェクションの構築

## 概念の説明

### イベントストアのインターフェース

```rust
trait EventStore {
    // イベントを追加
    fn append(&mut self, events: Vec<Event>) -> Result<(), Error>;

    // 集約のイベントを取得
    fn get_events(&self, aggregate_id: &str) -> Vec<Event>;

    // 特定バージョン以降のイベントを取得
    fn get_events_since(&self, aggregate_id: &str, version: u64) -> Vec<Event>;

    // 全イベントを取得（プロジェクション構築用）
    fn get_all_events(&self) -> Vec<Event>;
}
```

### バージョニングと楽観的ロック

並行アクセス時の競合を防ぐため、楽観的ロックを使用します：

```
┌─────────────────────────────────────────────────────────────────┐
│                    楽観的ロックの流れ                            │
│                                                                 │
│  Client A                    Event Store                        │
│     │                            │                              │
│     │  1. Get events (v=3)       │                              │
│     │ ─────────────────────────→ │                              │
│     │ ←───────────────────────── │                              │
│     │                            │                              │
│     │  2. Append (expected v=3)  │                              │
│     │ ─────────────────────────→ │  ✓ Success (v=4)            │
│     │ ←───────────────────────── │                              │
│                                  │                              │
│  Client B                        │                              │
│     │                            │                              │
│     │  3. Append (expected v=3)  │                              │
│     │ ─────────────────────────→ │  ✗ Conflict! (current v=4)  │
│     │ ←───────────────────────── │                              │
│     │                            │                              │
│     │  4. Retry: Get events      │                              │
│     │ ─────────────────────────→ │                              │
└─────────────────────────────────────────────────────────────────┘
```

### ストリームの概念

イベントストアでは、関連するイベントを「ストリーム」としてグループ化します：

```
┌─────────────────────────────────────────────────────────────────┐
│                        Event Streams                            │
│                                                                 │
│  Stream: order-001                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ [OrderPlaced] → [OrderPaid] → [OrderShipped]             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Stream: order-002                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ [OrderPlaced] → [OrderCancelled]                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Stream: $all (全イベント)                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ [1:OrderPlaced] → [2:OrderPlaced] → [3:OrderPaid] → ...  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 実装タスク

### タスク1: 基本的なインメモリイベントストアを実装

イベントを保存・取得できる基本的なイベントストアを実装してください。

**要件:**
- `append()`でイベントを追加
- `get_events()`で集約のイベントを取得
- グローバルなシーケンス番号を付与

### タスク2: 楽観的ロックを実装

並行アクセス時の競合を検出する楽観的ロックを実装してください。

**要件:**
- 期待するバージョンを指定してappend
- バージョン不一致時はエラーを返す
- 現在のバージョンを取得するメソッド

### タスク3: イベントのフィルタリングと検索

特定の条件でイベントを検索する機能を実装してください。

**要件:**
- イベントタイプでフィルタリング
- 時間範囲でフィルタリング
- ページネーション対応

## ヒント

<details>
<summary>タスク1のヒント</summary>

```rust
use std::collections::HashMap;

struct StoredEvent {
    sequence: u64,
    aggregate_id: String,
    event_type: String,
    data: String,  // JSON
    timestamp: DateTime<Utc>,
}

struct InMemoryEventStore {
    events: Vec<StoredEvent>,
    streams: HashMap<String, Vec<usize>>,  // aggregate_id -> event indices
    next_sequence: u64,
}
```

</details>

<details>
<summary>タスク2のヒント</summary>

```rust
#[derive(Debug)]
enum EventStoreError {
    ConcurrencyConflict {
        expected: u64,
        actual: u64,
    },
}

fn append(
    &mut self,
    aggregate_id: &str,
    events: Vec<Event>,
    expected_version: u64,
) -> Result<u64, EventStoreError> {
    let current_version = self.get_version(aggregate_id);
    if current_version != expected_version {
        return Err(EventStoreError::ConcurrencyConflict {
            expected: expected_version,
            actual: current_version,
        });
    }
    // イベントを追加...
}
```

</details>

<details>
<summary>タスク3のヒント</summary>

```rust
struct EventQuery {
    aggregate_id: Option<String>,
    event_types: Option<Vec<String>>,
    from_time: Option<DateTime<Utc>>,
    to_time: Option<DateTime<Utc>>,
    limit: usize,
    offset: usize,
}

fn query(&self, query: EventQuery) -> Vec<StoredEvent> {
    self.events.iter()
        .filter(|e| /* フィルタ条件 */)
        .skip(query.offset)
        .take(query.limit)
        .cloned()
        .collect()
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
// エラー型
// ============================================================

#[derive(Debug, Clone)]
pub enum EventStoreError {
    ConcurrencyConflict { expected: u64, actual: u64 },
    StreamNotFound(String),
    SerializationError(String),
}

impl std::fmt::Display for EventStoreError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::ConcurrencyConflict { expected, actual } => {
                write!(f, "Concurrency conflict: expected version {}, but was {}", expected, actual)
            }
            Self::StreamNotFound(id) => write!(f, "Stream not found: {}", id),
            Self::SerializationError(msg) => write!(f, "Serialization error: {}", msg),
        }
    }
}

impl std::error::Error for EventStoreError {}

// ============================================================
// 保存されるイベント
// ============================================================

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoredEvent {
    /// グローバルシーケンス番号
    pub sequence: u64,
    /// ストリーム内のバージョン
    pub version: u64,
    /// イベントID
    pub event_id: Uuid,
    /// 集約ID（ストリームID）
    pub aggregate_id: String,
    /// イベントタイプ
    pub event_type: String,
    /// イベントデータ（JSON）
    pub data: String,
    /// メタデータ（JSON）
    pub metadata: String,
    /// タイムスタンプ
    pub timestamp: DateTime<Utc>,
}

// ============================================================
// クエリ
// ============================================================

#[derive(Debug, Default)]
pub struct EventQuery {
    pub aggregate_id: Option<String>,
    pub event_types: Option<Vec<String>>,
    pub from_time: Option<DateTime<Utc>>,
    pub to_time: Option<DateTime<Utc>>,
    pub from_sequence: Option<u64>,
    pub limit: Option<usize>,
    pub offset: usize,
}

impl EventQuery {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn for_aggregate(mut self, id: impl Into<String>) -> Self {
        self.aggregate_id = Some(id.into());
        self
    }

    pub fn with_types(mut self, types: Vec<String>) -> Self {
        self.event_types = Some(types);
        self
    }

    pub fn from_time(mut self, time: DateTime<Utc>) -> Self {
        self.from_time = Some(time);
        self
    }

    pub fn to_time(mut self, time: DateTime<Utc>) -> Self {
        self.to_time = Some(time);
        self
    }

    pub fn limit(mut self, limit: usize) -> Self {
        self.limit = Some(limit);
        self
    }

    pub fn offset(mut self, offset: usize) -> Self {
        self.offset = offset;
        self
    }
}

// ============================================================
// イベントストアトレイト
// ============================================================

pub trait EventStore: Send + Sync {
    fn append(
        &self,
        aggregate_id: &str,
        events: Vec<(String, String)>,  // (event_type, data)
        expected_version: Option<u64>,
    ) -> Result<u64, EventStoreError>;

    fn get_events(&self, aggregate_id: &str) -> Vec<StoredEvent>;

    fn get_events_since(&self, aggregate_id: &str, version: u64) -> Vec<StoredEvent>;

    fn get_version(&self, aggregate_id: &str) -> u64;

    fn query(&self, query: EventQuery) -> Vec<StoredEvent>;

    fn get_all_events(&self) -> Vec<StoredEvent>;
}

// ============================================================
// インメモリ実装
// ============================================================

#[derive(Debug)]
struct InMemoryEventStoreInner {
    events: Vec<StoredEvent>,
    streams: HashMap<String, Vec<usize>>,
    next_sequence: u64,
}

#[derive(Debug, Clone)]
pub struct InMemoryEventStore {
    inner: Arc<RwLock<InMemoryEventStoreInner>>,
}

impl InMemoryEventStore {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(RwLock::new(InMemoryEventStoreInner {
                events: Vec::new(),
                streams: HashMap::new(),
                next_sequence: 1,
            })),
        }
    }

    pub fn event_count(&self) -> usize {
        self.inner.read().unwrap().events.len()
    }

    pub fn stream_count(&self) -> usize {
        self.inner.read().unwrap().streams.len()
    }
}

impl Default for InMemoryEventStore {
    fn default() -> Self {
        Self::new()
    }
}

impl EventStore for InMemoryEventStore {
    fn append(
        &self,
        aggregate_id: &str,
        events: Vec<(String, String)>,
        expected_version: Option<u64>,
    ) -> Result<u64, EventStoreError> {
        let mut inner = self.inner.write().unwrap();

        // 楽観的ロックのチェック
        let current_version = inner
            .streams
            .get(aggregate_id)
            .map(|indices| indices.len() as u64)
            .unwrap_or(0);

        if let Some(expected) = expected_version {
            if current_version != expected {
                return Err(EventStoreError::ConcurrencyConflict {
                    expected,
                    actual: current_version,
                });
            }
        }

        // イベントを追加
        let stream_indices = inner
            .streams
            .entry(aggregate_id.to_string())
            .or_insert_with(Vec::new);

        let mut new_version = current_version;

        for (event_type, data) in events {
            new_version += 1;
            let sequence = inner.next_sequence;
            inner.next_sequence += 1;

            let stored_event = StoredEvent {
                sequence,
                version: new_version,
                event_id: Uuid::new_v4(),
                aggregate_id: aggregate_id.to_string(),
                event_type,
                data,
                metadata: "{}".to_string(),
                timestamp: Utc::now(),
            };

            let index = inner.events.len();
            inner.events.push(stored_event);
            stream_indices.push(index);
        }

        Ok(new_version)
    }

    fn get_events(&self, aggregate_id: &str) -> Vec<StoredEvent> {
        let inner = self.inner.read().unwrap();

        inner
            .streams
            .get(aggregate_id)
            .map(|indices| {
                indices
                    .iter()
                    .map(|&i| inner.events[i].clone())
                    .collect()
            })
            .unwrap_or_default()
    }

    fn get_events_since(&self, aggregate_id: &str, version: u64) -> Vec<StoredEvent> {
        self.get_events(aggregate_id)
            .into_iter()
            .filter(|e| e.version > version)
            .collect()
    }

    fn get_version(&self, aggregate_id: &str) -> u64 {
        let inner = self.inner.read().unwrap();
        inner
            .streams
            .get(aggregate_id)
            .map(|indices| indices.len() as u64)
            .unwrap_or(0)
    }

    fn query(&self, query: EventQuery) -> Vec<StoredEvent> {
        let inner = self.inner.read().unwrap();

        let iter: Box<dyn Iterator<Item = &StoredEvent>> = if let Some(ref agg_id) = query.aggregate_id {
            if let Some(indices) = inner.streams.get(agg_id) {
                Box::new(indices.iter().map(|&i| &inner.events[i]))
            } else {
                return Vec::new();
            }
        } else {
            Box::new(inner.events.iter())
        };

        iter.filter(|e| {
            // イベントタイプフィルタ
            if let Some(ref types) = query.event_types {
                if !types.contains(&e.event_type) {
                    return false;
                }
            }
            // 時間範囲フィルタ
            if let Some(from) = query.from_time {
                if e.timestamp < from {
                    return false;
                }
            }
            if let Some(to) = query.to_time {
                if e.timestamp > to {
                    return false;
                }
            }
            // シーケンスフィルタ
            if let Some(from_seq) = query.from_sequence {
                if e.sequence < from_seq {
                    return false;
                }
            }
            true
        })
        .skip(query.offset)
        .take(query.limit.unwrap_or(usize::MAX))
        .cloned()
        .collect()
    }

    fn get_all_events(&self) -> Vec<StoredEvent> {
        self.inner.read().unwrap().events.clone()
    }
}

// ============================================================
// 使用例
// ============================================================

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== Event Store Demo ===\n");

    let store = InMemoryEventStore::new();

    // --- 基本的な追加と取得 ---
    println!("--- Basic Append and Get ---\n");

    let order_id = "order-001";

    // イベントを追加（バージョン0から開始）
    let v1 = store.append(
        order_id,
        vec![("OrderPlaced".to_string(), r#"{"customer_id":"C001","amount":15000}"#.to_string())],
        Some(0),
    )?;
    println!("After OrderPlaced: version = {}", v1);

    let v2 = store.append(
        order_id,
        vec![("OrderPaid".to_string(), r#"{"payment_id":"P001"}"#.to_string())],
        Some(v1),
    )?;
    println!("After OrderPaid: version = {}", v2);

    // イベントを取得
    let events = store.get_events(order_id);
    println!("\nEvents for {}:", order_id);
    for event in &events {
        println!("  [{}] {} (v{}): {}", event.sequence, event.event_type, event.version, event.data);
    }

    // --- 楽観的ロックのテスト ---
    println!("\n--- Optimistic Locking Test ---\n");

    // 古いバージョンで追加を試みる
    let result = store.append(
        order_id,
        vec![("OrderShipped".to_string(), r#"{"tracking":"T001"}"#.to_string())],
        Some(1),  // 期待: v1、実際: v2
    );

    match result {
        Ok(_) => println!("Unexpected success"),
        Err(EventStoreError::ConcurrencyConflict { expected, actual }) => {
            println!("Concurrency conflict detected!");
            println!("  Expected version: {}", expected);
            println!("  Actual version: {}", actual);
        }
        Err(e) => println!("Other error: {}", e),
    }

    // 正しいバージョンで再試行
    let v3 = store.append(
        order_id,
        vec![("OrderShipped".to_string(), r#"{"tracking":"T001"}"#.to_string())],
        Some(v2),
    )?;
    println!("Retry succeeded: version = {}", v3);

    // --- クエリのテスト ---
    println!("\n--- Query Test ---\n");

    // 別の注文を追加
    store.append(
        "order-002",
        vec![
            ("OrderPlaced".to_string(), r#"{"customer_id":"C002","amount":8000}"#.to_string()),
            ("OrderCancelled".to_string(), r#"{"reason":"Customer request"}"#.to_string()),
        ],
        None,
    )?;

    // 全イベントを取得
    println!("All events:");
    for event in store.get_all_events() {
        println!("  [{}] {}/{}: {}", event.sequence, event.aggregate_id, event.event_type, event.data);
    }

    // OrderPlacedイベントのみ取得
    println!("\nOrderPlaced events only:");
    let placed_events = store.query(
        EventQuery::new().with_types(vec!["OrderPlaced".to_string()])
    );
    for event in placed_events {
        println!("  {}: {}", event.aggregate_id, event.data);
    }

    println!("\n=== Demo Complete ===");
    println!("Total events: {}", store.event_count());
    println!("Total streams: {}", store.stream_count());

    Ok(())
}
```

### コード解説

#### RwLockによる並行アクセス制御

```rust
inner: Arc<RwLock<InMemoryEventStoreInner>>
```

- `Arc`: 複数スレッドで共有
- `RwLock`: 読み取りは並行、書き込みは排他

#### 楽観的ロックの実装

```rust
if let Some(expected) = expected_version {
    if current_version != expected {
        return Err(EventStoreError::ConcurrencyConflict { ... });
    }
}
```

期待するバージョンと実際のバージョンが一致しない場合、エラーを返します。

## 発展課題

### 課題1: スナップショット機能

大量のイベントがある場合のパフォーマンス改善のため、スナップショット機能を追加してください。

### 課題2: イベントの圧縮

古いイベントを圧縮して保存する機能を実装してください。

### 課題3: ファイルベースの永続化

イベントをファイルに永続化し、再起動後も復元できるようにしてください。

## よくある間違い

### ❌ 間違い1: イベントを更新・削除する

```rust
// 悪い例: イベントを更新
fn update_event(&mut self, id: Uuid, new_data: String) {
    // イベントストアでは更新は禁止！
}

// 良い例: 補正イベントを追加
fn append_correction(&mut self, original_id: Uuid, correction: String) {
    self.append("OrderAmountCorrected", correction);
}
```

### ❌ 間違い2: バージョンチェックを省略

```rust
// 悪い例: バージョンチェックなし
store.append(aggregate_id, events, None);  // 競合を検出できない

// 良い例: 常にバージョンを指定
let version = store.get_version(aggregate_id);
store.append(aggregate_id, events, Some(version))?;
```

### ❌ 間違い3: 大きすぎるイベントデータ

```rust
// 悪い例: バイナリデータを直接保存
struct FileUploaded {
    file_content: Vec<u8>,  // 数MBのデータ
}

// 良い例: 参照のみ保存
struct FileUploaded {
    file_id: String,
    storage_path: String,
}
```

## まとめ

この章では、イベントストアの設計と実装を学びました。

### 学んだこと

1. **イベントストアの役割**: 追記のみ、不変、時系列順序
2. **インメモリ実装**: HashMap + Vecによるストリーム管理
3. **楽観的ロック**: バージョンによる並行制御
4. **クエリ機能**: フィルタリングとページネーション

### 次の章への準備

次の章では、イベントストアを活用したイベントソーシングパターンを学びます。イベントから状態を再構築する方法を習得します。

## 参考文献

- [Event Store Documentation](https://www.eventstore.com/docs/)
- [Martin Fowler - Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Greg Young - CQRS and Event Sourcing](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)

---

[← 前の章: 非同期イベント処理](./03-async-events.md) | [次の章: イベントソーシング →](./05-event-sourcing.md)
