# Recipient List

## 概念図

```
         ┌─────────────────────────────────────────┐
         │           Recipient List                │
         │                                         │
         │    ┌─────┐      ┌─────┐                │
         │    │     │      │     │────────────────┼───▶
         │    │     │      │     │                │
    ────▶│───▶│  ◁  │─────▶│     │────────────────┼───▶
         │    │     │      │     │                │
         │    │     │      │     │────────────────┼───▶
         │    └─────┘      └─────┘                │
         │                                         │
         └─────────────────────────────────────────┘
```

---

## 定義

Recipient Listは、電子メールのTo（宛先）およびCc（カーボンコピー）フィールドに類似したパターンである。電子メールで任意の数の意図された受信者を指定するように、Recipient Listでも複数の受信者にメッセージを送信する。

---

## 受信者の決定方法

| 決定方法 | 説明 |
|---------|------|
| **事前決定型** | 送信されるメッセージの種類に応じて受信者があらかじめ決まっている |
| **動的決定型** | ビジネスルールのセットによって受信者が決定される（Dynamic Router (237) の特性を持つ） |

---

## システム構成図（Figure 7.4）

```
                                                     ┌─────────┐
                                                ┌───▶│ 📄      │───▶ (A)
                                                │    └─────────┘
                                                │
  ┌─────────┐                                   │    ┌─────────┐
  │  TypeC  │        ┌───────────────┐          ├───▶│ 📄      │───▶ (B)
  │ Message │───────▶│  Recipient    │──────────┤    └─────────┘
  │  📄     │        │    List       │          │
  └─────────┘        │   ┌───┐       │          │         (C)
                     │   │ ◁ │       │          │
                     │   └───┘       │          │    ┌─────────┐
                     └───────────────┘          └───▶│ 📄      │───▶ (D)
                                                     └─────────┘

※ Recipient Listを使用して、特定のMessage (130) を受信すべきアクターを識別する
```

---

## 具体例：価格見積システム

この例は価格見積を行うシステムである。`MountaineeringSuppliesOrderProcessor`が`RequestForQuotation`メッセージを受信すると、ビジネスルールのセットに基づいてRecipient Listを計算する。これはDynamic Router (237) の一種である。

### 見積エンジン（Quote Engines）

`MountaineeringSuppliesOrderProcessor`は任意の数の見積サービスから`PriceQuoteInterest`メッセージを受信する。各見積エンジンはアクターである：

| 見積エンジン | 対象価格帯 | 説明 |
|------------|----------|------|
| BudgetHikersPriceQuotes | $1.00 〜 $1,000.00 | 低価格帯向け |
| HighSierraPriceQuotes | $100.00 〜 $10,000.00 | 中価格帯向け |
| MountainAscentPriceQuotes | $70.00 〜 $5,000.00 | 中価格帯向け |
| PinnacleGearPriceQuotes | $250.00 〜 $500,000.00 | 高価格帯向け（エベレスト遠征など） |
| RockBottomOuterwearPriceQuotes | $0.50 〜 $7,500.00 | 幅広い価格帯 |

### 関心登録の仕組み

各見積エンジンアクターは作成時に`MountaineeringSuppliesOrderProcessor`への参照を受け取る。見積エンジンアクターにとって、この参照は「関心レジストラ」に過ぎない。見積エンジンアクターは即座に`PriceQuoteInterest`メッセージをレジストラに送信し、どの条件下で`RequestPriceQuote`メッセージを受け入れるかを示す。

### 関心登録の例

```scala
// Budget Hikers: $1〜$1,000の注文を受け入れる
interestRegistrar ! PriceQuoteInterest(
            self.path.toString, self, 1.00, 1000.00)

// Pinnacle Gear: $250〜$500,000の注文を受け入れる
interestRegistrar ! PriceQuoteInterest(
            self.path.toString, self, 250.00, 500000.00)
```

---

## 実装例（Scala/Akka）

### メッセージ定義

