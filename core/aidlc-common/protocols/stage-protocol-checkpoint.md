# Checkpoint Protocol Module

エンジンが `directive.protocol_modules` に `checkpoint` を挙げている場合にこのモジュールを読み込む: Unit またはスケルトンのチェックポイント（`directive.construction_checkpoint`）、スウォームのバッチチェックポイント（`directive.swarm_checkpoint`）、そしてすべての `invoke-swarm` が該当する。

### Checkpoint verification and approval

`construction_checkpoint` を持つ `run-stage` は、`kind`（`unit` または
`skeleton`）、`unit`、`stages`、`fingerprint`、`ready`、`verified`、`approved`、
`human_required`、`verification_command`、`command_authorized`、`errors`、
`proof_path` を持つ。これは既存の作業に対する検証/承認のための再入である。
`verification_command` は記録された正規のコマンド全体であり、決して切り詰められた
表示用ラベルではない。エンジンが出力したユニットと kind をそのまま使う:

```bash
{{INVOKE}} engine bolt checkpoint --action status --unit "<unit>" --kind <unit|skeleton>
```

`verify` は、インテントに記録された、人間が承認した **Construction Verification
Command** を実行する。このコマンドはすべてのユニット/バッチのチェックポイントで再利用される。
`construction_checkpoint.command_authorized` が false の場合（記録されたコマンドがない、
一致する受領記録がない、または状態フィールドが変更された）、**`verify` を実行しない**。代わりに、プロジェクトのスキャン結果から
実際のプロジェクトチェック（`bun test`、`pytest`、
`make check` など）を提案する。そのチェックは、スケルトンの統合スライスをエンドツーエンドで実証し、
完了したユニットの動作結果を確認するものでなければならない。また、Testing Contract が求める
すべてのテストレベル（該当する場合はユニット、インテグレーション、エンドツーエンド）と
カバレッジチェックを実行しなければならない。必要なレベルをこのマシンで実行できない場合（コンテナ
ランタイムがない、サービスが起動していない）は、コマンドが実行しないテストとその理由を
Approve 選択肢の説明に追記する。たとえば "Runs typecheck and unit tests.
Does not run the integration tests: they need Docker, which is not installed."（型チェックとユニットテストを実行する。インテグレーションテストは実行しない: Docker が必要だが、インストールされていない。）のように書く。
ランタイムがないことを避けるために、その旨を言わずに範囲の狭いコマンドを選んでは決してならない。また、
プランが CI Pipeline をスキップしている場合やリポジトリに CI 設定がない場合に、その不足に対して "CI runs them" と答えては
決してならない。前後の空白を取り除いた後で 1024 文字以内の、空でない 1 行を使う。
ツールは制御文字（改行、CR、タブ、NUL を含む）と表示を偽装する文字を拒否する:
Unicode の書式文字（ゼロ幅文字や双方向制御文字を含む）、行/段落の
区切り文字、ノーブレークスペース（U+00A0）である。空白を取り除いたコマンドは、そのまま記録・ハッシュ化・
実行される。複数行のチェックはスクリプトにまとめ、その呼び出しを記録する。
コマンドを提示する前に、ハーネスのファイル書き込みツール（Write/edit）を使って、UTF-8 テキストとして
`<record>/verification-command.txt` に書き込む。シェルの `echo` やヒアドキュメントは決して使わない。
リポジトリ由来のコマンドテキストを、シェルのコマンドラインに決して
埋め込んではならない: シェルの置換が、人間が承認する前に実行されてしまう可能性がある。
以下のとおり、記録ディレクトリからの相対ファイルパスだけを渡す。
コマンドは自分が実行されているセッションを自ら見つけるため、セッションを渡さず、
セッションを調べることも決してしない:

```bash
{{INVOKE}} engine log decision --stage "<directive.stage>" --checkpoint verification-command --command-file verification-command.txt --decision "Use this command to verify each completed Unit?" --options "Approve,Request Changes"
```

この構造化質問は、ハーネスの質問バインディングを通じてレンダリングする。`log decision`
ツールの JSON 出力にある `command` フィールドから、正規のコマンド全体を正確にコピーして
質問のコードスパンに入れる。決して省略したり、要約・プレフィックス・ダイジェストで置き換えたりしない。
コマンド内のバッククォートを保持できる長さのコードスパン区切りを使う。人間は
`<record>/verification-command.txt` を開いて確認することもできる。自律完了の場合でも、人間の応答を待つ:

```question
prompt: "Use this command to verify each completed Unit? `<full command>`"
header: Verification
multiSelect: false
options:
  - label: Approve
    description: Record this command for all Unit and batch checkpoints in this intent.
  - label: Request Changes
    description: Propose a different project check before running verification.
```

