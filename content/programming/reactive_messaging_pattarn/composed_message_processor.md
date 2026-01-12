# Composed Message Processor

## 概念図

```
         ┌─────────────────────────────────────────┐
         │      Composed Message Processor         │
         │                                         │
         │    ┌─────┐  ┌─────┐  ┌─────┐           │
         │    │     │  │     │  │     │           │
    ────▶│───▶│ ──▶ │─▶│ ──▶ │─▶│ ◀── │───────────┼───▶
         │    │     │  │     │  │     │           │
         │    └─────┘  └──┬──┘  └─────┘           │
         │                │                        │
         │             ┌──┴──┐                     │
         │             │     │                     │
         │             └─────┘                     │
         └─────────────────────────────────────────┘
```

---

## 定義

Composed Message Processorは、以下の組み合わせによって構成されるより大きなパターンである：

| 組み合わせ | 構成要素 |
|-----------|---------|
| 組み合わせ1 | Recipient List (245) + Aggregator (257) |
| 組み合わせ2 | Content-Based Router (228) + Splitter (254) |

この構成をComposed Message Processorとして見ることの利点は、結合されたコンポーネントが**Pipes and Filters (135)** スタイルの単一フィルタとしてより容易に機能できることである。

---

## システム構成図（Figure 7.7）

Scatter-Gather形式のComposed Message Processorの詳細を示す。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Composed Message Processor 詳細                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                          ┌─────────────────┐                            │
│                          │  Order          │                            │
│                          │  Processor      │                            │
│                          └────────┬────────┘                            │
│                                   │                                     │
│  ┌─────────────┐                  │         ┌───────────────────┐       │
│  │  Requestor  │                  │         │ RequestPriceQuote │       │
│  └──────┬──────┘                  │         │       [!]         │       │
│         │                         │         └─────────┬─────────┘       │
│         │ RequestForQuotation     │                   │                 │
│         │       [!]               │                   ▼                 │
│         ▼                         │         ┌─────────────────┐         │
│  ┌─────────────┐                  │    ┌───▶│  Price Quotes   │         │
│  │             │──────────────────┘    │    └────────┬────────┘         │
│  └─────────────┘                       │             │                  │
│         ▲                              │             │                  │
│         │                              │    ┌────────┴────────┐         │
│         │ BestPriceQuotation           │    │  Price Quotes   │         │
│         │       [📄]                   │    └────────┬────────┘         │
│         │                              │             │                  │
│  ┌──────┴──────┐                       │             ▼                  │
│  │  PriceQuote │◀──────────────────────┤      ┌───────────┐             │
│  │  Aggregator │                       │      │ PriceQuote│             │
│  │   [◀──]     │                       │      │    [📄]   │             │
│  └─────────────┘                       │      └─────┬─────┘             │
│         ▲                              │            │                   │
│         │ PriceQuoteFulfilled          │            ▼                   │
│         │       [⚡]                   │   ┌──────────────────┐         │
│         │                              │   │ OrderProcessor   │         │
│         └──────────────────────────────┴───│    (転送)        │         │
│                                            └──────────────────┘         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 抽象化された表現（Figure 7.8）

このComposed Message Processorを抽象化すると、はるかにシンプルなフィルタとして表現できる。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ┌───────────────────┐      ┌─────────────────┐      ┌───────────────┐  │
│  │                   │      │                 │      │               │  │
│  │ RequestForQuotation│─────▶│  Composed       │─────▶│ BestPrice     │  │
│  │       [!]         │      │  Message        │      │ Quotation     │  │
│  │                   │      │  Processor      │      │    [📄📄📄]   │  │
│  └───────────────────┘      │   ┌───────┐     │      └───────────────┘  │
│                             │   │ □ ─▶ □│     │                         │
│                             │   │   □   │     │                         │
│                             │   └───────┘     │                         │
│                             └─────────────────┘                         │
│                                                                         │
│  ※ Scatter-Gather処理を単一アクターに構成できる                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

Composed Message Processorの例については、Scatter-Gather (272) を参照。

---

# Scatter-Gather

## 定義

Scatter-Gatherは、Recipient ListとAggregatorの組み合わせによる実装を既に見てきた。これは2つのScatter-Gatherバリアントの最初のものを提供する。

2番目のバリアント—**Publish-Subscribe Channel (154)** を使用して`RequestPriceQuote`メッセージを関心のある参加者に送信する方法—をここで説明する。

---

## システム構成図（Figure 7.9）

