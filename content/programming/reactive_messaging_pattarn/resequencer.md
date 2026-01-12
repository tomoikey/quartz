# Resequencer

## 概念図

```
         ┌─────────────────────────────────────────┐
         │             Resequencer                 │
         │                                         │
         │    ┌─────┐      ┌─────┐                │
    ────▶│───▶│  3  │      │     │                │
         │    ├─────┤      │     │────────────────┼───▶ 1
    ────▶│───▶│  1  │─────▶│ ──▶ │────────────────┼───▶ 2
         │    ├─────┤      │     │────────────────┼───▶ 3
    ────▶│───▶│  2  │      │     │                │
         │    └─────┘      └─────┘                │
         │   (順不同)       (並べ替え)              │
         └─────────────────────────────────────────┘
```

---

## 定義

Resequencerは、元のシーケンスから外れたメッセージのセットを受信し、最終宛先に送信する前に必要なシーケンスに戻すパターンである。

---

## Akkaにおけるメッセージ順序保証

Request-Reply (209) の議論に関連して、メッセージ配信の順序について疑問が生じることがある。

### Akkaドキュメントからのメッセージ保証

Akkaおよび他のアクターモデルシステムでは、一般的に、あるアクターから別のアクターへの直接メッセージ送信の結果としてメッセージが受信されるシーケンスについて心配する必要はない。

Akkaドキュメント [Akka-Message-Guarantees] によると、あるアクターから別のアクターへの直接メッセージは、最初のアクターが送信した順序で常に受信される。

### シナリオ例

以下のシナリオを想定：
- アクターA1がメッセージM1、M2、M3をA2に送信
- アクターA3がメッセージM4、M5、M6をA2に送信

これらの2つのシナリオに基づく事実：

| 番号 | 事実 |
|-----|------|
| 1 | M1が配信される場合、M2とM3より前に配信される必要がある |
| 2 | M2が配信される場合、M3より前に配信される必要がある |
| 3 | M4が配信される場合、M5とM6より前に配信される必要がある |
| 4 | M5が配信される場合、M6より前に配信される必要がある |
| 5 | **A2はA1からのメッセージとA3からのメッセージがインターリーブされて見える可能性がある** |
| 6 | デフォルトでは保証配信がないため、メッセージのいずれかがドロップされる可能性がある（A2に到着しない） |

**結論**：あるアクターから別のアクターに直接送信される基本的なメッセージのシーケンスが順序通りに受信されないことを心配する必要はない。それは起こらない。

---

## Resequencerが必要なケース

複数の送信アクターが存在する複雑なメッセージルーティングシナリオでは、メッセージのシーケンスが順序通りに受信されない可能性がある（前述の#5、A1とA3からのメッセージがA2に到着する際にインターリーブされる）。

これは、例えばContent-Based Router (228) やSplitter (254) を使用する場合に発生する可能性がある。大きなメッセージが複数の小さなメッセージに分割され、小さなメッセージの内容に基づいてルーティングされる場合を想像してほしい。すべての細粒度メッセージを処理するプロセスの数と処理時間により、最終宛先で結果が順序通りに受信されない可能性が容易に生じる。

### Resequencerが必要な場合と不要な場合

| ケース | Resequencerの必要性 |
|-------|-------------------|
| Scatter-Gather (272) の例 | 不要な場合がある |
| 元のシーケンスに従って受信する必要がある場合 | **必要** |

---

## システム構成図（Figure 7.6）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│     ┌───┐      ┌───┐      ┌───┐                                         │
│     │ 1 │      │   │      │   │                                         │
│     └─┬─┘      │ 2 │      │ 3 │                                         │
│       │        └─┬─┘      └─┬─┘                                         │
│       │          │          │                                           │
│   ┌───┴───┐  ┌───┴───┐  ┌───┴───┐                                       │
│   │  📄   │  │  📄   │  │  📄   │                                       │
│   └───┬───┘  └───┬───┘  └───┬───┘                                       │
│       │          │          │                                           │
│       └──────────┼──────────┘                                           │
│                  │                                                      │
│                  ▼                                                      │
│           ┌─────────────┐                                               │
│           │             │                                               │
│           │ Resequencer │                                               │
│           │  ┌───────┐  │                                               │
│           │  │ □ ─▶ □│  │                                               │
│           │  │   □   │  │                                               │
│           │  └───────┘  │                                               │
│           └──────┬──────┘                                               │
│                  │                                                      │
│                  ▼                                                      │
│     ┌───┐      ┌───┐      ┌───┐                                         │
│     │ 3 │      │ 2 │      │ 1 │                                         │
│     └─┬─┘      └─┬─┘      └─┬─┘                                         │
│       │          │          │                                           │
│   ┌───┴───┐  ┌───┴───┐  ┌───┴───┐                                       │
│   │  📄   │  │  📄   │  │  📄   │                                       │
│   └───────┘  └───────┘  └───────┘                                       │
│                                                                         │
│   （順序が正しく並べ替えられた出力）                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

