# Content-Based Router

## 概念図

```
    ┌─────────────────────────────────────────────────────────────┐
    │                   Content-Based Router                      │
    │  ┌─────────┐                                                │
    │  │ Message │──┐      ┌──────────────┐      ┌──────────────┐ │
    │  └─────────┘  │      │              │      │  Channel A   │ │
    │               ├─────▶│    Router    │─────▶│              │ │
    │  ┌─────────┐  │      │   (内容判定)  │      └──────────────┘ │
    │  │ Message │──┘      │              │      ┌──────────────┐ │
    │  └─────────┘         │              │─────▶│  Channel B   │ │
    │                      └──────────────┘      └──────────────┘ │
    └─────────────────────────────────────────────────────────────┘
```

---

## 定義

Content-Based Routerは、メッセージの内容を分析し、その内容に基づいて適切な出力チャネルにメッセージ全体をルーティングするパターンである。

---

## 関連パターンとの比較

| パターン | 動作 | 目的 |
|---------|------|------|
| **Content-Based Router** | メッセージ全体を内容に基づいてルーティング | 互換性のあるシステムへメッセージを振り分ける |
| **Splitter** | 1つのメッセージを複数のメッセージに分割 | 複合メッセージの構成部品を分離 |
| **Message Filter** | 不要なメッセージをチャネルから除去 | 消費できないメッセージを排除 |

### Splitterとの違い
両者ともメッセージ内容に基づいてルーティングするが、Splitterは1つのメッセージの構成部品を複数メッセージに分解する。Content-Based Routerはメッセージを分解せず、全体をそのままルーティングする。

### Message Filterとの違い
Message Filterは受信側Message Channelから不要メッセージを除去する。Content-Based Routerは不要メッセージが互換性のないシステムに到達しないことを保証しつつ、すべてのメッセージを互換性のあるシステムにルーティングする。

---

## 具体例：注文システム

Enterprise Integration Patterns [EIP] の例として、注文システムが在庫確認のために各注文を在庫システムにルーティングするケースがある。

### 前提条件
- 注文は1つまたは2つ以上の在庫システムのいずれかで管理される商品のみを含む
- 1つの注文に複数の在庫システムにまたがる商品が含まれる場合はSplitterを使用する

### システム構成図

```
  ┌───────────────┐    ┌───────────────┐
  │ OrderPlaced   │    │ OrderPlaced   │
  │   TypeXYZ     │    │   TypeABC     │
  │   ┌───┐       │    │   ┌───┐       │
  │   │ ⚡│       │    │   │ ⚡│       │
  │   └───┘       │    │   └───┘       │
  └───────┬───────┘    └───────┬───────┘
          │                    │
          └─────────┬──────────┘
                    │
                    ▼
          ┌─────────────────┐
          │   OrderRouter   │
          │  ┌───┬───┬───┐  │
          │  │   │ ⚡│   │  │
          │  ├───┼───┼───┤  │
          │  │   │   │   │  │
          │  └───┴───┴───┘  │
          └────────┬────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
┌───────────────┐     ┌───────────────┐
│InventorySystemA│    │InventorySystemX│
│   ┌───┬───┐   │     │   ┌───┬───┐   │
│   │ ⚡│ ⚡│   │     │   │ ⚡│ ⚡│   │
│   └───┴───┘   │     │   └───┴───┘   │
└───────────────┘     └───────────────┘

※ TypeABC → InventorySystemA
※ TypeXYZ → InventorySystemX
```

---

## 実装例（Scala/Akka）

### データモデル

```scala
package co.vaughnvernon.reactiveenterprise.contentbasedrouter

import scala.collection.Map
import akka.actor._
import co.vaughnvernon.reactiveenterprise._

// 注文アイテム
case class OrderItem(
    id: String, 
    itemType: String,
    description: String, 
    price: Double) {
  override def toString = {
    s"OrderItem($id, $itemType, '$description', $price)"
  }
}

// 注文
case class Order(
    id: String, 
    orderType: String,
    orderItems: Map[String, OrderItem]) {
  val grandTotal: Double =
    orderItems.values.map(orderItem =>
      orderItem.price).sum

  override def toString = {
    s"Order($id, $orderType, $orderItems,↩
    Totaling: $grandTotal)"
  }
}

// 注文確定メッセージ
case class OrderPlaced(order: Order)
```

### Content-Based Router 実装