このScatter-Gather処理は最良価格の見積を見つけるために使用される。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Scatter-Gather 処理フロー                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                          ┌─────────────────┐                            │
│                          │  Order          │                            │
│                          │  Processor      │                            │
│                          └────────┬────────┘                            │
│                                   │                                     │
│  ┌─────────────┐                  │         ┌───────────────────┐       │
│  │  Requestor  │                  │         │ RequestPriceQuote │       │
│  └──────┬──────┘                  │         │ (Publish)   [!]   │       │
│         │                         │         └─────────┬─────────┘       │
│         │ RequestForQuotation     │                   │                 │
│         │       [!]               │                   ▼                 │
│         ▼                         │         ┌─────────────────┐         │
│  ┌─────────────┐                  │    ┌───▶│  Price Quotes   │         │
│  │             │──────────────────┘    │    │  (Subscriber)   │         │
│  └─────────────┘                       │    └────────┬────────┘         │
│         ▲                              │             │                  │
│         │                              │             │                  │
│         │ BestPriceQuotation           │    ┌────────┴────────┐         │
│         │       [📄]                   │    │  Price Quotes   │         │
│         │                              │    │  (Subscriber)   │         │
│  ┌──────┴──────┐                       │    └────────┬────────┘         │
│  │  PriceQuote │◀──────────────────────┤             │                  │
│  │  Aggregator │                       │             ▼                  │
│  │   [◀──]     │                       │      ┌───────────┐             │
│  └─────────────┘                       │      │ PriceQuote│             │
│         ▲                              │      │    [📄]   │             │
│         │ PriceQuoteFulfilled          │      └─────┬─────┘             │
│         │       [⚡]                   │            │                   │
│         │                              │            ▼                   │
│         └──────────────────────────────┴───┌──────────────────┐         │
│                                            │ OrderProcessor   │         │
│                                            │    (転送)        │         │
│                                            └──────────────────┘         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Recipient List との違い

Recipient List (245) およびAggregator (257) サンプルの`MountaineeringSuppliesOrderProcessor`は既に`interestRegistry`を維持している。これは事実だが、**Publish-Subscribe Channel (154)** と同じではない。

| 観点 | Recipient List アプローチ | Publish-Subscribe アプローチ |
|-----|-------------------------|---------------------------|
| フィルタリング | `calculateRecipientList()`でビジネスルールにより参加者をフィルタ | すべての登録済み関心サブスクライバに公開 |
| 制御 | 送信側がどの関心者が見積に参加するかを決定 | 関心のある見積エンジンが見積を提供するかどうかを選択可能 |

この例では、各見積エンジンの希望する注文合計価格範囲を事前に検査する形式性を省略する。代わりに、`RequestPriceQuote`メッセージを各登録済み関心サブスクライバに公開し、関心のある見積エンジンが見積を提供することを許可する。

---

## 制御の側面

この方式により、`MountaineeringSuppliesOrderProcessor`はプロセスの多くの制御を放棄する。それでも、少なくとも2つの側面を制御できる：**時間**と**最良見積**。

| 制御項目 | 説明 |
|---------|------|
| **時間（タイムアウト）** | 一定期間後に特定の注文の見積プロセス全体をシャットダウン（実際には`PriceQuoteAggregator`がタイムアウトを管理） |
| **最良見積** | `PriceQuoteAggregator`が最良の見積を選択—オークションで最良の入札を受け入れるのと同様—誰がビジネスを獲得するかを決定 |

---

## 実装例（Scala/Akka）

### メッセージ定義

```scala
package co.vaughnvernon.reactiveenterprise.scattergather

import java.util.concurrent.TimeUnit
import scala.concurrent._
import scala.concurrent.duration._
import ExecutionContext.Implicits.global
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

case class RetailItem(
        itemId: String,
        retailPrice: Double)

// 価格見積リクエスト
case class RequestPriceQuote(
        rfqId: String,
        itemId: String,
        retailPrice: Money,
        orderTotalRetailPrice: Money)

// 価格見積（quoterIdを追加）
case class PriceQuote(
        quoterId: String,
        rfqId: String,
        itemId: String,
        retailPrice: Money,
        discountPrice: Money)

// 見積フルフィルメント通知
case class PriceQuoteFulfilled(priceQuote: PriceQuote)

// タイムアウト通知
case class PriceQuoteTimedOut(rfqId: String)

// フルフィルメント追跡要求
case class RequiredPriceQuotesForFulfillment(
        rfqId: String,
        quotesRequested: Int)

// 見積フルフィルメント
case class QuotationFulfillment(
        rfqId: String,
        quotesRequested: Int,
        priceQuotes: Seq[PriceQuote],
        requester: ActorRef)

// 最良価格見積（最終結果）
case class BestPriceQuotation(
        rfqId: String,
        priceQuotes: Seq[PriceQuote])

// 価格見積リクエストへのサブスクリプション
case class SubscribeToPriceQuoteRequests(
        quoterId: String,
        quoteProcessor: ActorRef)
```