```scala
package co.vaughnvernon.reactiveenterprise.recipientlist

import akka.actor._
import co.vaughnvernon.reactiveenterprise._

// 見積依頼
case class RequestForQuotation(
        rfqId: String,
        retailItems: Seq[RetailItem]) {
  val totalRetailPrice: Double =
        retailItems.map(retailItem =>
              retailItem.retailPrice).sum
}

// 小売アイテム
case class RetailItem(
        itemId: String,
        retailPrice: Double)

// 価格見積への関心登録
case class PriceQuoteInterest(
        path: String,
        quoteProcessor: ActorRef,
        lowTotalRetail: Money,
        highTotalRetail: Money)

// 価格見積リクエスト
case class RequestPriceQuote(
        rfqId: String,
        itemId: String,
        retailPrice: Money,
        orderTotalRetailPrice: Money)

// 価格見積
case class PriceQuote(
        rfqId: String,
        itemId: String,
        retailPrice: Money,
        discountPrice: Money)
```

### ドライバアプリケーション

```scala
object RecipientList extends CompletableApp(5) {
  // 注文プロセッサ（Recipient List管理）
  val orderProcessor =
          system.actorOf(
            Props[MountaineeringSuppliesOrderProcessor],
            "orderProcessor")

  // 見積エンジンの作成
  system.actorOf(
            Props(classOf[BudgetHikersPriceQuotes],
                    orderProcessor),
            "budgetHikers")
  system.actorOf(
            Props(classOf[HighSierraPriceQuotes],
                    orderProcessor),
            "highSierra")
  system.actorOf(
            Props(classOf[MountainAscentPriceQuotes],
                    orderProcessor),
            "mountainAscent")
  system.actorOf(
            Props(classOf[PinnacleGearPriceQuotes],
                    orderProcessor),
            "pinnacleGear")
  system.actorOf(
            Props(classOf[RockBottomOuterwearPriceQuotes],
                    orderProcessor),
            "rockBottomOuterwear")

  // 見積依頼の送信
  orderProcessor ! RequestForQuotation("123",
      Vector(RetailItem("1", 29.95),
            RetailItem("2", 99.95),
            RetailItem("3", 14.95)))

  orderProcessor ! RequestForQuotation("125",
      Vector(RetailItem("4", 39.99),
            RetailItem("5", 199.95),
            RetailItem("6", 149.95),
            RetailItem("7", 724.99)))

  orderProcessor ! RequestForQuotation("129",
      Vector(RetailItem("8", 119.99),
            RetailItem("9", 499.95),
            RetailItem("10", 519.00),
            RetailItem("11", 209.50)))

  orderProcessor ! RequestForQuotation("135",
      Vector(RetailItem("12", 0.97),
            RetailItem("13", 9.50),
            RetailItem("14", 1.99)))

  orderProcessor ! RequestForQuotation("140",
      Vector(RetailItem("15", 107.50),
            RetailItem("16", 9.50),
            RetailItem("17", 599.99),
            RetailItem("18", 249.95),
            RetailItem("19", 789.99)))

  ...
}
```

---

## MountaineeringSuppliesOrderProcessor（Recipient List管理）

```scala
import scala.collection.mutable.Map

class MountaineeringSuppliesOrderProcessor
    extends Actor {

  // 関心登録レジストリ
  val interestRegistry = Map[String, PriceQuoteInterest]()

  // Recipient List の計算
  def calculateRecipientList(
      rfq: RequestForQuotation): Iterable[ActorRef] = {
    for {
      interest <- interestRegistry.values
      if (rfq.totalRetailPrice >= interest.lowTotalRetail)
      if (rfq.totalRetailPrice <= interest.highTotalRetail)
    } yield interest.quoteProcessor
  }

  // Recipient List への配信
  def dispatchTo(
      rfq: RequestForQuotation,
      recipientList: Iterable[ActorRef]) = {
    recipientList.map { recipient =>
      rfq.retailItems.map { retailItem =>
        println("OrderProcessor: "
              + rfq.rfqId
              + " item: "
              + retailItem.itemId
              + " to: "
              + recipient.path.toString)
        recipient ! RequestPriceQuote(
                      rfq.rfqId,
                      retailItem.itemId,
                      retailItem.retailPrice,
                      rfq.totalRetailPrice)
      }
    }
  }

  def receive = {
    // 関心登録
    case interest: PriceQuoteInterest =>
      interestRegistry(interest.path) = interest

    // 価格見積の受信
    case priceQuote: PriceQuote =>
      println(s"OrderProcessor: received: $priceQuote")

    // 見積依頼の処理
    case rfq: RequestForQuotation =>
      val recipientList = calculateRecipientList(rfq)
      dispatchTo(rfq, recipientList)

    case message: Any =>
      println(s"OrderProcessor: unexpected: $message")
  }
}
```

