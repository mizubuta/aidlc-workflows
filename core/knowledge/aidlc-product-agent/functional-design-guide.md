# Functional Design Guide

## Business Logic Modeling

### Logic Decomposition Approach
複雑なビジネスロジックを、組み合わせ可能なレイヤーに分解する:

1. **入力検証レイヤー**: データ形式、範囲、必須フィールドを検証する
2. **ビジネスルールレイヤー**: ドメインルール、制約、計算を適用する
3. **状態遷移レイヤー**: エンティティのライフサイクルと有効な状態変化を管理する
4. **副作用レイヤー**: 通知、監査ログ、外部連携を起動する

### Business Rule Specification Format
各ルールについて、次を文書化する（トレーサビリティセンサーが認識する `BR{group}.{seq}` 形式の ID を使う。例: `BR1.1`）:
```
Rule ID: BRx.y
Name: [descriptive name]
Trigger: [when this rule is evaluated]
Condition: [the logical expression]
Action: [what happens when condition is true]
Exception: [what happens when condition is false or invalid]
Priority: [execution order when multiple rules apply]
Source: [requirement ID or stakeholder that defined this rule]
```

### Rule Conflict Resolution
同じ操作に複数のルールが適用される場合:
- **優先度順**: 優先度の高いルールから実行する
- **最初の一致を採用**: 最初に一致したルールで評価を打ち切る
- **すべての一致を累積**: 一致したすべてのルールを適用する（互いに矛盾してはならない）
- ルールグループごとに、選んだ戦略を必ず文書化する

## Domain Entity Design

### Entity Identification Checklist
次の場合はエンティティとしてモデル化する:
- 時間が経っても持続する一意の識別子を持つ
- 明確に区別される状態を持つライフサイクルがある
- システムの複数の部分から参照される
- その振る舞いを規定するビジネスルールがある
- 他のエンティティとの関係に参加する

### Entity Specification Template

| 属性 | 型 | 必須 | 制約 | 既定値 | 備考 |
|-----------|------|----------|-------------|---------|-------|
| id | UUID | Yes | システム生成 | Auto | 主識別子 |
| status | Enum | Yes | [有効な値] | Initial | ステートマシンを参照 |
| ... | ... | ... | ... | ... | ... |

### Relationship Types and Design Rules
- **1 対 1**: アクセスパターンに基づいて埋め込むか分離するかを決める（常に一緒に取得されるなら埋め込む）
- **1 対多**: 親がコレクションを所有し、子は ID で親を参照する
- **多対多**: 独自の属性（タイムスタンプ、ステータス）を持つ中間エンティティを使う
- カスケードの振る舞いを必ず定義する: 親が削除されたら子はどうなるか？
- 参照整合性の制約を明示的に文書化する

## Business Rule Specification Patterns

### Calculation Rules
算出値については、次を明記する:
- **計算式**: 変数の定義を含む計算式
- **精度**: 丸めのルール、小数点以下の桁数、通貨の扱い
- **エッジケース**: ゼロ除算、オーバーフロー、null 入力
- **例**: 入力と期待される出力を示した計算例を少なくとも 3 つ

### Validation Rules
各入力フィールドについて:
| フィールド | 型 | 必須 | 最小 | 最大 | パターン | 独自ルール |
|-------|------|----------|-----|-----|---------|-------------|
| email | string | Yes | 5 | 254 | RFC 5322 | 一意でなければならない |
| amount | decimal | Yes | 0.01 | 999999.99 | 小数点以下 2 桁 | 口座残高を超えてはならない |

### Authorization Rules
操作ごとにアクセス制御を文書化する:
```
Operation: [Create/Read/Update/Delete] [Entity]
Allowed Roles: [role list]
Additional Conditions: [ownership check, status check, time window]
Denied Response: [error code and message]
Audit: [whether to log access attempts]
```

## Workflow Design Methodology

### Workflow Specification Template
各業務ワークフローについて:

1. **トリガー**: 何がワークフローを開始するか（ユーザー操作、スケジュールされたイベント、外部シグナル）
2. **事前条件**: ワークフローを開始する前に何が成り立っていなければならないか
3. **ステップ**: 判断ポイントを含む、順序付けられたアクションの一覧
4. **アクター**: 各ステップを誰または何が実行するか（ユーザー、システム、外部サービス）
5. **事後条件**: ワークフローが正常に完了したときに何が成り立っていなければならないか
6. **エラー経路**: 各ステップが失敗したときに何が起きるか
7. **タイムアウト時の振る舞い**: ワークフローがいずれかのステップで止まった場合に何が起きるか

### State Machine Design
複雑なライフサイクルを持つエンティティについて:

| 現在の状態 | イベント | ガード条件 | 次の状態 | アクション |
|--------------|-------|-----------------|------------|---------|
| Draft | Submit | すべての必須フィールドが入力済み | Pending Review | レビュアーに通知し、イベントを記録する |
| Pending Review | Approve | レビュアーに権限がある | Approved | 提出者に通知し、タイムスタンプを更新する |
| Pending Review | Reject | 却下理由が示されている | Draft | 理由を添えて提出者に通知する |
| Approved | Activate | 開始日に達した | Active | 機能を有効にする |
| Active | Expire | 終了日に達した | Expired | 機能を無効にし、所有者に通知する |
| Any | Cancel | キャンセルポリシーを満たす | Cancelled | ステークホルダーに通知し、リソースを解放する |

### State Machine Validation Rules
- すべての状態が初期状態から到達可能でなければならない
- 終端でないすべての状態は、少なくとも 1 つの出ていく遷移を持たなければならない
- 終端状態（Cancelled、Expired、Completed）は出ていく遷移を持たない
- 暗黙の遷移は認めない -- すべての状態変化には明示的なイベントが必要である
- ガード条件はテスト可能でなければならない（主観的な基準は認めない）

## Functional Design Document Structure

各機能設計仕様は次のように構成する:
1. **概要**: その機能の目的とスコープ
2. **エンティティ**: 属性と関係を含むデータモデル
3. **ビジネスルール**: 優先度付きの完全なルールセット
4. **ワークフロー**: 判断ポイントを含むステップごとのプロセス
5. **ステートマシン**: 状態を持つエンティティのライフサイクル図
6. **インターフェース**: 各操作の入出力仕様
7. **検証**: フィールド単位およびフィールド間の検証ルール
8. **エラーカタログ**: コードとメッセージを含むすべてのエラー条件
9. **トレーサビリティ**: 要件への対応付け（FR-NNN 参照）
