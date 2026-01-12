# Message Filter

## 概念図

```
         ┌─────────────────────────────────┐
         │        Message Filter           │
         │                                 │
         │           ┌─────┐               │
         │           │ ▼   │               │
         │           │ フィ │               │
         │           │ ル  │               │
         │           │ タ  │               │
         │           │ ー  │               │
         │           └──┬──┘               │
         │              │                  │
         └──────────────┼──────────────────┘
                        ▼
                  不要メッセージを
                     破棄
```

---

## 定義

Message Filterは、システムが関心のないメッセージや互換性のないメッセージを受信する可能性がある場合に、それらの不要なメッセージを破棄するために使用するパターンである。

---

## Content-Based Router との比較

| 観点 | Content-Based Router | Message Filter |
|-----|---------------------|----------------|
| **配置場所** | 送信システム側またはハブ | 受信システム側（ターゲットシステム） |
| **動作** | メッセージタイプに基づいて特定システムにルーティング | 処理目標と互換性のないメッセージを除外 |
| **知識** | 送信側がルーティング先を把握 | 受信側は送信側の知識がない、または古い知識しかない |
| **結果** | ターゲットシステムに互換性のないメッセージは届かない | 互換性のあるメッセージのみコア処理に転送 |

### Content-Based Router の特徴
- 特定システムが特定メッセージタイプをサポートする場合、そのタイプのメッセージはそのシステムにルーティングされる
- ルーターは送信システムの一部として配置されるか、実際の宛先システムへのプロキシとして機能するハブとして存在する
- Content-Based Routerを使用する場合、ターゲットシステムに処理目標と互換性のないメッセージが送信されることはない

### Message Filter の特徴
- 受信システムが処理目標と互換性のないメッセージを受け取る可能性がある（送信システムが知識を持たない、または古い知識しか持たないため）
- ターゲットシステムはコアビジネスプロセスを実行する前に、互換性のないメッセージをフィルタリングする必要がある
- Message Filterはターゲットシステムに見えるが、実際にはコア処理を管理するアクターへのプロキシに過ぎない

---

## 具体例：注文システム

Content-Based Router (228) で議論した `OrderPlaced` イベントを `InventorySystemA` と `InventorySystemX` に送信するケースを考える。

### Message Filter例の設定
Content-Based Routerとは異なり、特定のメッセージタイプを特定の在庫システムにルーティングするのではなく、`"TypeABC"` と `"TypeXYZ"` の両方の注文を `InventorySystemA` と `InventorySystemX` の**両方**に送信する。

### システム構成図

```
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ OrderPlaced │ │ OrderPlaced │ │ OrderPlaced │ │ OrderPlaced │
│   TypeABC   │ │   TypeABC   │ │   TypeXYZ   │ │   TypeABC   │
│   ┌───┐     │ │   ┌───┐     │ │   ┌───┐     │ │   ┌───┐     │
│   │ ⚡│     │ │   │ ⚡│     │ │   │ ⚡│     │ │   │ ⚡│     │
│   └───┘     │ │   └───┘     │ │   └───┘     │ │   └───┘     │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
       │               │               │               │
       └───────────────┴───────┬───────┴───────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    OrderFilter      │
                    │      ┌─────┐        │
                    │      │  ▼  │        │
                    │      └──┬──┘        │
                    └─────────┼───────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
    ┌─────────────────────┐       ┌─────────────────────┐
    │  InventorySystemA   │       │  InventorySystemA   │
    │     ┌───┬───┐       │       │     ┌───┬───┐       │
    │     │ ⚡│ ⚡│       │       │     │ ⚡│ ⚡│       │
    │     └───┴───┘       │       │     └───┴───┘       │
    └─────────────────────┘       └─────────────────────┘

    ※ 各在庫システムが自身でフィルタリングを行う
    ※ TypeABC → InventorySystemA が処理
    ※ TypeXYZ → InventorySystemX が処理
    ※ 互換性のないタイプは各システムで破棄
```

---

## 実装例（Scala/Akka）

### ドライバアプリケーション

