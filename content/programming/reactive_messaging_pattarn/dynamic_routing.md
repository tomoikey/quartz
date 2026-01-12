# Dynamic Router

## 概念図

```
              ┌─────────────────────────────────────────┐
              │           Dynamic Router                │
              │                                         │
              │    ┌─────┐      ┌─────┐                │
     ─────────┼───▶│     │      │     │────────────────┼───▶
              │    │  R  │ .... │  R  │                │
              │    │  U  │      │  U  │                │
              │    │  L  │      │  L  │                │
              │    │  E  │      │  E  │                │
              │    └─────┘      └─────┘                │
              │         ▲                              │
              │         │                              │
              │    ┌────┴────┐                         │
              │    │ Control │                         │
              │    │ Channel │                         │
              └────┴─────────┴─────────────────────────┘
                        ▲
                        │
                   登録/解除
```

---

## 定義

Dynamic Routerは、ルールベースを使用してメッセージをルーティングするパターンである。Splitter (254) や Content-Based Router (228) と比較して、より挑戦的な設計を提供する。動的（dynamic）な要素があり、ルールを使用するため興味深い側面を持つ。

---

## 他のルーティングパターンとの比較

| パターン | 特徴 |
|---------|------|
| Splitter | 比較的シンプル |
| Content-Based Router | 比較的シンプル |
| **Dynamic Router** | より挑戦的な設計、動的なルール、複数の可動部品 |

Dynamic Routerのルールは、SplitterやContent-Based Routerで観察されるものより改善されているが、それほど複雑ではない。

---

## 登録プロセス

Dynamic Routerからメッセージを受信するには、アクターが特定のメッセージに対する関心を登録する必要がある。

### 基本的な登録プロセス
最も基本的な登録プロセスでは、アクターが特定のメッセージタイプに関心があることをDynamic Routerに伝える。

### 高度な登録プロセス
より複雑なルールを使用して、Dynamic Routerがマルチレベルルックアップを実行し、特定のメッセージを受信するアクターを解決することも可能。

---

## システム構成図（Figure 7.3）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ┌─────────┐                                                            │
│  │  TypeC  │     Input          TypedMessage              Output        │
│  │ Message │    Channel        InterestRouter            Channel        │
│  │  ┌───┐  │                   ┌───────────┐                            │
│  │  │ 📄│  │─────────────────▶│  ┌─┬─┬─┐  │─────────────────▶  (A)     │
│  │  └───┘  │                   │  │ │:│ │  │                            │
│  └─────────┘                   │  ├─┼─┼─┤  │        Output              │
│                                │  │ │ │ │  │       Channel              │
│                                │  └─┴─┴─┘  │─────────────────▶  (B)     │
│                                └─────┬─────┘                            │
│                                      │              Output              │
│                                      │             Channel              │
│                                      │      ─────────────────▶  (C)     │
│                                      │                                  │
│                           ┌──────────┴──────────┐                       │
│                           │                     │                       │
│                           ▼                     │                       │
│                    ┌─────────────┐              │                       │
│                    │   Dynamic   │              │                       │
│                    │  Rule Base  │◀─────────────┘                       │
│                    │   ┌─────┐   │         Control                      │
│                    │   │ DB  │   │         Channel                      │
│                    │   └─────┘   │                                      │
│                    └─────────────┘                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

※ Dynamic Routerは関心を登録したアクターにのみメッセージをルーティングする
```

---

## ルールベースの設計

### Enterprise Integration Patterns [EIP] のアプローチ
- **last-registration-wins** ルールベースを採用
- 特定のメッセージタイプに対して最後に登録した関係者が、そのタイプのすべてのメッセージを受信
- ルールはメッセージの名前のみ
- ハッシュテーブルがメッセージタイプをキー、受信者のキュー名を値として格納
- 最新の登録者のキュー名が、以前に関連付けられていた値を置き換える

### 本例のアプローチ
- キュー名ではなく**アクター**を登録
- 同じメッセージタイプに複数のアクターが登録した場合、**セカンダリ受信者**として保持
- セカンダリ受信者はプライマリと同じメッセージを受信するわけではない（Recipient Listの実装違反になるため）
- プライマリアクターが特定のメッセージタイプへの関心を解除した場合、セカンダリがプライマリの代わりとして入れ替わる

---

## 実装例（Scala/Akka）

### メッセージ定義

```scala
package co.vaughnvernon.reactiveenterprise.dynamicrouter

import reflect.runtime.currentMirror
import akka.actor._
import co.vaughnvernon.reactiveenterprise._

