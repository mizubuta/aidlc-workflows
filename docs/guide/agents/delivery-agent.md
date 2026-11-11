# Delivery Agent

> **エージェント詳細解説** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [詳細解説](README.md) · 技術リファレンス: [delivery-agent](../../reference/agents/delivery-agent.md)

aidlc-delivery-agent は、あなたのエンジニアリングマネージャー兼デリバリープランナーです。スコープ定義とアーキテクチャ設計を、チームの割り当て、ビルド順序、シーケンスを含む実行可能なデリバリー計画に落とし込みます。Ideation と Construction をつなぐイニシアチブブリーフを担当し、完全なトレーサビリティを保ちながらフェーズ間の引き継ぎを円滑に進めます。

aidlc-delivery-agent は、Ideation と Inception にまたがる 3 つのステージを主導します。設計されたものが実際にデリバリー可能かどうか、そしてどの順序でデリバリーするかを保証する責任を持つエージェントです。

## Stages Led

| ステージ | フェーズ | 説明 |
|-------|-------|-------------|
| 1.5 Team Formation | Ideation | 必要なスキルを評価し、チーム構成を組み立てる |
| 1.7 Approval & Handoff | Ideation | フェーズゲート承認のためにイニシアチブブリーフをまとめる |
| 2.9 Delivery Planning | Inception | Bolt のシーケンス（2.7 の DAG を通じた経済的な順序付け）を、チーム配分、リスク／シーケンスの根拠、外部依存関係マップとともに計画する |

## Stages Supported

| ステージ | フェーズ | 貢献内容 |
|-------|-------|-------------|
| 1.4 Scope Definition | Ideation | デリバリーの実現可能性の観点からスコープを検証する |
| 2.7 Units Generation | Inception | ユニットの粒度を計画上のニーズに合わせる |

## What to Expect

aidlc-delivery-agent がアクティブなときは、シーケンスと実現可能性に焦点を当てます。チームの規模、利用可能な専門知識、デリバリーに関する希望について質問します。作業ユニットをビルド順序に対応付け、クリティカルパスを特定し、進行を妨げうる依存関係を指摘する、構造化された計画を作成します。

## How It Collaborates

aidlc-delivery-agent は、aidlc-product-agent からスコープと優先順位を、aidlc-architect-agent からユニット仕様と複雑度の見積もりを受け取ります。作成したデリバリー計画は、すべての Construction エージェントがビルド順序とチームの割り当てを把握するために利用します。

## Key Principles

- 計画は生きたドキュメントである — 変更できない計画は失敗する
- 小さなバッチ、速いフィードバック — 小さな増分ほどリスクを早く表面化させる
- すべての作業ユニットは要件までさかのぼれなければならない
- フェーズの移行には明示的な完全性チェックが必要である
- 信頼は Bolt ごとに積み上げる — 出荷した Bolt が次の Bolt のリスクを下げる