### ドライバアプリケーション

```scala
object ScatterGather extends CompletableApp(5) {
  val priceQuoteAggregator =
          system.actorOf(
                Props[PriceQuoteAggregator],
                "priceQuoteAggregator")

  val orderProcessor =
      system.actorOf(
          Props(
            classOf[MountaineeringSuppliesOrderProcessor],
                priceQuoteAggregator),
          "orderProcessor")

  // 見積エンジンの作成（orderProcessorをPublisherとして参照）
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
      Vector(RetailItem("15", 1295.50),
            RetailItem("16", 9.50),
            RetailItem("17", 599.99),
            RetailItem("18", 249.95),
            RetailItem("19", 789.99)))

  awaitCompletion
  println("Scatter-Gather: is completed.")
}
```

`BestPriceQuotation`メッセージタイプは、`MountaineeringSuppliesOrderProcessor`に送信される最終メッセージである。プロセッサが各`RequestForQuotation`に対して1つずつ、5つのそのようなメッセージを受信すると、Scatter-Gatherサンプルプロセスは完了する。

---

## MountaineeringSuppliesOrderProcessor（Publish-Subscribe版）

```scala
import scala.collection.mutable.Map

class MountaineeringSuppliesOrderProcessor(
        priceQuoteAggregator: ActorRef)
    extends Actor {

  // サブスクライバ管理（Publish-Subscribe）
  val subscribers =
        Map[String, SubscribeToPriceQuoteRequests]()

  // すべてのサブスクライバに配信（フィルタリングなし）
  def dispatch(rfq: RequestForQuotation) = {
    subscribers.values.map { subscriber =>
      val quoteProcessor = subscriber.quoteProcessor
      rfq.retailItems.map { retailItem =>
        println("OrderProcessor: "
              + rfq.rfqId
              + " item: "
              + retailItem.itemId
              + " to: "
              + subscriber.quoterId)
        quoteProcessor !
            RequestPriceQuote(
                rfq.rfqId,
                retailItem.itemId,
                retailItem.retailPrice,
                rfq.totalRetailPrice)
      }
    }
  }

  def receive = {
    // サブスクリプション登録
    case subscriber: SubscribeToPriceQuoteRequests =>
      subscribers(subscriber.path) = subscriber

    // 価格見積をAggregatorに転送
    case priceQuote: PriceQuote =>
      priceQuoteAggregator !
            PriceQuoteFulfilled(priceQuote)
      println(s"OrderProcessor: received: $priceQuote")

    // 見積依頼の処理
    case rfq: RequestForQuotation =>
      priceQuoteAggregator !
            RequiredPriceQuotesForFulfillment(
                  rfq.rfqId,
                  subscribers.size * rfq.retailItems.size)
      dispatch(rfq)

    // 最良価格見積の受信
    case bestPriceQuotation: BestPriceQuotation =>
      println(s"OrderProcessor: received:↩
      $bestPriceQuotation")
      ScatterGather.completedStep()

    case message: Any =>
      println(s"OrderProcessor: unexpected: $message")
  }
}
```

### Recipient List版との主な違い

このバージョンの`dispatch()`は、ビジネスルールで見積プロセッサを制約するのではなく、すべてのサブスクライバに`RequestPriceQuote`メッセージを送信する。

---

## PriceQuoteAggregator（タイムアウト＆最良価格選択）

