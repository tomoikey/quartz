# Event-Driven Consumer

## EIPにおけるEvent-Driven Consumer

Event-Driven Consumerは、メッセージングシステムによってメッセージがコンシューマーのチャネルに到着すると自動的に呼び出されるオブジェクトである。

### 問題

アプリケーションは利用可能になった直後にメッセージを自動的に消費する必要がある。どのようにしてアプリケーションが配信されたメッセージを自動的に消費できるか。

### 解決策

Event-Driven Consumerを使用する。これは、チャネルで配信されたメッセージが自動的に渡されるものである。このパターンは非同期レシーバーとしても知られており、メッセージの配信がイベントとして機能し、レシーバーを起動する。

### 特徴

- メッセージ到着時に自動的に呼び出される
- アクティブなスレッドがない状態で休止できる
- メッセージシステムがコールバック経由でアプリケーションにメッセージを渡す

## アクターモデルにおけるEvent-Driven Consumer

アクターモデルにおけるアクターは自然にEvent-Driven Consumersであり、各アクターのメールボックスがPoint-to-Point Channelとして機能する。アクターは直接的な非同期メッセージングを使用するため、別のアクターからメッセージを送信されたアクターは、そのメッセージを非同期に消費する。

### 「Event-Driven」の意味

Event-Driven Consumerを作るのは、必ずしもEvent Messageであるわけではない。Command MessageやDocument Messageであっても構わない。アクターがあらゆる種類のメッセージを受信したときにリアクティブであるという事実が、それをEvent-Driven Consumerにするのである。

したがって、「event-driven」という用語は、コンシューマーが受信しているメッセージの種類を説明するためではなく、「polling」と対比して使用される。ポーリングでは、コンシューマーが明示的にメッセージを要求するが、event-drivenでは、メッセージの到着がコンシューマーを起動する「イベント」として機能する。

### Polling ConsumerとEvent-Driven Consumerの違い

| 特性 | Polling Consumer | Event-Driven Consumer |
|-----|-----------------|----------------------|
| メッセージ取得方式 | 明示的に要求 | 自動的に配信される |
| スレッドの使用 | ポーリング中はスレッドを使用 | メッセージ到着時のみスレッドを使用 |
| 制御の主体 | コンシューマーが制御 | メッセージングシステムが制御 |

## 関連パターン

| パターン | 関係 |
|---------|------|
| [[programming/reactive_messaging_pattern/chapter9/polling_consumer\|Polling Consumer]] | Event-Driven Consumerの対となるパターン。明示的にメッセージを要求する |
| [[programming/reactive_messaging_pattern/chapter9/competing_consumers\|Competing Consumers]] | 複数のEvent-Driven Consumerが同じチャネルから消費する |
| [[programming/reactive_messaging_pattern/chapter9/message_dispatcher\|Message Dispatcher]] | Event-Driven Consumerにメッセージをディスパッチする |
| [[programming/reactive_messaging_pattern/chapter9/message_selector\|Selective Consumer]] | 特定の条件に一致するメッセージのみを処理する |
| [[programming/reactive_messaging_pattern/chapter9/transactional_client\|Transactional Client]] | メッセージ処理をトランザクション内で行う |

## 参考資料

- [Enterprise Integration Patterns - Event-Driven Consumer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/EventDrivenConsumer.html)
