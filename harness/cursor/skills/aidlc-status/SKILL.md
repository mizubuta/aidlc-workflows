---
name: aidlc-status
description: >
  Show the current AI-DLC workflow status without advancing or mutating it.
  A Cursor-native shortcut for `/aidlc --status`.
argument-hint: ""
user-invocable: true
disable-model-invocation: true
classification: read-only
---

# AI-DLC status

`/aidlc --status` と同じ決定的な経路で、現在のワークフローの状態を表示する。

1. 次を実行する:

   ```bash
   {{INVOKE}} engine orchestrate next --status
   ```

2. 返されたディレクティブに対し、`aidlc` スキルの転送ループの記述どおりに
   動作する。想定される終端の `print` ディレクティブは読み取り専用の
   状態表示コマンドを指定している。そのコマンドを実行し、標準出力をそのまま表示して停止する。

ディレクティブの JSON を状態レポートとして表示してはならず、ワークフローの状態を
前進させたり変更したりしてもならない。
