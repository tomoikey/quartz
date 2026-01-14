# Simple Routers

## 概要
単一メッセージの振り分けを行う基本的なルーティングパターン群。

## パターン比較表

| パターン | 消費 | 発行 | ステートフル | 特徴 |
|---------|------|------|------------|------|
| [[message_router\|Message Router]] | 1 | 1 | No | ルーティングの基本概念 |
| [[content_based_router\|Content-Based Router]] | 1 | 1 | No | 内容に基づき単一宛先へ |
| [[message_filter\|Message Filter]] | 1 | 0-1 | No | 条件に合わないものを破棄 |
| [[dynamic_router\|Dynamic Router]] | 1 | 1 | No | 制御メッセージでルール更新 |
| [[recipient_list\|Recipient List]] | 1 | N | No | 複数宛先へコピー送信 |
| [[splitter\|Splitter]] | 1 | N | No | メッセージを分割 |
| [[aggregator\|Aggregator]] | N | 1 | **Yes** | 関連メッセージを集約 |
| [[resequencer\|Resequencer]] | N | N | **Yes** | 順序を復元 |

## パターン選択ガイド

```mermaid
flowchart TD
    START[メッセージをどう処理したい？]

    START --> Q1{振り分け先は？}
    Q1 -->|単一| Q2{振り分け基準は？}
    Q1 -->|複数| Q3{振り分け先の決定方法は？}
    Q1 -->|分割/集約| Q4{どの操作？}

    Q2 -->|内容に基づく| CBR2[Content-Based Router]
    Q2 -->|動的ルール| DR2[Dynamic Router]
    Q2 -->|条件で破棄| MF2[Message Filter]

    Q3 -->|メッセージ内容から計算| RL2[Recipient List]
    Q3 -->|全員に放送| PS[Publish-Subscribe Channel]

    Q4 -->|1→N分割| SP2[Splitter]
    Q4 -->|N→1集約| AG2[Aggregator]
    Q4 -->|順序復元| RS2[Resequencer]
```

## トレードオフ

### ステートレス vs ステートフル

| 特性 | ステートレス | ステートフル |
|-----|------------|------------|
| パターン | Router, Filter, Splitter, Recipient List | Aggregator, Resequencer |
| スケーラビリティ | 容易 | 状態共有が課題 |
| 障害復旧 | シンプル | 状態永続化が必要 |
| メモリ使用量 | 低 | 高（バッファ保持） |

## 関連パターン

- [[index|Message Routing]] - 親カテゴリ
- [[composed-routers/index|Composed Routers]] - 複合パターン
- [[architectural-routers/index|Architectural Routers]] - アーキテクチャパターン
