# DevSecOps Agent

> **エージェント詳解** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [詳解一覧](README.md) · 技術リファレンス: [devsecops-agent](../../reference/agents/devsecops-agent.md)

aidlc-devsecops-agent は、あなたのセキュリティエンジニアです。セキュリティを最後に後付けするのではなく、ライフサイクルのすべてのフェーズに組み込むことを保証します。Ideation で特定されたコンプライアンス要件を受け取り、セキュリティコントロール、脅威モデル、スキャンパイプライン、ランタイム監視として実装します。アプリケーションセキュリティ、クラウドセキュリティ、パイプラインセキュリティを担当します。

aidlc-compliance-agent と同様に、aidlc-devsecops-agent はもっぱら支援役として動作します。Inception、Construction、Operation にまたがる 5 つのステージでセキュリティの専門知識を提供します。セキュリティスキャンツールを実行するために Bash へのアクセス権を持ちます。

## Stages Led

aidlc-devsecops-agent がリードするステージはありません。

## Stages Supported

| ステージ | フェーズ | 貢献内容 |
|-------|-------|-------------|
| 2.2 Practices Discovery | Inception | collaborators が on のとき、互いの内容を見ない（mutually blind）セキュリティプラクティス担当のスポークとして参加し、自身の貢献ファイルを書く |
| 3.2 NFR Requirements | Construction | セキュリティコントロール、脅威モデル、STRIDE 分析 |
| 3.4 Infrastructure Design | Construction | IAM ポリシーのレビュー、セキュリティグループの検証 |
| 3.6 Build and Test | Construction | SAST/DAST スキャン、依存関係の脆弱性、IaC の lint |
| 4.2 Environment Provisioning | Operation | セキュリティ態勢の検証（Security Hub、Inspector、GuardDuty） |

## What to Expect

aidlc-devsecops-agent が（支援エージェントとして）稼働しているときは、攻撃対象領域、信頼境界、セキュリティコントロールに注目します。設計にセキュリティのアンチパターンがないかをレビューし、機密データのフローが暗号化されアクセス制御されていることを検証し、サードパーティの依存関係に既知の脆弱性がないかを評価します。

## How It Collaborates

aidlc-devsecops-agent は、aidlc-compliance-agent から規制要件を、aidlc-architect-agent からシステム設計を受け取ります。セキュアコーディングのプラクティスについては aidlc-developer-agent と、インフラのハードニングについては aidlc-aws-platform-agent と、セキュリティテストの要件については aidlc-quality-agent と連携します。セキュリティゲートとスキャン設定は aidlc-pipeline-deploy-agent に引き渡されます。

## Key Principles

- 多層防御 — 単一のセキュリティコントロールが単一障害点になってはならない
- あらゆる場所で最小権限 — すべてのユーザー、サービス、プロセスに最小限の権限を与える
- 侵害を前提とする — 内部コンポーネント同士も相互に認証・認可しなければならない
- デフォルト設定は安全でなければならない
- すべての入力は検証されるまで敵対的とみなし、すべての外部データはサニタイズされるまで汚染されているとみなす
- セキュリティは要件であり、後回しにできる機能ではない
