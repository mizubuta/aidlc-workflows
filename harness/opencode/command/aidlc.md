---
description: >
  AI-DLC. Start, resume, or manage an AI-driven
  development lifecycle. Utilities: --status, --doctor, --stage, --phase,
  --scope, --depth, --test-strategy, --version, --help, plus the intent and
  space verbs. Or describe what you want to build and the agent proposes a
  plan for you to confirm.
---
以下の AI-DLC スキルに厳密に従う。スキルはここに全文が含まれているため、
追加で読み込むものはない。スキルは AI-DLC の転送ループを定義している。
以下の引数をそのまま渡して `{{INVOKE}} engine orchestrate next` を実行し、
返された 1 つのディレクティブに従って行動し、結果を報告する。これをエンジンが完了を告げるまで繰り返す。

$ARGUMENTS

!`{{INVOKE}} engine skill-text --harness-dir {{HARNESS_DIR}}`
