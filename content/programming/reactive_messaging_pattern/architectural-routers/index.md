# Architectural Routers

## このカテゴリについて

Architectural Routersは、**システム全体のメッセージングアーキテクチャを定義する**パターン群です。

[[programming/reactive_messaging_pattern/simple-routers/index|Simple Routers]]や[[programming/reactive_messaging_pattern/composed-routers/index|Composed Routers]]が「個々のメッセージをどう処理するか」を扱うのに対し、Architectural Routersは「システム全体をどのような構造で設計するか」という、より大きな視点での設計方針を示します。

## なぜArchitectural Routersが必要なのか

メッセージングシステムを構築する前に、「システム全体をどのような形で組み立てるか」を決める必要があります。この基本的な構造が決まっていないと、個々のルーティングパターンを適用しても、全体として整合性のあるシステムにはなりません。

例えるなら、家を建てる前に「一戸建てにするか、マンションにするか」を決めるようなものです。この基本構造が決まって初めて、各部屋のレイアウトを考えることができます。

## パターン一覧

| パターン | 何をするか | 使う場面 |
|---------|-----------|---------|
| [[pipes_and_filters\|Pipes and Filters]] | 処理をパイプ（チャネル）とフィルター（処理ステップ）に分けて、柔軟に組み合わせる | データ変換パイプライン、段階的なメッセージ処理 |
| [[message_broker\|Message Broker]] | 中央にブローカーを置いて、全てのメッセージを仲介する | 異種システム間の統合、エンタープライズ統合 |

## 2つのアーキテクチャスタイル

### Pipes and Filters（パイプとフィルター）

**考え方：** 処理を「フィルター」という独立したステップに分け、それらを「パイプ」（チャネル）で接続します。

```mermaid
graph LR
    IN[入力] --> F1[フィルター1<br/>暗号解除]
    F1 --> F2[フィルター2<br/>認証]
    F2 --> F3[フィルター3<br/>変換]
    F3 --> OUT[出力]

    style F1 fill:#e1f5fe
    style F2 fill:#e1f5fe
    style F3 fill:#e1f5fe
```

**身近な例：** 工場の組み立てライン

自動車工場では、車が組み立てラインを流れながら、各ステーションで部品が取り付けられていきます。
- ステーション1：エンジンを取り付け
- ステーション2：ドアを取り付け
- ステーション3：塗装

各ステーションは自分の仕事だけを行い、前後のステーションのことを知る必要がありません。

**特徴：**
- 各フィルターは独立しており、再利用しやすい
- フィルターの追加・削除・入れ替えが容易
- フィルターを並列化してスケールアウトしやすい
- フィルター同士は隣接するフィルターとしか通信しない

### Message Broker（メッセージブローカー）

**考え方：** 中央に「ブローカー」を置き、全てのメッセージがブローカーを経由して送受信されます。

```mermaid
graph TB
    A[アプリA] --> MB{Message<br/>Broker}
    B[アプリB] --> MB
    C[アプリC] --> MB
    MB --> X[アプリX]
    MB --> Y[アプリY]
    MB --> Z[アプリZ]

    style MB fill:#ffcc80
```

**身近な例：** 空港のハブ

航空会社の「ハブ・アンド・スポーク」モデルでは、各地からの便が中央のハブ空港に集まり、そこから各地へ乗り継ぎます。
- 福岡 → 羽田（ハブ） → 札幌
- 大阪 → 羽田（ハブ） → 仙台

各空港は羽田への接続だけを考えればよく、他の全ての空港との直行便を持つ必要がありません。

**特徴：**
- 送信者と受信者が互いを知らなくて良い（疎結合）
- メッセージフローを中央で管理・監視できる
- データフォーマットの変換を一箇所で行える
- ただし、ブローカーがボトルネックや障害点になる可能性がある

## パターンの比較

| 観点 | Pipes and Filters | Message Broker |
|-----|-------------------|----------------|
| 構造 | 線形のチェーン | ハブ・アンド・スポーク |
| 結合度 | 低い（隣接フィルターのみ） | 低い（ブローカー経由） |
| スケーラビリティ | 高い（フィルターごとにスケール） | 中程度（階層化で対応） |
| 主な用途 | データ処理パイプライン | 異種システム間の統合 |

## パターンの選び方

| 要件 | 推奨パターン |
|-----|------------|
| データを段階的に変換・処理したい | Pipes and Filters |
| 異なるシステム間でメッセージをやり取りしたい | Message Broker |
| 処理を並列化してスケールアウトしたい | Pipes and Filters |
| データフォーマットの変換を一元化したい | Message Broker |
| 処理の順番を柔軟に変更したい | Pipes and Filters |
| システム全体のメッセージフローを監視したい | Message Broker |

### 選択のポイント

**Pipes and Filters を選ぶ場合**
- メッセージに対して複数の処理を順番に適用したい
- 各処理ステップを独立して開発・テスト・デプロイしたい
- 処理の組み合わせを柔軟に変更したい

**Message Broker を選ぶ場合**
- 多数の異なるシステムを接続したい
- 送信者と受信者を完全に分離したい
- メッセージフローを中央で管理・監視したい

## 組み合わせて使う

実際のシステムでは、Pipes and FiltersとMessage Brokerを組み合わせて使うことが多いです。

例えば、Message Brokerの中でPipes and Filtersを使ってメッセージを変換する、といった構成が考えられます。

```mermaid
graph TB
    subgraph "システム全体（Message Broker）"
        A[アプリA] --> MB{Message<br/>Broker}
        B[アプリB] --> MB

        subgraph "ブローカー内部（Pipes and Filters）"
            MB --> F1[変換]
            F1 --> F2[検証]
            F2 --> F3[エンリッチ]
        end

        F3 --> X[アプリX]
        F3 --> Y[アプリY]
    end
```

## 次に読むべき内容

1. [[pipes_and_filters|Pipes and Filters]] - Message Routerが動作する基本的なアーキテクチャ
2. [[message_broker|Message Broker]] - エンタープライズ統合の中心的なパターン

## 関連するカテゴリ

- [[programming/reactive_messaging_pattern/index|Message Routing]] - 親カテゴリ
- [[programming/reactive_messaging_pattern/simple-routers/index|Simple Routers]] - Pipes and Filtersの中で使われるルーティングパターン
- [[programming/reactive_messaging_pattern/composed-routers/index|Composed Routers]] - 複合パターン