```scala
import scala.collection.mutable.Map

class PriceQuoteAggregator extends Actor {
  val fulfilledPriceQuotes =
        Map[String, QuotationFulfillment]()

  // 最良価格見積の作成
  def bestPriceQuotationFrom(
        quotationFulfillment: QuotationFulfillment)
        : BestPriceQuotation = {
    val bestPrices = Map[String, PriceQuote]()

    // 各アイテムについて最低価格を選択
    quotationFulfillment.priceQuotes.map { priceQuote =>
      if (bestPrices.contains(priceQuote.itemId)) {
        if (bestPrices(priceQuote.itemId).discountPrice >
            priceQuote.discountPrice) {
          bestPrices(priceQuote.itemId) = priceQuote
        }
      } else {
        bestPrices(priceQuote.itemId) = priceQuote
      }
    }

    BestPriceQuotation(
        quotationFulfillment.rfqId,
        bestPrices.values.toVector)
  }

  def receive = {
    // フルフィルメント追跡開始＆タイムアウト設定
    case required: RequiredPriceQuotesForFulfillment =>
      fulfilledPriceQuotes(required.rfqId) =
              QuotationFulfillment(
                  required.rfqId,
                  required.quotesRequested,
                  Vector(),
                  sender)

      // 2秒のタイムアウトを設定
      val duration = Duration.create(2, TimeUnit.SECONDS)

      context.system.scheduler.scheduleOnce(
            duration, self,
            PriceQuoteTimedOut(required.rfqId))

    // 価格見積の受信
    case priceQuoteFulfilled: PriceQuoteFulfilled =>
      priceQuoteRequestFulfilled(priceQuoteFulfilled)
      println(s"PriceQuoteAggregator: fulfilled price↩
      quote: $priceQuoteFulfilled")

    // タイムアウト処理
    case priceQuoteTimedOut: PriceQuoteTimedOut =>
      priceQuoteRequestTimedOut(priceQuoteTimedOut.rfqId)

    case message: Any =>
      println(s"PriceQuoteAggregator: unexpected: $message")
  }

  // 価格見積リクエストのフルフィルメント処理
  def priceQuoteRequestFulfilled(
        priceQuoteFulfilled: PriceQuoteFulfilled) = {
    if (fulfilledPriceQuotes.contains(
          priceQuoteFulfilled.priceQuote.rfqId)) {
      val previousFulfillment =
              fulfilledPriceQuotes(
                priceQuoteFulfilled.priceQuote.rfqId)
      val currentPriceQuotes =
              previousFulfillment.priceQuotes :+
                    priceQuoteFulfilled.priceQuote
      val currentFulfillment =
          QuotationFulfillment(
              previousFulfillment.rfqId,
              previousFulfillment.quotesRequested,
              currentPriceQuotes,
              previousFulfillment.requester)

      // すべての見積が揃ったら最良価格を送信
      if (currentPriceQuotes.size >=
          currentFulfillment.quotesRequested) {
        quoteBestPrice(currentFulfillment)
      } else {
        fulfilledPriceQuotes(
              priceQuoteFulfilled.priceQuote.rfqId) =
                    currentFulfillment
      }
    }
  }

  // タイムアウト処理
  def priceQuoteRequestTimedOut(rfqId: String) = {
    if (fulfilledPriceQuotes.contains(rfqId)) {
      // 受信済みの見積から最良価格を作成
      quoteBestPrice(fulfilledPriceQuotes(rfqId))
    }
  }

  // 最良価格の送信
  def quoteBestPrice(
        quotationFulfillment: QuotationFulfillment) = {
    if (fulfilledPriceQuotes.contains(
          quotationFulfillment.rfqId)) {
      quotationFulfillment.requester !
          bestPriceQuotationFrom(quotationFulfillment)
      fulfilledPriceQuotes.remove(
            quotationFulfillment.rfqId)
    }
  }
}
```

### PriceQuoteAggregatorの重要な機能

`PriceQuoteAggregator`はいくつかの重要なことを管理する：

| 機能 | 説明 |
|-----|------|
| **タイムアウト設定** | `RequiredPriceQuotesForFulfillment`メッセージに反応してタイムアウトを設定し、各`RequestForQuotation`の見積プロセス全体を期間に制限する。この例では2秒を使用しているが、ビジネスが決定する実用的な値に変更可能 |
| **タイムアウト処理** | タイムアウトが発生すると、`PriceQuoteTimedOut`メッセージが`PriceQuoteAggregator`によって受信され、その`RequestForQuotation`のプロセスは終了する。プロセスの終了は失敗を意味しない。むしろ、`PriceQuoteAggregator`が既に受信した`PriceQuoteFulfilled`メッセージの数に基づいて`MountaineeringSuppliesOrderProcessor`に`BestPriceQuotation`メッセージを提供するよう通知する |
| **最良価格選択** | `quoteBestPrice()`および`bestPriceQuotationFrom()`操作：集約完了時に`BestPriceQuotation`を`MountaineeringSuppliesOrderProcessor`に送信。`bestPriceQuotationFrom()`を使用して`BestPriceQuotation`を作成し、最低の`discountPrice`を持つ`PriceQuote`インスタンスのみを保持 |

