# Stage Protocol: Phase Boundary Verification

このファイルはフェーズ遷移時（Ideation、Inception、Construction の終了時）に読み込む。注: Initialization→Ideation の遷移にはガバナンス境界チェックはない。
これは `stage-protocol.md` の補足であり、メインのプロトコルも引き続き適用される。

> 修正内容を永続的なルールとして記録する処理は、`stage-protocol-learnings.md` の条件付き §13 Learnings Ritual が担う。これは `directive.protocol_modules` に `learnings` が含まれる場合にのみ読み込まれ（`aidlc-learnings.ts` による、ツールをアクターとするループ）、ここでは扱わない。このファイルはフェーズ境界のトレーサビリティ検証のみを扱う。

---

## 13. Phase Boundary Verification

各フェーズ遷移（Ideation→Inception（approval-handoff→reverse-engineering）、Inception→Construction（delivery-planning→functional-design）、Construction→Operation（ci-pipeline→deployment-pipeline））で、トレーサビリティ検証を実行する。

### When to verify
- 各フェーズの最後のステージが承認された後
- 次のフェーズの最初のステージが始まる前
- ユーザーが `/aidlc --status` で検証を要求した場合は随時

### Verification process
1. `{{HARNESS_DIR}}/knowledge/aidlc-shared/verification.md` から検証方法論を読み込む
2. フェーズ固有のトレーサビリティチェックを実行する
3. 結果を `<record>/verification/[phase-boundary]-verification.md` に書き込む
4. 検証が失敗した場合は、先に進む前に問題をユーザーに提示する:
   - 欠落しているトレーサビリティリンク（例: 設計のない要件）
   - 孤立した成果物（要件のない設計）
   - フェーズ出力間の不整合
5. `PHASE_VERIFIED` は、フェーズ境界を越えたときにエンジンが発行する（状態ツールが最終ゲートの承認時に `PHASE_COMPLETED` と対にして発行する）。自分で追記してはならない

### Phase boundary checks
**Ideation → Inception**: インテントが記録され、スコープが定義され、実現可能性が確認され、イニシアチブが承認されている
**Inception → Construction**: すべての要件が設計にトレースされ、ユニットが定義され、デリバリー計画が承認されている
**Construction → Operation**: すべてのユニットがビルドおよびテストされ、CI パイプラインが構成され、インフラストラクチャが設計されている