### calculateRecipientList の動作

`MountaineeringSuppliesOrderProcessor`の`calculateRecipientList()`メソッドはScalaの**for内包表記（for comprehension）**を使用して、登録されたビジネスルールに基づいてすべての受信者を決定する。

### 配信ロジック

`RequestForQuotation`メッセージを受信すると、Recipient Listを計算してからアイテムをディスパッチする：
1. `RequestForQuotation`の`totalRetailPrice`が関心の`lowTotalRetail`と`highTotalRetail`の間にあるかチェック
2. 条件を満たす見積エンジンがRecipient Listに含まれる
3. 各受信者に対して、注文内の各アイテムについて`RequestPriceQuote`メッセージを送信

---

## 見積エンジンの実装

各見積エンジンの基本的な違いは`discountPercentage()`の実装である。意図的に抽象ベースクラスの継承（クラス拡張）を避けている。異なる小売業者の価格エンジンが同じベースクラスを継承することは非常に考えにくいため、類似していても各エンジンの独立した実装があることを示している。

### BudgetHikersPriceQuotes

```scala
class BudgetHikersPriceQuotes(interestRegistrar: ActorRef)
            extends Actor {
  // 関心登録: $1〜$1,000
  interestRegistrar ! PriceQuoteInterest(
                        self.path.toString,
                        self, 1.00, 1000.00)

  def receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(
                        rpq.orderTotalRetailPrice) *
                        rpq.retailPrice
      sender ! PriceQuote(rpq.rfqId, rpq.itemId,
                        rpq.retailPrice,
                        rpq.retailPrice - discount)

    case message: Any =>
      println(s"BudgetHikersPriceQuotes: unexpected:↩
      $message")
  }

  def discountPercentage(
      orderTotalRetailPrice: Double) = {
    if (orderTotalRetailPrice <= 100.00) 0.02
    else if (orderTotalRetailPrice <= 399.99) 0.03
    else if (orderTotalRetailPrice <= 499.99) 0.05
    else if (orderTotalRetailPrice <= 799.99) 0.07
    else 0.075
  }
}
```

### HighSierraPriceQuotes

```scala
class HighSierraPriceQuotes(interestRegistrar: ActorRef)
            extends Actor {
  // 関心登録: $100〜$10,000
  interestRegistrar ! PriceQuoteInterest(
                        self.path.toString, self,
                        100.00, 10000.00)

  def receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(
                        rpq.orderTotalRetailPrice) *
                        rpq.retailPrice
      sender ! PriceQuote(rpq.rfqId, rpq.itemId,
                        rpq.retailPrice,
                        rpq.retailPrice - discount)

    case message: Any =>
      println(s"HighSierraPriceQuotes: unexpected:↩
      $message")
  }

  def discountPercentage(
      orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 150.00) 0.015
    else if (orderTotalRetailPrice <= 499.99) 0.02
    else if (orderTotalRetailPrice <= 999.99) 0.03
    else if (orderTotalRetailPrice <= 4999.99) 0.04
    else 0.05
  }
}
```

### MountainAscentPriceQuotes

```scala
class MountainAscentPriceQuotes(interestRegistrar: ActorRef)
            extends Actor {
  // 関心登録: $70〜$5,000
  interestRegistrar ! PriceQuoteInterest(
                        self.path.toString, self,
                        70.00, 5000.00)

  def receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(
                        rpq.orderTotalRetailPrice) *
                        rpq.retailPrice
      sender ! PriceQuote(rpq.rfqId, rpq.itemId,
                        rpq.retailPrice,
                        rpq.retailPrice - discount)

    case message: Any =>
      println(s"MountainAscentPriceQuotes: unexpected:↩
      $message")
  }

  def discountPercentage(
      orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 99.99) 0.01
    else if (orderTotalRetailPrice <= 199.99) 0.02
    else if (orderTotalRetailPrice <= 499.99) 0.03
    else if (orderTotalRetailPrice <= 799.99) 0.04
    else if (orderTotalRetailPrice <= 999.99) 0.045
    else if (orderTotalRetailPrice <= 2999.99) 0.0475
    else 0.05
  }
}
```

