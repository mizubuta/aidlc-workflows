# AI-DLC - one core, many harnesses

AI-DLC（AI-Driven Development Life Cycle）は、AI コーディングアシスタントを、
構造化され検証可能なソフトウェアデリバリーのワークフローへと変えます。ハーネスに依存しない
1 つのコアが、Claude Code、Kiro CLI、Kiro IDE、Codex CLI、Cursor、opencode、
GitHub Copilot でネイティブに動作します。

![version](https://img.shields.io/badge/version-2.11.3-blue)
![license](https://img.shields.io/badge/license-MIT--0-green)

以下のクイックスタートでは、AI-DLC の最新の安定版リリースをインストールします。

## Quick Start

### 1. Install AI-DLC

macOS、Linux、または WSL:

```bash
curl -fsSL https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.sh | sh
```

Windows PowerShell の場合:

```powershell
irm https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.ps1 | iex
```

インストーラーは、ネイティブの `aidlc` コマンドとすべてのハーネス用ランタイムという、
AI-DLC に必要なものをすべてセットアップします。Bun や Node.js をインストールする必要はありません。
ネイティブインストールでは、フックとコマンドを `aidlc` 経由で実行し、`bun` は呼び出しません。
Windows では現在のアカウント向けにインストールされ、
bin ディレクトリが自動的にユーザーの PATH に登録されます。通常の
PowerShell ウィンドウから実行してください。"Run as administrator"（管理者として実行）で開いたウィンドウでは、
管理者としてのインストールは安全性が低いため、警告と確認が表示されます。
別のセッションで `aidlc` が見つからない場合は、新しいターミナルを開いてください。永続的な PATH の変更と
現在のプロセスの PATH の変更をどちらも行わないようにするには、
[`-NoModifyPath`](docs/guide/18-install-and-lifecycle.md#windows-powershell) を使います。
Windows でのアンインストールでは、インストーラーが追加したと記録されているユーザー PATH のエントリだけが削除されます。
macOS、Linux、WSL では、必要に応じてインストーラーが表示する PATH の指示に従ってください。

プロジェクトのファイルを自分で管理したい場合や、ネイティブの実行ファイルをインストールできない場合は、
手動コピーの方法を使えます。Bun が必要になるのはこの方法だけです:
[Bun](https://bun.sh/) をインストールし、[リリース](https://github.com/awslabs/aidlc-workflows/releases/latest) から
`aidlc-copy-runtime-X.Y.Z.tar.gz` をダウンロードして、
`runtime/<harness>/` ディレクトリ全体をプロジェクトにコピーします。この方法では
ネイティブの `aidlc` コマンドは不要です。

### 2. Configure a project

プロジェクトのルートで、使用するハーネスを選択します:

```bash
cd /path/to/your-project
aidlc config --harness claude
aidlc doctor
```

`claude` は `kiro`、`kiro-v2`、`codex`、`cursor`、`opencode`、
`copilot` のいずれかに置き換えてください。`--harness` を付けずに `aidlc config` を実行すると、
ターミナルが利用可能な場合は対話式のセットアップが始まります。まだ信頼していない
プロジェクトフォルダで Kiro IDE 自身のターミナルを使うと、Kiro はまずそのフォルダを信頼するかどうかを尋ねます。
信頼すると、そのフォルダの `.kiro` フックがあなたのマシン上でコマンドを実行できるようになるため、
**Trust Folder & Continue** を選ぶのは自分のプロジェクトか、確認済みのプロジェクトの場合だけにしてください。
それ以外の場合は **Cancel** を選び、先にフォルダの内容を確認してください
（[初回実行](docs/guide/harnesses/kiro.md#first-run) を参照）。

### 3. Start a workflow

設定したプロジェクトでハーネスを開き、作業内容を記述します:

```text
/aidlc Build a REST API for inventory management
```

Codex CLI では `/aidlc` の代わりに `$aidlc` を使います。Kiro IDE では、まずチャットパネルの
エージェント選択で **aidlc** を選びます。AI-DLC は依頼内容からワークフローを選択し、
不足している判断について質問し、先へ進む前に承認ゲートで停止します。

プロバイダーの設定、信頼の確認、ハーネス固有の前提条件については、下の表にある
ガイドを参照してください。完全な手順は
[はじめに](docs/guide/01-getting-started.md) にあります。

## Pick your harness

| ハーネス | 設定 | 起動 | 呼び出し | ガイド |
| --- | --- | --- | --- | --- |
| Claude Code | `aidlc config --harness claude` | `claude` | `/aidlc` | [はじめに](docs/guide/01-getting-started.md) |
| Kiro: Kiro IDE >= 1.1.70 / Kiro CLI >= 2.24.1 (v3) | `aidlc config --harness kiro` | Kiro IDE でプロジェクトを開いてチャットパネルのエージェント選択で **aidlc** を選ぶか、`kiro-cli --v3` を実行する（Kiro がフォルダを信頼するか尋ねてきたら、中身を把握している場合は **Trust, continue** を選ぶ） | `/aidlc` | [Kiro](docs/guide/harnesses/kiro.md) |
| Kiro CLI (V2) >= 2.6 | `aidlc config --harness kiro-v2` | `kiro-cli --v2`（Kiro CLI 2.21.4 以降。それより古い場合は [Kiro CLI (V2) の前提条件](docs/guide/harnesses/kiro.md#kiro-cli-v2-prerequisites) を参照） | `/aidlc` | [Kiro CLI (V2)](docs/guide/harnesses/kiro.md#kiro-cli-v2) |
| Codex CLI >= 0.145.0 | `aidlc config --harness codex` | `codex` | `$aidlc` | [Codex CLI](docs/guide/harnesses/codex-cli.md) |
| Cursor | `aidlc config --harness cursor` | Cursor を開くか、`agent` を実行する | `/aidlc` | [Cursor](docs/guide/harnesses/cursor.md) |
| opencode >= 1.18.29 / OpenCode 2 | `aidlc config --harness opencode` | `opencode` | `/aidlc` | [opencode](docs/guide/harnesses/opencode.md) |
| GitHub Copilot CLI >= 1.0.74 / VS Code >= 1.130 | `aidlc config --harness copilot` | Copilot CLI または VS Code | `/aidlc` | [GitHub Copilot](docs/guide/harnesses/copilot.md) |

AI-DLC 2.11.3 以前、および 2026-11-04 と 2026-11-05 の 2.11.4 プレビューでは、2 つの Kiro の名前の対応が逆になっています: Kiro が `--harness kiro-ide`、Kiro CLI (V2) が `--harness kiro` であり、`kiro-v2` は認識されません。`--harness kiro-ide` は、それらのリリースでもこのリリースでも Kiro を指します。

モデルプロバイダーの設定はハーネス側の役割です。同梱のプロジェクト設定は、
ユーザーがすでに選択しているプロバイダーとモデルをそのまま維持します。`aidlc config
providers` は、対応しているプロジェクト環境では Amazon Bedrock の設定を適用でき、
その他のハーネスについては手動セットアップの内容を記録できます。Kiro CLI と Kiro IDE では、
モデルへのアクセスが Kiro に含まれているため、プロバイダーに関する回答は不要です。Kiro CLI
2.24.1 以降で Kiro の v3 エンジンを使う場合は、`kiro` を使って `kiro-cli --v3` を起動します（フォルダを信頼するか
尋ねられたら、中身を把握している場合は **Trust, continue** を選びます）。`kiro-v2` は Kiro CLI (V2) です
（既存の Kiro CLI (V2) プロジェクトの切り替え方法は [Kiro CLI (V2)](docs/guide/harnesses/kiro.md#kiro-cli-v2) を
参照してください）。方法論そのものは
プロバイダーに依存しません。

## Recommended Model

AI-DLC は、高い推論能力を持つモデルで最もよく機能します。現在の推奨モデルは
Claude Opus 4.8 です。Opus を利用できない場合や、ハーネスでモデルと effort をどこで設定すればよいか
わからない場合は、
[モデルと effort の選び方](docs/guide/18-install-and-lifecycle.md#choosing-a-model-and-effort) を参照してください。

## Why AI-DLC

場当たり的な AI コーディングでは、プロジェクトが大きくなるにつれてコンテキストが失われます。AI-DLC は、要件、
判断、実装、テスト、運用作業を、監査された 1 つのライフサイクルを通じて
つなぎ続けます:

- 初期化から運用までの 5 フェーズと 33 ステージ
- 14 のエージェント: 11 のドメインエキスパート、2 つのレビュアー、適応型のコンポーザー
- 機能開発、バグ修正、インフラ、セキュリティ、PoC、エンタープライズ向けデリバリー、
  その他の一般的な作業向けの 11 のワークフロープロファイル
- 人による承認ゲートと、ソースに紐づくレビューのエビデンス
- 134 種類のイベントからなる監査証跡に加え、永続的な状態、チームのナレッジ、学習済みのルール
- サポートするすべてのハーネスで共通の決定的なエンジン

まずは [ワークフロープロファイル](docs/guide/workflow-profiles.md) で、
Classic、Express、および特定用途向けのワークフローを比較してください。アーキテクチャと方法論については
[AI-DLC Workflows 2.0 仕様書](assets/AI-DLC-Workflows-2.0-Specification.pdf)
を参照してください。

> [!IMPORTANT]
> 生成 AI は誤りを犯すことがあります。生成された出力とコストは、それに基づいて行動する前に
> 確認してください。[AWS 責任ある AI ポリシー](https://aws.amazon.com/ai/responsible-ai/policy/) を参照してください。

## Documentation

| ガイド | 用途 |
| --- | --- |
| [はじめに](docs/guide/01-getting-started.md) | インストール、設定、最初のワークフローの実行 |
| [ユーザーガイド](docs/guide/00-introduction.md) | ワークフロー、プロファイル、エージェント、ナレッジ、承認ゲートの使い方 |
| [ハーネス別ガイド](docs/guide/harnesses/README.md) | プロバイダー、信頼設定、ランタイムの違いへの対処 |
| [インストールとライフサイクル](docs/guide/18-install-and-lifecycle.md) | 更新、バージョン固定、オフラインインストール、ミラーの利用、アンインストール、プロジェクトからの AI-DLC の削除 |
| [ハーネスエンジニアガイド](docs/harness-engineering/00-overview.md) | ステージ、エージェント、ルール、センサー、ナレッジのカスタマイズ |
| [開発とリリース](DEVELOPERS.md) | PR を AI レビュー、プレビューテスト、安定版の公開まで進める |
| [開発者リファレンス](docs/reference/00-overview.md) | エンジン、フック、パッケージング、テストの変更 |

## Repository Layout

- `core/` - 手書きの、ハーネスに依存しない方法論とエンジン
- `core/tools/` - aidlc-*.ts エンジンとオーサリングツール
- `harness/<name>/` - 薄いハーネス固有のマニフェストと統合
- `plugins/<name>/` - オプションの AIDLC プラグイン
- `scripts/` - パッケージング、バイナリ、インストーラー、リリース用のツール
- `tests/` - スモーク、単体、結合、エンドツーエンドのテスト
- `docs/` - ユーザー向け、ハーネスエンジニアリング向け、開発者向けのドキュメント
- `dist/` と `dist-release/` - 生成物（git では無視されるローカル出力）

編集するのは `core/` または `harness/<name>/` であり、生成された `dist*` の出力は決して編集しないでください。

## Development

依存関係をインストールし、すべてのハーネスを生成します:

```bash
bun install --frozen-lockfile
bun scripts/package.ts
```

便利なコマンド:

```bash
bun scripts/package.ts <name>     # generate one harness
bun scripts/package.ts --check    # determinism guard
bun tests/run-tests.ts --ci       # smoke, unit, and integration
bun tests/run-tests.ts --release  # full release acceptance
```

開発ワークフローの全体については [コントリビューションガイド](docs/reference/11-contributing.md) を、
別のランタイムを追加する方法については [新しいハーネスへの移植](docs/harness-engineering/09-porting-to-a-new-harness.md)
を参照してください。

## Troubleshooting

まずプロジェクトのルートで `aidlc doctor` を実行してください。よくある対処法:

| 症状 | 対処 |
| --- | --- |
| `aidlc` が見つからない | Windows では、ターミナルアプリまたは IDE を終了して開き直すか（インストール中に開いていたアプリ内で新しいタブを開いても古い PATH のままです）、[`-NoModifyPath`](docs/guide/18-install-and-lifecycle.md#windows-powershell) 指定時に表示される直接実行用のコマンドを使ってください。Unix では、インストーラーが表示する PATH の指示を適用してください。 |
| プロジェクトとランタイムのバージョンの不一致 | 実行中のワークフローを完了させてから `aidlc config` を実行する |
| Codex のフックが実行されない | [Codex ガイド](docs/guide/harnesses/codex-cli.md) の説明に従ってプロジェクトのフックを信頼する |
| Bedrock へのアクセスに失敗する | 設定したモデルを有効化し、AWS の認証情報とリージョンを確認する |
| 更新後にプラグインのステージが消える | `/aidlc plugin sync` を実行する |
| 更新したスキルが反映されない | ハーネスの新しいセッションを開始する |

診断と復旧の手順については [トラブルシューティング](docs/guide/15-troubleshooting.md) を
参照してください。

## References

- [AWS AI-DLC ブログ記事](https://aws.amazon.com/blogs/devops/ai-driven-development-life-cycle/)
- [AI-DLC 手法定義ペーパー](https://prod.d13rzhkk8cj2z0.amplifyapp.com/)
- [ロードマップ](https://awslabs.github.io/aidlc-workflows/roadmap.html)
- [ライセンス](LICENSE)
