# Process Manager パターン

## 概要

メッセージを複数の処理ステップを通してルーティングする。必要なステップが**設計時に不明**な場合や、**シーケンシャルでない**場合にも対応可能。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Process Manager パターン                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                            ┌─────┐                                      │
│                            │Step │                                      │
│                           /└──┬──┘                                      │
│                          /    │                                         │
│         ┌───────────────┐     │                                         │
│         │    Process    │─────┼─────┐                                   │
│         │    Manager    │     │     │                                   │
│         └───────────────┘\    │     │                                   │
│                           \   v     v                                   │
│                            ┌─────┐ ┌─────┐ ┌─────┐                      │
│                            │Step │ │Step │ │Step │  ← 並列処理可能      │
│                            └─────┘ └─────┘ └─────┘                      │
│                                                                         │
│   ・条件分岐、ループ、並列処理をサポート                                 │
│   ・各ステップの順序は動的に決定可能                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

## Routing Slip との比較

| 観点 | Routing Slip (285) | Process Manager |
|------|-------------------|-----------------|
| 処理ステップ | 固定、線形 | 動的、非線形可能 |
| 条件分岐 | 不可 | 可能 |
| ループ | 不可 | 可能 |
| 並列処理 | 不可 | 可能 |
| 選択基準 | シンプルな直列処理 | 複雑なフロー制御が必要な場合 |

---

## 実装アプローチ（2種類）

### 1. DSL + インタプリタ方式

```
┌────────────────────────────────────────────────────────────────┐
│  DSL (Domain-Specific Language) + インタプリタ                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────┐      ┌──────────────────┐               │
│  │  プロセス定義     │ ───> │  インタプリタ     │               │
│  │  (例: BPEL)      │      │  (汎用的)        │               │
│  └──────────────────┘      └──────────────────┘               │
│                                                                │
│  特徴:                                                         │
│  ・Process Manager はビジネスドメインに特化しない               │
│  ・高レベルプログラミング言語のコンパイラ/インタプリタと同様     │
│  ・高度に再利用可能                                            │
└────────────────────────────────────────────────────────────────┘
```

### 2. ドメイン固有 Process Manager（本の例）

```
┌────────────────────────────────────────────────────────────────┐
│  ドメイン固有 Process Manager                                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────┐     │
│  │              LoanBroker (Process Manager)            │     │
│  │  ・銀行ローン見積りリクエスト専用に構築               │     │
│  │  ・フロー制御ルールを直接コード化                     │     │
│  └──────────────────────────────────────────────────────┘     │
│                          │                                     │
│                          v                                     │
│  ┌──────────────────────────────────────────────────────┐     │
│  │         LoanRateQuote (Process Instance Entity)      │     │
│  │  ・プロセスの状態を維持                               │     │
│  │  ・プロセス状態に基づき遷移方向を提供                 │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
│  Domain-Driven Design アプローチ [IDDD] に適している           │
└────────────────────────────────────────────────────────────────┘
```

---

## 例：銀行ローン見積りシステム（Figure 7.10）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         銀行ローン見積りシステム                             │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌──────────────┐
                              │ CreditBureau │
                              └──────┬───────┘
                                     │ 1 (CheckCredit/CreditChecked)
                                     │
  ┌─────────────┐                    v                        ┌─────────┐
  │   Process   │              ┌──────────────┐          ┌───>│  Bank1  │
  │   Entity    │<─────────────│  LoanBroker  │          │    └─────────┘
  │(LoanRate    │              │   (Process   │──────────┤          2
  │   Quote)    │──────────────│   Manager)   │    2     │    ┌─────────┐
  └─────────────┘              └──────────────┘          ├───>│  Bank2  │
        ^                            ^                   │    └─────────┘
        │                            │                   │          2
        │                      ┌─────┴─────┐             │    ┌─────────┐
        │                      │QuoteBest  │             └───>│  Bank3  │
        │                      │ LoanRate  │                  └─────────┘
        │                      └───────────┘
        │
        └─ 各銀行への処理ステップは並列（同じシーケンス番号 2）
