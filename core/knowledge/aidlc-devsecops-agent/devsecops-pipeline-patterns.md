# DevSecOps Pipeline Patterns

CI/CD パイプラインのすべての段階にセキュリティを組み込み、検出をシフトレフトし、自動化されたゲートを適用する。

## Shift-Left Security Principles

- 脆弱性はできるだけ早く見つける: IDE で、コミット時に、そして CI の中で見つけ、デプロイ後まで持ち越さない。
- 初期の段階ではセキュリティチェックを高速かつ非ブロッキング（警告）にし、後の段階で強制（ゲート）にする。
- セキュリティの指摘はバグと同様に扱う: 追跡し、優先順位を付け、通常のスプリントサイクルの中で修正する。
- セキュリティは共有責任であり、門番の役割ではない。ツールと教育で開発者を支援する。

## SAST — Static Application Security Testing

**目的**: ソースコードを実行せずに分析し、脆弱性を見つける。

**ツール**:
- **Amazon CodeGuru Reviewer**: Java と Python 向けの ML によるコードレビュー。CodeCommit や GitHub と統合できる。
- **SonarQube / SonarCloud**: 多言語対応の静的解析。セキュリティ、信頼性、保守性をカバーする。
- **Semgrep**: 軽量なパターンベースのスキャン。高速で、カスタムルールをサポートし、チームの標準を強制するのに適している。
- **Bandit**（Python）、**ESLint のセキュリティプラグイン**（JavaScript/TypeScript）。

**パイプラインへの統合**:
- すべてのプルリクエストで SAST を実行する。指摘は PR のコメントまたはアノテーションとして報告する。
- 重大度のしきい値を定める: Critical/High の指摘ではマージをブロックし、Medium では警告する。
- 受け入れたリスクについては抑制ファイルを（根拠と有効期限を付けて）管理する。

## DAST — Dynamic Application Security Testing

**目的**: 細工したリクエストを実行中のアプリケーションに送り、実行時の脆弱性を見つける。

- 本番ではなく、ステージング環境またはエフェメラル環境に対して実行する。
- ツール: OWASP ZAP（オープンソース）、Burp Suite（商用）、ネットワークスキャン用の AWS Inspector。
- テスト環境へのデプロイ後に、CD パイプラインに組み込む。
- DAST は SAST を補完する。それぞれが異なる種類の脆弱性を検出する。

## Dependency Vulnerability Scanning

**目的**: サードパーティライブラリと推移的依存関係に含まれる既知の CVE を検出する。

**ツール**:
- **Amazon Inspector**: EC2 インスタンス、Lambda 関数、ECR イメージのソフトウェア脆弱性をスキャンする。
- **Snyk**: 開発者向けで、npm、pip、Maven、Go モジュールをサポートする。修正用の PR を提供する。
- **Dependabot**（GitHub ネイティブ）: 脆弱性アラートと連動した、依存関係更新 PR の自動作成。
- **npm audit / pip-audit / cargo audit**: ローカルで手早く確認するための、言語レベルの組み込みツール。

**パイプラインへの統合**:
- すべてのビルドでスキャンする。既知のエクスプロイトがある Critical/High の CVE ではビルドを失敗させる。
- サプライチェーンの透明性のため、Syft または Trivy を使って SBOM（Software Bill of Materials）を生成する。
- 依存関係は少なくとも毎月見直して更新する。Dependabot または Renovate で自動化する。

## IaC Security Scanning

**目的**: デプロイ前に Infrastructure as Code の設定ミスを検出する。

**ツール**:
- **cfn-lint**: CloudFormation のリンター。構文とベストプラクティスを検証する。
- **cfn-nag**: CloudFormation の静的解析。過度に許可的な IAM ポリシーや暗号化されていないリソースを見つける。
- **Checkov**: 複数フレームワーク対応のスキャナー（CloudFormation、Terraform、CDK、Kubernetes）。カスタムルールによる Policy as Code。
- **cdk-nag**: CDK ネイティブ。CDK コンストラクトを AWS Solutions のルールや NIST/HIPAA のパックに照らしてチェックする。

**パイプラインへの統合**:
- `cdk synth` や `cfn deploy` の前に IaC スキャンを実行する。High の指摘ではパイプラインを失敗させる。
- cdk-nag を CDK Aspect として使い、違反を合成後ではなく合成時に検出する。
- 例外ルールは（帯域外ではなく）コード内で管理し、根拠を示すコメントを必須にする。

## Secret Detection

**目的**: 認証情報、API キー、トークンがソース管理にコミットされるのを防ぐ。

**ツール**:
- **git-secrets**（AWS Labs）: AWS の認証情報に一致するパターンをブロックする pre-commit フック。
- **truffleHog**: git の履歴をスキャンし、エントロピーの高い文字列や既知のシークレットのパターンを探す。
- **Gitleaks**: 高速で設定可能。CI と pre-commit をサポートする。デフォルトのルールセットが優れている。
- **GitHub secret scanning**: GitHub リポジトリに組み込まれており、既知のプロバイダーのシークレットがコミットされるとアラートを出す。

**ベストプラクティス**:
- シークレット検出用の pre-commit フックを、すべての開発者のマシンにインストールする。
- pre-commit フックの漏れに備え、CI でもシークレットスキャンを実行する。
- シークレットがコミットされた場合: ただちに失効させ、ローテーションし、その後 git の履歴をクリーンアップする。
- シークレットは AWS Secrets Manager または SSM Parameter Store に保存し、コードや環境ファイルには決して置かない。

## Container Image Scanning

**目的**: コンテナイメージに含まれる OS レベルおよびアプリケーションレベルの脆弱性を検出する。

- **Amazon ECR のイメージスキャン**: Basic（Clair ベース）と Enhanced（Inspector による）のスキャン。
- **Trivy**: 包括的なオープンソースのスキャナー。OS パッケージ、言語ライブラリ、IaC ファイルをカバーする。
- ECR へのプッシュ時とデプロイ前にイメージをスキャンする。Critical の指摘があるイメージはデプロイをブロックする。
- 攻撃対象領域を減らすため、最小限のベースイメージ（distroless、Alpine）を使う。
- 新たに公開された CVE を取り込むため、イメージを毎週再ビルドして再スキャンする。

## Security Gates in CI/CD

パイプラインの各段階で、明確な合否基準を定める:

| 段階 | ゲート | 失敗時のアクション |
|-------|------|-------------------|
| コミット | シークレット検出 | コミットをブロック（pre-commit フック） |
| PR | SAST スキャン | Critical/High でマージをブロック |
| ビルド | 依存関係スキャン | エクスプロイトのある Critical でビルドを失敗 |
| ビルド | IaC スキャン | High でビルドを失敗 |
| ステージングへのデプロイ | DAST スキャン | 本番への昇格をブロック |
| 本番へのデプロイ | イメージスキャン | Critical でデプロイをブロック |
| デプロイ後 | Inspector の継続的スキャン | アラートを出してチケットを作成 |

例外は、セキュリティチームの承認を必要とし自動的に失効する期限付きの免除として自動化する。
