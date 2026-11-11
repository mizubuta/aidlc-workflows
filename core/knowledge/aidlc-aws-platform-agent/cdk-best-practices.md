# AWS CDK Best Practices

## Purpose
AWS Cloud Development Kit を使って、保守しやすく、安全で、テスト可能なインフラストラクチャを構築するためのガイドライン。これらのプラクティスは、TypeScript（大半のチームに推奨される言語）を用いた CDK v2 に適用される。

## Construct Levels

### L1 (Cfn Resources)
- CloudFormation リソースを直接ラップしたもの（例: `CfnBucket`）
- L2 コンストラクトが必要なプロパティを公開していない場合にのみ使う
- すべてのプロパティを手動で設定する必要がある（デフォルト値なし）

### L2 (Curated Constructs)
- 妥当なデフォルト値を備えた、AWS がメンテナンスするコンストラクト（例: `Bucket`、`Function`、`Table`）
- ヘルパーメソッドを含む（例: `bucket.grantRead(lambda)`）
- 大半のユースケースで推奨 — AWS のベストプラクティスが組み込まれている

### L3 (Patterns)
- 複数のリソースを組み合わせた、より高レベルのコンストラクト（例: `LambdaRestApi`）
- パターンがニーズに完全に合致する場合に使う
- 大幅なカスタマイズが必要な場合は避け、代わりに L2 を使う

## Construct Design Patterns

### Single Responsibility
各カスタムコンストラクトは、1つの論理的な単位（サービス、データパイプラインの1段階、監視スタック）を表すようにする。互いに無関係なリソースを構築するコンストラクトを作成してはならない。

### Props Interface Pattern
```typescript
export interface OrderServiceProps {
  readonly vpc: ec2.IVpc;
  readonly table: dynamodb.ITable;
  readonly environment: string;         // 'dev' | 'staging' | 'prod'
  readonly alarmTopic?: sns.ITopic;     // optional props use ?
}

export class OrderService extends Construct {
  public readonly api: apigateway.RestApi;  // expose outputs as public readonly

  constructor(scope: Construct, id: string, props: OrderServiceProps) {
    super(scope, id);
    // ...
  }
}
```

### Rules
- 依存関係は props 経由で受け取り（依存性注入）、共有リソースをコンストラクトの内部で作成しない
- props には具象型ではなくインターフェース型（`IVpc`、`ITable`）を使う — スタック間の参照が可能になる
- 利用側のコンストラクトのために、出力を public readonly プロパティとして公開する
- 省略可能な props には、デフォルトの動作を説明するドキュメントを付ける

## Stack Organization

### Recommended Structure
```
/infrastructure
  /bin
    app.ts              # CDK app entry point, environment configuration
  /lib
    /constructs         # Reusable L3 constructs
      order-service.ts
      monitoring.ts
    /stacks
      network-stack.ts      # VPC, subnets, security groups
      data-stack.ts         # DynamoDB, S3, RDS
      compute-stack.ts      # Lambda, ECS, API Gateway
      monitoring-stack.ts   # CloudWatch, alarms, dashboards
  /test
    order-service.test.ts
    data-stack.test.ts
```

### Stack Separation Guidelines
- ライフサイクルでスタックを分ける: 一緒に変更されるリソースは同じスタックに置く
- ステートフルなリソース（データベース、S3 バケット）は、ステートレスなリソース（Lambda、API Gateway）とは別のスタックに置く
- ステートフルなスタックはめったに変更されず、ステートレスなスタックは頻繁にデプロイされる
- スタック間の参照は控えめに使う — デプロイ時の結合を生む

## Environment-Aware Stacks

### Pattern
```typescript
// bin/app.ts
const app = new cdk.App();
const env = app.node.tryGetContext('env') || 'dev';

const config = {
  dev:     { instanceType: 't3.small', minCapacity: 1, maxCapacity: 2 },
  staging: { instanceType: 't3.medium', minCapacity: 2, maxCapacity: 4 },
  prod:    { instanceType: 't3.large', minCapacity: 3, maxCapacity: 10 },
}[env];

new ComputeStack(app, `ComputeStack-${env}`, {
  env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: 'us-east-1' },
  config,
});
```

