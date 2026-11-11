# Code Analysis Guide

## Package & Build System Discovery

プロジェクトのルートと一般的なサブディレクトリを走査し、次のマーカーを探す:

| ファイル | ビルドシステム | 言語/ランタイム |
|------|-------------|-----------------|
| `package.json` | npm/yarn/pnpm | JavaScript/TypeScript |
| `tsconfig.json` | TypeScript コンパイラ | TypeScript |
| `requirements.txt` / `pyproject.toml` / `setup.py` | pip/poetry/setuptools | Python |
| `Cargo.toml` | Cargo | Rust |
| `go.mod` | Go modules | Go |
| `pom.xml` | Maven | Java/Kotlin |
| `build.gradle` / `build.gradle.kts` | Gradle | Java/Kotlin |
| `Gemfile` | Bundler | Ruby |
| `*.csproj` / `*.sln` | dotnet/MSBuild | C# |
| `Makefile` | Make | 任意 |
| `Dockerfile` / `docker-compose.yml` | Docker | コンテナ化 |
| `serverless.yml` / `template.yaml` | Serverless/SAM | クラウド関数 |
| `cdk.json` / `cdktf.json` | CDK/CDKTF | インフラストラクチャ |

## Framework Detection Patterns

インポートと設定を走査してフレームワークを特定する:
- **React**: `import React`、`jsx`/`tsx` ファイル、`react-dom`
- **Next.js**: `next.config.js`、`pages/` または `app/` のディレクトリ構成
- **Express**: `require('express')`、`app.get/post/use` のパターン
- **FastAPI**: `from fastapi import`、`@app.get` デコレーター
- **Django**: `INSTALLED_APPS` を含む `settings.py`、`urls.py`、`models.py`
- **Spring Boot**: `@SpringBootApplication`、`application.properties/yml`
- **Rails**: `config/routes.rb`、`app/controllers/`、`ActiveRecord`

## What to Skip

人が書いたものを走査する。ファイルはファイルツールを使い、フォルダーを1つずつ
探す。ディレクトリ読み取りツールがあればそれを使い、なければフォルダーに対して
素の `ls` を1回だけ実行する。リポジトリの `.gitignore` ファイルを読み、そこで無視
されているものは除外する。また、どこでも無視されていない生成物やインストール物も
スキップする:

- **ビルド出力**: `.csproj`、`.fsproj`、`.vbproj` の隣にある .NET の `bin/` と
  `obj/`（および `dotnet publish -o out` による `out/`）、`dist/`、`build/`、
  `target/`、`coverage/`、ならびに `*.dll`、`*.pdb`、`*.class`、`*.pyc` などの
  コンパイル済みファイル
- **依存関係**: `node_modules/`、`.venv/` と `venv/`、`vendor/`、および
  NuGet のソリューションレベルの `packages/`
- **IDE とツールのキャッシュ**: `.vs/`、`.idea/`、`.gradle/`、`__pycache__/`、
  `.pytest_cache/`、`.next/`

フォルダーを開いて判断するのではなく、その隣にあるファイルから判断する。
`.csproj`、`.fsproj`、`.vbproj` の隣にある `bin/`、`obj/`、`out/` はビルド出力であり、
`.sln` の隣にある `packages/` は NuGet のものである。これらは中身を一覧表示したり
読んだりせずにスキップする。同じ名前でも、別の場所では手書きのコードを含む。
Node の `bin/www`、Rails の `bin/` スクリプト、ヘキサゴナルアーキテクチャの
`adapter/out/` パッケージ、あるいは `package.json` の `workspaces` や
`pnpm-workspace.yaml` に記載された JavaScript モノレポの `packages/` などである。
隣のファイルからどちらか判断できない場合に限り、フォルダー内のファイルを1つだけ開く。
ブリーフが走査対象として指定したフォルダーや、`.aidlc-source-paths.json` に
記載されたフォルダーは、隣に何があってもソースとして扱う。

スキップしたフォルダーは、スキャンのカバレッジの **Skimmed only** ではなく
**Left out** の下に列挙する。

## Source File Classification

"What to Skip" で除外されずに残ったすべてのソースファイルを、次のいずれかの
カテゴリに分類する:
- **Model/Entity**: データ構造、データベースモデル、DTO、スキーマ
- **Controller/Handler**: リクエストのルーティング、入力の解析、レスポンスの整形
- **Service/UseCase**: ビジネスロジック、オーケストレーション、ドメイン操作
- **Repository/DAO**: データアクセス、クエリ、永続化の抽象化
- **Utility/Helper**: 横断的な関数、フォーマッター、バリデーター
- **Configuration**: アプリ設定、環境構成、依存性注入
- **Middleware**: リクエスト/レスポンスのパイプライン（認証、ログ記録、エラー処理）
- **Test**: ユニットテスト、統合テスト、フィクスチャ、ファクトリ
- **Migration**: データベーススキーマの変更、データ移行
- **Static/Asset**: テンプレート、スタイルシート、画像、静的コンテンツ

## Dependency Graph Extraction

各ソースファイルについて、次を抽出する:
1. **直接インポート** -- このファイルが依存するモジュール/パッケージ
2. **エクスポートされるシンボル** -- このファイルが提供する関数/クラス/定数
3. **外部依存関係** -- 使用しているサードパーティパッケージ
4. **循環参照** -- 相互にインポートし合っているファイル（これらにはフラグを立てる）

依存関係の隣接リストを構築する: `file -> [dependency1, dependency2, ...]`

## Code Quality Quick Assessment

次の各項目を3段階（good/fair/poor）で評価する:
- **命名の明確さ**: 変数、関数、ファイルの名前がそれ自体で意味を説明しているか。
- **関数のサイズ**: 関数は30行未満で、単一責任になっているか。
- **エラー処理**: エラーが適切に捕捉、記録、伝播されているか。
- **テストの有無**: 重要な経路に対応するテストファイルがあるか。
- **重複**: 抽象化すべきコピー＆ペーストのパターンがあるか。
- **デッドコード**: 未使用のインポート、到達不能な分岐、コメントアウトされたブロックがあるか。

## API Endpoint Inventory

検出した各エンドポイントについて、次を記録する:
- HTTP メソッドとパス（または GraphQL のオペレーション名）
- リクエストパラメーター（パス、クエリ、ボディ、ヘッダー）
- レスポンスの形とステータスコード
- 認証/認可の要件
- レート制限またはスロットリングの設定
- 関連するミドルウェアチェーン

## Technical Debt Indicators

コードの走査中に、次のパターンにフラグを立てる:
- TODO/FIXME/HACK コメント（件数を数えて分類する）
- 抑制されたリンター警告（`// eslint-disable`、`# noqa`、`@SuppressWarnings`）
- ハードコードされた認証情報、URL、マジックナンバー
- 深くネストした条件分岐（3階層超）
- 神クラス/神ファイル（複数の責務を持つ500行超のもの）
- I/O 操作でのエラー処理の欠如
- 古い依存関係（メジャーバージョンが遅れているもの）