利用者の返信を読み、利用者が行った選択を記録する。
human-turn フックが、利用者がこの質問に返信したことと、その正確な文言を保持する。
新しいチャットで入力された返信も、それが最も新しく尋ねられた質問である間はこの質問への返信として保持される。
別の質問への返信は数えない。利用者が
承認した場合は、その回答を記録し、コマンドを設定する:

```bash
{{INVOKE}} engine log answer --stage "<directive.stage>" --checkpoint verification-command --command-file verification-command.txt --details "Approve"
{{INVOKE}} engine state set-construction-verification-command --command-file verification-command.txt
```

**Request Changes** の場合は、同じ `log answer` を
`--details "Request Changes"` で記録し、セッターは呼び出さず、別の
コマンドを提案する。コマンドを決して捏造したり自動承認したりしない。権限の根拠は状態フィールドではなく、
ツールが管理する承認の受領記録である: その受領記録なしにフィールドを書き込んだり、
汎用の `state set` を使ったりしては決してならない。後でコマンドを変更する場合も、これと同じ新たな
decision/answer/setter のフローが必要である。すでに承認されたユニットとバッチは承認を
維持し、新しいコマンドはまだ承認されていないものを確認する。セッターの
`notice` が、利用者に伝える 1 行である。記録した後に `next` を再実行し、検証の前に
新しいディレクティブに従う。実行可能なプロジェクトチェックがまだ存在しない場合は、
その不足を人間と一緒に解決する。プレースホルダーで代用したり、合格したと主張したりしない。

新しいディレクティブで `command_authorized: true` が確認できたら、
記録されたコマンドで検証する:

```bash
{{INVOKE}} engine bolt checkpoint --action verify --unit "<unit>" --kind <unit|skeleton>
```

検証ツールは、現在の成果物、ソース、試行、承認済みコマンドの SHA-256 に結び付いた証明を、
表示用ラベルとしての正規のコマンド全体とともに保存する。ファイルの存在、実証したという主張、プレースホルダーのコマンド、以前の
合格は、検証ではない。チェックが失敗したら停止する。実行中にユニットの
ファイルを変更するチェック（フォーマッター、ジェネレーター）は、その時点のファイルに対して
もう一度実行される。それでも再びファイルを変更した場合は、証明の `error` が、ファイルを
そのままにしておくチェックを使うよう指示する。出力が長いことを理由に、合格したチェックが失敗になることは決してない。検証の後に `next` を再実行する。
その結果のディレクティブが、準備状況と検証に関する次の信頼できる情報源となる。`verify` を再実行すると、
このインテントについて、どのセッションのものであれ、開いているすべてのチェックポイントの質問と
取得済みのチェックポイントの応答が取り下げられる。再び質問するのは、
新しい検証が `verified: true` を報告した後に限る。質問する前に、検証が返す各 `change_notices`
の行を、書かれているとおりに伝える。
検証ツールは、ツールが管理する `CHECKPOINT_VERIFICATION_RECORDED` の受領記録を
証明ファイルと並べて記録し、承認にはその受領記録が必要である。手書きの
証明ファイルでユニットを検証することはできない。証明ファイルがまったくないチェックアウト（新しい
クローン、別のマシン）では、証拠が変わっていないすでに承認済みのユニットについて、その受領記録が証明の代わりとなるため、
何も再実行されない。

