---
id: type-check
kind: deterministic
command: {{INVOKE}} engine sensor-type-check
default_severity: advisory
description: Wraps the project's configured type-checker (tsc by default for v0.5.0); fires on TS/TSX code outputs
category: code-quality
matches: "**/*.{ts,tsx}"
input_schema:
  file_path: string
output_schema:
  pass: boolean
  errors:
    - file: string
      line: number
      column: number
      message: string
timeout_seconds: 1200
---

# type-check sensor

プロジェクトで設定された型チェッカーをラップする。v0.5.0 のデフォルトは tsc である。
多言語の自動検出（mypy、go vet、cargo-check）は v0.6.0 以降に
先送りされている。

harness-engineering の記事にある Fowler の「型チェッカー（type checkers）」の例に倣ったものである。

## Failure mode

`SENSOR_FAILED` を発行し、詳細を
`aidlc/spaces/<active-space>/intents/<active-intent>/.aidlc-engine/sensors/<stage-slug>/type-check-<fire-id>.md`
に書き込む。スペースとインテントはアクティブなカーソルから決まる。fire id は、
アクティブな記録の `audit/<host>-<clone-id>.md` シャードにある `SENSOR_FIRED` 行の
8 桁 16 進の相関 ID である。詳細には型チェッカーの
構造化された出力が含まれる。

## v0.6.0 carry-forward

フレームワーク境界での多言語検出（プラクティスの `## Tech Stack` セクションから
プロジェクトの種類を読み取り、適切な型チェッカーにディスパッチする）。
