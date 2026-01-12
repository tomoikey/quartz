# Splitter

## 概念図

```
         ┌─────────────────────────────────────────┐
         │              Splitter                   │
         │                                         │
         │    ┌─────┐      ┌─────┐                │
         │    │     │      │     │────────────────┼───▶ Part A
         │    │ 複合 │      │     │                │
    ────▶│───▶│ MSG │─────▶│ ──▶ │────────────────┼───▶ Part B
         │    │     │      │     │                │
         │    │     │      │     │────────────────┼───▶ Part C
         │    └─────┘      └─────┘                │
         │                                         │
         └─────────────────────────────────────────┘
```

---

## 定義

Splitterは、大きな複合メッセージを個別のパーツに分離し、それぞれを小さなメッセージとして送信する必要がある場合に使用するパターンである。

---

## Content-Based Router との比較

| パターン | 関心事 |
|---------|-------|
| **Splitter** | 単一の複合メッセージの個別パーツを別々のサブシステムにルーティング |
| **Content-Based Router** | メッセージ全体を包括的なメッセージタイプに基づいて特定のサブシステムにルーティング |

Splitterは、パーツのルーティング方法を決定するのがメッセージのパーツ内容であるため、Content-Based Router (228) に似ていると考えられるかもしれない。しかし、Content-Based Routerは主に包括的なメッセージタイプに基づいてメッセージ全体を特定のサブシステムにルーティングすることに関心がある。一方、Splitterは単一の複合メッセージの個別パーツを別々のサブシステムにルーティングすることに関心がある。

---

## 具体例：注文の分割

`OrderPlaced`メッセージを個別の`Type[?]ItemOrdered`メッセージに分割する例を示す。

### システム構成図

```
                                    ┌────────────────────────┐
                                    │ OrderItemTypeAProcessor│
                              ┌────▶│                        │
                              │     │ TypeAItemOrdered       │
                              │     └────────────────────────┘
┌──────────────┐              │
│  OrderPlaced │              │     ┌────────────────────────┐
│  ┌─────────┐ │   ┌────────┐ │     │ OrderItemTypeBProcessor│
│  │ TypeA   │ │   │        │ ├────▶│                        │
│  │ TypeB   │ │──▶│ Order  │─┤     │ TypeBItemOrdered       │
│  │ TypeC   │ │   │ Router │ │     └────────────────────────┘
│  └─────────┘ │   │        │ │
└──────────────┘   └────────┘ │     ┌────────────────────────┐
                              │     │ OrderItemTypeCProcessor│
                              └────▶│                        │
                                    │ TypeCItemOrdered       │
                                    └────────────────────────┘
```

---

## 実装例（Scala/Akka）

### メッセージ定義

```scala
package co.vaughnvernon.reactiveenterprise.splitter

import scala.collection.Map
import akka.actor._
import co.vaughnvernon.reactiveenterprise._

// 注文アイテム
case class OrderItem(
        id: String,
        itemType: String,
        description: String,
        price: Money) {
  override def toString = {
    s"OrderItem($id, $itemType, '$description', $price)"
  }
}

// 注文（複合メッセージ）
case class Order(orderItems: Map[String, OrderItem]) {
  val grandTotal: Double =
        orderItems.values.map(_.price).sum

  override def toString = {
    s"Order(Order Items: $orderItems Totaling:↩
    $grandTotal)"
  }
}

// 注文確定メッセージ（複合メッセージ）
case class OrderPlaced(order: Order)

// 分割後のメッセージ
case class TypeAItemOrdered(orderItem: OrderItem)
case class TypeBItemOrdered(orderItem: OrderItem)
case class TypeCItemOrdered(orderItem: OrderItem)
```

### ドライバアプリケーション

```scala
object Splitter extends CompletableApp(4) {
  val orderRouter =
          system.actorOf(
            Props[OrderRouter],
            "orderRouter")

  // 異なるタイプのアイテムを含む注文を作成
  val orderItem1 = OrderItem("1", "TypeA",
                    "An item of type A.", 23.95)
  val orderItem2 = OrderItem("2", "TypeB",
                    "An item of type B.", 99.95)
  val orderItem3 = OrderItem("3", "TypeC",
                    "An item of type C.", 14.95)

  val orderItems = Map(
      orderItem1.itemType -> orderItem1,
      orderItem2.itemType -> orderItem2,
      orderItem3.itemType -> orderItem3)

  // 複合メッセージを送信
  orderRouter ! OrderPlaced(Order(orderItems))

  awaitCompletion
  println("Splitter: is completed.")
}
```

---

## OrderRouter（Splitter実装）

