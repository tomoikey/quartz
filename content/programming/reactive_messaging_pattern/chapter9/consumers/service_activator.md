# Service Activator (Messaging Adapter)

## パターンの概要

Service Activator（サービスアクティベーター）は、メッセージングチャネル上のメッセージをサービス呼び出しに接続するパターンです。「Messaging Adapter」とも呼ばれ、メッセージングインフラストラクチャと内部のビジネスサービスの間の橋渡し役として機能します。このパターンにより、サービスはメッセージング経由で呼び出されていることを意識することなく、様々な外部インターフェースからアクセス可能になります。

## EIPにおけるService Activator

### 解決すべき問題

アプリケーションが提供するサービスを、**メッセージング技術と非メッセージング技術の両方を通じて利用可能にしたい**という要件があります。

例えば、以下のようなシナリオを考えてみましょう。

**マルチチャネルアクセス**では、同じビジネスサービスに対して、Webアプリケーション、モバイルアプリ、バッチ処理システム、他のマイクロサービスなど、様々なクライアントからアクセスしたい場合があります。各クライアントは異なるプロトコル（HTTP、メッセージキュー、gRPCなど）を使用する可能性があります。

**レガシーシステムの統合**では、既存のサービスをメッセージングシステムに接続したい場合があります。サービス自体を変更せずに、メッセージングインターフェースを追加したいという要件があります。

**疎結合の実現**では、サービスの実装をメッセージングインフラストラクチャから分離し、サービスがメッセージングの詳細を知らなくても良いように設計したい場合があります。

### 解決策

メッセージチャネルからのメッセージをサービス呼び出しに変換する**Service Activator**を設計します。

Service Activatorは、メッセージングチャネルとサービスの間に位置し、以下の役割を担います。

**メッセージの受信**として、Service Activatorはメッセージングチャネルからメッセージを受信します。これはEvent-Driven ConsumerまたはPolling Consumerとして動作できます。

**パラメータの変換**として、受信したメッセージの内容をサービスが期待するパラメータ形式に変換します。メッセージのプロパティやペイロードから必要な情報を抽出し、サービスのAPIに適した形に整形します。

**サービスの呼び出し**として、変換されたパラメータを使用して内部サービスを呼び出します。呼び出しはシンプルなメソッド呼び出しである場合もあれば、リフレクションを使用した動的な呼び出しである場合もあります。

**応答の処理**として、サービスからの応答を受け取り、必要に応じてメッセージ形式に変換して返送します。

### Service Activatorの動作モード

Service Activatorは、二つの動作モードをサポートします。

**一方向（One-Way）モード**では、Service Activatorはメッセージを受信し、サービスを呼び出しますが、応答を期待しません。Fire-and-Forget方式で、メッセージの送信者は結果を待たずに処理を続行します。イベント通知やログ記録などの非同期処理に適しています。

**双方向（Request-Reply）モード**では、Service Activatorはメッセージを受信し、サービスを呼び出し、その結果を応答メッセージとして返します。送信者は応答を受け取って処理を続行します。クエリ操作や確認が必要な操作に適しています。

### サービスの独立性

Service Activatorの重要な特徴は、サービス自体がメッセージング経由で呼び出されていることを認識しないように設計できることです。

**メッセージング詳細の隠蔽**として、Service Activatorがすべてのメッセージング関連の処理（メッセージの受信、パラメータの抽出、応答の送信など）を担当します。サービスは純粋なビジネスロジックに集中できます。

**テスト容易性の向上**として、サービスがメッセージングに依存しないため、単体テストが容易になります。Service Activatorとサービスを独立してテストできます。

**再利用性の向上**として、同じサービスを異なるService Activatorから呼び出すことで、複数のプロトコルやインターフェースをサポートできます。

## アクターモデルにおけるService Activator

### Hexagonal Architectureとの関係

Service Activatorは、Domain-Driven Design（DDD）で提唱されている**Hexagonal Architecture（ヘキサゴナルアーキテクチャ）**、別名**Ports and Adaptersアーキテクチャ**と密接に関連しています。

このアーキテクチャでは、アプリケーションは複数の層で構成されます。

**Domain Model（ドメインモデル）**は、アプリケーションの中心に位置し、ビジネスロジックとビジネスルールを含みます。この層は外部の技術的な詳細から完全に独立しています。

**Application Service（アプリケーションサービス）**は、ドメインモデルを取り囲み、ユースケースを実装する層です。アプリケーションサービスは、ドメインモデルを調整し、ビジネスプロセスを実行します。

**Adapter（アダプター）**は、外部との接続を担当する層です。Web、データベース、ファイルシステム、クラウドサービス、メッセージングシステムなど、様々な外部リソースとの橋渡しを行います。

### Service Activatorの役割