※ Message (130) が任意の順序で到着することが許容できない場合、
   Resequencerを使用して必要な順序に並べ替える
```

---

## 実装例（Scala/Akka）

### メッセージ定義

```scala
package co.vaughnvernon.reactiveenterprise.resequencer

import java.util.concurrent.TimeUnit
import java.util.Date
import scala.concurrent._
import scala.concurrent.duration._
import scala.util._
import ExecutionContext.Implicits.global
import akka.actor._
import co.vaughnvernon.reactiveenterprise._

// シーケンス付きメッセージ
case class SequencedMessage(
        correlationId: String,  // Correlation Identifier (215)
        index: Int,             // シーケンス内の位置
        total: Int)             // メッセージの総数

// 再シーケンス処理用のメッセージコレクション
case class ResequencedMessages(
        dispatchableIndex: Int,
        sequencedMessages: Array[SequencedMessage]) {

  def advancedTo(dispatchableIndex: Int) = {
    ResequencedMessages(
          dispatchableIndex,
          sequencedMessages)
  }
}
```

### ドライバアプリケーション

```scala
object Resequencer extends CompletableApp(10) {
  // 最終宛先（正しい順序でメッセージを受信する必要がある）
  val sequencedMessageConsumer = system.actorOf(
            Props[SequencedMessageConsumer],
            "sequencedMessageConsumer")

  // Resequencer（順序を並べ替える）
  val resequencerConsumer =
      system.actorOf(
          Props(classOf[ResequencerConsumer],
                  sequencedMessageConsumer),
          "resequencerConsumer")

  // ChaosRouter（意図的に順序を乱す）
  val chaosRouter = system.actorOf(
            Props(classOf[ChaosRouter],
                  resequencerConsumer),
            "chaosRouter")

  // ABCシーケンスのメッセージを送信
  for (index <- 1 to 5)
        chaosRouter !
            SequencedMessage("ABC", index, 5)

  // XYZシーケンスのメッセージを送信
  for (index <- 1 to 5)
        chaosRouter !
            SequencedMessage("XYZ", index, 5)

  awaitCompletion
  println("Resequencer: is completed.")
}
```

### 3つのアクターの役割

| アクター | 役割 |
|---------|------|
| **SequencedMessageConsumer** | 正しい順序でメッセージを受信する必要がある最終宛先 |
| **ResequencerConsumer** | 順序通りでないメッセージを受信し、正しい順序に戻す |
| **ChaosRouter** | 正しい順序で受信したメッセージを意図的に順序通りでなくする |

`SequencedMessage`には`correlationId`、メッセージシーケンスの`index`、メッセージの`total`数が含まれる。`correlationId`はCorrelation Identifier (215) で説明されている。

---

## ChaosRouter（意図的な順序乱し）

```scala
class ChaosRouter(consumer: ActorRef) extends Actor {
  val random = new Random((new Date()).getTime)