```

---

## Table 7.1: コンポーネント一覧

| Actor | 説明 |
|-------|------|
| **LoanBroker** | Process Manager。*Message Broker (308)* として実装。`QuoteBestLoanRate` メッセージで見積りリクエストを受信し、最終的に `BestLoanRateQuoted` または `BestLoanRateDenied` を出力 |
| **LoanRateQuote** | プロセスインスタンスエンティティ。プロセス状態を維持し、遷移方向を Process Manager に提供。`QuoteBestLoanRate` ごとに1つ作成され、ローン見積りプロセスの最初から最後まで使用。すべての銀行見積りを収集し、全て受信したら最良の見積りを `LoanBroker` に通知。メッセージ *Aggregator (257)* として機能 |
| **CreditBureau** | ローン見積りをリクエストする各当事者のクレジットスコアを提供。`CheckCredit` *Command Message (202)* を受け取り、`CreditChecked` *Event Message (207)* で応答 |
| **Bank1, Bank2, Bank3** | 実際には1つの `Bank` アクタータイプだが、ドライバーアプリケーションが3つのインスタンスを作成。異なる構成とランダムな乗数により、各銀行は `QuoteBestLoanRate` リクエストごとに異なるローン金利を見積もる |

---

## 集中化 vs 分散化

一般的には集中制御ポイントを避けることが好ましいが、Process Manager は集中化する傾向がある。ただし、分散化も可能。

```
┌─────────────────────────────────────────────────────────────────┐
│  分散 Process Manager の実現方法                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DistributedPubSubMediator を使用                               │
│  (Publish-Subscribe Channel (154) で説明)                       │
│                                                                 │
│  本の例は集中化されたアクターに基づいた設計                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## ドライバーアプリケーション

```scala
object ProcessManagerDriver extends CompletableApp(5) {
  val creditBureau =
      system.actorOf(
              Props[CreditBureau],
              "creditBureau")

  val bank1 =
      system.actorOf(
              Props(classOf[Bank], "bank1", 2.75, 0.30),
              "bank1")
  val bank2 =
      system.actorOf(
              Props(classOf[Bank], "bank2", 2.73, 0.31),
              "bank2")
  val bank3 =
      system.actorOf(
              Props(classOf[Bank], "bank3", 2.80, 0.29),
              "bank3")

  val loanBroker = system.actorOf(
    Props(classOf[LoanBroker],
            creditBureau,
            Vector(bank1, bank2, bank3)),
    "loanBroker")

  loanBroker ! QuoteBestLoanRate("111-11-1111", 100000, 84)
  awaitCompletion
}
```

**処理の流れ:**
1. `CreditBureau` アクターを作成
2. 3つの `Bank` アクターを作成（それぞれ異なるプライムレートとプレミアム）
3. `LoanBroker`（Process Manager）を作成し、`CreditBureau` と各 `Bank` への参照を渡す
4. `QuoteBestLoanRate` メッセージを `LoanBroker` に送信（**トリガーメッセージ**）

クライアントから見ると、単純な *Request-Reply (209)* に見える。

---

## ProcessManager 抽象基底クラス

`LoanBroker` は `ProcessManager` 抽象基底クラスを継承。この基底クラスは `Actor` を継承し、基本的な再利用可能な Process Manager 動作を提供。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ProcessManager 基底クラス                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  abstract class ProcessManager extends Actor                    │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  private var processes = Map[String, ActorRef]()                │   │
│  │                                                                 │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │ processOf(processId): ActorRef                          │   │   │
│  │  │   → プロセスエンティティアクターへの参照を取得           │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │ startProcess(processId, process): Unit                  │   │   │
│  │  │   → プロセスインスタンスエンティティを永続化             │   │   │
│  │  │   → self ! ProcessStarted(processId, process)           │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │ stopProcess(processId): Unit                            │   │   │
│  │  │   → 永続化された状態を削除                               │   │   │
│  │  │   → self ! ProcessStopped(processId, process)           │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  永続化: この実装ではインメモリのみだが、データベース永続化も可能       │
│                                                                         │
│  ライフサイクルメッセージ:                                              │
│  ・ProcessStarted → 具体的な Process Manager で特別な処理を実行可能     │
│  ・ProcessStopped → プロセスインスタンスエンティティアクターを停止など  │
└─────────────────────────────────────────────────────────────────────────┘
```

```scala
case class ProcessStarted(
    processId: String,
    process: ActorRef)

case class ProcessStopped(
    processId: String,
    process: ActorRef)

abstract class ProcessManager extends Actor {
  private var processes = Map[String, ActorRef]()
  val log: LoggingAdapter = Logging.getLogger(context.system, self)

