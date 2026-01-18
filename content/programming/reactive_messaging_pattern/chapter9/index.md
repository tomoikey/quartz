# Message Endpoints (メッセージエンドポイント)

## このドキュメントについて

このドキュメントは、「Reactive Messaging Patterns with the Actor Model」のChapter 9「Message Endpoints」を要約したものです。

## Message Endpointsとは何か

Chapter 4「Messaging with Actors」で、Message Endpointsはアクターモデルにおけるアクターとして紹介されました。この章では、様々な種類のエンドポイントについて学びます。

## カテゴリ

### [[programming/reactive_messaging_pattern/chapter9/consumers/index|Consumers (コンシューマーパターン)]]

メッセージを受信・消費するためのパターン集です。

| パターン | 説明 |
|---------|------|
| [[programming/reactive_messaging_pattern/chapter9/consumers/polling_consumer\|Polling Consumer]] | ポーリングは、リソース情報が提供されるまでコンシューマーがブロックすることを必要とする。アクターモデルではそのようには動作しない。Request-Replyを使ってポーリングを大まかに模倣する方法と、Akkaアクターがブロッキングを最小限または無しでハードウェアリソースをポーリングする方法を学ぶ |
| [[programming/reactive_messaging_pattern/chapter9/consumers/event_driven_consumer\|Event-Driven Consumer]] | このパターンはEvent Messageを送ることについてではなく、それを含む。Event-Driven Consumerは、送られてきたあらゆるメッセージにリアクティブに反応するもの |
| [[programming/reactive_messaging_pattern/chapter9/consumers/competing_consumers\|Competing Consumers]] | Competing Consumersは、特殊なグループとして、複数のメッセージに同時に反応する。Polling ConsumerとMessage Dispatcherの実装に応じて、自然なCompeting Consumersとなりうる |
| [[programming/reactive_messaging_pattern/chapter9/consumers/selective_consumer\|Selective Consumer]] | Selective ConsumerはMessage Filterの一種。アクターが様々なタイプのメッセージを受信するが、一部のメッセージタイプしか処理できない場合、処理するように設計されていないものは破棄できる |
| [[programming/reactive_messaging_pattern/chapter9/consumers/durable_subscription\|Durable Subscriber]] | Durable Subscriberを使って、ハンドラーがアクティブにリスニングしていない間に送信されたメッセージを逃さないようにする |
| [[programming/reactive_messaging_pattern/chapter9/consumers/idempotent_receiver\|Idempotent Receiver]] | 同じメッセージを複数回受信し、その都度処理すると問題が発生する可能性がある場合、Idempotent Receiverとなるようにアクターを設計する。アクターモデルの標準的なメッセージ配信契約は「at most once」であるため、これは問題にならないように思えるかもしれない。しかし、`AtLeastOnceDelivery`ミックスインを使用すると問題が発生しうる |
| [[programming/reactive_messaging_pattern/chapter9/consumers/message_dispatcher\|Message Dispatcher]] | Message DispatcherはContent-Based Routerに匹敵する。Message Dispatcherがメッセージの内容（メッセージタイプなど）を見て、各タイプのメッセージに対応するアクターにディスパッチすることは可能。両者の違いは、Message Dispatcherは一般的にワークロードに関心があり、即座に反応できるアクターにのみメッセージをディスパッチする点 |
| [[programming/reactive_messaging_pattern/chapter9/consumers/service_activator\|Service Activator]] | 1つ以上の外部アクセス可能なリソースを使用してリクエストされる可能性のある内部サービスがある場合、Service Activatorを使用する。アプリケーションの外部エッジでメッセージを受信し、サービスやドメインモデルアクターに向けて内部にディスパッチするすべてのアクターはService Activatorsである |

### [[programming/reactive_messaging_pattern/chapter9/gateways/index|Gateways (ゲートウェイパターン)]]

アプリがメッセージングシステムを利用するための補助・隠蔽を行うパターン集です。

| パターン | 説明 |
|---------|------|
| [[programming/reactive_messaging_pattern/chapter9/gateways/messaging_gateway\|Messaging Gateway]] | Akkaアクターシステムとそのアクターは自然なMessaging Gatewayを形成する。一つのアクターが別のアクターにメッセージを送るために必要なコードはシンプル |
| [[programming/reactive_messaging_pattern/chapter9/gateways/messaging_mapper\|Messaging Mapper]] | Messaging Mapperを使って、1つ以上のドメインオブジェクトの一部をメッセージにマッピングする |
| [[programming/reactive_messaging_pattern/chapter9/gateways/transactional_client\|Transactional Client/Actor]] | Akkaアクターで使用されるこのパターンは、クライアント/送信側アクターと受信側アクターの両方でのトランザクションに関するもの |

## パターンの組み合わせ

この章で見つかるパターンを組み合わせて、リアクティブな設計目標を達成する必要があるかもしれません。Polling Consumerで示されているように、ソリューションのワークコンシューマーは実際にはCompeting ConsumerとPolling Consumerの両方を使用しています。同じソリューションでは、ワークプロバイダーはMessage Dispatcherであり、ワークロードと応答性に関心を持ち、ワークコンシューマーが処理できると言っているアイテム数のみをディスパッチします。ワークコンシューマーが行う作業の種類によっては、Transactional Actorである必要があるかもしれません。これはIDDDのAggregateパターンに沿っています。

## まとめ

この章では、12種類のMessage Endpointsについて詳細に扱った。これには、アクターがトランザクションをサポートするための2つの方法が含まれる。

- アクターモデルが自然なMessaging Gatewayを提供することと、Messaging Mapperを実装する方法を学んだ
- Event Sourcingを使用するTransactional Actorを通じてAggregates [IDDD]をサポートする方法を学んだ
- ノンブロッキングなPolling Consumerを作成する方法と、アクターが定義上Event-Driven Consumersであることを学んだ
- Akkaの標準ルーターを活用してCompeting Consumersを作成する方法を学んだ
- Message Dispatchersによるワークロードトラフィッキングのサポートの重要性を学んだ
- アクターがSelective Consumersとなり、特定の種類のMessagesにのみ反応できることを学んだ
- Akka Persistenceを使用してDurable Subscribersをサポートする方法を発見した
- Guaranteed Deliveryを使用する場合、アクターをIdempotent Receiversとして設計することが重要であることを学んだ
- 最後に、アプリケーションの外縁でメッセージを受信し、サービスまたはドメインモデルアクターに内向きにディスパッチするすべてのアクターがService Activatorsであることを学んだ

## 参考資料

- [Enterprise Integration Patterns - Messaging Endpoints Intro](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingEndpointsIntro.html)
- [Enterprise Integration Patterns - Message Endpoint](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageEndpoint.html)