  def receive = {
    case sequencedMessage: SequencedMessage =>
      // 1〜100ミリ秒のランダムな遅延を設定
      val millis = random.nextInt(100) + 1
      println(s"ChaosRouter: delaying delivery↩
      of $sequencedMessage for $millis milliseconds")

      val duration =
          Duration.create(
            millis,
            TimeUnit.MILLISECONDS)

      // 遅延後にconsumer（ResequencerConsumer）に送信
      context.system.scheduler.scheduleOnce(
            duration,
            consumer,
            sequencedMessage)

    case message: Any =>
      println(s"ChaosRouter: unexpected: $message")
  }
}
```

`ChaosRouter`は直接のコンシューマー（`ResequencerConsumer`）への`ActorRef`を保持している。`ChaosRouter`の基本的な責任は、メッセージシーケンシングに混乱を生じさせることである。1〜100ミリ秒のランダムな時間のタイマーを設定し、タイマーが経過すると、関連する`SequencedMessage`をコンシューマーである`ResequencerConsumer`にディスパッチする。

---

## ResequencerConsumer（Resequencer本体）

```scala
class ResequencerConsumer(
        actualConsumer: ActorRef)
    extends Actor {

  // correlationId → ResequencedMessages のマップ
  val resequenced =
        scala.collection.mutable.Map[
          String,
          ResequencedMessages]()

  // 順序通りのメッセージをすべて配信
  def dispatchAllSequenced(
        correlationId: String) = {
    val resequencedMessages = resequenced(correlationId)
    var dispatchableIndex =
        resequencedMessages.dispatchableIndex

    resequencedMessages.sequencedMessages.map {
        sequencedMessage =>
      if (sequencedMessage.index == dispatchableIndex) {
        actualConsumer ! sequencedMessage
        dispatchableIndex += 1
      }
    }

    // 次に配信可能なインデックスを更新
    resequenced(correlationId) =
        resequencedMessages.advancedTo(dispatchableIndex)
  }

  // ダミーメッセージの配列を生成（プレースホルダー）
  def dummySequencedMessages(
        count: Int): Seq[SequencedMessage] = {
    for {
      index <- 1 to count
    } yield {
      SequencedMessage("", -1, count)
    }
  }

  def receive = {
    case unsequencedMessage: SequencedMessage =>
      println(s"ResequencerConsumer: received:↩
      $unsequencedMessage")
      resequence(unsequencedMessage)
      dispatchAllSequenced(unsequencedMessage.correlationId)
      removeCompleted(unsequencedMessage.correlationId)

    case message: Any =>
      println(s"ResequencerConsumer: unexpected: $message")
  }

  // 完了したシーケンスを削除
  def removeCompleted(correlationId: String) = {
    val resequencedMessages = resequenced(correlationId)

    if (resequencedMessages.dispatchableIndex >
        resequencedMessages.sequencedMessages(0).total) {
      resequenced.remove(correlationId)
      println(s"ResequencerConsumer: removed completed:↩
      $correlationId")
    }
  }

  // メッセージを正しい位置に配置
  def resequence(
        sequencedMessage: SequencedMessage) = {
    // 新しいcorrelationIdの場合、配列を初期化
    if (!resequenced.contains(
          sequencedMessage.correlationId)) {
      resequenced(sequencedMessage.correlationId) =
          ResequencedMessages(
              1,
              dummySequencedMessages(
                  sequencedMessage.total).toArray)
    }

    // 正しいインデックス位置にメッセージを配置
    resequenced(sequencedMessage.correlationId)
        .sequencedMessages
        .update(sequencedMessage.index - 1,
              sequencedMessage)
  }
}
```

### ResequencerConsumerの動作

`correlationId`に基づいて、`ResequencerConsumer`は受信した各`SequencedMessage`を徐々に元の正しいシーケンスに配置する。`SequencedMessageConsumer`が受信する必要がある順序で`SequencedMessage`インスタンスを持つとすぐに、それらをディスパッチする。

メッセージの受信順序に応じて、これは単一のバースト、複数のバースト、または1つずつ発生する可能性がある。例えば、シーケンス5、4、3、2、そして1のメッセージが受信された場合、メソッド`dispatchAllSequenced()`がインデックス1のメッセージを見ると、メッセージ1〜5のバーストを`actualConsumer`に送信する。

---

## SequencedMessageConsumer（最終宛先）

```scala
class SequencedMessageConsumer extends Actor {
  def receive = {
    case sequencedMessage: SequencedMessage =>
      println(s"SequencedMessageConsumer: received:↩
      $sequencedMessage")
      Resequencer.completedStep()

    case message: Any =>
      println(s"SequencedMessageConsumer: unexpected:↩
      $message")
  }
}
```

`ResequencerConsumer`の`actualConsumer`は単純な`SequencedMessageConsumer`である。受信した各`SequencedMessage`を表示するだけで、正しいシーケンス順序で受信されていることを証明する。

---

## 処理フロー図

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Resequencer 処理フロー                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. メッセージ送信（順序通り）                                           │
│                                                                         │
│  ┌──────────────┐                                                       │
│  │   Driver     │  ABC: 1,2,3,4,5                                       │
│  │              │────────────────────▶ ChaosRouter                      │
│  │              │  XYZ: 1,2,3,4,5                                       │
│  └──────────────┘                                                       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  2. 意図的な順序乱し                                                     │
│                                                                         │
│  ┌──────────────┐                                                       │
│  │ ChaosRouter  │                                                       │
│  │              │  ランダム遅延（1-100ms）後に配信                       │
│  │ scheduler    │                                                       │
│  │ .scheduleOnce│──────────────────▶ ResequencerConsumer               │
│  └──────────────┘                                                       │
│                                                                         │
│  到着順序例: XYZ-4, ABC-3, XYZ-5, ABC-5, XYZ-1, ABC-2...                │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  3. 再シーケンス処理                                                     │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │ ResequencerConsumer                                          │       │
│  │                                                              │       │
│  │  resequenced["ABC"] = [_, _, msg3, _, msg5]                  │       │
│  │  resequenced["XYZ"] = [msg1, _, _, msg4, msg5]               │       │
│  │                                                              │       │
│  │  dispatchableIndex: 次に配信可能なインデックス               │       │
│  │                                                              │       │
│  │  dispatchAllSequenced():                                     │       │
│  │    インデックス1から順番に、                                  │       │
│  │    配信可能なメッセージをすべて送信                          │       │
│  │                                                              │       │
│  └──────────────────────────────────────────────────────────────┘       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  4. 正しい順序で最終配信                                                 │
│                                                                         │
│                          SequencedMessageConsumer                       │
│                                    │                                    │
│                                    ▼                                    │
│                          ABC-1, ABC-2, ABC-3...                         │
│                          XYZ-1, XYZ-2, XYZ-3...                         │
│                          （正しい順序で受信）                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 実行結果（抜粋）

```
ChaosRouter: delaying delivery of SequencedMessage(↩
ABC,1,5) for 14 milliseconds
ChaosRouter: delaying delivery of SequencedMessage(↩
ABC,2,5) for 71 milliseconds
ChaosRouter: delaying delivery of SequencedMessage(↩
ABC,3,5) for 1 milliseconds
ChaosRouter: delaying delivery of SequencedMessage(↩
XYZ,5,5) for 63 milliseconds
...
ResequencerConsumer: received: SequencedMessage(XYZ,4,5)
ResequencerConsumer: received: SequencedMessage(ABC,3,5)
ResequencerConsumer: received: SequencedMessage(XYZ,5,5)
ResequencerConsumer: received: SequencedMessage(ABC,5,5)
ResequencerConsumer: received: SequencedMessage(XYZ,1,5)
ResequencerConsumer: received: SequencedMessage(ABC,2,5)
SequencedMessageConsumer: received: SequencedMessage(↩
XYZ,1,5)
ResequencerConsumer: received: SequencedMessage(XYZ,2,5)
SequencedMessageConsumer: received: SequencedMessage(↩
XYZ,2,5)
SequencedMessageConsumer: received: SequencedMessage(↩
ABC,1,5)
SequencedMessageConsumer: received: SequencedMessage(↩
ABC,2,5)
SequencedMessageConsumer: received: SequencedMessage(↩
ABC,3,5)
ResequencerConsumer: received: SequencedMessage(XYZ,3,5)
ResequencerConsumer: removed completed: XYZ
SequencedMessageConsumer: received: SequencedMessage(↩
XYZ,3,5)
SequencedMessageConsumer: received: SequencedMessage(↩
XYZ,4,5)
ResequencerConsumer: received: SequencedMessage(ABC,4,5)
ResequencerConsumer: removed completed: ABC
SequencedMessageConsumer: received: SequencedMessage(↩
XYZ,5,5)
SequencedMessageConsumer: received: SequencedMessage(↩
ABC,4,5)
SequencedMessageConsumer: received: SequencedMessage(↩
ABC,5,5)
Resequencer: is completed.
```

### 結果の解説

- `ChaosRouter`がランダムな遅延でメッセージを順不同に配信
- `ResequencerConsumer`がバラバラに到着するメッセージを受信
- `SequencedMessageConsumer`には正しい順序（1,2,3,4,5）でメッセージが配信される
- ABCとXYZはそれぞれ独立した`correlationId`で管理される

---

## バースト配信の動作

メッセージの受信順序に応じて、配信は以下のように発生する可能性がある：

| パターン | 説明 |
|---------|------|
| **単一バースト** | インデックス5,4,3,2,1の順で受信した場合、インデックス1を見た時点で1〜5をまとめて配信 |
| **複数バースト** | インデックス2,1,4,3,5の順で受信した場合、1を見た時点で1-2を配信、3を見た時点で3-4を配信、5を見た時点で5を配信 |
| **1つずつ** | インデックス1,2,3,4,5の順で受信した場合、それぞれ即座に配信 |

---

## 参照

- Request-Reply (209)
- Correlation Identifier (215)
- Content-Based Router (228)
- Splitter (254)
- Scatter-Gather (272)
- Message (130)
- Akka-Message-Guarantees