  def processOf(processId: String): ActorRef = {
    if (processes.contains(processId)) {
      processes(processId)
    } else {
      null
    }
  }

  def startProcess(processId: String, process: ActorRef) = {
    if (!processes.contains(processId)) {
      processes = processes + (processId -> process)
      self ! ProcessStarted(processId, process)
    }
  }

  def stopProcess(processId: String) = {
    if (processes.contains(processId)) {
      val process = processes(processId)
      processes = processes - processId
      self ! ProcessStopped(processId, process)
    }
  }
}
```

---

## LoanBroker の外部コントラクト

```scala
// 入力: 見積りリクエスト (Command Message)
case class QuoteBestLoanRate(
    taxId: String,
    amount: Integer,
    termInMonths: Integer)

// 出力: 成功時 (Event Message)
case class BestLoanRateQuoted(
    bankId: String,
    loanRateQuoteId: String,
    taxId: String,
    amount: Integer,
    termInMonths: Integer,
    creditScore: Integer,
    interestRate: Double)

// 出力: 失敗時 (Event Message)
case class BestLoanRateDenied(
    loanRateQuoteId: String,
    taxId: String,
    amount: Integer,
    termInMonths: Integer,
    creditScore: Integer)
```

---

## プロセスフロー（Figure 7.11）

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              Process Manager のアクティビティ                            │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│   Receive    │
│ QuoteBest    │
│  LoanRate    │
└──────┬───────┘
       │
       v
┌──────────────┐         ┌─────────────┐
│    Create    │         │ CheckCredit │
│ LoanRateQuote│         └──────┬──────┘
└──────┬───────┘                │
       │                        v
       v                 ┌─────────────┐
┌──────────────┐         │ CreditCheck │
│    Start     │         │     ed      │
│   Process    │         └──────┬──────┘
└──────┬───────┘                │
       │                        v
       v                 ┌──────────────────────┐
┌──────────────┐         │EstablishCreditScore  │
│  StartLoan   │         │  ForLoanRateQuote    │
│  RateQuote   │         └──────────┬───────────┘
└──────┬───────┘                    │
       │                    ┌───────┴───────┐
       v                    │               │
┌──────────────┐    credit score    credit score
│  LoanRate    │      <= 399          >= 400
│ QuoteStarted │            │               │
└──────────────┘            v               v
                   ┌────────────────┐ ┌────────────────┐
                   │CreditScoreFor  │ │CreditScoreFor  │
                   │LoanRateQuote   │ │LoanRateQuote   │
                   │    Denied      │ │  Established   │
                   └───────┬────────┘ └───────┬────────┘
                           │                  │
                           │                  │    Send QuoteLoanRate
                           │                  │    to all Banks (並列)
                           │                  v
                           │          ┌─────────────────┐
                           │          │  Bank1/2/3      │
                           │          │ QuoteLoanRate   │
                           │          │   Received      │
                           │          └────────┬────────┘
                           │                   │
                           │                   v   Aggregated
                           │          ┌─────────────────┐
                           │          │  BankLoan       │
                           │          │  RateQuoted     │
                           │          └────────┬────────┘
                           │                   │
                           │                   v
                           │          ┌─────────────────┐
                           │          │  LoanRateBest   │
                           │          │  QuoteFilled    │
                           │          └────────┬────────┘
                           │                   │
                           v                   v
                   ┌────────────────┐  ┌────────────────┐
                   │  BestLoanRate  │  │  BestLoanRate  │
                   │     Denied     │  │    Quoted      │
                   └───────┬────────┘  └───────┬────────┘
                           │                   │
                           └─────────┬─────────┘
                                     │
                                     v
                                   (●) 終了
```

---

## LoanBroker 実装