### PinnacleGearPriceQuotes

```scala
class PinnacleGearPriceQuotes(interestRegistrar: ActorRef)
            extends Actor {
  // 関心登録: $250〜$500,000
  interestRegistrar ! PriceQuoteInterest(
                        self.path.toString, self,
                        250.00, 500000.00)

  def receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(
                        rpq.orderTotalRetailPrice) *
                        rpq.retailPrice
      sender ! PriceQuote(rpq.rfqId, rpq.itemId,
                        rpq.retailPrice,
                        rpq.retailPrice - discount)

    case message: Any =>
      println(s"PinnacleGearPriceQuotes: unexpected:↩
      $message")
  }

  def discountPercentage(
      orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 299.99) 0.015
    else if (orderTotalRetailPrice <= 399.99) 0.0175
    else if (orderTotalRetailPrice <= 499.99) 0.02
    else if (orderTotalRetailPrice <= 999.99) 0.03
    else if (orderTotalRetailPrice <= 1199.99) 0.035
    else if (orderTotalRetailPrice <= 4999.99) 0.04
    else if (orderTotalRetailPrice <= 7999.99) 0.05
    else 0.06
  }
}
```

### RockBottomOuterwearPriceQuotes

```scala
class RockBottomOuterwearPriceQuotes(
    interestRegistrar: ActorRef)
    extends Actor {
  // 関心登録: $0.50〜$7,500
  interestRegistrar ! PriceQuoteInterest(
                        self.path.toString, self,
                        0.50, 7500.00)

  def receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(
                        rpq.orderTotalRetailPrice) *
                        rpq.retailPrice
      sender ! PriceQuote(rpq.rfqId, rpq.itemId,
                        rpq.retailPrice,
                        rpq.retailPrice - discount)

    case message: Any =>
      println(s"RockBottomOuterwearPriceQuotes:↩
      unexpected: $message")
  }

  def discountPercentage(
      orderTotalRetailPrice: Double): Double = {
    if (orderTotalRetailPrice <= 100.00) 0.015
    else if (orderTotalRetailPrice <= 399.99) 0.02
    else if (orderTotalRetailPrice <= 499.99) 0.03
    else if (orderTotalRetailPrice <= 799.99) 0.04
    else if (orderTotalRetailPrice <= 999.99) 0.05
    else if (orderTotalRetailPrice <= 2999.99) 0.06
    else if (orderTotalRetailPrice <= 4999.99) 0.07
    else if (orderTotalRetailPrice <= 5999.99) 0.075
    else 0.08
  }
}
```

---