チェックポイントが `rereview` を持つ場合、そのユニットのレビュー済みのコードやドキュメントが
レビューの後に変更されており（利用者による編集、フォーマッター、いかなるレビューも確認していない変更）、それ以外に欠けているものはない。まず利用者に何も尋ねずに、
ただちに `rereview.command` を実行する（これによりユニットの再チェックが開かれる。パス上限によって
拒否されることは決してない。利用者がユニットを承認するたびに新しい再チェックが開かれる）。そのリクエストのために
レビュアーモジュールを通じて `rereview.reviewer` をディスパッチし、リクエストが返した
`recordVerdict` コマンドでその判定を記録し、`next` を再実行してから検証する。どちらの
判定も、この再チェックについては最終的なものである。再チェックが実行されるのは Guard Policy が
`strict` の場合のみである。`relaxed` と `off` では、レビュー後にユニットのコードやドキュメントに加えられた変更は
受け入れられる: どのチェックポイントも `rereview` を持たず、ユニットの
承認や準備状況はそのまま維持され、その旨の 1 行が `change_notices` で届く。
レビューがオフの場合も同様である: `relaxed` と `off` では、承認済みの
Unit の作業に後から変更が加えられても承認は維持され、その旨の 1 行が、次のチェックポイントの `verify` または Construction ステージ
自身のチェックから `change_notices` で届く。承認の質問を尋ねた後、
利用者が回答する前に加えられた変更も同じように受け入れられる: 利用者の `approve` がそれを記録し、その旨の 1
行がそのコマンドの `change_notices` で届く。
`unfinished` を持つ `rereview` は、代わりに、完了しなかったユニット自身のレビューを表す
（`no-verdict`: 一度も回答されなかった。`not-ready`: パスが残っている状態での NOT-READY であり、まず
指摘されたものを修復する）。同じように実行する。利用者がユニットを
そのまま承認するよう述べた場合は、代わりに `--over-unfinished-review` を付けて `verify` を実行する。するとチェックポイントは
`approved_in_words` 付きの `review_not_finished` を持つ: 利用者の言葉が
承認であるため、何も尋ねずにただちに `approve` を実行し、それが返す 1 行を
伝える。承認の後にそのレビューの判定が届いた場合は、通常どおり
記録され、ユニットは承認されたままになる。レビューが必須のまま残るのは、チームが
ロックした strict の場合だけである: そのとき `verify` はレビューのステップを示し、
利用者に尋ねずにそれを実行する。レビューが
レビュアーモジュールの NOT-READY フォールバックで終わったユニットは、
`question` 付きの `review_not_finished` を持つ。これは "Approve this completed <unit>?" の代わりとなるもので、
合格させる選択肢はない: その受領記録はどのレビュアーの判定でもないからである。
`first` を持つ `rereview` は、その作業の今回の実行において一度も依頼されなかったユニットのレビューを表す
（ジャンプバックや再オープンは新しい実行を開始するため、それ以前のレビューは
数えない）。どの Guard Policy の下でも、同じように実行する。これは必須であるため、`--over-unfinished-review`
は適用されない。

それ以外の場合で、`ready` が false であるか証拠が古くなっているときは、`errors` を説明する。
指摘された欠けているレビューや受領記録は、それを担う手順を通じて修復し、
必要に応じて修復について人間に相談する。チェックポイントが `gate: true` を使っているというだけで
本体全体を再実行してはならない。未検証の間は、チェックポイントの
承認の質問を開いたり、検証が成功したと主張したりしない。修復の後は、新しい
ディレクティブを取得し、現在の結果を検証する。実際のプロジェクトチェックが存在しない場合は、
合格を主張する前にその不足を人間と一緒に解決する。

承認できるのは検証済みのチェックポイントだけである。スケルトンには常に、実際の
人間による承認が必要である。通常のユニットでは、`human_required` が true の場合
（`unset`/`gated`）に人間の承認が必要となる。明示的な自律の許可がある場合は、コンダクターが
検証済みの通常のユニットを自動的に承認してよい。人間によるチェックポイントでは、`directive.protocol_modules` に `learnings` が
含まれている場合に限り、対象のステージについてセクション
13 の学びのステップを実施する。モジュールが含まれている場合、それが尋ねるのは
それらのステージで利用者が行った修正についてだけである（いつ尋ねるかはモジュールが定めている）:
各質問とその回答を `--stage "<directive.stage>"` で記録し、利用者が残すものだけを
担当ステージの学びツールで永続化し、その後、以下の `ask` で
チェックポイントの承認を別の質問・別のターンとして開く。
自動実行中は、`directive.protocol_modules` に `learnings` が含まれている場合に限り、利用者が行った修正は次の人間によるチェックポイント
または最終的な引き渡しまで保留される。利用者の回答を推測したり、
利用者が残さなかったものを永続化したりしない。モジュールが
ない場合は、日誌を付けず、学びに関する質問もしない。人間が必要な場合は、そのまま
チェックポイントの承認手順に進む。`verify` が
`verified: true` を報告し、現在のディレクティブが `ready: true` を持つようになって初めて、`ask` を実行する。
このコマンドは、準備ができていない、または未検証のチェックポイントを拒否する。**Approve** /
**Request Changes** を提示する前に、質問を現在のチェックポイントの証明と
承認済みコマンドのダイジェストに結び付ける。コマンドは自分が実行されているセッションを自ら見つけるため、
セッションを渡さず、セッションを調べることも決してしない:

```bash
{{INVOKE}} engine bolt checkpoint --action ask --unit "<unit>" --kind <unit|skeleton>
```

`ask` が `earlier_reply` を返した場合、利用者はユニットのレビューの実行中に、このチャットですでに
回答している: それは利用者の言葉であり、この質問への
返信として保持されている。質問を表示せず、利用者の言葉を読み、利用者が選んだとおりに `approve` または `reject`
を、その言葉を `--user-input` として実行する。利用者の言葉が何にも
答えていない場合（利用者自身の質問、別の話題）は、以下のとおり質問を
提示して待つ。利用者の返信は、それらの言葉と合わせて扱われる。