```scala
class LoanBroker(
    creditBureau: ActorRef,
    banks: Seq[ActorRef])
extends ProcessManager {

  def receive = {
    // 見積りリクエスト受信 → プロセス開始
    case message: QuoteBestLoanRate =>
      val loanRateQuoteId = LoanRateQuote.id
      log.info(s"$message for: $loanRateQuoteId")

      val loanRateQuote =
          LoanRateQuote(
                context.system,
                loanRateQuoteId,
                message.taxId,
                message.amount,
                message.termInMonths,
                self)

      startProcess(loanRateQuoteId, loanRateQuote)

    // プロセス開始完了 → LoanRateQuote に開始指示
    case message: ProcessStarted =>
      log.info(s"$message")
      message.process ! StartLoanRateQuote(banks.size)

    // LoanRateQuote 開始完了 → CreditBureau に与信チェック依頼
    case message: LoanRateQuoteStarted =>
      log.info(s"$message")
      creditBureau ! CheckCredit(
          message.loanRateQuoteId,
          message.taxId)

    // 与信チェック完了 → LoanRateQuote にクレジットスコア設定
    case message: CreditChecked =>
      log.info(s"$message")
      processOf(message.creditProcessingReferenceId) !
        EstablishCreditScoreForLoanRateQuote(
            message.creditProcessingReferenceId,
            message.taxId,
            message.score)

    // クレジットスコア不足で拒否 → プロセス終了、拒否応答
    case message: CreditScoreForLoanRateQuoteDenied =>
      log.info(s"$message")
      processOf(message.loanRateQuoteId) !
          TerminateLoanRateQuote()
      ProcessManagerDriver.completeAll
      val denied =
        BestLoanRateDenied(
            message.loanRateQuoteId,
            message.taxId,
            message.amount,
            message.termInMonths,
            message.score)
      log.info(s"Would be sent to original requester: $denied")

    // クレジットスコア確立 → 全銀行に見積り依頼
    case message: CreditScoreForLoanRateQuoteEstablished =>
      log.info(s"$message")
      banks map { bank =>
        bank ! QuoteLoanRate(
          message.loanRateQuoteId,
          message.taxId,
          message.score,
          message.amount,
          message.termInMonths)
      }
      ProcessManagerDriver.completedStep

    // 銀行から見積り受信 → LoanRateQuote に記録
    case message: BankLoanRateQuoted =>
      log.info(s"$message")
      processOf(message.loadQuoteReferenceId) !
        RecordLoanRateQuote(
            message.bankId,
            message.bankLoanRateQuoteId,
            message.interestRate)

    // 見積り記録完了
    case message: LoanRateQuoteRecorded =>
      log.info(s"$message")
      ProcessManagerDriver.completedStep

    // 最良見積り確定 → プロセス終了、成功応答
    case message: LoanRateBestQuoteFilled =>
      log.info(s"$message")
      ProcessManagerDriver.completedStep
      stopProcess(message.loanRateQuoteId)
      val best = BestLoanRateQuoted(
          message.bestBankLoanRateQuote.bankId,
          message.loanRateQuoteId,
          message.taxId,
          message.amount,
          message.termInMonths,
          message.creditScore,
          message.bestBankLoanRateQuote.interestRate)
      log.info(s"Would be sent to original requester: $best")

    // LoanRateQuote 終了通知
    case message: LoanRateQuoteTerminated =>
      log.info(s"$message")
      stopProcess(message.loanRateQuoteId)

    // プロセス停止 → プロセスインスタンスエンティティアクター停止
    case message: ProcessStopped =>
      log.info(s"$message")
      context.stop(message.process)
  }
}
```

---

## LoanRateQuote（プロセスインスタンスエンティティ）

### メッセージ定義

```scala
// LoanRateQuote のコントラクトメッセージ
case class StartLoanRateQuote(
    expectedLoanRateQuotes: Integer)

case class LoanRateQuoteStarted(
    loanRateQuoteId: String,
    taxId: String)

case class TerminateLoanRateQuote()

case class LoanRateQuoteTerminated(
    loanRateQuoteId: String,
    taxId: String)

case class EstablishCreditScoreForLoanRateQuote(
    loanRateQuoteId: String,
    taxId: String,
    score: Integer)

case class CreditScoreForLoanRateQuoteEstablished(
    loanRateQuoteId: String,
    taxId: String,
    score: Integer,
    amount: Integer,
    termInMonths: Integer)

case class CreditScoreForLoanRateQuoteDenied(
    loanRateQuoteId: String,
    taxId: String,
    amount: Integer,
    termInMonths: Integer,
    score: Integer)

case class RecordLoanRateQuote(
    bankId: String,
    bankLoanRateQuoteId: String,
    interestRate: Double)

case class LoanRateQuoteRecorded(
    loanRateQuoteId: String,
    taxId: String,
    bankLoanRateQuote: BankLoanRateQuote)

case class LoanRateBestQuoteFilled(
    loanRateQuoteId: String,
    taxId: String,
    amount: Integer,
    termInMonths: Integer,
    creditScore: Integer,
    bestBankLoanRateQuote: BankLoanRateQuote)

case class BankLoanRateQuote(
    bankId: String,
    bankLoanRateQuoteId: String,
    interestRate: Double)
```

