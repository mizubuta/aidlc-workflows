---
name: aidlc-jump
description: >
  Jump the active AI-DLC workflow to a stage or phase. A Cursor-native
  shortcut for `/aidlc --stage <target>` or `/aidlc --phase <target>`.
argument-hint: "--stage <slug|#> | --phase <name|#>"
user-invocable: true
disable-model-invocation: true
---

# AI-DLC jump

`/aidlc` と同じ決定論的な転送ループを通じてジャンプする。ユーザーの
`$ARGUMENTS` は一字一句そのまま渡す。引数には `--stage
<slug|#>` または `--phase <name|#>` のいずれかが含まれていなければならない。

1. エンジンを呼び出す前に呼び出し内容を検証する。`$ARGUMENTS` が空の場合、
   または `--stage <target>` と `--phase <target>` のうちちょうど1つを含んでいない場合は、
   `Usage: /aidlc-jump --stage <slug|#> | --phase <name|#>` を出力して停止する。
   ターゲットが欠けているときに、引数なしの `next` へ決してフォールバックしてはならない。

2. 次を実行する:

   ```bash
   {{INVOKE}} engine orchestrate next $ARGUMENTS
   ```

3. `aidlc` スキルの転送ループの説明どおりに、ディレクティブに従って動作する。
   アクティブなワークフローでは、エンジンは検証済みのジャンプコマンドを示す `print`
   ディレクティブを返す。それを実行してループを続ける。ディレクティブまたは人間によるゲートが
   停止を指示したときにのみ停止する。

ターゲットの検証、方向、スキップされたステージの扱い、およびすべての状態変更はエンジンが担う。
そのループの外でジャンプを推測したり適用したりしてはならない。