// 制御メッセージ
case class InterestedIn(messageType: String)
case class NoLongerInterestedIn(messageType: String)

// ルーティング対象メッセージ
case class TypeAMessage(description: String)
case class TypeBMessage(description: String)
case class TypeCMessage(description: String)
case class TypeDMessage(description: String)
```

### ドライバアプリケーション

```scala
object DynamicRouter extends CompletableApp(5) {
  // デッドレター/キャッチオールアクター
  val dunnoInterested =
          system.actorOf(
            Props[DunnoInterested],
            "dunnoInterested")

  // Dynamic Router本体
  val typedMessageInterestRouter =
      system.actorOf(
          Props(classOf[TypedMessageInterestRouter],
              dunnoInterested, 4, 1),
          "typedMessageInterestRouter")

  // 関心を持つアクターを生成
  val typeAInterest =
        system.actorOf(
          Props(classOf[TypeAInterested],
                  typedMessageInterestRouter),
          "typeAInterest")

  val typeBInterest =
      system.actorOf(
            Props(classOf[TypeBInterested],
                    typedMessageInterestRouter),
            "typeBInterest")

  val typeCInterest =
      system.actorOf(
        Props(classOf[TypeCInterested],
                typedMessageInterestRouter),
        "typeCInterest")

  // TypeCMessageに関心を持つ2番目のアクター（セカンダリ候補）
  val typeCAlsoInterested =
      system.actorOf(
            Props(classOf[TypeCAlsoInterested],
                    typedMessageInterestRouter),
            "typeCAlsoInterested")

  // 登録完了を待機
  awaitCanStartNow()

  // メッセージ送信
  typedMessageInterestRouter !
      TypeAMessage("Message of TypeA.")
  typedMessageInterestRouter !
      TypeBMessage("Message of TypeB.")
  typedMessageInterestRouter !
      TypeCMessage("Message of TypeC.")

  // プライマリ解除後の動作確認を待機
  awaitCanCompleteNow()

  // 追加メッセージ送信（セカンダリがプライマリに昇格後）
  typedMessageInterestRouter !
      TypeCMessage("Another message of TypeC.")
  typedMessageInterestRouter !
      TypeDMessage("Message of TypeD.")

  awaitCompletion
  println("DynamicRouter: is completed.")
}
```

### 同期ポイントの説明

| 同期ポイント | 目的 |
|------------|------|
| `awaitCanStartNow()` | 関心を持つアクターがDynamic Router（TypedMessageInterestRouter）に完全に登録されるまで待機 |
| `awaitCanCompleteNow()` | プライマリの`TypeCInterested`がセカンダリの`TypeCAlsoInterested`に置き換わるまで待機 |

最初の`TypeCMessage`は`TypeCInterested`に送信され、2番目の`TypeCMessage`は`TypeCAlsoInterested`に送信される。

---

## 実行結果

```
TypeAInterested: received: TypeAMessage(Message of TypeA.)
TypeBInterested: received: TypeBMessage(Message of TypeB.)
TypeCInterested: received: TypeCMessage(Message of TypeC.)
TypeCAlsoInterested: received: TypeCMessage(Another↩
 message of TypeC.)
DunnoInterest: received undeliverable message:↩
 TypeDMessage(Message of TypeD.)
