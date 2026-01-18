# Competing Consumers

## EIPにおけるCompeting Consumers

Competing Consumersは、複数のコンシューマーが単一のPoint-to-Point Channelからメッセージを受け取るために作成されるパターンである。

### 問題

アプリケーションがMessagingを使用しているが、チャネルに追加される速度と同じ速度でメッセージを処理できない。

### 解決策

複数のCompeting Consumersを単一チャネル上に作成し、コンシューマーが複数のメッセージを同時に処理できるようにする。

### 特徴

- Point-to-Pointチャネルでのみ機能する（Publish-Subscribeでは各コンシューマーがメッセージのコピーを受け取る）
- メッセージングシステムの実装により、どのコンシューマーが実際にメッセージを受け取るかが決定される
- コンシューマーはメッセージ受信後、アプリケーションの残りの部分に処理を委譲可能

## アクターモデルにおけるCompeting Consumers

Competing Consumersは、特殊なグループとして、複数のメッセージに同時に反応する。Polling ConsumerとMessage Dispatcherの実装に応じて、自然なCompeting Consumersとなりうる。

### Message DispatcherによるCompeting Consumers

Message Dispatcherを実装するワークディスパッチャーを考える。ワークディスパッチャーは複数のワークパフォーマーをCompeting Consumersとして持つ。ディスパッチャーが作業が必要であることを示すメッセージを受信すると、実際のワークタスクをワークパフォーマーの1つにディスパッチする。

どのワークパフォーマーにディスパッチするか？すべてのワーカーは作業を競っており、現在最も負荷が低いものが選ばれるべきである。完全にアイドル状態のワーカーがいれば、それが作業を与えるのに最適なワーカーである。

### SmallestMailboxPool

Akkaの標準ルーターの1つである`SmallestMailboxPool`は、Competing Consumerパターンを特にうまくサポートする。これは以下の特徴を持つ：

- 設定された数またはリサイズする数のプールされたroutees（ワークパフォーマー）を持つことができる
- 自身のメールボックスに最も少ないメッセージ数を持つ非サスペンドrouteesにメッセージを送信しようとする
- すべてのrouteeアクターが自身のメールボックスを持つ
- routeesのプールはローカルとリモートの両方が可能

ただし、リモートアクターrouteeはルーターがそのメールボックスサイズを見ることができないため不利である。リモートアクターへのルーティングは、最後の最も望ましくないルーティングオプションとして選ばれる。

## 関連パターン

| パターン | 関係 |
|---------|------|
| [[programming/reactive_messaging_pattern/chapter9/polling_consumer\|Polling Consumer]] | Competing Consumersがメッセージを取得する方法の1つ |
| [[programming/reactive_messaging_pattern/chapter9/event_driven_consumer\|Event-Driven Consumer]] | Competing Consumersがメッセージを取得する方法の1つ |
| [[programming/reactive_messaging_pattern/chapter9/message_dispatcher\|Message Dispatcher]] | Competing Consumersにメッセージを配布する |
| [[programming/reactive_messaging_pattern/chapter9/transactional_client\|Transactional Client]] | 処理の信頼性を確保する |

## 参考資料

- [Enterprise Integration Patterns - Competing Consumers](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CompetingConsumers.html)
