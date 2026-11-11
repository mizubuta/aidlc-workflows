# Architect Agent

> **エージェント詳解** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [詳解一覧](README.md) · 技術リファレンス: [architect-agent](../../reference/agents/architect-agent.md)

aidlc-architect-agent は、あなたのソリューションアーキテクトです。要件を堅牢なシステムアーキテクチャに落とし込み、アーキテクチャ決定記録（ADR）を作成し、ドメインモデルを設計し、プロジェクトを実装可能な作業ユニットに分解します。パターンとトレードオフで考え、開発者がそのまま実装できる設計を生み出します。

aidlc-architect-agent は、ライフサイクルの中で単一エージェントとしては最も多い合計 7 つのステージをリードし、Ideation、Inception、Construction にまたがります。設計における第一の権限者であり、他の 7 つの高判断エージェントとともに `judgment` ティアに属します。そのため、特定のモデルに固定するのではなく、あなたのセッション自身のモデルと effort を継承します。`templated` ティアに属するのは delivery、pipeline-deploy、operations だけです。これらの出力は大部分がテンプレート化されているため、出荷時のベースラインでもセッションのモデルと effort を継承します。

## Stages Led

| ステージ | フェーズ | 説明 |
|-------|-------|-------------|
| 1.3 Feasibility & Constraints | Ideation | 技術的な実現可能性の評価と制約の分析 |
| 2.6 Domain Design | Inception | コンポーネントカタログ、ドメインモデリング、ADR |
| 2.7 Units Generation | Inception | 設計を実装可能な作業ユニットに分解する |
| 2.8 Contract Design | Inception | ユニット間の境界および公開/外部 API の API コントラクト |
| 3.1 Functional Design | Construction | 詳細なドメインモデルとビジネスロジック（ユニットごと） |
| 3.2 NFR Requirements | Construction | 測定可能な目標値を持つ非機能要件（ユニットごと） |
| 3.3 NFR Design | Construction | キャッシュ、レジリエンス、セキュリティ、オブザーバビリティの技術的アプローチ（ユニットごと） |

また、ステージ 2.1（Reverse Engineering）の統合ステップもリードします。ここでは aidlc-developer-agent からコードスキャン結果を受け取り、9 つのアーキテクチャ成果物を作成します。collaborators が off の場合は developer がそれらを書き、architect はディスパッチされません。

## Stages Supported

| ステージ | フェーズ | 貢献内容 |
|-------|-------|-------------|
| 1.1 Intent Capture | Ideation | 技術的なコンテキストを提供する |
| 2.1 Reverse Engineering（パイプラインの最後の工程としてディスパッチされる） | Inception | コードスキャン結果を一貫したアーキテクチャモデルに統合する |
| 2.9 Delivery Planning | Inception | ビルド順序をアーキテクチャ上の依存関係に照らして検証する |

## What to Expect

aidlc-architect-agent が稼働しているときは、境界、パターン、トレードオフに注目します。既存システムの制約、技術の好み、スケーラビリティ要件、運用上の懸念について質問します。明示的な判断根拠を伴う構造化された設計ドキュメント、Markdown で記述したコンポーネント図、そして重要な選択ごとの ADR を作成します。

## How It Collaborates

aidlc-architect-agent は、aidlc-product-agent から要件を、aidlc-developer-agent からコードスキャン結果を受け取ります。AWS サービスへのマッピングについては aidlc-aws-platform-agent と、セキュアな設計については aidlc-devsecops-agent と、規制上の制約については aidlc-compliance-agent と連携します。その出力（ユニット仕様、API コントラクト、NFR 目標値）は aidlc-developer-agent、aidlc-quality-agent、aidlc-aws-platform-agent が利用します。

## Key Principles

- すべての設計成果物は、明示的な根拠を持つ判断に遡れなければならない
- コンポーネントの境界を正しく定めることは、内部の詳細よりも重要である
- コンポーネント間の依存関係を積極的に最小化する
- 再利用ではなく変更のために設計する — 変更容易性を最適化する
- 隠れた前提を明示する — データフロー、所有権、障害モードを表に出す
- 可逆な判断を優先し、不可逆な判断には追加の精査が必要であることを明示する
