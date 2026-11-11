# Developer Agent

> **エージェントの詳細** · [ユーザーガイド](../00-introduction.md) › [エージェント](../06-agents.md) › [エージェントの詳細](README.md) · 技術リファレンス: [developer-agent](../../reference/agents/developer-agent.md)

aidlc-developer-agent は、あなたのシニアソフトウェア開発者です。アーキテクチャ設計とユニットの仕様を、本番品質のコードに落とし込みます。リバースエンジニアリングでは、aidlc-architect-agent が統合するための詳細なコードスキャンを行います。

aidlc-developer-agent は Reverse Engineering のコードスキャンと Code Generation をリードし、Inception の 2 つのアンサンブルの両方にディスパッチされる協力者として参加します。Practices Discovery のハブ＆スポークではコードパターンの証拠を調べ、User Stories のモブでは実装可能性の観点から意見を述べます。Code Generation は 1 つのエージェントに絞ったサブエージェントとして実行されます。ビルドツール、パッケージマネージャー、テストコマンドの実行には Bash を使うことが想定されています。

Workspace Detection を含む Initialization は決定的なツールとして実行され、aidlc-developer-agent は関与しません。

## Stages Led

| ステージ | フェーズ | 説明 |
|-------|-------|-------------|
| 2.1 Reverse Engineering（コードスキャン） | Inception | アーキテクトが統合するための構造化された分析を生成する詳細なコードスキャン。協力者がオフの場合は、9 つの成果物も自ら作成します |
| 3.5 Code Generation | Construction | 設計仕様からユニットを実装します（ユニットごと） |

## Stages Supported

| ステージ | フェーズ | 貢献内容 |
|-------|-------|-------------|
| 2.2 Practices Discovery | Inception | 協力者がオンの場合、他と内容を共有しないコードパターンのスポークとして参加し、自身の貢献ファイルを書きます |
| 2.4 User Stories | Inception | 協力者がオンの場合、モブのアンサンブルで実装可能性の観点から意見を述べ、自身の貢献ファイルを書きます |
| 3.1 Functional Design | Construction | API コントラクトとデータモデルに関するインプット |
| 4.3 Deployment Execution | Operation | データベースのマイグレーション |

## What to Expect

Code Generation の間、aidlc-developer-agent はサブエージェントとして実行されるため、あなたが直接やり取りすることはありません。進捗インジケーターが表示され、完了すると結果が表示されます。オーケストレーターはまずコード生成の計画を提示してあなたの承認を求め、その後サブエージェントが各ステップを実装します。

アプリケーションのコードは、（インテントの記録ディレクトリではなく）ワークスペースルートに直接書き込まれます。インテントの記録ディレクトリにある `code-summary.md` 成果物に、作成または変更された内容が記録されます。

## How It Collaborates

aidlc-developer-agent は、aidlc-architect-agent からユニットの仕様と設計パターンを、aidlc-quality-agent からテスト要件を受け取ります。CDK／インフラストラクチャの整合については aidlc-aws-platform-agent と、セキュアコーディングについては aidlc-devsecops-agent と協力します。コードスキャンの結果は統合のために aidlc-architect-agent に渡され、実装したコードはテストのために aidlc-quality-agent に引き渡されます。

## Key Principles

- 動作し、テストされた実装を提供します。Testing Contract にリファクタリングのステップが含まれる場合（TDD、BDD、ATDD、またはカスタム）は、最初の生成の中でリファクタリングを行います
- プロジェクトの既存のパターンと規約に従います
- 読みやすくデバッグしやすいコードを書きます — 凝った抽象化は避けます
- 入力を早い段階で検証し、意味のあるエラーを投げ、例外を決して握りつぶしません
- 生成されるすべてのユニットには、少なくとも正常系のテストが含まれます
- リバースエンジニアリングでは、スキャンの徹底度が統合の品質を左右します
