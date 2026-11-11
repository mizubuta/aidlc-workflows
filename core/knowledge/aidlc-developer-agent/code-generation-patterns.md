# Code Generation Patterns

## Purpose
クリーンで保守しやすいコードを生成するための標準とパターン。これらの原則は言語を問わず適用され、生成されるコードがプロトタイプ品質ではなく本番に耐える品質であることを保証する。

## Clean Code Principles

### Naming
- **変数**: 型ではなく、何を保持するかを表す。`str1` や `data` ではなく `customerEmail` を使う。
- **関数**: 何をするかを動詞で表す。`process()` や `doWork()` ではなく `calculateShippingCost()` を使う。
- **真偽値**: `is`, `has`, `can`, `should` の接頭辞を使う。`active` や `flag` ではなく `isActive` を使う。
- **定数**: 真の定数には SCREAMING_SNAKE_CASE を使う。`num` ではなく `MAX_RETRY_COUNT` を使う。
- **クラス**: 名詞を使う。`ProcessOrders` や `OrderHelper` ではなく `OrderProcessor` を使う。

### Functions
- 1 つのことだけを行う。関数名に「and」が含まれるなら、2 つの関数に分割する。
- 引数の数は少なく保つ（0〜3 が理想。3 を超えるならパラメータオブジェクトが必要なサインである）。
- 振る舞いを切り替える真偽値の引数は避け、代わりに明確な名前を持つ 2 つの関数を使う。
- ネストを浅くするため、ガード節では早期リターンする。

### Files
- 1 ファイルにつき主要な概念は 1 つ（1 クラス、1 モジュール、1 コンポーネント）。
- ファイルは 300 行未満に保つ。それより長いなら、その概念は多くのことをやりすぎている可能性が高い。
- 関連ファイルは種類ではなく機能単位でまとめる（後述の「Code Organization」を参照）。

## SOLID Principles Applied

### Single Responsibility (S)
各モジュール/クラスが変更される理由は 1 つだけである。
```
// Bad: UserService handles auth, profile, and notifications
// Good: AuthService, ProfileService, NotificationService
```

### Open/Closed (O)
既存コードを変更せずに振る舞いを拡張する。インターフェースとコンポジションを使う。
```
// Bad: Adding a new payment type requires modifying PaymentProcessor
// Good: PaymentProcessor accepts a PaymentStrategy interface; add new strategies without touching the processor
```

### Liskov Substitution (L)
サブタイプは、振る舞いを壊すことなく基底型と置き換え可能でなければならない。メソッドのオーバーライドで契約が変わるなら、継承階層が誤っている。

### Interface Segregation (I)
クライアントは、使わないメソッドに依存すべきではない。大きなインターフェースより、小さく焦点の絞られたインターフェースを優先する。
```
// Bad: interface Repository { find, save, delete, export, import, backup }
// Good: interface Readable { find }, interface Writable { save, delete }
```

### Dependency Inversion (D)
上位モジュールは具体的な実装ではなく抽象に依存する。依存関係は外から渡し、内部で生成しない。
```
// Bad: class OrderService { db = new PostgresDB() }
// Good: class OrderService { constructor(db: Database) }
```

## Error Handling Patterns

### Result Type (Preferred)
想定される失敗に対しては、例外を投げる代わりに成功/失敗を明示的に返す。
```typescript
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

function parseEmail(input: string): Result<Email, ValidationError> {
  if (!isValidEmail(input)) {
    return { ok: false, error: new ValidationError('Invalid email format') };
  }
  return { ok: true, value: new Email(input) };
}
```

### Try-Catch Hierarchy
例外を使う場合は、次の階層に従う:
1. **まず特定の例外を捕捉する** — 既知の、回復可能なエラーを処理する
2. **未知の例外は伝播させる** — `Exception` や `Error` を広く捕捉しない
3. **境界で捕捉する** — API ハンドラー、イベントプロセッサー、CLI のエントリポイントが、すべてを捕捉する場所として適切である
4. **決して例外を黙って握りつぶさない** — `catch (e) {}` はバグを隠す

