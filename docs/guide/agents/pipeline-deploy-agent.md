# Pipeline & Deploy Agent

> **エージェントの詳細** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [エージェントの詳細](README.md) · 技術リファレンス: [pipeline-deploy-agent](../../reference/agents/pipeline-deploy-agent.md)

aidlc-pipeline-deploy-agent は、あなたの CI/CD エンジニア兼リリースマネージャーです。ビルド仕様とインフラストラクチャのターゲットを、品質ゲート、安全なロールバック、完全な監査可能性を備え、コードをコミットから本番環境まで運ぶ完全に自動化されたパイプラインに落とし込みます。

aidlc-pipeline-deploy-agent は、Inception、Construction、Operation にまたがる 4 つのステージをリードします。パイプラインツール、デプロイスクリプト、スモークテストのコマンドを実行するための Bash へのアクセス権を持ちます。

## Stages Led

| ステージ | フェーズ | 説明 |
|-------|-------|-------------|
| 2.2 Practices Discovery | Inception | ハブ＆スポークの調査結果を起草し、インタビューを行い、統合します。承認されたプラクティスを、アクティブなスペースのチーム／プロジェクトのメモリに昇格させます |
| 3.7 CI Pipeline | Construction | 品質ゲートを備えた CI パイプラインの構成 |
| 4.1 Deployment Pipeline | Operation | デプロイ戦略とロールバック手順を備えた CD パイプライン |
| 4.3 Deployment Execution | Operation | デプロイの実行、スモークテストの実行、正常性の監視 |

## Stages Supported

aidlc-pipeline-deploy-agent は、助言的な立場でサポートするステージを持ちません。

## What to Expect

Practices Discovery では、aidlc-pipeline-deploy-agent がまず草案を作成し、利用者へのインタビューの後に、互いの内容を知らずに作成された quality、developer、devsecops の貢献を統合します（協力者がオフの場合、草案はそのままインタビューに進みます）。デリバリーのステージでは、CI/CD インフラストラクチャ、デプロイ先、ブランチ戦略、ロールバックの要件について質問し、パイプラインの構成、デプロイ戦略、ロールバックのランブックを作成し、Deployment Execution を監視します。

## How It Collaborates

aidlc-pipeline-deploy-agent は、aidlc-developer-agent からビルド可能なソースとテストスイートを、aidlc-quality-agent から品質ゲートの定義を、aidlc-aws-platform-agent から環境のエンドポイントを受け取ります。デプロイされたサービスはオブザーバビリティの設定のために aidlc-operations-agent に引き渡され、デプロイの成果物はパフォーマンスの検証のために aidlc-quality-agent に渡されます。

## Key Principles

- すべてのコミットはリリース候補です。すべてのゲートを通過すれば、本番環境に出せる状態です
- すべてのデプロイには、テスト済みのロールバック経路が必要です
- CI パイプラインは数時間ではなく数分で完了すべきです。遅いパイプラインは変更をまとめて投入することを助長します
- 品質ゲートは、欠陥のある成果物が利用者に届くのを防ぐために存在します
- スモークテストでサービスが正常であることを確認するまで、デプロイは完了していません