### Rules
- アカウント ID やリージョンを決してハードコードしない — 環境変数またはコンテキストを使う
- すべての環境で同じコードを使い、違いは設定（config）でパラメータ化する
- 本番スタックでは明示的な `env`（アカウント + リージョン）を必ず指定する — デフォルトに頼らない

## CDK Testing

### Assertion Tests (Fine-Grained)
```typescript
test('DynamoDB table has encryption enabled', () => {
  const app = new cdk.App();
  const stack = new DataStack(app, 'TestStack');
  const template = Template.fromStack(stack);

  template.hasResourceProperties('AWS::DynamoDB::Table', {
    SSESpecification: {
      SSEEnabled: true,
    },
  });
});
```

### Snapshot Tests (Regression Detection)
```typescript
test('stack matches snapshot', () => {
  const app = new cdk.App();
  const stack = new DataStack(app, 'TestStack');
  const template = Template.fromStack(stack);
  expect(template.toJSON()).toMatchSnapshot();
});
```
- 意図的な変更を行った後に、意図してスナップショットを更新する（`jest --updateSnapshot`）
- プルリクエストでスナップショットの差分をレビューする — インフラストラクチャの何が変わるかが正確にわかる

### What to Test
- セキュリティ関連のプロパティ: 暗号化の有効化、パブリックアクセスのブロック、最小権限のポリシー
- 重要な設定: 保持ポリシー、バックアップ設定、オートスケーリングのパラメータ
- リソース数: Lambda 関数、テーブル、キューの想定数
- CDK の内部や CloudFormation の実装の詳細はテストし**ない**

## Security Defaults

### Encryption
- S3: `encryption: s3.BucketEncryption.S3_MANAGED`（最低限）、または機密データには KMS
- DynamoDB: `encryption: dynamodb.TableEncryption.AWS_MANAGED`（デフォルト）、またはカスタマー管理の KMS
- SQS: 機密性の高いメッセージ内容には `encryption: sqs.QueueEncryption.KMS`
- EBS: アカウント設定でデフォルトの暗号化を有効にする

### Access Control
- S3: `blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL`（パブリックな静的コンテンツを配信する場合を除き、常に）
- Lambda: IAM ポリシーを手書きする代わりに `grant*` メソッドを使う
- API Gateway: すべてのルートに認可を追加する（IAM、Cognito、または Lambda オーソライザー）

### Logging
- S3: 専用のログ用バケットへのアクセスログを有効にする
- API Gateway: アクセスログと実行ログを有効にする
- Lambda: ログは自動的に CloudWatch に送られる。保持期間を設定する（`logRetention: logs.RetentionDays.ONE_MONTH`）

### Least Privilege
```typescript
// Good: specific grant
table.grantReadData(lambdaFunction);

// Bad: overly broad
lambdaFunction.addToRolePolicy(new iam.PolicyStatement({
  actions: ['dynamodb:*'],
  resources: ['*'],
}));
```

## CDK Aspects for Compliance

### Purpose
Aspect はツリー内のすべてのコンストラクトを巡回し、リソースを検証・警告・変更できる。

```typescript
class EncryptionChecker implements cdk.IAspect {
  public visit(node: IConstruct): void {
    if (node instanceof s3.CfnBucket) {
      if (!node.bucketEncryption) {
        Annotations.of(node).addError('S3 bucket must have encryption enabled');
      }
    }
  }
}

// Apply to the entire app
Aspects.of(app).add(new EncryptionChecker());
```

### Common Compliance Aspects
- すべての S3 バケットで暗号化とパブリックアクセスのブロックが設定されていることを検証する
- すべての DynamoDB テーブルでポイントインタイムリカバリが有効になっていることを検証する
- すべての Lambda 関数で予約済み同時実行数が設定されていることを検証する
- すべてのセキュリティグループが 0.0.0.0/0 からのインバウンドを許可していないことを検証する
- 必須のコスト配分タグをすべてのリソースに付与する
