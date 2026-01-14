# Aggregator

## 概念図

```
         ┌─────────────────────────────────────────┐
         │              Aggregator                 │
         │                                         │
         │    ┌─────┐                              │
    ────▶│───▶│     │      ┌─────┐                │
         │    │     │      │     │                │
    ────▶│───▶│     │─────▶│     │────────────────┼───▶
         │    │     │      │     │                │
    ────▶│───▶│     │      └─────┘                │
         │    └─────┘                              │
         │                                         │
         └─────────────────────────────────────────┘
```

---

## 定義

Aggregatorは、複数の個別メッセージを収集し、それらを単一の統合されたメッセージに結合するパターンである。

---

## Correlation Identifier との関係

Recipient List (245) の例では、`PriceQuote`応答が`MountaineeringSuppliesOrderProcessor`によってどのように同化されるかを示していなかった。`PriceQuote`応答を元の`RequestForQuotation`に関連付けるには、各メッセージと共に渡された一意の`rfqId`（**Correlation Identifier (215)**）を使用する必要がある。

```scala
// リクエスト送信時
orderProcessor ! RequestForQuotation("123", ...)
...
// 見積エンジンへのディスパッチ時
recipient ! RequestPriceQuote(rfq.rfqId, ...)
...
// 応答時
sender ! PriceQuote(rpq.rfqId, ...)
```

---

## システム構成図（Figure 7.5）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                      │
│  │ PriceQuote  │  │ PriceQuote  │  │ PriceQuote  │                      │
│  │ Fulfilled   │  │ Fulfilled   │  │ Fulfilled   │                      │
│  │    📄       │  │    📄       │  │    📄       │                      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                      │
│         │                │                │                             │
│         └────────────────┼────────────────┘                             │
│                          │                                              │
│                          ▼                                              │
│                   ┌─────────────┐                                       │
│                   │             │                                       │
│                   │  Aggregator │                                       │
│                   │   ┌───┐     │                                       │
│                   │   │ ─▶│     │                                       │
│                   │   └───┘     │                                       │
│                   └──────┬──────┘                                       │
│                          │                                              │
│                          ▼                                              │
│                   ┌─────────────┐                                       │
│                   │  Quotation  │                                       │
│                   │ Fulfillment │                                       │
│                   │    📄📄📄   │                                       │
│                   └─────────────┘                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

※ Aggregatorは複数の個別フルフィルメントMessage (130) を
   1つの完全な見積（Quotation）にまとめる
```

---

## 具体例：価格見積の集約

Recipient List (245) の例を拡張し、要求されたすべての価格見積のフルフィルメントを追跡するAggregatorを含める。複数の`PriceQuoteFulfillment`イベントメッセージ (207) が単一の`QuotationFulfillment`ドキュメントメッセージ (204) に集約される。

---

## 実装例（Scala/Akka）

### 新しいメッセージ定義

```scala
// 価格見積フルフィルメント（個別の見積完了通知）
case class PriceQuoteFulfilled(priceQuote: PriceQuote)

// フルフィルメントに必要な見積数の要求
case class RequiredPriceQuotesForFulfillment(
        rfqId: String,
        quotesRequested: Int)

// 見積フルフィルメント（集約結果）
case class QuotationFulfillment(
        rfqId: String,
        quotesRequested: Int,
        priceQuotes: Seq[PriceQuote],
        requester: ActorRef)
```

### ドライバアプリケーション

```scala
object Aggregator extends CompletableApp(5) {
  // Aggregatorの作成
  val priceQuoteAggregator =
              system.actorOf(
                Props[PriceQuoteAggregator],
                "priceQuoteAggregator")

  // OrderProcessorにAggregatorを注入
  val orderProcessor = system.actorOf(
        Props(classOf[MountaineeringSuppliesOrderProcessor],
                      priceQuoteAggregator),
        "orderProcessor")

  ...
}
```

---

## MountaineeringSuppliesOrderProcessor（Aggregator連携版）

```scala
import scala.collection.mutable.Map