```scala
object ContentBasedRouter extends CompletableApp(3) {
  val orderRouter = system.actorOf(
            Props[OrderRouter], "orderRouter")

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

  // TypeABC 注文を送信
  orderRouter ! OrderPlaced(Order(
              "123", "TypeABC", orderItemsOfTypeA))

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

  // TypeXYZ 注文を送信
  orderRouter ! OrderPlaced(Order("124", "TypeXYZ",
                      orderItemsOfTypeX))

  awaitCompletion
  println("ContentBasedRouter: is completed.")
}
```

### OrderRouter（ルーティングロジック）

```scala
class OrderRouter extends Actor {
  // 在庫システムAのアクター
  val inventorySystemA =
            context.actorOf(Props[InventorySystemA],
                          "inventorySystemA")
  // 在庫システムXのアクター
  val inventorySystemX =
            context.actorOf(Props[InventorySystemX],
                          "inventorySystemX")

  def receive = {
    case orderPlaced: OrderPlaced =>
      // orderType に基づいてルーティング先を決定
      orderPlaced.order.orderType match {
        case "TypeABC" =>
          println(s"OrderRouter: routing $orderPlaced")
          inventorySystemA ! orderPlaced
        case "TypeXYZ" =>
          println(s"OrderRouter: routing $orderPlaced")
          inventorySystemX ! orderPlaced
      }
      ContentBasedRouter.completedStep()
    case _ =>
      println("OrderRouter: received unexpected message")
  }
}
```

### 在庫システム

```scala
// 在庫システムA
class InventorySystemA extends Actor {
  def receive = {
    case OrderPlaced(order) =>
      println(s"InventorySystemA: handling $order")
      ContentBasedRouter.completedStep()
    case _ =>
      println("InventorySystemA: unexpected message")
  }
}

// 在庫システムX
class InventorySystemX extends Actor {
  def receive = {
    case OrderPlaced(order) =>
      println(s"InventorySystemX: handling $order")
      ContentBasedRouter.completedStep()
    case _ =>
      println("InventorySystemX: unexpected message")
  }
}
```

---

## 実行結果

```
OrderRouter: routing OrderPlaced(Order(123, TypeABC,↩
 Map(TypeABC.4 -> OrderItem(1, TypeABC.4, 'An item of↩
 type ABC.4.', 29.95), TypeABC.1 -> OrderItem(2,↩
TypeABC.1, 'An item of type ABC.1.', 99.95), TypeABC.9↩
 -> OrderItem(3, TypeABC.9, 'An item of type ABC.9.',↩
 14.95)), Totaling: 144.85))
...
ContentBasedRouter: is completed.
InventorySystemX: handling Order(124, TypeXYZ, Map(↩
TypeXYZ.2 -> OrderItem(4, TypeXYZ.2, 'An item of type↩
 XYZ.2.', 74.95), TypeXYZ.1 -> OrderItem(5, TypeXYZ.1,↩
 'An item of type XYZ.1.', 59.95), TypeXYZ.7 -> ↩
OrderItem(6, TypeXYZ.7, 'An item of type XYZ.7.',↩
 29.95), TypeXYZ.5 -> OrderItem(7, TypeXYZ.5, 'An item↩
 of type XYZ.5.', 9.95)), Totaling: 174.80)
```

---

## ルーティングロジックの処理フロー

```
┌─────────────────────────────────────────────────────────────────┐
│                      OrderRouter の動作                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OrderPlaced メッセージ受信                                      │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────┐                                        │
│  │ order.orderType を  │                                        │
│  │      検査           │                                        │
│  └──────────┬──────────┘                                        │
│             │                                                   │
│     ┌───────┴───────┐                                           │
│     │               │                                           │
│     ▼               ▼                                           │
│ "TypeABC"       "TypeXYZ"                                       │
│     │               │                                           │
│     ▼               ▼                                           │
│ inventorySystemA  inventorySystemX                              │
│ へ転送             へ転送                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 設計上の考慮事項

### ルーティング判定基準
この例では `orderType` コンテンツのみを使用してルーティング先を決定しているが、他のコンテンツや追加コンテンツを使用してより詳細なルーティング要件を実現することも可能である。

### ルーティングロジックの配置
`OrderPlaced` メッセージ自体にルーティングを支援する振る舞いを持たせることで、`OrderRouter` が `OrderPlaced` の内部構造を深く知る必要をなくすことも検討できる。ただし、チーム構成や責任範囲によっては、他チームへの依存が発生する可能性がある。

### メッセージの前処理
`OrderPlaced` がドメインオブジェクトの完全なコピーではなく、**Content Filter** (321) や **Content Enricher** (317) によって生成されたものであれば、ルーティングの容易性に貢献する場合がある。
