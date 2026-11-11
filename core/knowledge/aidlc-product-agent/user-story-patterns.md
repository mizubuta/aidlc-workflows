# User Story Patterns

## Purpose
素早くデリバリーできるほど小さく、作業を正当化できるほど価値があり、曖昧さなくテストできるほど明確なユーザーストーリーを書く。

## Standard Format
```
As a [role], I want to [action], so that [benefit].
```

「so that」節が最も重要な部分である — 価値を言語化することを強制するからだ。便益を述べられないなら、そのストーリーが必要かどうかを疑う。

## INVEST Criteria

すべてのストーリーは 6 つの基準をすべて満たすべきである:

- **Independent（独立している）** — 他のストーリーに依存せずに開発・デリバリーできる
- **Negotiable（交渉可能である）** — 詳細は議論の余地がある。ストーリーは契約ではない
- **Valuable（価値がある）** — ユーザーまたはビジネスに価値をもたらす（「データベースをセットアップする」ではない）
- **Estimable（見積もり可能である）** — 必要な工数をチームが妥当に見積もれる
- **Small（小さい）** — 1 スプリント内で完了できる（理想は 1〜3 日分の作業）
- **Testable（テスト可能である）** — 明確な受け入れ基準が「完了」を曖昧さなく定義している

## Story Splitting Patterns

ストーリーが大きすぎる場合は、次のいずれかのパターンで分割する:

### 1. Workflow Steps
複数ステップのプロセスを個々のステップに分割する。
- 「ユーザーがチェックアウトを完了する」は次のようになる: カートに追加、配送先の入力、支払い情報の入力、注文の確定、確認通知の受け取り

### 2. Business Rule Variations
各ビジネスルールをそれぞれ独立したストーリーに切り出す。
- 「送料を計算する」は次のようになる: 国内一律料金、国内重量制、国際配送、送料無料のしきい値

### 3. Data Variations
扱うデータの種類や複雑さで分割する。
- 「顧客データをインポートする」は次のようになる: CSV からのインポート、API からのインポート、重複排除付きのインポート、バリデーションエラーを伴うインポート

### 4. Interface Variations
プラットフォームや操作モードで分割する。
- 「ユーザーがダッシュボードを閲覧する」は次のようになる: デスクトップ向けレイアウト、モバイル対応のレスポンシブレイアウト、キーボード操作に対応した版

### 5. Operations / CRUD
作成、参照、更新、削除を別々のストーリーに分割する。
- 参照（最も単純で、すぐに価値が出る）から始め、次に作成、更新、削除の順に進める

### 6. Performance / Scale
「動くようにする」と「速くする」を分ける。
- ストーリー 1: 「ユーザーが商品を検索する（基本的なクエリ、結果 1000 件未満）」
- ストーリー 2: 「ユーザーが 100 万件以上のカタログ全体を対象に、200ms 未満の応答で商品を検索する」

## Acceptance Criteria: Given/When/Then

### Format
```
Given [precondition / initial state]
When [action / trigger]
Then [expected outcome / observable result]
```

### Example
```
Story: As a customer, I want to reset my password, so that I can regain access to my account.

AC 1:
Given I am on the login page
When I click "Forgot password" and enter my registered email
Then I receive a password reset link within 5 minutes

AC 2:
Given I have received a password reset link
When I click the link after 24 hours
Then I see a message that the link has expired with an option to request a new one

AC 3:
Given I am resetting my password
When I enter a password shorter than 8 characters
Then I see a validation error and the password is not changed
```

### Tips
- 受け入れ基準はストーリーごとに 3〜6 個書く — 少なすぎれば曖昧さが残り、多すぎればストーリーが大きすぎることを意味する
- ハッピーパスだけでなく、必ずサッドパス（エラー、エッジケース、タイムアウト）も含める
- 受け入れ基準はテスト可能なアサーションである — QA がそのまま自動化できるようにする

## Anti-Patterns to Avoid

### Too Large (Epic in Disguise)
- **症状**: 「ユーザーとして、完全なレポーティングシステムがほしい」
- **対処**: 個々のレポートに分割し、さらに各レポートをフィルター／エクスポート／スケジュールで分割する

### Too Technical (Implementation Story)
- **症状**: 「開発者として、データベースを PostgreSQL に移行したい」
- **対処**: ユーザー価値を軸に捉え直す: 「ユーザーとして、検索結果を 200ms 未満で得たい」（その実現に移行が必要になる）

### No Value Statement
- **症状**: 「ユーザーとして、送信ボタンをクリックしたい」 — 「so that」がない
- **対処**: 意味のある便益にたどり着くまで「なぜユーザーはそれを気にするのか？」と問い続ける

### Compound Stories (AND in the Title)
- **症状**: 「ユーザーとして、結果を検索し、かつ絞り込み、かつ並べ替えたい」
- **対処**: 検索、絞り込み、並べ替えの 3 つのストーリーに分割し、それぞれを独立してデリバリー可能にする

### Solution-Prescriptive
- **症状**: 「ユーザーとして、チェックボックス付きのドロップダウンメニューがほしい」
- **対処**: UI ではなくニーズを記述する: 「ユーザーとして、結果を絞り込むために複数のカテゴリを選択したい」

## Story Mapping

ストーリーを 2 次元のマップに整理する:
- **横軸**: ユーザージャーニーのステップ（左から右へ）
- **縦軸**: 優先度（上 = 必須、下 = あれば望ましい）

マップを横切る水平線を引いて MVP を定義する — 線より上にあるものすべてを最初にリリースする。これにより、個々のストーリーの一覧では見落とされるジャーニー上の抜けが明らかになる。

## Definition of Ready Checklist
- [ ] ストーリーが明確な価値記述を含む標準フォーマットに従っている
- [ ] INVEST 基準を満たしている
- [ ] Given/When/Then 形式で 3〜6 個の受け入れ基準が書かれている
- [ ] 依存関係が特定され、解消済みまたは受容済みである
- [ ] UX デザインがレビュー済みである（UI に関わる場合）
- [ ] チームがストーリーを見積もり済みである