その後、選択肢を提示して人間の応答を待つ。`ask` が
`check_findings`、`left_out`、`changed_files`、または `check_offer` を返した場合は、質問の直前にそれぞれを書かれているとおりに伝える。承認の質問では、記録された
コマンドを省略せずに全体を示す: "Verified with
`<full command>` (exit 0). Approve this completed <unit>?" 現在のツール出力にある完全な
`verification_command` を、バッククォートを保持できるコードスパン区切りで
使う。チェックポイントが `rechecked` を持つ場合、これが
再チェックに関する唯一の質問となる: その前に学びに関する質問は入らず、
その行が "Verified with `<full command>` (exit 0)." の後の
"Approve this completed <unit>?" の代わりとなるため、利用者に尋ねるのは 1 回だけである:
`rechecked.approved_before` が true の場合は
"<unit>'s <changed> changed since you approved it, so it was re-checked:
<verdict>. Approve it?"、それ以外の場合は "<unit>'s <changed> changed after
its review, so it was re-checked: <verdict>. Approve it?" とし、`<changed>` には
`rechecked.changed`（code または documents）を、判定には平易な言葉（ready
または not ready）を入れる。`rechecked.redone` が true の場合は、利用者がその作業の
やり直しを求めたため、代わりに "<unit>'s design was redone and its review
says <verdict>. Approve it?" という行にする（
`rechecked.changed` が code の場合は "design" の代わりに "code"）。判定が `NOT-READY` の場合は、
レビュアーモジュールが回復判定の後に求めるとおり、まず Review brief を出力する:
`bun {{HARNESS_DIR}}/tools/aidlc-review-brief.ts review --stage "<directive.stage>" --unit "<unit>" --why stale`.
このチェックポイントの質問に対する人間の返信は、このチャットでも新しいチャットでも、
そこから読み取った操作を承認する: 承認するか、利用者が変更を求めた内容とともに却下するかである
（利用者の言葉は記録とともに保持される。さらに説明したい場合は `--reason` を追加する）。
利用者の返信が Request Changes を選んだだけの場合は、記録する前に何を変えるべきかを
尋ねる。新しいチャットで入力された返信は、それが最も新しく尋ねられた質問である間、
その質問への返信として保持される。そこで approve または reject のコマンドをセッションなしで実行し、
同じ質問を再び尋ねることは決してしない。別の質問への返信は数えない。利用者が承認したうえで変更を求めた場合は、
承認してから変更を行い、何を変更したかを 1 行で伝える。利用者が修正を求めた
レビュー指摘ごとに `approve` へ
`--fix-finding "<review-artifact>#R-NN"` を追加し、利用者が却下した指摘ごとに `reject` へ `--reject-finding '<review-artifact>#R-NN=<their reason>'`
を追加する。応答は
1 回限りであり、このユニット、kind、現在のフィンガープリント、検証の証明 ID、
承認済みコマンドのダイジェストに結び付いている。チェックポイントが変わった場合は、新しいディレクティブを取得し、
再検証してから再び質問する。再検証の前に取得した返信で、
新しい証明を承認することはできない。自動承認（`human_required: false`）には `ask` も
`--user-input` も不要だが、人間による Request Changes には常に、この検証済みの
質問と回答のフローが必要である。

```bash
# Only after the human chose Approve:
{{INVOKE}} engine bolt checkpoint --action approve --unit "<unit>" --kind <unit|skeleton> --user-input 'Approve'
# Automatic approval: verified ordinary Unit and human_required: false only.
{{INVOKE}} engine bolt checkpoint --action approve --unit "<unit>" --kind unit
# Only after the human chose Request Changes and supplied feedback:
{{INVOKE}} engine bolt checkpoint --action reject --unit "<unit>" --kind <unit|skeleton> --user-input 'Request Changes' --reason '<human feedback>'
```

承認または却下の後は、`next` を再実行する。チェックポイントの承認を、ユニット全体に対する
`report --stage code-generation --result approved` として使っては決してならない。
すべてのユニットの承認が記録されると、エンジンは
`completion_only: true` 付きの通常のステージゲートを出力することがある。それらは construction モジュールの
`completion_only` ルーティング分岐を通じて処理する
（単独の unit-major ウォークでは、素の `next` がそれらを自ら記録する）。
明示的な stage-major のゲート付き実行では、通常のステージレビューが維持される。
自律実行では、定型的な人間による完了の質問はスキップされる。
