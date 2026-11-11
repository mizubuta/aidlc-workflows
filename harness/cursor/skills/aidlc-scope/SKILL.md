---
name: aidlc-scope
description: >
  Set or change the AI-DLC workflow scope. A Cursor-native shortcut for
  `/aidlc --scope <name>`.
argument-hint: "<scope> [description | --depth <level> | --test-strategy <level>]"
user-invocable: true
disable-model-invocation: true
---

# AI-DLC scope

`/aidlc` と同じ決定的な転送ループを通じてスコープを設定する。
最初の引数はスコープ名である。組み込みの `--scope` フラグの後に、すべての引数を
そのまま渡す。

1. `$ARGUMENTS` が空の場合は、
   `Usage: /aidlc-scope <scope> [description | --depth <level> | --test-strategy <level>]`
   を出力して停止する。スコープの欠落が、引数なしの `next` に格下げされることは決してあってはならない。

2. 次を実行する:

   ```bash
   {{INVOKE}} engine orchestrate next --scope $ARGUMENTS
   ```

3. `aidlc` スキルの転送ループの説明どおりにディレクティブに従って動作する。
   アクティブなワークフローでは、エンジンがスコープ変更コマンドを指名し、その実行後に
   ループが継続する。新規のワークスペースでは、通常のワークフロー作成コマンドを指名する。
   ディレクティブまたは人間によるゲートが停止を指示した場合にのみ停止する。

スコープの検証とすべての状態変更はエンジンが担う。状態ファイルを直接編集してはならない。