---

## 見積エンジンの実装（Publish-Subscribe版）

この例では、`BudgetHikersPriceQuotes`と`RockBottomOuterwearPriceQuotes`が$1,000および$2,000を超える注文の見積を無視する。これにより、一部の見積についてプロセスがタイムアウトし、売り手間で取引が分散される。

### BudgetHikersPriceQuotes

```scala
class BudgetHikersPriceQuotes(
        priceQuoteRequestPublisher: ActorRef)
    extends Actor {

  val quoterId = self.path.name

  // サブスクリプション登録
  priceQuoteRequestPublisher !
        SubscribeToPriceQuoteRequests(quoterId, self)

  def receive = {
    case rpq: RequestPriceQuote =>
      // $1,000未満の注文のみ見積を提供
      if (rpq.orderTotalRetailPrice < 1000.00) {
        val discount = discountPercentage(
                    rpq.orderTotalRetailPrice) *
                    rpq.retailPrice
        sender ! PriceQuote(
                    quoterId,
                    rpq.rfqId,
                    rpq.itemId,
                    rpq.retailPrice,
                    rpq.retailPrice - discount)
      } else {
        println(s"BudgetHikersPriceQuotes: ignoring: $rpq")
      }

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
class HighSierraPriceQuotes(
        priceQuoteRequestPublisher: ActorRef)
    extends Actor {

  val quoterId = self.path.name

  priceQuoteRequestPublisher !
        SubscribeToPriceQuoteRequests(quoterId, self)

  def receive = {
    case rpq: RequestPriceQuote =>
      // すべての見積リクエストに応答
      val discount = discountPercentage(
                  rpq.orderTotalRetailPrice) *
                  rpq.retailPrice
      sender ! PriceQuote(
                  quoterId,
                  rpq.rfqId,
                  rpq.itemId,
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
class MountainAscentPriceQuotes(
        priceQuoteRequestPublisher: ActorRef)
    extends Actor {

  val quoterId = self.path.name

  priceQuoteRequestPublisher !
        SubscribeToPriceQuoteRequests(quoterId, self)

  def receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(
                  rpq.orderTotalRetailPrice) *
                  rpq.retailPrice
      sender ! PriceQuote(
                  quoterId,
                  rpq.rfqId,
                  rpq.itemId,
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
class PinnacleGearPriceQuotes(
        priceQuoteRequestPublisher: ActorRef)
    extends Actor {

  val quoterId = self.path.name

  priceQuoteRequestPublisher !
        SubscribeToPriceQuoteRequests(quoterId, self)

  def receive = {
    case rpq: RequestPriceQuote =>
      val discount = discountPercentage(
                  rpq.orderTotalRetailPrice) *
                  rpq.retailPrice
      sender ! PriceQuote(
                  quoterId,
                  rpq.rfqId,
                  rpq.itemId,
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
        priceQuoteRequestPublisher: ActorRef)
    extends Actor {

  val quoterId = self.path.name

  priceQuoteRequestPublisher !
        SubscribeToPriceQuoteRequests(quoterId, self)

  def receive = {
    case rpq: RequestPriceQuote =>
      // $2,000未満の注文のみ見積を提供
      if (rpq.orderTotalRetailPrice < 2000.00) {
        val discount = discountPercentage(
                    rpq.orderTotalRetailPrice) *
                    rpq.retailPrice
        sender ! PriceQuote(
                    quoterId,
                    rpq.rfqId,
                    rpq.itemId,
                    rpq.retailPrice,
                    rpq.retailPrice - discount)
      } else {
        println(s"RockBottomOuterwearPriceQuotes:↩
        ignoring: $rpq")
      }

    case message: Any =>
      println(s"RockBottomOuterwearPriceQuotes: unexpected:↩
      $message")
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

## 実行結果（抜粋）

```
OrderProcessor: 123 item: 1 to: rockBottomOuterwear
OrderProcessor: 123 item: 2 to: rockBottomOuterwear
OrderProcessor: 123 item: 3 to: rockBottomOuterwear
OrderProcessor: 123 item: 1 to: highSierra
OrderProcessor: 123 item: 2 to: highSierra
OrderProcessor: 123 item: 3 to: highSierra
OrderProcessor: 123 item: 1 to: pinnacleGear
OrderProcessor: 123 item: 2 to: pinnacleGear
OrderProcessor: 123 item: 3 to: pinnacleGear
OrderProcessor: 123 item: 1 to: budgetHikers
OrderProcessor: 123 item: 2 to: budgetHikers
OrderProcessor: 123 item: 3 to: budgetHikers
OrderProcessor: 123 item: 1 to: mountainAscent
OrderProcessor: 123 item: 2 to: mountainAscent
OrderProcessor: 123 item: 3 to: mountainAscent
...
BudgetHikersPriceQuotes: ignoring: RequestPriceQuote(↩
125,4,39.99,1114.88)
BudgetHikersPriceQuotes: ignoring: RequestPriceQuote(↩
125,5,199.95,1114.88)
...
PriceQuoteAggregator: fulfilled price quote:↩
 PriceQuoteFulfilled
