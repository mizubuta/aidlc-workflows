# Reverse Engineering Artifact Templates

## Output Structure

RE の成果物はすべて `aidlc/spaces/<active-space>/codekb/<repo>/` 配下に作成する。これはインテント間で共有される、リポジトリごとの永続的なコードナレッジベースである（`codekb-path --repo <repo>` ツールが解決するスペースレベルのディレクトリ）。

### Required Artifacts

1. **business-overview.md** — ビジネスドメインのコンテキスト、目的、主要な機能
2. **architecture.md** — システムアーキテクチャ、パターン、コンポーネント間の関係、Mermaid 図
3. **code-structure.md** — パッケージ/モジュールの構成、ファイルの分類、コードパターン
4. **api-documentation.md** — 外部および内部の API サーフェス、エンドポイント、コントラクト
5. **component-inventory.md** — 責務と依存関係を含む、完全なコンポーネント一覧
6. **technology-stack.md** — 言語、フレームワーク、ライブラリとそのバージョン
7. **dependencies.md** — 外部依存関係、内部のパッケージ間依存関係
8. **code-quality-assessment.md** — テストカバレッジ、リンティング、CI/CD、ドキュメントの品質、技術的負債
9. **reverse-engineering-timestamp.md** - リバースエンジニアリングを実施した日時（日付、可能ならコミットハッシュ）を Run Record セクションに記録し、加えて構造化された Scope of Analysis ブロックを含める（テンプレートは後述）。スコープブロックは次回の再実行時に `codekb-scope-diff` によって機械的に読まれるため、その正確さによって、将来のインテントが検証済みのカバレッジを再利用できるか、マージ/置換しなければならないかが決まる。

### Developer Code Scan Template

```markdown
## Developer Code Scan Results

### Scan Coverage
- **Analyzed deeply**: [repo-relative dirs/files actually read and understood, one per line]
- **Skimmed only**: [areas noted at directory granularity without deep reading]
- **Left out**: [folders not opened because "What to Skip" in code-analysis-guide.md skips them, such as build output, one per line]

### Packages Found
- [package name] — [type] — [language] — [purpose]

### Build System
- **Type**: [build system]
- **Config Files**: [list]
- **Build Dependencies**: [package → package relationships]

### APIs Discovered
- [API type] — [location] — [endpoints/methods count]

### Frameworks & Libraries
- [name] — [version] — [purpose]

### Test Coverage
- **Test Directories**: [list]
- **Test Frameworks**: [list]
- **Coverage Config**: [present/absent]

### Code Quality Indicators
- **Linting**: [tool and config location]
- **CI/CD**: [pipeline files found]
- **Documentation**: [README presence, doc comments quality]

### Technical Debt Signals
- [signal description and location]

## Handoff Summary
- **Intent-relevant finding**: [the finding most relevant to the active intent, with file/line evidence]
- **Risks / follow-up**: [facts the architect or next stage must preserve; "None" if absent]
```

### Architecture Synthesis Template

```markdown
## Architecture Analysis

### System Overview
[High-level description of the system]

### Architectural Style
[Monolithic / Microservices / Serverless / Hybrid — with evidence]

### Component Relationships
[Mermaid diagram showing component interactions]

### Data Flow
[How data moves through the system]

### Key Design Decisions
[Notable architectural choices and their implications]

### Improvement Opportunities
[Areas where the architecture could be strengthened]

## Interaction Diagrams
[Mermaid sequence or flow diagrams showing how key business transactions are implemented across components]
```

AI-DLC には Mermaid のバリデーターが同梱されていないため、探す必要はない。各図は
読んで確認し、`flowchart` と `sequenceDiagram` の構文にとどめ、図の下に
テキストによる代替表現を置く。

### Run Record (reverse-engineering-timestamp.md)

reverse-engineering-timestamp.md は次のセクションで始める。これにより読み手は、後述の
Scope of Analysis ブロックより先に、スキャンがいつ、どのコミットに対して実行されたかを
確認できる:

```markdown
# Reverse Engineering Timestamp

## Run Record

- Date: [ISO-8601 date of this run]
- Commit: [the commit the brief gives from the snapshot, or "unknown"]
```

### Scope of Analysis Block (reverse-engineering-timestamp.md)

reverse-engineering-timestamp.md の末尾は、必ず次のフェンス付きブロックそのもので終える。
開発者が報告した Scan Coverage に基づいて正直に記入すること - ステージがカバーしようとした
範囲ではなく、その実行で**実際に**深く分析した範囲を記録する:

````markdown
## Scope of Analysis

```yaml
scope_version: 1
kind: partial
intent: [active intent slug]
fingerprint: [output of the mint command below - verbatim; it prints "unknown" when not computable]
analyzed:
  paths:
    - [repo-relative dir (trailing slash) or file analyzed deeply, one per line]
  components:
    - [component names exactly as they appear in component-inventory.md]
shallow:
  paths:
    - [areas only skimmed]
```
````

