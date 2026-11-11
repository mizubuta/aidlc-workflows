# Compliance Agent

> **エージェント詳細解説** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [詳細解説](README.md) · 技術リファレンス: [compliance-agent](../../reference/agents/compliance-agent.md)

aidlc-compliance-agent は、GRC（Governance, Risk, and Compliance：ガバナンス・リスク・コンプライアンス）のアナリストです。ライフサイクルのすべてのステージで、適用される規制上の義務や組織のコンプライアンスポリシーが考慮されるようにします。規制要件を早い段階で洗い出し、それらを技術的コントロールに対応付け、コンプライアンスリスクに関する RAID ログを管理し、設計が監査上の期待を満たしているかを検証します。

aidlc-compliance-agent はもっぱら支援役として動作し、どのステージもリードしません。その代わりに、Ideation、Construction、Operation にまたがる 4 つのステージでコンプライアンスの専門知識を提供します。

## Stages Led

aidlc-compliance-agent がリードするステージはありません。

## Stages Supported

| ステージ | フェーズ | 貢献内容 |
|-------|-------|-------------|
| 1.3 Feasibility & Constraints | Ideation | 規制上の制約の特定、コンプライアンス面の実現可能性、RAID ログの初期化 |
| 3.2 NFR Requirements | Construction | 規制に関する NFR の対応付け、コンプライアンスコントロール要件、データ分類 |
| 3.4 Infrastructure Design | Construction | データレジデンシーの検証、暗号化要件、IAM のコンプライアンスコントロール |
| 4.2 Environment Provisioning | Operation | コンプライアンスコントロールの検証、監査ログ記録、規制関連の設定チェック |

## What to Expect

aidlc-compliance-agent が（リードエージェントと並ぶ支援エージェントとして）有効になっているとき、規制フレームワーク、データ分類、コントロールの対応付けに注力します。適用される規制（GDPR、HIPAA、PCI-DSS、SOC 2）、データの機密レベル、既存のコンプライアンスポリシーについて質問します。コンプライアンスコントロールのマトリクスを作成し、是正が必要なギャップを指摘します。

## How It Collaborates

aidlc-compliance-agent は、aidlc-architect-agent からシステム設計とデータフローの情報を、aidlc-devsecops-agent からセキュリティコントロールの詳細を受け取ります。そして、設計に組み込むためのコンプライアンス要件と制約を aidlc-architect-agent に返し、実装のためのセキュリティコントロール仕様を aidlc-devsecops-agent に渡します。

## Key Principles

- コンプライアンスは後付けではなく制約です。リリース時に見つかったギャップはプロジェクトの失敗です
- データ分類があらゆるコントロールの判断を左右します
- コンプライアンスの主張には監査可能な証拠が必要です。証拠のないコントロールは存在しないのと同じです
- 是正は、最も機密性の高いデータと、最も罰則の重い規制に集中させます
- 規制への理解はチーム全体で取り組むものです。aidlc-compliance-agent が教え、チームが実行します
