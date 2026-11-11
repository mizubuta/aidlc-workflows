# Operations Agent

> **エージェント詳細解説** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [詳細解説](README.md) · 技術リファレンス: [operations-agent](../../reference/agents/operations-agent.md)

aidlc-operations-agent は、サイト信頼性エンジニア兼インシデントマネージャーです。デプロイされたシステムが観測可能で、レジリエントで、継続的に改善されていくことを保証します。CloudWatch のダッシュボードとアラームから、X-Ray トレーシング、SLO の追跡、インシデント対応ランブック、カオスエンジニアリングによる検証まで、運用レイヤーを担当します。特に重要なのは、本番環境から得られた知見を次のイテレーションに向けて Ideation へ還流させ、フィードバックループを閉じることです。

aidlc-operations-agent は Operation フェーズの 3 つのステージをリードします。監視のセットアップコマンド、ランブックスクリプト、診断ツールを実行するために Bash へのアクセス権を持っています。

## Stages Led

| ステージ | フェーズ | 説明 |
|-------|-------|-------------|
| 4.4 Observability Setup | Operation | ダッシュボード、アラーム、トレーシング、構造化ログ、カスタムメトリクス |
| 4.5 Incident Response | Operation | SSM ランブック、インシデント計画、エスカレーションマトリクス、オンコール体制 |
| 4.7 Feedback & Optimization | Operation | SLO レポート、コスト分析、ドリフト検出、フィードバックループ |

4.6 Performance Validation は aidlc-quality-agent がリードします。このエージェントが 4.4 で整備する運用テレメトリとベースラインは非公式にその作業へ活用されますが、4.6 の正式なサポートエージェントではありません。

## Stages Supported

ありません。aidlc-operations-agent はリード専任のエージェントであり、他のエージェントのステージはサポートしません。リードするステージはすべて Operation に属し、そこでライフサイクルのループを閉じます。

## What to Expect

aidlc-operations-agent がアクティブになると、監視に関する希望、SLO の目標値、インシデント対応プロセス、オンコール体制について質問します。CloudWatch ダッシュボードの設定、しきい値と通知先を含むアラーム定義、X-Ray トレーシングのセットアップ、よくあるシナリオ（サービスの再起動、キャッシュのフラッシュ、フェイルオーバー）向けの SSM ランブック、エスカレーションパスを含むインシデント重大度の定義を作成します。

Feedback & Optimization（最終ステージ）では、本番環境のメトリクスを分析して最適化の機会を特定し、運用上の知見を次の開発サイクルに向けて aidlc-product-agent へ還流させるフィードバックループ文書を作成します。

## How It Collaborates

aidlc-operations-agent は、aidlc-aws-platform-agent からプロビジョニング済みのインフラを、aidlc-pipeline-deploy-agent からデプロイ済みのサービスを受け取ります。パフォーマンスのベースラインと SLO の検証では aidlc-quality-agent と、アプリケーションレベルのログ改善では aidlc-developer-agent と連携します。そのフィードバックレポートは Operation から Ideation へ戻る橋渡しとなり、ライフサイクルのループを完成させます。

## Key Principles

- 包括的にテレメトリを収集しつつ、アラートはユーザーに影響する問題に限る — アラート疲れは対応の質を下げます
- SLO が信頼性の目標を定め、それ以外はすべてそこから導かれます
- すべてのインシデントは学びの機会です — 非難しないポストモーテムによってインシデントを改善につなげます
- テストされていないレジリエンスの仕組みは仮定にすぎません — カオスエンジニアリングで検証します
- Ideation へ還流しない本番環境の知見は、無駄になった学びです
- 手作業の運用トイルは排除しなければなりません — 繰り返し可能なランブックの手順はすべて自動化します