class MountaineeringSuppliesOrderProcessor(
        priceQuoteAggregator: ActorRef)
    extends Actor {

  val interestRegistry =
        Map[String, PriceQuoteInterest]()

  def calculateRecipientList(
        rfq: RequestForQuotation): Iterable[ActorRef] = {
    for {
      interest <- interestRegistry.values
      if (rfq.totalRetailPrice >= interest.lowTotalRetail)
      if (rfq.totalRetailPrice <= interest.highTotalRetail)
    } yield interest.quoteProcessor
  }

  def dispatchTo(
        rfq: RequestForQuotation,
        recipientList: Iterable[ActorRef]) = {
    var totalRequestedQuotes = 0

    recipientList.map { recipient =>
      rfq.retailItems.map { retailItem =>
        println("OrderProcessor: " + rfq.rfqId
              + " item: " + retailItem.itemId + " to: "
              + recipient.path.toString)
        recipient ! RequestPriceQuote(
              rfq.rfqId, retailItem.itemId,
              retailItem.retailPrice, rfq.totalRetailPrice)
      }
    }
  }

  def receive = {
    case interest: PriceQuoteInterest =>
      interestRegistry(interest.quoterId) = interest

    // 価格見積を受信したらAggregatorに転送
    case priceQuote: PriceQuote =>
      priceQuoteAggregator !
            PriceQuoteFulfilled(priceQuote)
      println(s"OrderProcessor: received: $priceQuote")

    case rfq: RequestForQuotation =>
      val recipientList = calculateRecipientList(rfq)

      // Aggregatorにフルフィルメント追跡を依頼
      priceQuoteAggregator !
            RequiredPriceQuotesForFulfillment(
                  rfq.rfqId,
                  recipientList.size
                    * rfq.retailItems.size)

      dispatchTo(rfq, recipientList)

    // 集約完了通知を受信
    case fulfillment: QuotationFulfillment =>
      println(s"OrderProcessor: received: $fulfillment")
      Aggregator.completedStep()

    case message: Any =>
      println(s"OrderProcessor: unexpected: $message")
  }
}
```

### 重要な変更点

`MountaineeringSuppliesOrderProcessor`が計算されたRecipient List (245) にディスパッチする際、`PriceQuoteAggregator`に`RequiredPriceQuotesForFulfillment`メッセージを送信して、すべての`PriceQuote`インスタンスをフルフィルメントポイントまで追跡するよう依頼する。

---

## PriceQuoteAggregator（Aggregator本体）

```scala
import scala.collection.mutable.Map

class PriceQuoteAggregator extends Actor {
  // rfqId → QuotationFulfillment のマップ
  val fulfilledPriceQuotes =
        Map[String, QuotationFulfillment]()