### コンパニオンオブジェクトと実装

```scala
object LoanRateQuote {
  val randomLoanRateQuoteId = new Random()

  def apply(
      system: ActorSystem,
      loanRateQuoteId: String,
      taxId: String,
      amount: Integer,
      termInMonths: Integer,
      loanBroker: ActorRef): ActorRef = {
    val loanRateQuote =
      system.actorOf(
        Props(
            classOf[LoanRateQuote],
            loanRateQuoteId, taxId,
            amount, termInMonths, loanBroker),
        "loanRateQuote-" + loanRateQuoteId)

    loanRateQuote
  }

  def id() = {
    randomLoanRateQuoteId.nextInt(1000).toString
  }
}

class LoanRateQuote(
    loanRateQuoteId: String,
    taxId: String,
    amount: Integer,
    termInMonths: Integer,
    loanBroker: ActorRef)
extends Actor {

  var bankLoanRateQuotes = Vector[BankLoanRateQuote]()
  var creditRatingScore: Int = _
  var expectedLoanRateQuotes: Int = _

  // 最良の銀行見積りを選択
  private def bestBankLoanRateQuote() = {
    var best = bankLoanRateQuotes(0)

    bankLoanRateQuotes map { bankLoanRateQuote =>
      if (best.interestRate >
          bankLoanRateQuote.interestRate) {
        best = bankLoanRateQuote
      }
    }

    best
  }

  // クレジットスコアが見積り可能か判定（399より大きい必要あり）
  private def quotableCreditScore(
      score: Integer): Boolean = {
    score > 399
  }

  def receive = {
    // 開始指示受信 → 期待される見積り数を記録、開始通知
    case message: StartLoanRateQuote =>
      expectedLoanRateQuotes =
        message.expectedLoanRateQuotes
      loanBroker !
        LoanRateQuoteStarted(
            loanRateQuoteId,
            taxId)

    // クレジットスコア設定 → スコアに応じて分岐
    case message: EstablishCreditScoreForLoanRateQuote =>
      creditRatingScore = message.score
      if (quotableCreditScore(creditRatingScore))
        loanBroker !
          CreditScoreForLoanRateQuoteEstablished(
              loanRateQuoteId,
              taxId,
              creditRatingScore,
              amount,
              termInMonths)
      else
        loanBroker !
          CreditScoreForLoanRateQuoteDenied(
              loanRateQuoteId,
              taxId,
              amount,
              termInMonths,
              creditRatingScore)

    // 銀行見積り記録 → 全て揃ったら最良見積りを通知
    case message: RecordLoanRateQuote =>
      val bankLoanRateQuote =
        BankLoanRateQuote(
                message.bankId,
                message.bankLoanRateQuoteId,
                message.interestRate)
      bankLoanRateQuotes =
        bankLoanRateQuotes :+ bankLoanRateQuote
      loanBroker !
        LoanRateQuoteRecorded(
            loanRateQuoteId,
            taxId,
            bankLoanRateQuote)

      if (bankLoanRateQuotes.size >=
            expectedLoanRateQuotes)
        loanBroker !
          LoanRateBestQuoteFilled(
              loanRateQuoteId,
              taxId,
              amount,
              termInMonths,
              creditRatingScore,
              bestBankLoanRateQuote)

    // 終了指示 → 終了通知
    case message: TerminateLoanRateQuote =>
      loanBroker !
        LoanRateQuoteTerminated(
            loanRateQuoteId,
            taxId)
  }
}
```

### ビジネスルール

```
┌─────────────────────────────────────────────────────────────────┐
│                    クレジットスコアによる分岐                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   quotableCreditScore(score) = score > 399                     │
│                                                                 │
│   ┌─────────────────┐                                          │
│   │ Credit Score    │                                          │
│   └────────┬────────┘                                          │
│            │                                                    │
│     ┌──────┴──────┐                                            │
│     │             │                                            │
│   <= 399        >= 400                                         │
│     │             │                                            │
│     v             v                                            │
│  ┌──────────┐  ┌──────────────────────────┐                   │
│  │  拒否    │  │  銀行への見積り依頼      │                   │
│  │ Denied   │  │  (並列で全銀行に送信)    │                   │
│  └──────────┘  └──────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## CreditBureau 実装

```scala
case class CheckCredit(
    creditProcessingReferenceId: String,
    taxId: String)