```scala
object MessageFilter extends CompletableApp (4) {
  // InventorySystemA（フィルタ内蔵型）
  val inventorySystemA =
          system.actorOf(
            Props[InventorySystemA],
            "inventorySystemA")

  // 実際のInventorySystemX
  val actualInventorySystemX =
          system.actorOf(
            Props[InventorySystemX],
            "inventorySystemX")

  // InventorySystemX用のMessage Filter（別アクター）
  val inventorySystemX =
      system.actorOf(
          Props(classOf[InventorySystemXMessageFilter],
              actualInventorySystemX),
          "inventorySystemXMessageFilter")

  // TypeABC の注文アイテム
  val orderItem1 = OrderItem("1", "TypeABC.4",
                    "An item of type ABC.4.", 29.95)
  val orderItem2 = OrderItem("2", "TypeABC.1",
                    "An item of type ABC.1.", 99.95)
  val orderItem3 = OrderItem("3", "TypeABC.9",
                    "An item of type ABC.9.", 14.95)

  val orderItemsOfTypeA = Map(
          orderItem1.itemType -> orderItem1,
          orderItem2.itemType -> orderItem2,
          orderItem3.itemType -> orderItem3)

  // TypeABC注文を両方の在庫システムに送信
  inventorySystemA ! OrderPlaced(Order("123", "TypeABC",
                          orderItemsOfTypeA))
  inventorySystemX ! OrderPlaced(Order("123", "TypeABC",
                          orderItemsOfTypeA))

  // TypeXYZ の注文アイテム
  val orderItem4 = OrderItem("4", "TypeXYZ.2",
                    "An item of type XYZ.2.", 74.95)
  val orderItem5 = OrderItem("5", "TypeXYZ.1",
                    "An item of type XYZ.1.", 59.95)
  val orderItem6 = OrderItem("6", "TypeXYZ.7",
                    "An item of type XYZ.7.", 29.95)
  val orderItem7 = OrderItem("7", "TypeXYZ.5",
                    "An item of type XYZ.5.", 9.95)

  val orderItemsOfTypeX = Map(
          orderItem4.itemType -> orderItem4,
          orderItem5.itemType -> orderItem5,
          orderItem6.itemType -> orderItem6,
          orderItem7.itemType -> orderItem7)

  // TypeXYZ注文を両方の在庫システムに送信
  inventorySystemA ! OrderPlaced(Order("124", "TypeXYZ",
              orderItemsOfTypeX))
  inventorySystemX ! OrderPlaced(Order("124", "TypeXYZ",
              orderItemsOfTypeX))

  awaitCompletion
  println("MessageFilter: is completed.")
}
```

---

## 2つの実装アプローチ

両方の在庫システムが両方のタイプのメッセージを受信するため、各システムがサポートしないメッセージタイプをフィルタリングする責任を持つ。各在庫システムは異なるアプローチを取る。

### アプローチ1：フィルタ内蔵型（InventorySystemA）

アクターの `receive` ブロックの実装方法を利用して、Message Filterを `InventorySystemA` アクター自体に設計する。

```scala
class InventorySystemA extends Actor {
  def receive = {
    // ガード条件でTypeABCのみ処理
    case OrderPlaced(order) if (order.isType("TypeABC")) =>
      println(s"InventorySystemA: handling $order")
      MessageFilter.completedStep()

    // 互換性のない注文はフィルタリング
    case incompatibleOrder =>
      println(s"InventorySystemA: filtering out:↩
      $incompatibleOrder")
      MessageFilter.completedStep()
  }
}
```

### アプローチ2：別アクター型（InventorySystemX）

Message Filterを別のアクターとして実装する。

```scala
// 実際の在庫システムX
class InventorySystemX extends Actor {
  def receive = {
    case OrderPlaced(order) =>
      println(s"InventorySystemX: handling $order")
      MessageFilter.completedStep()
    case _ =>
      println("InventorySystemX: unexpected message")
      MessageFilter.completedStep()
  }
}

// InventorySystemX用のMessage Filter
class InventorySystemXMessageFilter(
          actualInventorySystemX: ActorRef)
  extends Actor {
  def receive = {
    // TypeXYZのみ実際のシステムに転送
    case orderPlaced: OrderPlaced
        if (orderPlaced.order.isType("TypeXYZ")) =>
      actualInventorySystemX forward orderPlaced
      MessageFilter.completedStep()

    // 互換性のない注文はフィルタリング
    case incompatibleOrder =>
      println(s"InventorySystemXMessageFilter: filtering:↩
      $incompatibleOrder")
      MessageFilter.completedStep()
  }
}
```

### ドライバから見た構成

```scala
object MessageFilter extends CompletableApp (4) {
  ...
  // 実際のInventorySystemX
  val actualInventorySystemX =
          system.actorOf(
            Props[InventorySystemX],
            "inventorySystemX")

  // Message Filter（ドライバからはInventorySystemXとして参照）
  val inventorySystemX =
          system.actorOf(
            Props(classOf[InventorySystemXMessageFilter],
                actualInventorySystemX),
            "inventorySystemXMessageFilter")
  ...
}
```

ドライバアプリケーションから見ると、`InventorySystemXMessageFilter` は `InventorySystemX` として参照される（`inventorySystemX` という名前で参照）。

実際には、ドライバの `InventorySystemX` アクター参照が保持するMessage Filterは、システム互換のメッセージの転送とその他すべてのフィルタリングのみに関心を持つ。`"TypeXYZ"` の `Order` を含む `OrderPlaced` イベントを受信すると、Message Filterは `actualInventorySystemX` が参照するアクター（実際の在庫システムのエントリポイント）にイベントを転送する。

---

## 処理フロー図