ツールがこのブロックをどう読むか（記入に必要な情報はこれですべて）:

- `fingerprint:` 行は
  `{{INVOKE}} engine workspace codekb-scope-diff --repo <repo> --mint --paths <analyzed.paths, comma-separated>`
  で生成し（`--repo` を省略するのは、記録されていないプロジェクトルートのリポジトリの場合のみ）、
  その出力を一字一句そのまま貼り付ける。
- フィンガープリントは、コミット済みかどうかにかかわらず、`analyzed.paths` 配下のファイルを
  ディスク上の状態のまま対象とする。無視ファイル、プロジェクトファイルと同じ階層にある .NET の
  `bin/`、`obj/`、`out/`、さらにリポジトリがワークスペースのルートである場合は AI-DLC 自身の
  ワークスペース、インストール、設定を除外する。したがって、ビルド出力を開かなかった
  フルスキャンでは `./` が正しい。
- `shallow.paths` は人間向けの記録であり、どのツールもこれをハッシュ化したり比較したりしない。
  次回実行時のカバレッジ比較が読むのは `analyzed.paths` と `components` である。
  空のリストは `paths: []` と書く。
- `intent:` はツールが表示するだけで、どのツールも検査しない。
  `{{INVOKE}} engine workspace codekb --repo <repo> --json` が出力する `intent` を書く。
- 公開時には、ステージングディレクトリにちょうど 9 つのファイルがあること、このブロックが
  後述のルールに従って解析できること、スナップショットのパスが `analyzed.paths` をカバーして
  いること、`fingerprint:` が `analyzed.paths` に対して新たに生成した値と等しいことが検査される。
  Run Record は読まれない。
- ステージングされた `reverse-engineering-timestamp.md` 内の完成したブロックを
  `{{INVOKE}} engine workspace codekb-scope-diff --repo <repo> --check <that file>`
  で検査する（`--repo` のルールは生成時と同じ）。`VALID` と読み取った内容、および
  フィンガープリントが現在のソースと一致するかどうか、または `INVALID` と修正すべき点が
  出力される。これは公開時に使われるのと同じパーサーである。

ルール:
- `kind: full` にするのは、スキャンが本当にリポジトリ全体を深くカバーした場合のみであり、その場合 `analyzed.paths` には**必ず**リポジトリのルート（`./`）を含める。それに満たない場合は `kind: partial` とする。
- `kind: partial` では、`analyzed.paths` に `./` を**決して**含めてはならない。
- `analyzed.paths` のエントリはリポジトリ相対パスとし、ディレクトリは `/` で終え、glob 文字は使わない。
- コンポーネント名は `component-inventory.md` の見出しと一字一句一致させなければならない - 再実行ガードは文字どおりに比較する。
- スキャンで除外したフォルダー（ビルド出力など、開発者の **Left out** リストにあるもの）は `shallow.paths` に入れない。ブロックには含めない。
- フル再スキャンでは 9 つの成果物すべてを丸ごと置き換え、このブロックは新しい実行の結果のみから構築する。
- 既存のストアに対する絞り込みスキャンでは、まず既存の 9 つの成果物すべてと、以前の Scope of Analysis ブロックを読む。新たに分析した領域については記述を更新・拡充し、それ以外の領域の既存の記述は保持する。
- ストアが CURRENT の場合、`analyzed.paths` と `analyzed.components` はストアと今回の実行の和集合としてマージする。CURRENT かつ `kind: full` のストアは full のままで `./` を保持する。それ以外の場合、マージ後のブロックは partial となり、`./` を主張できない。
- ストアが STALE または UNVERIFIED の場合、`analyzed.paths` と `analyzed.components` には今回の実行分のみを記録し、既存の記述は保持し、ストアの以前の分析済みパスは `shallow.paths` に格下げする。
- UNKNOWN_SCOPE のレガシーストアの場合、既存の記述はベストエフォートでマージするが、新しいスコープブロックには今回の実行分のみを記録する。
- `fingerprint` は、マージ後または置換後のブロックにおける最終的な `analyzed.paths` に対して生成する。
- 9 つの候補成果物はすべて、一時ディレクトリ `<record>/.aidlc-engine/codekb-stage-<repo>/`（同じコマンドが出力する `stage`）の下に作成する。書き終えたら同じコマンドを再度実行し、その `staged` にちょうど 9 つの名前が列挙されていなければならない。累積的なマージを共有 CodeKB に直接書き込んでは決してならない。
- スキャン前の `codekb-snapshot` のパスが、検証済みカバレッジの範囲を規定する。スキャン中にその範囲外の深いパスが見つかった場合は、新しいスナップショットを取得し、拡張した範囲でスキャンをやり直す。
- 公開は、スナップショットのストア世代とソースのフィンガープリントを指定した `codekb-publish` を通じてのみ行う。ストア世代の競合が起きた場合は、勝った側のストアを読み直して再マージする必要がある。ソースの競合が起きた場合は、新たにスキャンする必要がある。
