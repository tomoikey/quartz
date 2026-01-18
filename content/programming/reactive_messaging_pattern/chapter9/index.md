# Message Endpoints (メッセージエンドポイント)

## このドキュメントについて

このドキュメントは、「Reactive Messaging Patterns with the Actor Model」のChapter 9「Message Endpoints」の内容と、Enterprise Integration Patterns（EIP）の公式ドキュメントを統合して解説したものです。

## Message Endpointsとは何か

エンタープライズ統合において、アプリケーションがメッセージングシステムと通信するためには、何らかの接続点が必要です。この接続点こそが**Message Endpoint**（メッセージエンドポイント）と呼ばれるコンポーネントです。

EIPの公式ドキュメントでは、Message Endpointを「アプリケーションをメッセージングチャネルに接続し、メッセージの送受信を可能にするメッセージングシステムのクライアント」と定義しています。つまり、エンドポイントはアプリケーションの世界とメッセージングシステムの世界を橋渡しする役割を担っています。

## なぜMessage Endpointsが重要なのか

### 統合の抽象化という課題

理想的なシステム設計において、アプリケーションはメッセージングを使用していることを意識すべきではありません。ビジネスロジックは、メッセージの形式やチャネルの詳細といった技術的な関心事から切り離されているべきです。この分離を実現するために、統合ポイントには薄いコード層を設けることが推奨されます。

### エンドポイントの二つの責務

Message Endpointは、以下の二つの重要な責務を担っています。

**送信時の責務**として、エンドポイントはアプリケーションのコマンドやデータをメッセージ形式に変換し、適切なメッセージチャネルに送信する役割を果たします。アプリケーション側は「何を伝えたいか」だけを考え、「どのように伝えるか」はエンドポイントが処理します。

**受信時の責務**として、エンドポイントはメッセージチャネルからメッセージを取得し、その内容を抽出してアプリケーションが理解できる形で提供します。メッセージングプロトコルの詳細はエンドポイント内で完結し、アプリケーションには必要な情報だけが渡されます。

### スループット制御の重要性

メッセージングシステムを設計する際に避けて通れない課題の一つが、メッセージ消費速度の制御（スロットリング）です。サーバーに過剰な負荷がかかることを防ぐため、アプリケーションがメッセージを処理する速度を適切に管理する必要があります。Message Endpointsパターン群は、この制御を実現するための様々なアプローチを提供しています。

## カテゴリ

Message Endpointsは、その役割と機能に応じて二つの大きなカテゴリに分類されます。

### [[programming/reactive_messaging_pattern/chapter9/consumers/index|Consumers (コンシューマーパターン)]]

コンシューマーパターンは、メッセージをどのように受信し、消費するかに焦点を当てたパターン群です。メッセージの取得タイミング、複数コンシューマー間での負荷分散、メッセージの選択的処理など、受信側の様々な要件に対応します。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/polling_consumer|Polling Consumer]]** - 能動的にメッセージを問い合わせる方式です。コンシューマーが「新しいメッセージはありますか？」と定期的に確認を行います。従来のポーリングはブロッキング処理を伴いますが、アクターモデルではRequest-Replyパターンを活用することで、ノンブロッキングなポーリングを実現できます。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/event_driven_consumer|Event-Driven Consumer]]** - 受動的にメッセージを待ち受ける方式です。メッセージが到着したときに初めて処理が開始されます。アクターモデルでは、アクター自体がメッセージ駆動で動作するため、すべてのアクターは本質的にEvent-Driven Consumerとして機能します。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/competing_consumers|Competing Consumers]]** - 同じチャネルから複数のコンシューマーが競合的にメッセージを取得する方式です。負荷分散やスケーラビリティの向上を目的として使用されます。複数のワーカーが同じキューからタスクを取り出して並列処理するシナリオが典型例です。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/selective_consumer|Selective Consumer]]** - 特定の条件を満たすメッセージのみを処理する方式です。アクターが複数種類のメッセージを受信できるが、一部のメッセージタイプのみを処理対象とする場合に有効です。処理対象外のメッセージは破棄されます。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/durable_subscription|Durable Subscriber]]** - コンシューマーが一時的にオフラインになっても、その間に発行されたメッセージを見逃さないようにする方式です。サブスクリプション状態がメッセージングシステムに永続化されるため、再接続時に未処理メッセージを受け取ることができます。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/idempotent_receiver|Idempotent Receiver]]** - 同じメッセージを複数回受信しても、副作用が一度しか発生しないように設計された受信者です。「少なくとも1回の配信（at-least-once delivery）」を保証する環境では、重複メッセージが発生する可能性があるため、このパターンが重要になります。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/message_dispatcher|Message Dispatcher]]** - 受信したメッセージを適切なハンドラーに振り分ける役割を担います。Content-Based Routerと似ていますが、Message Dispatcherはワークロードのバランシングにも関心を持ち、処理可能な状態にあるハンドラーにのみメッセージを割り当てます。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/service_activator|Service Activator]]** - メッセージングシステムと内部サービスの橋渡しを行います。アプリケーションの境界でメッセージを受信し、適切なサービスやドメインモデルに処理を委譲する役割を果たします。