```
┌─────────────────────────────────────────────────────────────────┐
│           アプローチ1: フィルタ内蔵型 (InventorySystemA)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OrderPlaced 受信                                               │
│        │                                                        │
│        ▼                                                        │
│  ┌──────────────────┐                                           │
│  │ order.isType     │                                           │
│  │  ("TypeABC") ?   │                                           │
│  └────────┬─────────┘                                           │
│      ┌────┴────┐                                                │
│      │         │                                                │
│     Yes        No                                               │
│      │         │                                                │
│      ▼         ▼                                                │
│   処理実行   フィルタリング                                        │
│              (破棄)                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           アプローチ2: 別アクター型 (InventorySystemX)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         InventorySystemXMessageFilter (プロキシ)          │   │
│  │                                                          │   │
│  │  OrderPlaced 受信                                        │   │
│  │        │                                                 │   │
│  │        ▼                                                 │   │
│  │  ┌──────────────────┐                                    │   │
│  │  │ order.isType     │                                    │   │
│  │  │  ("TypeXYZ") ?   │                                    │   │
│  │  └────────┬─────────┘                                    │   │
│  │      ┌────┴────┐                                         │   │
│  │      │         │                                         │   │
│  │     Yes        No                                        │   │
│  │      │         │                                         │   │
│  │      ▼         ▼                                         │   │
│  │   forward   フィルタリング                                 │   │
│  │      │       (破棄)                                      │   │
│  └──────┼───────────────────────────────────────────────────┘   │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              InventorySystemX (実際のシステム)             │   │
│  │                                                          │   │
│  │  OrderPlaced 受信 → 処理実行                              │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 実行結果

```
InventorySystemA: handling Order(123, TypeABC,↩
 Map(TypeABC.4 -> OrderItem(1, TypeABC.4, 'An item↩
 of type ABC.4.', 29.95), TypeABC.1 -> OrderItem(2,↩
TypeABC.1, 'An item of type ABC.1.', 99.95),↩
TypeABC.9 -> OrderItem(3, TypeABC.9, 'An item↩
 of type ABC.9.', 14.95)), Totaling: 144.85))

InventorySystemXMessageFilter: filtering: OrderPlaced(↩
Order(123, TypeABC, Map(TypeABC.4 -> OrderItem(1,↩
 TypeABC.4, 'An item of type ABC.4.', 29.95), TypeABC.1↩
 -> OrderItem(2, TypeABC.1, 'An item of type ABC.1.',↩
 99.95), TypeABC.9 -> OrderItem(3, TypeABC.9, 'An item↩
 of type ABC.9.', 14.95)), Totaling: 144.85))

InventorySystemA: filtering: OrderPlaced(Order(124,↩
 TypeXYZ, Map(TypeXYZ.2 -> OrderItem(4, TypeXYZ.2,↩
 'An item of type XYZ.2.', 74.95), TypeXYZ.1 ->↩
OrderItem(5, TypeXYZ.1, 'An item of type XYZ.1.',↩
 59.95), TypeXYZ.7 -> OrderItem(6, TypeXYZ.7, 'An↩
 item of type XYZ.7.', 29.95), TypeXYZ.5 -> OrderItem(↩
7, TypeXYZ.5, 'An item of type XYZ.5.', 9.95)), Totaling:↩
 174.79999999999998))

InventorySystemX: handling Order(124, TypeXYZ,↩
 Map(TypeXYZ.2 -> OrderItem(4, TypeXYZ.2, 'An item↩
 of type XYZ.2.', 74.95), TypeXYZ.1 -> OrderItem(5,↩
TypeXYZ.1, 'An item of type XYZ.1.', 59.95), TypeXYZ.7↩
 -> OrderItem(6, TypeXYZ.7, 'An item of type XYZ.7.',↩
 29.95), TypeXYZ.5 -> OrderItem(7, TypeXYZ.5, 'An item↩
 of type XYZ.5.', 9.95)), Totaling: 174.79999999999998)

MessageFilter: is completed.
```

---

## 2つのアプローチの比較

| 観点 | フィルタ内蔵型 (InventorySystemA) | 別アクター型 (InventorySystemX) |
|-----|--------------------------------|-------------------------------|
| **保守性** | 新しいメッセージタイプのサポート追加や既存タイプのサポート終了時にアクター変更が必要 | Message Filterを `InventorySystemX` 自体とは別に保守できる |
| **オーバーヘッド** | 追加アクターなし | フィルタリング用の別アクター導入による若干のオーバーヘッド |
| **アーキテクチャ** | シンプル | Pipes and Filters (135) アーキテクチャの強みを活用 |

### 別アクター型の利点
`InventorySystemXMessageFilter` を別に実装する主な利点は、`InventorySystemX` 自体とは別に保守できることである。対照的に、`InventorySystemA` アクターは新しいメッセージタイプがサポートされるたび、または以前サポートされていたメッセージタイプが互換性がなくなるたびに変更する必要がある。

### 別アクター型の欠点
`InventorySystemX` の設計の欠点は、フィルタリング用に別のアクターを導入することによる若干のオーバーヘッドがあることだが、オーバーヘッドは最小限である。

### 推奨
**Pipes and Filters** (135) アーキテクチャの強みを活かすため、`InventorySystemX` とその Message Filter のアーキテクチャを選択することが推奨される。


