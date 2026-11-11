---
id: linter
kind: deterministic
command: {{INVOKE}} engine sensor-linter
default_severity: advisory
description: Wraps the project's configured linter (eslint by default for v0.5.0); fires on TS/JS code outputs
category: code-quality
matches: "**/*.{ts,js}"
input_schema:
  file_path: string
output_schema:
  pass: boolean
  violations:
    - file: string
      line: number
      rule: string
      message: string
timeout_seconds: 1200
---

# linter sensor

プロジェクトで設定されたリンターをラップする。v0.5.0 の既定は eslint で、多言語の
自動検出（ruff、golangci-lint、clippy）は v0.6.0 以降に先送りされている。

harness-engineering の記事にある Fowler の "Eslint, Semgrep" の例に倣ったものである。

## Failure mode

`SENSOR_FAILED` を出力し、詳細を
`aidlc/spaces/<active-space>/intents/<active-intent>/.aidlc-engine/sensors/<stage-slug>/linter-<fire-id>.md` に書き込む。
スペースとインテントはアクティブなカーソルから決まる。fire id は、
アクティブなレコードの `audit/<host>-<clone-id>.md` シャードにある `SENSOR_FIRED` 行の
8桁16進の相関 ID である。詳細にはリンターの構造化された
出力（違反ごとの file、line、rule、message）が含まれる。

## v0.6.0 carry-forward

フレームワーク境界での多言語検出（practices の `## Tech Stack` セクションからプロジェクトの種類を読み取り、
適切なリンターをディスパッチする）。
