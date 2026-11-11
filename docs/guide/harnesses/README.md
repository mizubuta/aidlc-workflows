# Running on other harnesses

AI-DLC は、ハーネスに依存しない 1 つのコアを、あなたが使う CLI 向けに描画したものです。
方法論 — [フェーズとステージ](../04-phases-and-stages.md)、
[エージェント](../06-agents.md)、[スコープ](../05-scopes-and-depth.md)、
[承認ゲート](../07-interaction-modes.md) — はどのハーネスでも同一です。
異なるのは *外殻* の部分です: ゲートの表示方法、サブエージェントのディスパッチ方法、
発火するセッションイベント、設定の置き場所などです。ここにある各章では、1 つの
ハーネスについて、インストール手順、前提条件、そしてハーネスに依存しない方法論と
異なるいくつかの振る舞いを説明します。

オンボーディングは、各ハーネスのネイティブな指示ファイルの場所にインストールされます:

| ハーネス | オンボーディングファイル |
|---------|-----------------|
| Claude Code | `.claude/CLAUDE.md`（完全なオンボーディング） |
| Kiro CLI と Kiro IDE | `.kiro/steering/aidlc-onboarding.md` |
| Codex CLI | `.codex/onboarding.md`（プロジェクトが信頼済みの場合は、`.codex/config.toml` の `developer_instructions` からも注入されます） |
| Cursor | `.cursor/rules/aidlc-onboarding.mdc` |
| opencode | `.aidlc/onboarding.md` |
| GitHub Copilot | ルートの `AGENTS.md`（メソッドのインポートを含む完全なオンボーディング） |

Kiro、Kiro IDE、Codex、Cursor、opencode は、インストール済みの他のハーネスと、
ハーネスに依存しないルートの `AGENTS.md` ブロックを共有します。それぞれのネイティブな
設定は上記のファイルに残ります。Copilot の完全なルートブロックは引き続き専有です。また、
異なるハーネスを共存させるには、それぞれ別のエンジンディレクトリも必要です。

## Install first

どのハーネスでも、初回実行の推奨手順は、チェックサム検証付きのネイティブ
インストーラーを実行し、続けて `aidlc config` を実行することです:

```bash
tmp="$(mktemp -d)"
curl -fsSL \
  https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.sh \
  -o "$tmp/install.sh"
sh "$tmp/install.sh"
rm -rf "$tmp"
cd your-project
aidlc config
```

インストーラーには常にすべてのハーネスのランタイムが含まれます。`aidlc config --harness <name>` でプロジェクト側の構成を選択します。ホスト側の前提条件は引き続き適用されます: Codex では、
プロジェクトのフック検出のために、対象プロジェクトが Git リポジトリである必要があります。

Windows では、`install.ps1` をダウンロードし、その PowerShell ウィンドウで
`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force` によりスクリプトの実行を許可してから、
`& $installer` を実行します。アカウントのスコープ、ユーザー PATH の自動登録、`-NoModifyPath` については
[Windows でのインストール](../18-install-and-lifecycle.md#windows-powershell) を参照してください。

ハーネスを選んでください:

| ハーネス | 呼び出し | 章 |
|---------|--------|---------|
| **Claude Code** | `/aidlc` | [ユーザーガイド](../00-introduction.md) 全体で扱っています（例は Claude Code で実行しています）。インストールは [Getting Started](../01-getting-started.md) を参照。 |
| **Kiro IDE**（≥ 1.1.70）と **Kiro CLI**（≥ 2.24.1、v3 エンジン）。セットアップでは Kiro（`--harness kiro`）として表示 | `/aidlc` | [Kiro で AI-DLC を実行する](kiro.md) — 両方の環境で共通の 1 つのツリー: 前提条件（Opus 4.8）、インストール、フック、Kiro での相違点。 |
| **Kiro CLI**（≥ 2.6）。セットアップでは Kiro CLI (V2)（`--harness kiro-v2`）として表示 | `/aidlc` | 同じページ内のセクション [Kiro CLI (V2)](kiro.md#kiro-cli-v2) — エンジンを固定しないハーネス: 前提条件、インストール、Kiro での相違点。Kiro CLI ≥ 2.24.1 で Kiro の v3 エンジンを使う場合は、上の Kiro を使ってください。 |
| **Codex CLI**（≥ 0.145.0） | `$aidlc` | [Codex CLI での AI-DLC](codex-cli.md) — 前提条件、信頼設定の事前投入、Bedrock の設定、Git リポジトリの要件。 |
| **Cursor** | `/aidlc` | [Cursor での AI-DLC](cursor.md) — Cursor IDE と CLI で共通の 1 つのツリー、ネイティブのサブエージェントとスキル、hooks.json アダプター、Cursor での相違点。 |
| **opencode**（1.18.29 以降、または OpenCode 2） | `/aidlc` | [opencode での AI-DLC](opencode.md) — `.aidlc/` + `.opencode/` に分かれたレイアウト、アダプタープラグイン、opencode での相違点。 |
| **GitHub Copilot**（CLI ≥ 1.0.74 / VS Code ≥ 1.130） | `/aidlc` | [GitHub Copilot での AI-DLC](copilot.md) — 両方の環境に対する 1 回のインストール、`.github/` のマージ、フォルダーの信頼、Copilot での相違点。 |

Kiro（IDE または CLI）上の AI-DLC は **Claude Opus 4.8** で最もよく動作します。これには **Kiro の有料プラン** が必要です。

手動でコピーする場合は、Bun をインストールし、特定リリースの
`aidlc-copy-runtime-X.Y.Z.tar.gz` をダウンロードして展開し、
`runtime/<harness>/` ディレクトリを丸ごとコピーしてから、そのセットアップを一度実行します（
[Copy Channel](../18-install-and-lifecycle.md#copy-channel) を参照）。ネイティブの `aidlc`
実行ファイルは不要です。コピーによってあなたの `.gitignore` や
`AGENTS.md` が置き換えられることはありません。AI-DLC は独自の行をそれらに追加します。
リポジトリのチェックアウトから生成済みツリーをコピーしないでください。フレームワーク開発者は代わりに、ソースのチェックアウトで
`bun scripts/package.ts` を実行して、無視対象（ignored）のローカル出力
`dist/` と `dist-release/` を生成できます。各ハーネスの章では、手動コピーの
手順を、明示的にラベル付けされた代替手段として記載しています。

`aidlc update` の後は、`aidlc doctor` を実行してプロジェクトとランタイムのバージョンのずれを確認し、
`aidlc config` で各プロジェクトを更新してください。ワークフローが開いている間に更新を行っても
更新は完了し、その作業が継続されるかどうかと、以前のリリースに
戻すためのコマンドが表示されます。

このセットは開かれています: 新しいハーネスは、同じテンプレートから追加された独自の章を
ここに持ちます。新しいハーネスを *構築する* 場合（ソース側の契約 — マニフェスト、フック
アダプター、`emit.ts`）は、Harness Engineer Guide の
[新しいハーネスへの移植](../../harness-engineering/09-porting-to-a-new-harness.md) を参照してください。

どのハーネスを使う場合でも、方法論は同じです — まずは
[最初のワークフロー](../02-your-first-workflow.md) と
[フェーズとステージ](../04-phases-and-stages.md) のツアーから始めてください。

ワークショップを運営したり、チームを支援したりしていますか？
[ファシリテーターガイド](../facilitator-guide.md#how-strongly-each-harness-enforces-the-workflow)
では、各ハーネスがワークフローをどの程度強く強制するかを比較し、
準備状況のチェックと復旧のプレイブックを紹介しています。
