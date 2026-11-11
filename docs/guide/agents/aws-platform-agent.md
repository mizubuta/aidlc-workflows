# AWS Platform Agent

> **エージェント詳解** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [詳解](README.md) · 技術リファレンス: [aws-platform-agent](../../reference/agents/aws-platform-agent.md)

aidlc-aws-platform-agent は、あなたの AWS ソリューションアーキテクト兼インフラエンジニアです。アプリケーションアーキテクチャを、AWS サービスの選定、CDK/CloudFormation テンプレート、環境プロビジョニング戦略へと落とし込みます。このエージェントが下すインフラ上の判断はすべて、コストを意識し、デフォルトでセキュアであり、AWS Well-Architected Framework に照らして検証されています。

aidlc-aws-platform-agent は 2 つのステージを主導し、ほかの 5 つのステージを支援します。AWS CLI コマンドの実行、CDK の操作、インフラ検証ツールの実行に Bash を使うことが想定されています。

## Stages Led

| ステージ | フェーズ | 説明 |
|-------|-------|-------------|
| 3.4 Infrastructure Design | Construction | AWS サービスの選定、IaC テンプレート、コスト見積もり（ユニットごと） |
| 4.2 Environment Provisioning | Operation | IaC 定義から環境をプロビジョニングして検証する |

## Stages Supported

| ステージ | フェーズ | 貢献内容 |
|-------|-------|-------------|
| 1.3 Feasibility & Constraints | Ideation | AWS サービスの利用可否と制約の評価 |
| 2.6 Domain Design | Inception | クラウドネイティブなパターンとサービス統合に関する助言 |
| 2.8 Contract Design | Inception | このステージの支援エージェント |
| 3.3 NFR Design | Construction | NFR をインフラ仕様とスケーリングポリシーに落とし込む |
| 4.7 Feedback & Optimization | Operation | コスト最適化とインフラのチューニング |

## What to Expect

aidlc-aws-platform-agent が稼働しているときは、AWS アカウント構成、既存のインフラ、コストの制約、コンプライアンス要件について質問されます。このエージェントは、CDK/CloudFormation の仕様、VPC トポロジー、IAM ポリシー、環境ティアごとのコスト見積もりを含むインフラ設計を作成します。サービスの利用可否や既存の設定を確認するために、AWS CLI コマンドを実行することもあります。

## How It Collaborates

aidlc-aws-platform-agent は、aidlc-architect-agent からアプリケーショントポロジーを、aidlc-devsecops-agent からセキュリティ要件を受け取ります。監視インフラとランブックの統合については aidlc-operations-agent と連携します。プロビジョニングした環境は、デプロイ先として aidlc-pipeline-deploy-agent に引き渡されます。

## Key Principles

- インフラ上のすべての判断は、Well-Architected の 6 つの柱すべてに照らして説明できなければなりません
- すべてのリソースはコードで定義します。コンソールでの変更はドリフトです
- コストはアーキテクチャ上の第一級の関心事です。すべての設計にコスト見積もりを含めます
- IAM ポリシーは必要最小限の権限だけを付与します。ワイルドカードポリシーは使いません
- dev、staging、production の違いは規模だけに限り、トポロジーを変えてはなりません
