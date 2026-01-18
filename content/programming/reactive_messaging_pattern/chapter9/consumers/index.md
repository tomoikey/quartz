# Consumers (コンシューマーパターン)

メッセージを受信・消費するためのパターン集です。

## パターン一覧

| パターン | 説明 |
|---------|------|
| [[programming/reactive_messaging_pattern/chapter9/consumers/polling_consumer\|Polling Consumer]] | リソース情報が提供されるまでポーリングするパターン |
| [[programming/reactive_messaging_pattern/chapter9/consumers/event_driven_consumer\|Event-Driven Consumer]] | 送られてきたメッセージにリアクティブに反応するパターン |
| [[programming/reactive_messaging_pattern/chapter9/consumers/competing_consumers\|Competing Consumers]] | 複数のメッセージに同時に反応する特殊なグループパターン |
| [[programming/reactive_messaging_pattern/chapter9/consumers/selective_consumer\|Selective Consumer]] | 特定のメッセージタイプのみを処理するフィルタリングパターン |
| [[programming/reactive_messaging_pattern/chapter9/consumers/durable_subscription\|Durable Subscriber]] | リスニングしていない間のメッセージも逃さないパターン |
| [[programming/reactive_messaging_pattern/chapter9/consumers/idempotent_receiver\|Idempotent Receiver]] | 同じメッセージを複数回受信しても問題ないように設計するパターン |
| [[programming/reactive_messaging_pattern/chapter9/consumers/message_dispatcher\|Message Dispatcher]] | ワークロードに応じてメッセージを適切な処理担当に振り分けるパターン |
| [[programming/reactive_messaging_pattern/chapter9/consumers/service_activator\|Service Activator]] | メッセージを受け取ってビジネスロジックを呼び出すパターン |