  def receive = {
    // フルフィルメント追跡の開始
    case required: RequiredPriceQuotesForFulfillment =>
      fulfilledPriceQuotes(required.rfqId) =
            QuotationFulfillment(
                  required.rfqId,
                  required.quotesRequested,
                  Vector(),
                  sender)

    // 個別の価格見積を受信・集約
    case priceQuoteFulfilled: PriceQuoteFulfilled =>
      val previousFulfillment =
              fulfilledPriceQuotes(
                priceQuoteFulfilled.priceQuote.rfqId)

      // 新しい見積を追加
      val currentPriceQuotes =
              previousFulfillment.priceQuotes :+
                    priceQuoteFulfilled.priceQuote

      val currentFulfillment =
          QuotationFulfillment(
              previousFulfillment.rfqId,
              previousFulfillment.quotesRequested,
              currentPriceQuotes,
              previousFulfillment.requester)

      // 全ての見積が揃ったかチェック
      if (currentPriceQuotes.size >=
          currentFulfillment.quotesRequested) {
        // 完了：リクエスターに送信してクリーンアップ
        currentFulfillment.requester ! currentFulfillment
        fulfilledPriceQuotes.remove(
              priceQuoteFulfilled.priceQuote.rfqId)
      } else {
        // 未完了：状態を更新
        fulfilledPriceQuotes(
              priceQuoteFulfilled.priceQuote.rfqId) =
                    currentFulfillment
      }

      println(s"PriceQuoteAggregator: fulfilled↩
      price quote: $priceQuoteFulfilled")

    case message: Any =>
      println(s"PriceQuoteAggregator: unexpected: $message")
  }
}
```

### PriceQuoteAggregatorの動作

1. `RequiredPriceQuotesForFulfillment`を受信すると、`fulfilledPriceQuotes`マップに新しい`QuotationFulfillment`エントリを作成
2. 以降、各`PriceQuoteFulfilled`を受信するたびに、`PriceQuoteAggregator`は各`PriceQuote`（`PriceQuoteFulfilled`メッセージに含まれる）を`QuotationFulfillment`に集約
3. 要求された各`PriceQuote`を受信すると、完了した`QuotationFulfillment`を`OrderProcessor`に送信

---

## 処理フロー図

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Aggregator 処理フロー                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. フルフィルメント追跡の開始                                           │
│                                                                         │
│  ┌─────────────────────┐                  ┌─────────────────────┐       │
│  │ OrderProcessor      │ ───────────────▶ │ PriceQuoteAggregator│       │
│  │                     │ RequiredPrice    │                     │       │
│  │ rfqId, count        │ QuotesFor        │ fulfilledPriceQuotes│       │
│  └─────────────────────┘ Fulfillment      │ [rfqId] = new       │       │
│                                           │ QuotationFulfillment│       │
│                                           └─────────────────────┘       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  2. 見積の収集・集約                                                     │
│                                                                         │
│  ┌────────────┐                                                         │
│  │ QuoteEngine│─┐                                                       │
│  └────────────┘ │  PriceQuote                                           │
│  ┌────────────┐ │      │                                                │
│  │ QuoteEngine│─┼──────┤                                                │
│  └────────────┘ │      │                                                │
│  ┌────────────┐ │      ▼                                                │
│  │ QuoteEngine│─┘  ┌─────────────────────┐                              │
│  └────────────┘    │ OrderProcessor      │                              │
│                    │                     │                              │
│                    │ PriceQuoteFulfilled │                              │
│                    └──────────┬──────────┘                              │
│                               │                                         │
│                               ▼                                         │
│                    ┌─────────────────────┐                              │
│                    │ PriceQuoteAggregator│                              │
│                    │                     │                              │
│                    │ currentPriceQuotes  │                              │
│                    │ :+ priceQuote       │                              │
│                    └─────────────────────┘                              │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  3. 完了判定と結果送信                                                   │
│                                                                         │
│                    ┌─────────────────────┐                              │
│                    │ PriceQuoteAggregator│                              │
│                    │                     │                              │
│                    │ if (size >= requested)                             │
│                    │   requester ! fulfillment                          │
│                    │   remove(rfqId)                                    │
│                    │ else                                               │
│                    │   update state                                     │
│                    └──────────┬──────────┘                              │
│                               │                                         │
│                               │ QuotationFulfillment                    │
│                               │ (when complete)                         │
│                               ▼                                         │
│                    ┌─────────────────────┐                              │
│                    │ OrderProcessor      │                              │
│                    │                     │                              │
│                    │ 全見積を含む        │                              │
│                    │ 集約結果を受信      │                              │
│                    └─────────────────────┘                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 実行結果（抜粋）

```
OrderProcessor: 123 item: 1 to: akka://default/↩
user/rockBottomOuterwear
OrderProcessor: 123 item: 2 to: akka://default/↩
user/rockBottomOuterwear
OrderProcessor: 123 item: 3 to: akka://default/↩
user/rockBottomOuterwear
OrderProcessor: 123 item: 1 to: akka://default/↩
user/mountainAscent
...
OrderProcessor: 140 item: 19 to: akka://default/↩
user/highSierra
OrderProcessor: received: PriceQuote(123,1,29.95,29.351)
PriceQuoteAggregator: fulfilled price quote:↩
 PriceQuoteFulfilled(PriceQuote(123,1,29.95,29.351))
OrderProcessor: received: PriceQuote(123,2,99.95,↩
97.95100000000001)
PriceQuoteAggregator: fulfilled price quote:↩
 PriceQuoteFulfilled(PriceQuote(123,2,99.95,↩
97.95100000000001))
OrderProcessor: received: PriceQuote(123,3,14.95,14.651)
PriceQuoteAggregator: fulfilled price quote:↩
 PriceQuoteFulfilled(PriceQuote(123,3,14.95,14.651))
...
PriceQuote(125,7,724.99,695.9904),Actor[akka://↩
default/user/orderProcessor])
OrderProcessor: received: QuotationFulfillment(129,16,↩
Vector(PriceQuote(129,8,119.99,112.7906),↩
 PriceQuote(129,9,499.95,469.953), PriceQuote(129,10,↩
519.0,487.86), PriceQuote(129,11,209.5,196.93),↩
...
PriceQuote(140,19,789.99,758.3904)),Actor[akka://↩
default/user/orderProcessor])
Aggregator: is completed.
```

---

## 終了条件（Termination Criteria）

Aggregatorは様々な終了条件で設計できる：

| 終了条件 | 説明 |
|---------|------|
| **Wait for All** | すべての期待される応答を待つ |
| **Timeout** | 指定時間経過後に終了 |
| **First Best** | 最初の最適な応答で終了 |
| **Timeout with Override** | タイムアウトだが、より良い応答があれば上書き |
| **External Event** | 外部イベントにより終了 |

この例では最初の**Wait for All**を使用している。

---

## Scatter-Gather パターンとの関係

Recipient List (245) とAggregatorの組み合わせは、**Scatter-Gather (272)** パターンを実装する1つの方法である。Scatter-Gatherは**Publish-Subscribe Channel (154)** を使用してRecipient List (245) の代わりに実装することも可能。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scatter-Gather パターン                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  実装方法1: Recipient List + Aggregator                  │   │
│  │                                                          │   │
│  │  ┌────────────┐        ┌────────────┐                    │   │
│  │  │ Recipient  │───────▶│ Aggregator │                    │   │
│  │  │    List    │        │            │                    │   │
│  │  └────────────┘        └────────────┘                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  実装方法2: Publish-Subscribe Channel + Aggregator       │   │
│  │                                                          │   │
│  │  ┌────────────┐        ┌────────────┐                    │   │
│  │  │ Pub-Sub    │───────▶│ Aggregator │                    │   │
│  │  │  Channel   │        │            │                    │   │
│  │  └────────────┘        └────────────┘                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Composed Message Processor との関係

Recipient List (245) とAggregator（またはPublish-Subscribe Channelを使用する場合）は、より大きなパターンである**Composed Message Processor (270)** を構成する。

この構成の利点は、結合されたコンポーネントが**Pipes and Filters (135)** スタイルの単一フィルタとしてより容易に機能できることである。

```
┌─────────────────────────────────────────────────────────────────┐
│              Composed Message Processor (270)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Single Filter                          │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐             │   │
│  │  │           │  │           │  │           │             │   │
│  │  │ Recipient │  │ Processing│  │ Aggregator│             │   │
│  │  │   List    │─▶│           │─▶│           │             │   │
│  │  │           │  │           │  │           │             │   │
│  │  └───────────┘  └───────────┘  └───────────┘             │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Pipes and Filters スタイルでの利用が容易                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 参照

- Recipient List (245)
- Correlation Identifier (215)
- Event Message (207)
- Document Message (204)
- Message (130)
- Scatter-Gather (272)
- Publish-Subscribe Channel (154)
- Composed Message Processor (270)
- Pipes and Filters (135)