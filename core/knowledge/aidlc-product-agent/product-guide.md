# Product Guide

## User Story Format

標準形式: **As a [persona], I want [action], so that [benefit].**（[ペルソナ] として、[便益] のために、[アクション] したい。）

### INVEST Criteria Checklist
すべてのストーリーは、6 つの基準すべてを満たさなければならない:
- **I - Independent（独立している）**: 他のストーリーに依存せずに開発・提供できる
- **N - Negotiable（交渉可能である）**: 詳細を話し合える。厳格な契約ではない
- **V - Valuable（価値がある）**: ユーザーまたはステークホルダーに、識別可能な価値を提供する
- **E - Estimable（見積もり可能である）**: チームが工数を見積もれる（見積もれない場合、そのストーリーは分解またはスパイクが必要）
- **S - Small（小さい）**: 1 回のイテレーション内で完了できる（実装作業で 1〜5 日）
- **T - Testable（テスト可能である）**: 検証可能な具体的な受け入れ基準がある

### Story Decomposition Patterns
ストーリーが大きすぎる（エピックである）場合は、次の戦略で分割する:
1. **ワークフローのステップ別**: Login -> Browse -> Select -> Purchase -> Confirm
2. **データのバリエーション別**: テキスト入力の処理 / ファイルアップロードの処理 / 画像アップロードの処理
3. **ビジネスルール別**: 基本的なバリデーション / 高度なバリデーション / フィールド間のバリデーション
4. **インターフェース別**: Web UI / API エンドポイント / 管理画面
5. **操作別**: Create / Read / Update / Delete（ただし垂直スライスを優先する）

## Persona Development

各ペルソナについて、次を定義する:
```
Name: [descriptive name, e.g., "Alex the Admin"]
Role: [their role in the system]
Goals: [what they want to accomplish, 2-3 items]
Pain Points: [current frustrations, 2-3 items]
Tech Comfort: [low / medium / high]
Frequency: [how often they use the system]
```


すべてのストーリーは、定義済みのペルソナを参照しなければならない。どのペルソナにも当てはまらないストーリーがある場合、ストーリーが間違っているか、ペルソナが欠けている。

## Prioritization Frameworks

### MoSCoW (preferred for MVP definition)
- **Must Have**: これがなければシステムは使えない。取り除くと、製品は中核的な目的を果たせない。
- **Should Have**: 重要だが、致命的ではない。回避策がある。時間が許せば含める。
- **Could Have**: あると望ましい。体験を向上させるが、最初のリリースでは期待されない。
- **Won't Have (this time)**: 明示的にスコープ外とする。将来の検討のために文書化しておく。

### RICE Scoring (preferred for backlog ranking)
Score = (Reach x Impact x Confidence) / Effort
- **Reach**: 一定期間あたりに影響を受けるユーザー数/セッション数（実数を使う）
- **Impact**: どれだけ大きく状況を動かすか（3=非常に大きい、2=大きい、1=中程度、0.5=小さい、0.25=最小）
- **Confidence**: 見積もりにどれだけ確信があるか（100%/80%/50%）
- **Effort**: 作業の人日（全ロールの合計）

## MVP Definition Criteria

MVP は次を満たさなければならない:
1. 主要ペルソナにとっての中核的な問題を解決する
2. Must Have のストーリーをすべて含み、Could/Won't のストーリーは含まない
3. 手作業の回避策なしにデプロイでき、利用できる
4. 基本的なエラーハンドリングを含む（正常系だけではない）
5. セキュリティとデータ整合性に関する最低限の NFR を満たす
6. ステークホルダーに 10 分以内でデモできる

## Delivery Planning Structure

ストーリーをイテレーションに編成する:
```
Iteration 0 (Foundation): Infrastructure, auth, core data model
Iteration 1 (Core Value): Primary user workflow end-to-end
Iteration 2 (Completeness): Secondary workflows, edge cases, admin features
Iteration 3 (Polish): Performance optimization, UX refinement, advanced features
```


各イテレーションについて、次を定義する:
- 開始基準（開始前に何が完了していなければならないか）
- 含まれるストーリー（依存関係の順序付き）
- 終了基準（このイテレーションの「完了」とはどういう状態か）
- デモシナリオ（インクリメントをどう披露するか）

## Story Mapping Layout

ストーリーを 2 次元のマップに配置する:
- **横軸**: ユーザージャーニーのステップ（左から右へ、順番に）
- **縦軸**: 優先度（上 = must-have、下 = nice-to-have）
- **水平線**: MVP の境界（線より上はすべて MVP）

この可視化により、次を容易に見つけられる:
- 欠けているジャーニーステップ（縦方向の隙間）
- 過剰に投資された領域（1 つの列にストーリーが多すぎる）
- 依存関係の連鎖（他のストーリーより上に置かなければならないストーリー）
