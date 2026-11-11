# Quality Agent

> **エージェント詳細解説** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [詳細解説](README.md) · 技術リファレンス: [quality-agent](../../reference/agents/quality-agent.md)

aidlc-quality-agent は、QA エンジニア兼パフォーマンスのスペシャリストです。テスト戦略を定義し、テストスイート（ユニット、統合、コントラクト、セキュリティ）を生成し、受け入れ基準に照らしてカバレッジを検証し、負荷テストを設計・実行し、非機能要件の目標値を検証します。実装されたすべてのユニットが受け入れ基準を満たし、システム全体が品質ゲートを満たすことを保証します。

aidlc-quality-agent は 2 つのステージ（Construction で 1 つ、Operation で 1 つ）をリードし、さらに 3 つのステージを支援します。ビルドツール、テストコマンド、パフォーマンステスト用ユーティリティを実行するための Bash アクセス権を持っています。

## Stages Led

| ステージ | フェーズ | 説明 |
|-------|-------|-------------|
| 3.6 Build and Test | Construction | ビルドを実行し、テストスイートを生成・実行し、品質ゲートを検証する |
| 4.6 Performance Validation | Operation | 負荷テスト、NFR 検証マトリクス、キャパシティプランニング |

## Stages Supported

| ステージ | フェーズ | 貢献内容 |
|-------|-------|-------------|
| 2.2 Practices Discovery | Inception | コラボレーターが有効な場合、互いの内容を見ない（mutually blind）テスト方針のスポークとして参加し、自身の貢献ファイルを書く |
| 2.4 User Stories | Inception | コラボレーターが有効な場合、モブの中でテスト容易性と受け入れ基準の観点を担い、自身の貢献ファイルを書く |
| 3.2 NFR Requirements | Construction | テスト可能な品質特性シナリオを定義する |

## What to Expect

aidlc-quality-agent が有効になると、ビルド手順とテストスイートを生成し、実装済みのコードに対して実行します。Build and Test では、プロジェクトのビルドシステムを実行し、ユニットテスト、統合テスト、およびプロジェクトに適したその他のテスト種別を実行します。合否結果、カバレッジ指標、品質ゲートの状態を報告します。

Operation フェーズの Performance Validation では、負荷テストを設計・実行し、NFR の目標値（レイテンシのパーセンタイル、スループット、可用性）を検証し、目標値と実測値を比較する NFR 検証マトリクスを作成します。

## How It Collaborates

aidlc-quality-agent は、aidlc-product-agent から受け入れ基準付きのユーザーストーリーを、aidlc-architect-agent から NFR の目標値を、aidlc-developer-agent から実装済みのコードを受け取ります。セキュリティテストの要件については aidlc-devsecops-agent と、CI への統合については aidlc-pipeline-deploy-agent と連携します。テスト結果とパフォーマンスのベースラインは aidlc-operations-agent に引き継がれます。

## Key Principles

- 実装ではなく要件をテストする
- テストピラミッドに従う: 高速なユニットテストを多く、統合テストは少なめに、エンドツーエンドテストは最小限に
- 欠陥を見つけたら、修正する前にそれを再現するテストを書く
- テストは実行順序や共有状態に依存してはならない
- カバレッジは目安であって目標ではない — 意味のないアサーションで 100% にするより、よく考えたテストで 70% のほうが優れている
