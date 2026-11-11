# REST API Design Guide

一貫性があり、予測可能で、進化させやすい HTTP API を設計するための実践的な原則。

## URL Naming Conventions

- コレクションには複数形の名詞を使う: `/orders`、`/users`、`/products`
- 所有関係はリソースをネストして表す: `/users/{userId}/orders/{orderId}`
- URL は浅く保つ（最大 2〜3 階層）。関係が弱い場合はフラットにする
- 複数語のセグメントには kebab-case を使う: `/order-items`（`/orderItems` ではない）
- URL に動詞を入れない。アクションは HTTP メソッドで表す
- フィルタリング、ソート、ページネーションにはクエリパラメータを使う: `/orders?status=pending&sort=-createdAt`

## HTTP Method Semantics

| メソッド | 目的 | 冪等 | 安全 |
|--------|---------|------------|------|
| GET | リソースの取得 | Yes | Yes |
| POST | リソースの作成、または処理の起動 | No | No |
| PUT | リソースの全体置換 | Yes | No |
| PATCH | リソースの部分更新 | No* | No |
| DELETE | リソースの削除 | Yes | No |

CRUD に対応しないアクションには POST を使う: `POST /orders/{id}/cancel`。

## Status Code Usage

- **200 OK** — GET、PUT、PATCH、またはアクション POST の成功
- **201 Created** — リソースを作成した POST の成功。Location ヘッダーを含める
- **204 No Content** — レスポンスボディのない DELETE または PUT の成功
- **400 Bad Request** — 構文の誤り、または不正なフィールド値
- **401 Unauthorized** — 認証情報がない、または無効
- **403 Forbidden** — 認証済みだが権限が不足している
- **404 Not Found** — リソースが存在しない
- **409 Conflict** — 状態の競合（重複、バージョン不一致）
- **422 Unprocessable Entity** — 構文は正しいがビジネスルールに違反している
- **429 Too Many Requests** — レート制限の超過。Retry-After ヘッダーを含める
- **500 Internal Server Error** — 処理されなかったサーバー障害

## Error Response Format

すべてのエラーで一貫したエンベロープを使う。機械が読めるコード、人間が読めるメッセージ、および任意でフィールド単位の詳細を含める:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "One or more fields failed validation.",
    "details": [
      { "field": "email", "reason": "Must be a valid email address." }
    ],
    "requestId": "abc-123"
  }
}
```

トレーサビリティのため、リクエスト ID を必ず含める。

## Pagination Patterns

- **オフセット方式**: `?offset=20&limit=10` — シンプルだが、OFFSET のコストにより大規模データセットでは性能が劣化する。
- **カーソル方式**: `?cursor=eyJpZCI6MTAwfQ&limit=10` — 最後に見たキーを不透明なトークンとしてエンコードする。DynamoDB や大規模データセットではこちらを推奨する。
- ページネーションのメタデータはレスポンスボディで返す: `nextCursor`、`hasMore`、`totalCount`（計算が安価な場合）。

## Versioning Strategies

- **URL パスによるバージョニング**（`/v1/orders`）— 最も明示的で、利用者にとって最も扱いやすい。公開 API ではこれを推奨する。
- **ヘッダーによるバージョニング**（`Accept: application/vnd.myapi.v2+json`）— URL はすっきりするが、発見しにくい。
- クエリパラメータによるバージョニング（`?version=2`）は避ける。フィルタリングと契約の選択を混同させる。
- バージョンを上げるのは破壊的変更を導入するときだけにする。追加的な変更（新しい任意フィールド）には新しいバージョンは不要である。

## OpenAPI and AsyncAPI

- OpenAPI 3.1 仕様を信頼できる唯一の情報源として維持する。サーバースタブとクライアント SDK はそこから生成する。
- イベント駆動 API（SNS、EventBridge、SQS）では、AsyncAPI を使ってメッセージスキーマとチャネルバインディングを文書化する。
- 仕様はコードと同じリポジトリに置き（`docs/openapi.yaml`）、CI で spectral や redocly-cli を使って検証する。

## HATEOAS Considerations

- レスポンスに `_links` を含め、関連するアクションやリソースへクライアントを導く。
- 利用可能な遷移が変化する複雑なステートマシン（注文のライフサイクル）で有用である。
- 内部のマイクロサービス API では、HATEOAS は多くの場合不要なオーバーヘッドである。発見しやすさが重要な公開 API やパートナー API に限定して使う。