このアーキテクチャにおいて、Service Activatorは**入力側（Inbound）のAdapter**として機能します。

**外部からのリクエストの受信**として、Service Activatorは外部クライアントからのリクエストを受け付けます。これはメッセージング、HTTP、gRPCなど、様々なプロトコルを通じて行われます。

**Application Serviceへの委譲**として、受信したリクエストを内部のApplication Serviceに委譲します。外部プロトコル固有のパラメータを、Application Serviceが期待するドメイン固有のパラメータに変換します。

**共通のApplication Service層の再利用**として、異なるタイプのService Activator（HTTP用、メッセージング用、RPC用など）が、同じApplication Service層を呼び出します。これにより、ビジネスロジックの重複を避けられます。

### 一方向と双方向の実装

アクターモデルでService Activatorを実装する際、一方向と双方向の両方のパターンをサポートできます。

**一方向（Fire-and-Forget）の実装**では、Service Activatorアクターが外部からメッセージを受信し、内部形式に変換してApplication Serviceアクターに送信します。応答は期待しないため、処理はそこで完了します。イベント通知や非同期コマンドの処理に適しています。

**双方向（Request-Reply）の実装**では、Service Activatorアクターがリクエストを受信し、Application Serviceアクターに転送します。Application Serviceからの応答を受け取ったら、外部プロトコルに適した形式に変換して、元のリクエスト元に返送します。同期的なクエリや確認が必要な操作に適しています。

### 複数プロトコルへの対応

Service Activatorパターンを使用すると、同じ内部サービスを複数のプロトコルから利用可能にできます。

**プロトコル固有のService Activator**として、HTTPリクエストを処理するService Activator、メッセージキューからのメッセージを処理するService Activator、WebSocket接続を処理するService Activatorなど、各プロトコルに特化したActivatorを作成できます。

**変換ロジックの分離**として、各Service Activatorは、そのプロトコル固有のデータ形式とApplication Serviceが期待する形式の間の変換ロジックを担当します。この変換ロジックは、各Activatorに閉じ込められ、Application Serviceには影響しません。

**段階的な移行**として、新しいプロトコルのサポートを追加する際、既存のApplication Serviceを変更せずに、新しいService Activatorを追加するだけで対応できます。

## Service Activatorの設計考慮事項

### エラーハンドリング

**サービス障害の処理**として、内部サービスが障害を起こした場合、Service Activatorは適切にエラーを処理する必要があります。エラーメッセージを返送するか、Invalid Message Channelにメッセージを転送するか、リトライするかを決定します。

**変換エラーの処理**として、メッセージからパラメータへの変換に失敗した場合の処理を定義します。不正なメッセージはログに記録し、Dead Letter Channelに送信することが一般的です。

### トランザクション管理

**メッセージ受信とサービス呼び出しの一貫性**として、メッセージの受信とサービスの呼び出しを同一トランザクション内で行う必要がある場合、Transactional Clientパターンと組み合わせることを検討します。

### スケーラビリティ

**複数のService Activatorインスタンス**として、負荷が高い場合、複数のService Activatorインスタンスを配置してCompeting Consumersパターンを形成できます。これにより、メッセージ処理のスループットを向上させられます。

## 関連パターン

- **[[programming/reactive_messaging_pattern/chapter9/gateways/messaging_gateway|Messaging Gateway]]** - メッセージングへのアクセスを簡素化するパターンです。Service Activatorがインバウンド（受信側）のアダプターであるのに対し、Messaging Gatewayはアウトバウンド（送信側）のアダプターとして機能します。

- **[[programming/reactive_messaging_pattern/chapter9/gateways/messaging_mapper|Messaging Mapper]]** - メッセージとドメインオブジェクト間のマッピングを行うパターンです。Service Activatorは、Messaging Mapperを使用してメッセージをサービスパラメータに変換できます。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/event_driven_consumer|Event-Driven Consumer]]** - Service Activatorがメッセージを受信する方法の一つです。メッセージの到着がService Activatorを起動します。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/polling_consumer|Polling Consumer]]** - Service Activatorがメッセージを受信するもう一つの方法です。Service Activatorが明示的にメッセージを要求します。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/competing_consumers|Competing Consumers]]** - 複数のService Activatorインスタンスで負荷を分散するパターンです。

- **[[programming/reactive_messaging_pattern/chapter9/consumers/message_dispatcher|Message Dispatcher]]** - メッセージを適切なService Activatorにディスパッチするパターンです。

- **[[programming/reactive_messaging_pattern/chapter9/gateways/transactional_client|Transactional Client]]** - メッセージ処理をトランザクション内で行うパターンです。Service Activatorと組み合わせて、信頼性の高いメッセージ処理を実現します。

## 参考資料

- [Enterprise Integration Patterns - Service Activator](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingAdapter.html)