case class CreditChecked(
    creditProcessingReferenceId: String,
    taxId: String,
    score: Integer)

class CreditBureau extends Actor {
  val creditRanges = Vector(300, 400, 500, 600, 700)
  val randomCreditRangeGenerator = new Random()
  val randomCreditScoreGenerator = new Random()

  def receive = {
    case message: CheckCredit =>
      val range =
          creditRanges(
              randomCreditRangeGenerator.nextInt(5))
      val score =
          range
          + randomCreditScoreGenerator.nextInt(20)

      sender !
        CreditChecked(
            message.creditProcessingReferenceId,
            message.taxId,
            score)
  }
}
```

**ランダムなクレジットスコア生成:**
- 範囲: 300, 400, 500, 600, 700 からランダムに選択
- スコア: 選択された範囲 + 0〜19 のランダム値

---

## Bank 実装

```scala
case class QuoteLoanRate(
    loadQuoteReferenceId: String,
    taxId: String,
    creditScore: Integer,
    amount: Integer,
    termInMonths: Integer)

case class BankLoanRateQuoted(
    bankId: String,
    bankLoanRateQuoteId: String,
    loadQuoteReferenceId: String,
    taxId: String,
    interestRate: Double)

class Bank(
    bankId: String,
    primeRate: Double,
    ratePremium: Double)
extends Actor {

  val randomDiscount = new Random()
  val randomQuoteId = new Random()

  private def calculateInterestRate(
      amount: Double,
      months: Double,
      creditScore: Double): Double = {

    val creditScoreDiscount = creditScore / 100.0 / 10.0 -
            (randomDiscount.nextInt(5) * 0.05)

    primeRate + ratePremium + ((months / 12.0) / 10.0) -
            creditScoreDiscount
  }

  def receive = {
    case message: QuoteLoanRate =>
      val interestRate =
        calculateInterestRate(
            message.amount.toDouble,
            message.termInMonths.toDouble,
            message.creditScore.toDouble)

      sender ! BankLoanRateQuoted(
        bankId, randomQuoteId.nextInt(1000).toString,
        message.loadQuoteReferenceId, message.taxId, interestRate)
  }
}
```

**金利計算式:**
```
金利 = プライムレート + レートプレミアム + (月数 / 12.0) / 10.0 - クレジットスコア割引

クレジットスコア割引 = クレジットスコア / 100.0 / 10.0 - (ランダム(0-4) * 0.05)
```

`CreditBureau` と `Bank` はどちらもランダムな結果を提供するように実装されている。`Bank` は個人のクレジットスコアを使用して可能な割引を決定し、クレジットスコアが高いほど良い割引が得られる。

---

## 成功時の出力例

```
QuoteBestLoanRate(111-11-1111,100000,84)  for: 151
ProcessStarted(151,Actor[akka://reactiveenterprise/user/
  loanRateQuote-151])
LoanRateQuoteStarted(151,111-11-1111)
CreditChecked(151,111-11-1111,610)
CreditScoreForLoanRateQuoteEstablished(151,111-11-1111,
  610,100000,84)
BankLoanRateQuoted(bank2,853,151,111-11-1111,3.33)
BankLoanRateQuoted(bank3,911,151,111-11-1111,3.23)
BankLoanRateQuoted(bank1,292,151,111-11-1111,3.34)
LoanRateQuoteRecorded(151,111-11-1111,BankLoanRateQuote(
  bank2,853,3.33))
LoanRateQuoteRecorded(151,111-11-1111,BankLoanRateQuote(
  bank3,911,3.23))
LoanRateQuoteRecorded(151,111-11-1111,BankLoanRateQuote(
  bank1,292,3.34))
LoanRateBestQuoteFilled(151,111-11-1111,100000,84,610,BankLoanRateQuote(bank3,911,3.23))
Would be sent to original requester: BestLoanRateQuoted(
  bank3,151,111-11-1111,100000,84,610,3.23)
ProcessStopped(151,Actor[akka://reactiveenterprise/user/
  loanRateQuote-151])
```

**結果解説:**
- クレジットスコア: 610（400以上なので見積り可能）
- 3つの銀行から見積り受信（並列処理）
    - bank1: 3.34%
    - bank2: 3.33%
    - bank3: 3.23%（最良）
- 最良見積り: bank3 の 3.23%