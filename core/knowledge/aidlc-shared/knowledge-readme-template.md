# Team Knowledge

ここに Markdown ファイルを追加すると、プロジェクトに合わせて AI-DLC エージェントの振る舞いを拡張できます。

## How It Works

これらのディレクトリに置いたファイルは、すべてのステージで（組み込みの方法論の後に）エージェントによって読み込まれます。次のような用途に使ってください:
- 社内のコーディング標準と規約
- アーキテクチャ上の決定と技術スタックの好み
- ドメイン固有の用語とビジネスルール
- プロジェクト固有のパターンとアンチパターン

## Directory Structure

> この表はスナップショットです — 正式な対応関係は `{{HARNESS_DIR}}/agents/*.md` にある各エージェントの frontmatter に記載されています。

| ディレクトリ | 目的 | ファイル例 |
|-----------|---------|---------------|
| `shared/` | チーム全体の標準 | coding-standards.md, api-conventions.md |
| `aidlc-architect-agent/` | アーキテクチャ上の決定 | tech-stack.md, infrastructure-preferences.md |
| `aidlc-developer-agent/` | コーディングパターン | db-conventions.md, error-handling.md |
| `aidlc-quality-agent/` | テスト標準 | test-strategy.md, coverage-requirements.md |
| `aidlc-design-agent/` | UX/UI ガイドライン | design-system.md, accessibility.md |
| `aidlc-product-agent/` | プロダクトのコンテキスト | roadmap.md, personas.md |
| `aidlc-devsecops-agent/` | セキュリティポリシー | security-baseline.md, compliance-rules.md |
| `aidlc-operations-agent/` | 運用ランブック | monitoring.md, incident-response.md |
| `aidlc-compliance-agent/` | コンプライアンス標準 | data-governance.md, audit-requirements.md |
| `aidlc-aws-platform-agent/` | クラウドインフラストラクチャ | account-structure.md, service-limits.md |
| `aidlc-pipeline-deploy-agent/` | CI/CD の構成 | pipeline-standards.md, deployment-gates.md |
| `aidlc-delivery-agent/` | プロジェクト管理 | sprint-cadence.md, definition-of-done.md |

## Format

ディレクトリに置いた `.md` ファイルはすべて読み込まれます。特別な命名は不要です。ファイルは焦点を絞ってください — 1ファイル1トピックが最もうまくいきます。