## 処理フロー図

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Recipient List 処理フロー                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 見積エンジンの関心登録                                               │
│                                                                         │
│  ┌──────────────────┐      PriceQuoteInterest      ┌────────────────┐   │
│  │ BudgetHikers     │─────────────────────────────▶│                │   │
│  │ ($1-$1,000)      │                              │                │   │
│  └──────────────────┘                              │                │   │
│  ┌──────────────────┐      PriceQuoteInterest      │  Mountaineering│   │
│  │ HighSierra       │─────────────────────────────▶│  Supplies      │   │
│  │ ($100-$10,000)   │                              │  Order         │   │
│  └──────────────────┘                              │  Processor     │   │
│  ┌──────────────────┐      PriceQuoteInterest      │                │   │
│  │ MountainAscent   │─────────────────────────────▶│ (interest      │   │
│  │ ($70-$5,000)     │                              │  Registry)     │   │
│  └──────────────────┘                              │                │   │
│  ┌──────────────────┐      PriceQuoteInterest      │                │   │
│  │ PinnacleGear     │─────────────────────────────▶│                │   │
│  │ ($250-$500,000)  │                              │                │   │
│  └──────────────────┘                              │                │   │
│  ┌──────────────────┐      PriceQuoteInterest      │                │   │
│  │ RockBottom       │─────────────────────────────▶│                │   │
│  │ ($0.50-$7,500)   │                              └────────────────┘   │
│  └──────────────────┘                                                   │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  2. 見積依頼の処理                                                       │
│                                                                         │
│  RequestForQuotation                                                    │
│  (totalRetailPrice)                                                     │
│         │                                                               │
│         ▼                                                               │
│  ┌─────────────────────────────────────┐                                │
│  │  calculateRecipientList()           │                                │
│  │                                     │                                │
│  │  for {                              │                                │
│  │    interest <- interestRegistry     │                                │
│  │    if total >= interest.low         │                                │
│  │    if total <= interest.high        │                                │
│  │  } yield interest.quoteProcessor    │                                │
│  └───────────────┬─────────────────────┘                                │
│                  │                                                      │
│                  ▼                                                      │
│  ┌─────────────────────────────────────┐                                │
│  │  dispatchTo(rfq, recipientList)     │                                │
│  │                                     │                                │
│  │  各受信者 × 各アイテム に対して     │                                │
│  │  RequestPriceQuote を送信           │                                │
│  └───────────────┬─────────────────────┘                                │
│                  │                                                      │
│         ┌───────┼───────┬───────┬───────┐                               │
│         ▼       ▼       ▼       ▼       ▼                               │
│       受信者A  受信者B  受信者C  受信者D  受信者E                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 実行結果（抜粋）

```
OrderProcessor: 123 item: 1 to: akka://mtnSupplies/↩
user/rockBottomOuterwear
OrderProcessor: 123 item: 2 to: akka://mtnSupplies/↩
user/rockBottomOuterwear
OrderProcessor: 123 item: 3 to: akka://mtnSupplies/↩
user/rockBottomOuterwear
OrderProcessor: 123 item: 1 to: akka://mtnSupplies/↩
user/mountainAscent
...
OrderProcessor: 140 item: 19 to: akka://mtnSupplies/↩
user/highSierra
OrderProcessor: received: PriceQuote(123,1,29.95,29.351)
OrderProcessor: received: PriceQuote(123,2,99.95,↩
97.95100000000001)
OrderProcessor: received: PriceQuote(123,3,14.95,14.651)
OrderProcessor: received: PriceQuote(123,1,29.95,29.351)
OrderProcessor: received: PriceQuote(123,2,99.95,↩
97.95100000000001)
OrderProcessor: received: PriceQuote(123,3,14.95,14.651)
OrderProcessor: received: PriceQuote(123,1,29.95,29.0515)
OrderProcessor: received: PriceQuote(123,2,99.95,↩
96.95150000000001)
...
OrderProcessor: received: PriceQuote(140,19,789.99,758.3904)
```

---

## Aggregator との関係

`MountaineeringSuppliesOrderProcessor`が各見積エンジンからのすべてのデータを購入者にとって意味のある単一の見積に結合する方法は、**Aggregator (257)** パターンの主題である。

Recipient ListとAggregatorを組み合わせると、**Scatter-Gather (272)** パターンの一種を形成する。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scatter-Gather パターン                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      ┌────────────┐                             │
│                 ┌───▶│ 受信者 A   │───┐                         │
│                 │    └────────────┘   │                         │
│                 │    ┌────────────┐   │                         │
│  ┌──────────┐   ├───▶│ 受信者 B   │───┤   ┌──────────────┐      │
│  │ Recipient│───┤    └────────────┘   ├──▶│  Aggregator  │      │
│  │   List   │   │    ┌────────────┐   │   └──────────────┘      │
│  │ (Scatter)│   ├───▶│ 受信者 C   │───┤       (Gather)          │
│  └──────────┘   │    └────────────┘   │                         │
│                 │    ┌────────────┐   │                         │
│                 └───▶│ 受信者 D   │───┘                         │
│                      └────────────┘                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 参照

- Dynamic Router (237)
- Message (130)
- Aggregator (257)
- Scatter-Gather (272)