```scala
class OrderRouter extends Actor {
  // タイプ別プロセッサの作成
  val orderItemTypeAProcessor = context.actorOf(
            Props[OrderItemTypeAProcessor],
            "orderItemTypeAProcessor")
  val orderItemTypeBProcessor = context.actorOf(
            Props[OrderItemTypeBProcessor],
            "orderItemTypeBProcessor")
  val orderItemTypeCProcessor = context.actorOf(
            Props[OrderItemTypeCProcessor],
            "orderItemTypeCProcessor")

  def receive = {
    case OrderPlaced(order) =>
      println(order)

      // 各アイテムを走査して分割・ルーティング
      order.orderItems foreach {
        case (itemType, orderItem) => itemType match {
          case "TypeA" =>
            println(s"OrderRouter: routing $itemType")
            orderItemTypeAProcessor !
                      TypeAItemOrdered(orderItem)
          case "TypeB" =>
            println(s"OrderRouter: routing $itemType")
            orderItemTypeBProcessor !
                      TypeBItemOrdered(orderItem)
          case "TypeC" =>
            println(s"OrderRouter: routing $itemType")
            orderItemTypeCProcessor !
                      TypeCItemOrdered(orderItem)
        }
      }

      Splitter.completedStep()

    case _ =>
      println("OrderRouter: received unexpected message")
  }
}
```

---

## タイプ別プロセッサ

### OrderItemTypeAProcessor

```scala
class OrderItemTypeAProcessor extends Actor {
  def receive = {
    case TypeAItemOrdered(orderItem) =>
      println(s"OrderItemTypeAProcessor: handling↩
      $orderItem")
      Splitter.completedStep()
    case _ =>
      println("OrderItemTypeAProcessor: unexpected")
  }
}
```

### OrderItemTypeBProcessor

```scala
class OrderItemTypeBProcessor extends Actor {
  def receive = {
    case TypeBItemOrdered(orderItem) =>
      println(s"OrderItemTypeBProcessor: handling↩
      $orderItem")
      Splitter.completedStep()
    case _ =>
      println("OrderItemTypeBProcessor: unexpected")
  }
}
```

### OrderItemTypeCProcessor

```scala
class OrderItemTypeCProcessor extends Actor {
  def receive = {
    case TypeCItemOrdered(orderItem) =>
      println(s"OrderItemTypeCProcessor: handling↩
      $orderItem")
      Splitter.completedStep()
    case _ =>
      println("OrderItemTypeCProcessor: unexpected")
  }
}
```

---

## 処理フロー図

```
┌─────────────────────────────────────────────────────────────────┐
│                      Splitter 処理フロー                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OrderPlaced(Order) 受信                                        │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────────────────────┐                        │
│  │  order.orderItems foreach {         │                        │
│  │    case (itemType, orderItem) =>    │                        │
│  │      itemType match { ... }         │                        │
│  │  }                                  │                        │
│  └───────────────┬─────────────────────┘                        │
│                  │                                              │
│         ┌────────┼────────┐                                     │
│         │        │        │                                     │
│         ▼        ▼        ▼                                     │
│      "TypeA"  "TypeB"  "TypeC"                                  │
│         │        │        │                                     │
│         ▼        ▼        ▼                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│  │ TypeA    │ │ TypeB    │ │ TypeC    │                         │
│  │ Item     │ │ Item     │ │ Item     │                         │
│  │ Ordered  │ │ Ordered  │ │ Ordered  │                         │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘                         │
│       │            │            │                               │
│       ▼            ▼            ▼                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│  │ TypeA    │ │ TypeB    │ │ TypeC    │                         │
│  │ Processor│ │ Processor│ │ Processor│                         │
│  └──────────┘ └──────────┘ └──────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 実行結果

```
Order(Order Items: Map(TypeA -> OrderItem(1,↩
 TypeA, 'An item of type A.', 23.95), TypeB ->↩
 OrderItem(2, TypeB, 'An item of type B.', 99.95),↩
 TypeC -> OrderItem(3, TypeC, 'An item of type C.',↩
 14.95)) Totaling: 138.85)
OrderRouter: routing TypeA
OrderRouter: routing TypeB
OrderRouter: routing TypeC
OrderItemTypeAProcessor: handling OrderItem(1, TypeA,↩
 'An item of type A.', 23.95)
OrderItemTypeBProcessor: handling OrderItem(2, TypeB,↩
 'An item of type B.', 99.95)
OrderItemTypeCProcessor: handling OrderItem(3, TypeC,↩
 'An item of type C.', 14.95)
Splitter: is completed.
```

---

## 動作の説明

サンプルの`Order`には3つの`OrderItem`インスタンスがあり、それぞれ異なるタイプを持つ。

`OrderRouter`が`OrderPlaced`メッセージを受信すると：
1. 各`OrderItem`インスタンスを走査（イテレート）する
2. `OrderItem`の`itemType`の値に基づいて、`OrderItem`を新しいメッセージにパッケージ化する
3. 特定のタイププロセッサにディスパッチする

これにより、単一の複合メッセージ（`OrderPlaced`）が複数の個別メッセージ（`TypeAItemOrdered`、`TypeBItemOrdered`、`TypeCItemOrdered`）に分割され、それぞれが適切なプロセッサで処理される。
