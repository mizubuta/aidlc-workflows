# Agent Deep Dives

ドメインエージェントごとに 1 ページを設け、そのドメイン、リードおよびサポートするステージ、
読み込むナレッジ、連携の仕方を説明します。14 エージェントの全体構成は、
11 のドメインエージェント、2 つのレビュー専用エージェント、そして adaptive-workflows のコンポーザーです。
この索引では、ステージの作業を実行する 11 のドメインエージェントを扱います。

レビュー専用エージェントは、プロダクトとアーキテクチャの独立したチェックを行います。
コンポーザーは、適応型のステージ計画を提案し、組み替えます。3 つの役割グループすべてと、
30 の狭いエージェントではなく 11 の幅広いドメインエージェントを採用した考え方については、
[エージェントの章](../06-agents.md)を参照してください。frontmatter の契約、ツールの許可リスト、
完全な比較マトリクスについては、開発者リファレンスの[エージェントリファレンス](../../reference/agents/README.md)を参照してください。

| # | エージェント | ドメイン |
|---|-------|--------|
| 1 | [aidlc-product-agent](product-agent.md) | 要件、スコープ、ユーザーストーリー、市場調査 |
| 2 | [aidlc-design-agent](design-agent.md) | UX/UI、ワイヤーフレーム、インタラクションデザイン、アクセシビリティ |
| 3 | [aidlc-delivery-agent](delivery-agent.md) | チーム編成、キャパシティ計画、デリバリーの順序付け |
| 4 | [aidlc-architect-agent](architect-agent.md) | ドメイン設計、ドメインモデリング、NFR、分解 |
| 5 | [aidlc-aws-platform-agent](aws-platform-agent.md) | AWS インフラストラクチャ、IaC、FinOps、環境のプロビジョニング |
| 6 | [aidlc-compliance-agent](compliance-agent.md) | GRC、規制マッピング、データ分類、リスク |
| 7 | [aidlc-devsecops-agent](devsecops-agent.md) | 脅威モデリング、セキュリティパイプライン、セキュア設計レビュー |
| 8 | [aidlc-developer-agent](developer-agent.md) | コード生成、リバースエンジニアリング、実装ガイダンス |
| 9 | [aidlc-quality-agent](quality-agent.md) | テスト戦略、受け入れ基準、パフォーマンス検証 |
| 10 | [aidlc-pipeline-deploy-agent](pipeline-deploy-agent.md) | CI/CD パイプライン、デプロイ戦略、リリースの実行 |
| 11 | [aidlc-operations-agent](operations-agent.md) | オブザーバビリティ、インシデント対応、フィードバックループ |

---

← [ユーザーガイド](../00-introduction.md)に戻る · [エージェントの章](../06-agents.md)
