# Introduction

> [AI-DLC ドキュメント](../README.md) の一部 · **ユーザーガイド** · [ハーネスエンジニアガイド](../harness-engineering/00-overview.md) · [開発者リファレンス](../reference/00-overview.md)

## What is AI-DLC?

AI-DLC（AI-Driven Development Life Cycle）は、AI を活用したソフトウェア開発を、繰り返し可能で追跡可能なフェーズに構造化するための方法論です。[AWS の AI-DLC 方法論](https://aws.amazon.com/blogs/devops/ai-driven-development-life-cycle/) に由来しています。このリポジトリは、1 つのハーネス非依存のコアからこの方法論をネイティブに実装しているため、すでにお使いの CLI ハーネス（現在は Claude Code、Kiro CLI、Kiro IDE、Codex CLI、Cursor、opencode、GitHub Copilot）の中で動作します。このガイドはハーネス非依存です。ハーネスによって詳細が異なる箇所ではその旨を記載し、該当するハーネスの章を案内します（[他のハーネスでの実行](harnesses/README.md) を参照）。特に断りのない限り、例は Claude Code で示しています。

1 つのコマンドで起動します:

```
/aidlc Build a REST API for inventory management
```

その後 AI-DLC は、インテントの把握から要件、設計、実装、テスト、デプロイまで、構造化されたワークフローを案内します。その間、すべての意思決定のポイントであなたが主導権を持ち続けます。

## Philosophy: Small Mob, Broad Agents

AI-DLC は、何十もの狭い専門家（ウォーターフォール型の引き継ぎの連鎖を再現してしまうアプローチ）ではなく、**幅広い能力を持つ 11 のエージェント**を使用し、それぞれが複数のステージやフェーズにまたがって参加します。各エージェントはステージをまたいでコンテキストを引き継ぐため、引き継ぎがなくなり、調整のオーバーヘッドが減ります。

これは、効果的な人間のチームの働き方を反映しています。3〜5 人のモブが 1 つの機能全体を担当し、それぞれが単一の狭い専門性ではなく幅広いスキルを持ち寄ります。

## How the Orchestrator Works

AI-DLC の中核では、シンプルなループが動いています。決定的な**エンジン**が次に何をするかを決め、**コンダクター**（`/aidlc` セッション、`SKILL.md`）がそれを実行し、エンジンに次の手を問い合わせます。そのループを通じて、フレームワークは次のことを行います:

1. **ステージファイルを読み込む** — 5 つのフェーズにわたる 33 のステージ定義で、それぞれが入力、ステップ、出力、リードエージェントを規定しています
2. **エージェントのペルソナを読み込む** — 専門的なナレッジを持つドメインエキスパートの視点（アーキテクト、開発者、プロダクトマネージャーなど）を起動します
3. **状態と監査を管理する** — `aidlc-state.md` で進捗を追跡し、トレーサビリティのためにすべての決定をインテントの `audit/` シャードに記録します
4. **ステージのトポロジーに応じて委任する** — 集中した自律作業やマルチエージェントの協働のために、サブエージェントをハブ＆スポーク、パイプライン、モブとしてディスパッチします
5. **承認ゲートを提示する**: Ideation 以降の各ステージの後、ワークフローが進む前にあなたがレビューして承認します（Construction では、定型的なチェックポイントを止まらずに通過させることを選べます）

エンジンはルーティング（次にどのステージに進むか、どのスコープか、いつ止まるか）を担い、コンダクターは実行の品質（ステージをうまく進めること、良い質問をすること、判断をあなたに示すこと）を担います。ほとんどのステージは**インライン**で実行されます。コンダクターがエージェントの視点を採用し、会話の中であなたと直接作業します。4 つのステージはディスパッチ型のトポロジーを使用します。Practices Discovery と Code Generation は `subagent` ハブとして、Reverse Engineering は `pipeline` として、User Stories は `mob` として実行されます。トポロジー全体の内訳は、インライン 29 / サブエージェント 2 / パイプライン 1 / モブ 1 です。これらの支援エージェントは、協力者がオンの場合に参加します（デフォルトでオンにするのは `enterprise` のみです）。それ以外の場合、これらの各ステージはリードエージェントが単独で実行し、Reverse Engineering のパイプラインは 2 リンクではなく 1 リンクになります。アーキテクチャの全体像については、開発者リファレンスの [エンジンとスキルシステム](../reference/17-skill-system.md) を参照してください。

## Who This Guide Is For

このガイドは、AI-DLC を**使って**ソフトウェアを構築するすべての方を対象としています:

- **初めての方** — [はじめに](01-getting-started.md) から始め、続いて [ガイド付きオンボーディング](onboarding.md)、[ワークフロープロファイル](workflow-profiles.md)、[最初のワークフロー](02-your-first-workflow.md)、[スペースとインテント](03-spaces-and-intents.md) をお読みください
- **日常的に使う方** — [CLI コマンド](12-cli-commands.md)、[スコープ、深さ、テスト戦略](05-scopes-and-depth.md)、[トラブルシューティング](15-troubleshooting.md) を参照してください
- **チームリード** — AI-DLC をチームの標準に合わせてカスタマイズするには、[ナレッジ](08-knowledge.md) と [ルールと学習ループ](09-rules-and-the-learning-loop.md) を参照してください

AI-DLC の振る舞い*そのもの*を作り変える（ステージやエージェントの追加、スコープの定義、ルールやセンサーの作成、チームのナレッジの追加。いずれも設定のみでコードは不要です）には、[ハーネスエンジニアガイド](../harness-engineering/00-overview.md) を参照してください。AI-DLC のコードベース自体を変更する場合は、[開発者リファレンス](../reference/00-overview.md) を参照してください。

## Key Numbers

| 指標 | 値 |
|--------|-------|
| フェーズ | 5（Initialization、Ideation、Inception、Construction、Operation） |
| ステージ | 33 |
| エージェント | 合計 14: ドメインエキスパート 11、レビュアー 2、コンポーザー |
| スコープ | 11（enterprise から express まで、および workshop）+ あなたの言葉からエージェントが提案する計画 |
| 深さのレベル | 3（Minimal、Standard、Comprehensive） |
| テスト戦略のレベル | 3（Minimal、Standard、Comprehensive） |
| 監査イベントの種類 | 134 |

## Guide Map

| 章 | 学べること |
|---------|------------------|
| [はじめに](01-getting-started.md) | 前提条件、インストール、最初のヘルスチェック |
| [オンボーディング: ガイド付きの最初の 1 週間](onboarding.md) | メンタルモデルと、初めて使うチーム向けの 5 回の実行で進める道筋 |
| [ワークフロープロファイル](workflow-profiles.md) | Classic、Express、その他のワークフローの選択肢の解説 |
| [最初のワークフロー](02-your-first-workflow.md) | 1 回の完全な実行を注釈付きでたどる解説 |
| [ビジョンドキュメントの書き方](writing-inputs/vision-document-guide.md) と [技術環境ドキュメントの書き方](writing-inputs/technical-environment-guide.md) | 大きめの作業やワークショップの前に書いておく 2 つのメモ |
| [スペースとインテント](03-spaces-and-intents.md) | ワークスペースのレイアウト: スペースとインテントをまたいで多くの作業を進める方法 |
| [フェーズとステージ](04-phases-and-stages.md) | 5 つのフェーズと 33 のステージの解説 |
| [スコープ、深さ、テスト戦略](05-scopes-and-depth.md) | スコープ／深さ／テスト戦略の選び方と上書きの方法 |
| [エージェント](06-agents.md) | 14 のエージェントの一覧: ドメインエキスパート 11、レビュアー 2、コンポーザー |
| [エージェントの詳細](agents/README.md) | エージェントごとのリファレンスページ: 責任、ステージ、ナレッジ |
| [インタラクションモード](07-interaction-modes.md) | Guide Me / Edit File / Chat と承認ゲート |
| [ナレッジ](08-knowledge.md) | 社内標準の追加とチームのドキュメントのカタログ化 |
| [ルールと学習ループ](09-rules-and-the-learning-loop.md) | 自己学習する振る舞いのルール |
| [状態と監査](10-state-and-audit.md) | 進捗と決定がどのように追跡されるか |
| [セッション管理](11-session-management.md) | 再開、やり直し、ジャンプ、復旧、セッションレポートのスキル |
| [CLI コマンド](12-cli-commands.md) | 例付きの完全なフラグリファレンス |
| [カスタマイズ](13-customization.md) | 設定、スコープの構成、エージェントの調整 |
| [成果物リファレンス](14-artifacts-reference.md) | インテントごとの記録ディレクトリ（`aidlc/spaces/<space>/intents/<YYMMDD>-<label>/`）の解説 |
| [トラブルシューティング](15-troubleshooting.md) | 症状別の問題解決 |
| [実践例](16-worked-examples.md) | bugfix と feature の完全なウォークスルー |
| [スキルとランナーコマンド](17-skills.md) | `/aidlc-*` のステージランナー／スコープランナーのコマンドと、独自のランナーを作成する方法 |
| [インストールとライフサイクル](18-install-and-lifecycle.md) | ネイティブインストール、init、アップグレード、ロールバック、ピン留め、オフラインパッケージ、アンインストール、プロジェクトからの AI-DLC の削除 |
| [マルチチームの Construction とワークショップモード](workshop-mode.md) | クローンや兄弟ワークツリーで作業するチーム向けの、claim、ビルド、ピン留めされたマージバック、リリース、ワークショップの流れ |
| [他のハーネスでの実行](harnesses/README.md) | Kiro CLI、Kiro IDE、Codex CLI、Cursor、opencode、GitHub Copilot でのインストールと実行、およびハーネスごとの違い |
| [用語集](glossary.md) | すべての用語の定義 |