### Error Messages
- 何が起きたか、なぜ起きたか、ユーザー/開発者がどう対処できるかを含める
- 関連するコンテキスト（ID、入力値、実行中の操作）を含める
- ユーザー向けのエラーで内部の実装詳細を露出しない

## Logging Standards

### Log Levels
- **ERROR**: 対応が必要な失敗が発生した。診断に十分なコンテキストを含める。
- **WARN**: 想定外のことが起きたが処理された。問題が進行しつつある兆候の可能性がある。
- **INFO**: 重要なビジネスイベント（注文の確定、ユーザー登録、決済処理）。1 操作につき 1 件。
- **DEBUG**: 開発向けの詳細な診断情報。どのレベルでも機密データは決してログに出さない。

### Structured Logging
```json
{
  "level": "INFO",
  "message": "Order placed successfully",
  "orderId": "ord-123",
  "customerId": "cust-456",
  "totalAmount": 99.99,
  "timestamp": "2024-01-15T10:30:00Z",
  "correlationId": "req-789"
}
```

### Rules
- ログは操作の境界（開始、成功、失敗）で出す — ループの内側では出さない
- サービス間の追跡のために相関 ID/リクエスト ID を含める
- 決してログに出さないもの: パスワード、トークン、API キー、PII、クレジットカード番号
- 本番のログには自由形式の文字列ではなく構造化形式（JSON）を使う

## Input Validation at Boundaries

### Boundary Definition
すべての信頼境界 — 信頼できないソースからシステムにデータが入ってくる場所 — で入力を検証する:
- API リクエストハンドラー（HTTP、gRPC、GraphQL）
- イベント/メッセージのコンシューマー（SQS、Kafka、EventBridge）
- ファイルアップロードの処理部
- CLI 引数のパーサー
- 外部システムからのデータベースクエリ結果

### Validation Strategy
```
External Input → Validate at boundary → Convert to domain type → Domain logic uses typed values
```

### What to Validate
- **存在**: 必須フィールドが存在し、null/空でない
- **型**: 値が想定どおりの型である（文字列、数値、日付）
- **範囲**: 数値が許容範囲内にあり、文字列が長さ制限内にある
- **形式**: メールアドレス、URL、日付、電話番号が想定されるパターンに一致する
- **ビジネスルール**: 値がその文脈で妥当である（ステータス遷移、参照整合性）

### After Validation
データが境界を通過してドメイン型に変換されたら、内部のコードで再検証してはならない（NOT）。境界を信頼する。これにより、ドメインロジックはクリーンに保たれ、ビジネスルールに集中できる。

## Code Organization

### By Feature (Recommended)
```
/src
  /orders
    order.ts              # domain model
    order-service.ts      # business logic
    order-repository.ts   # data access
    order-handler.ts      # API handler
    order.test.ts         # tests
  /customers
    customer.ts
    customer-service.ts
    ...
```

### By Layer (Avoid for Medium-Large Projects)
```
/src
  /models        # all domain models from all features
  /services      # all services from all features
  /repositories  # all repositories from all features
  /handlers      # all handlers from all features
```

### Why Feature-Based is Better
- 関連するコードが同じ場所に置かれる — 「orders」への変更は 1 つのディレクトリ内のファイルだけに及ぶ
- 1 つのディレクトリを見るだけで機能のスコープを把握しやすい
- 後からマイクロサービスへ切り出しやすい（各機能ディレクトリがその候補になる）
- レイヤー単位の構成では、関連するコードがプロジェクト全体に散らばる

## Code Generation Checklist
- [ ] 命名規約に従っている（説明的、一貫性がある、略語を使わない）
- [ ] 関数が小さく、単一の目的を持つ
- [ ] エラー処理が明示的である（Result 型、または境界での捕捉）
- [ ] すべての信頼境界で入力を検証している
- [ ] 操作の境界で構造化ログを出している
- [ ] シークレット、URL、設定値をハードコードしていない
- [ ] 依存関係は注入されており、内部で生成されていない
- [ ] 生成したすべてのコードにテストが付いている
- [ ] コードがレイヤー単位ではなく機能単位で構成されている