### [[programming/reactive_messaging_pattern/chapter9/gateways/index|Gateways (ゲートウェイパターン)]]

ゲートウェイパターンは、アプリケーションがメッセージングシステムを利用する際の複雑さを隠蔽し、よりシンプルなインターフェースを提供するパターン群です。

- **[[programming/reactive_messaging_pattern/chapter9/gateways/messaging_gateway|Messaging Gateway]]** - メッセージング固有のコードをカプセル化し、ドメイン固有のメソッドとしてアプリケーションに公開します。アプリケーションは「クレジットスコアを取得する」というビジネス操作を呼び出すだけで、その背後でメッセージの送受信が行われていることを意識する必要がありません。

- **[[programming/reactive_messaging_pattern/chapter9/gateways/messaging_mapper|Messaging Mapper]]** - ドメインオブジェクトとメッセージの間でデータ変換を行います。オブジェクト指向の世界とメッセージングの世界は根本的に異なるパラダイムを持つため、両者を独立させたままデータを移動させる仕組みが必要です。

- **[[programming/reactive_messaging_pattern/chapter9/gateways/transactional_client|Transactional Client/Actor]]** - メッセージの送受信にトランザクション境界を設定します。送信側ではコミットまでメッセージの送信を遅延し、受信側ではコミットまでメッセージの削除を遅延することで、データの整合性を保証します。

## パターンの組み合わせ

実際のシステム設計では、これらのパターンを単独で使用することは稀で、複数のパターンを組み合わせて使用することが一般的です。

例えば、ワーク処理システムを構築する場合を考えてみましょう。ワークコンシューマーは**Competing Consumer**と**Polling Consumer**の両方の特性を持つことがあります。つまり、複数のコンシューマーが競合的にワークキューからタスクを取得しつつ、各コンシューマーは能動的に新しいタスクの有無を確認します。

同時に、ワークプロバイダーは**Message Dispatcher**として機能し、各コンシューマーのワークロードと応答性を監視しながら、処理可能な状態にあるコンシューマーにのみワークアイテムを割り当てます。

さらに、ワークコンシューマーが行う処理の種類によっては、**Transactional Actor**として動作する必要があるかもしれません。これは、Domain-Driven DesignにおけるAggregateパターンと密接に関連しています。

このように、単一のMessage Endpointが複数のパターンを同時に実装することは珍しくありません。重要なのは、解決すべき問題に応じて適切なパターンの組み合わせを選択することです。

## まとめ

この章では、12種類のMessage Endpointsパターンを詳細に扱いました。これらのパターンを理解することで、以下のような設計上の知見を得ることができます。

**メッセージングの抽象化について**、アクターモデルが自然なMessaging Gatewayを提供すること、そしてMessaging Mapperを使用してドメインオブジェクトとメッセージ間の変換を適切に行う方法を学びました。

**トランザクション管理について**、Event Sourcingを使用するTransactional Actorを通じて、Domain-Driven DesignのAggregateパターンをサポートする方法を学びました。アクターがトランザクションをサポートするための二つのアプローチ（Service LayerアプローチとEvent Sourcingアプローチ）の違いについても理解を深めました。

**メッセージ消費のパターンについて**、ノンブロッキングなPolling Consumerの作成方法と、アクターが定義上Event-Driven Consumersであることを学びました。また、Akkaの標準ルーターを活用してCompeting Consumersを作成する方法、Message Dispatchersによるワークロード管理の重要性についても理解しました。

**信頼性と耐久性について**、Selective Consumersによる選択的メッセージ処理、Akka Persistenceを使用したDurable Subscribersのサポート、そしてGuaranteed Deliveryを使用する際のIdempotent Receiversの重要性を学びました。

**システム境界の設計について**、アプリケーションの外縁でメッセージを受信し、内部のサービスやドメインモデルにディスパッチするService Activatorsの役割を理解しました。

## 参考資料

- [Enterprise Integration Patterns - Messaging Endpoints Intro](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingEndpointsIntro.html)
- [Enterprise Integration Patterns - Message Endpoint](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageEndpoint.html)