DynamicRouter: is completed.
```

### 結果の解説
- `TypeDMessage`は専門のアクターに配信されない
- 代わりに`DunnoInterest`アクターが受信（デッドレター/キャッチオールアクターとして機能）

---

## DunnoInterested（デッドレター/キャッチオール）

```scala
class DunnoInterested extends Actor {
  def receive = {
    case message: Any =>
      println(s"DunnoInterest: received undeliverable↩
      message: $message")
      DynamicRouter.completedStep()
  }
}
```

登録された関心がない任意のメッセージタイプを受信するキャッチオールアクター。

---

## 関心を持つアクター

各アクターは構築時に`TypedMessageInterestRouter`への参照を受け取り（Actor-Endowment）、特定のメッセージタイプへの関心を登録する。

### TypeAInterested

```scala
class TypeAInterested(interestRouter: ActorRef)
  extends Actor {
  // 構築時に関心を登録
  interestRouter !
        InterestedIn(TypeAMessage.getClass.getName)

  def receive = {
    case message: TypeAMessage =>
      println(s"TypeAInterested: received: $message")
      DynamicRouter.completedStep()
    case message: Any =>
      println(s"TypeAInterested: unexpected: $message")
  }
}
```

### TypeBInterested

```scala
class TypeBInterested(interestRouter: ActorRef)
    extends Actor {
  interestRouter !
        InterestedIn(TypeBMessage.getClass.getName)

  def receive = {
    case message: TypeBMessage =>
      println(s"TypeBInterested: received: $message")
      DynamicRouter.completedStep()
    case message: Any =>
      println(s"TypeBInterested: unexpected: $message")
  }
}
```

### TypeCInterested（プライマリ候補）

```scala
class TypeCInterested(interestRouter: ActorRef)
    extends Actor {
  interestRouter !
        InterestedIn(TypeCMessage.getClass.getName)

  def receive = {
    case message: TypeCMessage =>
      println(s"TypeCInterested: received: $message")

      // 最初のメッセージ受信後、関心を解除
      interestRouter ! NoLongerInterestedIn(
                        TypeCMessage.getClass.getName)

      DynamicRouter.completedStep()

    case message: Any =>
      println(s"TypeCInterested: unexpected: $message")
  }
}
```

### TypeCAlsoInterested（セカンダリ候補）

```scala
class TypeCAlsoInterested(interestRouter: ActorRef)
    extends Actor {
  interestRouter !
        InterestedIn(TypeCMessage.getClass.getName)

  def receive = {
    case message: TypeCMessage =>
      println(s"TypeCAlsoInterested: received: $message")

      // メッセージ受信後、関心を解除
      interestRouter ! NoLongerInterestedIn(
                        TypeCMessage.getClass.getName)

      DynamicRouter.completedStep()

    case message: Any =>
      println(s"TypeCAlsoInterested: unexpected: $message")
  }
}
```

### TypeCMessage登録の動作

`TypeCMessage`に関心を登録する2つのアクターの動作：
1. 一方が先に登録し、**プライマリ**になる
2. 他方は**セカンダリ**として登録される
3. 通常は`TypeCInterested`がプライマリ、`TypeCAlsoInterested`がセカンダリ
4. プライマリが最初の`TypeCMessage`を受信後、`NoLongerInterestedIn`メッセージを`TypedMessageInterestRouter`に送信
5. これにより`TypedMessageInterestRouter`はプライマリを解除し、セカンダリをプライマリに昇格

---

## TypedMessageInterestRouter（Dynamic Router本体）

```scala
import scala.collection.mutable.Map

class TypedMessageInterestRouter(
    dunnoInterested: ActorRef,
    canStartAfterRegistered: Int,
    canCompleteAfterUnregistered: Int) extends Actor {

  // プライマリ登録用レジストリ
  val interestRegistry =
        Map[String, ActorRef]()

  // セカンダリ登録用レジストリ
  val secondaryInterestRegistry =
        Map[String, ActorRef]()

  def receive = {
    case interestedIn: InterestedIn =>
      registerInterest(interestedIn)
    case noLongerInterestedIn: NoLongerInterestedIn =>
      unregisterInterest(noLongerInterestedIn)
    case message: Any =>
      sendFor(message)
  }

  // 関心の登録
  def registerInterest(interestedIn: InterestedIn) = {
    val messageType =
        typeOfMessage(interestedIn.messageType)

    if (!interestRegistry.contains(messageType)) {
      // プライマリとして登録
      interestRegistry(messageType) = sender
    } else {
      // セカンダリとして登録
      secondaryInterestRegistry(messageType) = sender
    }

    // 登録数が閾値に達したら開始許可
    if (interestRegistry.size +
        secondaryInterestRegistry.size
        >= canStartAfterRegistered) {
      DynamicRouter.canStartNow()
    }
  }

  // メッセージ送信
  def sendFor(message: Any) = {
    val messageType =
          typeOfMessage(
            currentMirror
              .reflect(message)
              .symbol
              .toString)

    if (interestRegistry.contains(messageType)) {
      // 登録済みアクターに転送
      interestRegistry(messageType) forward message
    } else {
      // 登録なしの場合はデッドレターへ
      dunnoInterested ! message
    }
  }

  // メッセージタイプの正規化
  def typeOfMessage(rawMessageType: String): String = {
    rawMessageType
      .replace('$', ' ')
      .replace('.', ' ')
      .split(' ')
      .last
      .trim
  }

  var unregisterCount: Int = 0

  // 関心の解除
  def unregisterInterest(
        noLongerInterestedIn: NoLongerInterestedIn) = {
    val messageType =
        typeOfMessage(noLongerInterestedIn.messageType)

    if (interestRegistry.contains(messageType)) {
      val wasInterested = interestRegistry(messageType)

      // 送信者がプライマリ登録者の場合
      if (wasInterested.compareTo(sender) == 0) {
        if (secondaryInterestRegistry
              .contains(messageType)) {
          // セカンダリをプライマリに昇格
          val nowInterested =
              secondaryInterestRegistry
                .remove(messageType)

          interestRegistry(messageType) =
              nowInterested.get
        } else {
          // セカンダリなしの場合は削除
          interestRegistry.remove(messageType)
        }

        unregisterCount = unregisterCount + 1;

        if (unregisterCount >=
            this.canCompleteAfterUnregistered) {
          DynamicRouter.canCompleteNow()
        }
      }
    }
  }
}
```

---

## 処理フロー図

### 登録フロー

```
┌─────────────────────────────────────────────────────────────────┐
│                       登録プロセス                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  InterestedIn メッセージ受信                                    │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────┐                                    │
│  │ interestRegistry に     │                                    │
│  │ messageType が存在？     │                                    │
│  └───────────┬─────────────┘                                    │
│         ┌────┴────┐                                             │
│         │         │                                             │
│        No        Yes                                            │
│         │         │                                             │
│         ▼         ▼                                             │
│   プライマリ   セカンダリ                                         │
│   として登録   として登録                                         │
│   (interest   (secondary                                        │
│    Registry)   InterestRegistry)                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### メッセージルーティングフロー

```
┌─────────────────────────────────────────────────────────────────┐
│                    メッセージルーティング                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  任意のメッセージ受信                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────┐                                    │
│  │ interestRegistry に     │                                    │
│  │ messageType が存在？     │                                    │
│  └───────────┬─────────────┘                                    │
│         ┌────┴────┐                                             │
│         │         │                                             │
│        Yes        No                                            │
│         │         │                                             │
│         ▼         ▼                                             │
│   登録済み     dunnoInterested                                   │
│   アクターへ   (デッドレター)へ                                    │
│   forward     送信                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 解除フロー（セカンダリ昇格）

```
┌─────────────────────────────────────────────────────────────────┐
│                セカンダリ→プライマリ昇格                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NoLongerInterestedIn メッセージ受信                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────┐                                    │
│  │ 送信者がプライマリ？      │                                    │
│  └───────────┬─────────────┘                                    │
│              │ Yes                                              │
│              ▼                                                  │
│  ┌─────────────────────────┐                                    │
│  │ セカンダリが存在？        │                                    │
│  └───────────┬─────────────┘                                    │
│         ┌────┴────┐                                             │
│         │         │                                             │
│        Yes        No                                            │
│         │         │                                             │
│         ▼         ▼                                             │
│   セカンダリを   登録を                                          │
│   プライマリに   削除                                            │
│   昇格                                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 注意事項と考慮事項

### コードの目的
このアプローチは本番コード向けの推奨ソリューションとして紹介されているわけではない。例を簡略化するために使用されている。

### メッセージ順序と同期の問題
この例は、メッセージ順序の仮定、アクターの同期、その他の時間的依存関係から生じる問題を示している。

### 同期ポイントの問題
`DynamicRouter.canCompleteNow()`を使用して同期ポイントを構築することで、`TypeCMessage`に関心を持つプライマリアクターがセカンダリに置き換えられることを保証している。しかし、これは実際のアプリケーションには適さないアプローチである。

### 本番環境での考慮事項
- プライマリアクターは`NoLongerInterestedIn`を`TypedMessageInterestRouter`に送信した後でも、任意の数の`TypeCMessage`インスタンスを受け入れる準備が必要
- セカンダリアクターがプライマリとして登録される時間を確保する必要がある
- `TypeCInterested`と`TypeCAlsoInterested`アクターが登録のスワップのためのプロトコルを確立する必要がある可能性がある
- `TypedMessageInterestRouter`がネゴシエーションを管理するか、少なくともそれに含まれる場合に最も効果的

### メールボックスの競合に関する注意
`TypedMessageInterestRouter`は一度に1つのメッセージしか受信しないため、セカンダリ（例：`TypeCAlsoInterested`）が`TypedMessageInterestRouter`が次の`TypeCMessage`をディスパッチする前にプライマリ（例：`TypeCInterested`）を置き換えると考えるかもしれない。しかし、これは`TypedMessageInterestRouter`のメールボックスで特定の`NoLongerInterestedIn`より前に`TypeCMessage`インスタンスがない場合にのみ成り立つ。このような仮定に依存すると、問題が発生する可能性がある。