OrderProcessor: received: PriceQuote(highSierra,125,↩
4,39.99,38.3904)
OrderProcessor: received: PriceQuote(highSierra,125,↩
5,199.95,191.952)
OrderProcessor: received: PriceQuote(highSierra,125,↩
6,149.95,143.952)
...
OrderProcessor: received:↩
 BestPriceQuotation(140,Vector(PriceQuote(mountainAscent,↩
140,15,1295.5,1233.96375),↩
 PriceQuote(mountainAscent,140,18,249.95,238.077375),↩
 PriceQuote(mountainAscent,140,17,599.99,571.4904750000001),↩
 PriceQuote(mountainAscent,140,16,9.5,9.04875),↩
 PriceQuote(mountainAscent,140,19,789.99,752.465475)))
OrderProcessor: received:↩
 BestPriceQuotation(125,Vector(PriceQuote(rockBottomOuterwear,↩
125,5,199.95,187.953),↩
 PriceQuote(rockBottomOuterwear,125,7,724.99,681.4906),↩
 PriceQuote(rockBottomOuterwear,125,4,39.99,37.5906),↩
 PriceQuote(rockBottomOuterwear,125,6,149.95,140.953)))
OrderProcessor: received: BestPriceQuotation(129,↩
Vector(PriceQuote(rockBottomOuterwear,129,8,119.99,↩
112.7906), PriceQuote(rockBottomOuterwear,129,11,↩
209.5,196.93), PriceQuote(rockBottomOuterwear,129,9,↩
499.95,469.953), PriceQuote(rockBottomOuterwear,129,↩
10,519.0,487.86)))
Scatter-Gather: is completed.
```

### 結果の分析

`BudgetHikersPriceQuotes`と`RockBottomOuterwearPriceQuotes`がほとんどの価格見積入札の獲得を分け合っている。しかし、`MountainAscentPriceQuotes`は`RequestForQuotation`の1つを獲得できる。`BudgetHikersPriceQuotes`と`RockBottomOuterwearPriceQuotes`が$1,000および$2,000を超える注文の入札を拒否するため、サンプルプロセスは一部の見積でタイムアウトし、売り手間で取引が分散される。

---

## Scatter-Gatherの2つのバリアント

| バリアント | 実装方法 |
|-----------|---------|
| **バリアント1** | Recipient List (245) + Aggregator (257) |
| **バリアント2** | Publish-Subscribe Channel (154) + Aggregator (257) |

---

## なぜより大きなパターンとして見るのか？

Scatter-GatherパターンをRecipient List (245) + Aggregator (257) の組み合わせとして、またはPublish-Subscribe Channel (154) を使用して見ることは、**Composed Message Processor (270)** を構成する。

しかし、なぜ細粒度のパターンをより大きなパターンとして見るのか？これらをより大きなパターンに構成すると、**Composed Message Processor (270)** は**Pipes and Filters (135)** スタイルの単一フィルタとしてより容易に機能できる。

```
┌─────────────────────────────────────────────────────────────────┐
│             Pipes and Filters における活用                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐    ┌────────────────────┐    ┌─────────┐          │
│  │         │    │ Composed Message   │    │         │          │
│  │ Filter  │───▶│ Processor          │───▶│ Filter  │          │
│  │    A    │    │ (Scatter-Gather)   │    │    B    │          │
│  │         │    │                    │    │         │          │
│  └─────────┘    └────────────────────┘    └─────────┘          │
│                                                                 │
│  内部の複雑さを隠蔽し、単一フィルタとして振る舞う                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
