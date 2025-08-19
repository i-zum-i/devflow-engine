# インフラストラクチャ詳細設計書

## 1. 概要

本ドキュメントは「DevFlow Engine」のインフラストラクチャに関する詳細設計を定義する。
AWS上に、スケーラブルでセキュアなサーバーレスアーキテクチャを構築する。

- **クラウドプロバイダー:** Amazon Web Services (AWS)
- **プロビジョニング:** AWS CDK (TypeScript)
- **CI/CD:** GitHub Actions

## 2. 全体アーキテクチャ図

```mermaid
graph TD
    subgraph "開発者"
        A[ブラウザ]
    end

    subgraph "AWS Cloud"
        B[Route 53] --> C[CloudFront]
        C --> D[S3 Bucket]
        C --> E[API Gateway]

        E --> F{Lambda}
        F --> G[DynamoDB]
        F --> H[ECS on Fargate]
        F --> I[Secrets Manager]

        H --> J[ECR]
        H --> K[CloudWatch Logs]
    end

    subgraph "GitHub"
        L[GitHub API]
    end

    A --> B
    F --> L
    H --> L
```

## 3. AWSリソース詳細

### 3.1. ネットワーク

- **VPC:** 新規にVPCを作成する。
    - パブリックサブネットとプライベートサブネットをそれぞれAZに配置する。
    - NAT Gatewayをパブリックサブネットに配置し、プライベートサブネットからのアウトバウンド通信を可能にする。
- **セキュリティグループ:**
    - `sg-alb`: ALB用。ポート443のインバウンドを許可。
    - `sg-ecs`: ECSタスク用。`sg-alb`からのインバウンドと、必要なアウトバウンド（GitHub, ECRなど）を許可。
    - `sg-lambda`: Lambda関数用。VPC内リソースへのアクセスを制御。

### 3.2. フロントエンド

- **S3:** Reactアプリケーションの静的ファイルをホスティングする。
- **CloudFront:** S3をオリジンとし、コンテンツをグローバルに配信する。SSL証明書はACMで管理。
- **Route 53:** カスタムドメインをCloudFrontに紐付ける。

### 3.3. バックエンド

- **API Gateway:** REST APIを作成し、Lambda関数へのエンドポイントを提供する。
- **Lambda:** Python 3.11ランタイムで、各ビジネスロジックを実行する。
    - `devflow-start-session`, `devflow-execute-prompt`, `devflow-get-status`, `devflow-get-editor-url`, `devflow-stop-session`
- **DynamoDB:** `devflow-sessions`テーブルでセッション情報を管理し、24時間のTTL設定を行う。

### 3.4. コンテナ

- **ECS on Fargate:** `code-server`コンテナをオンデマンドで実行する。
    - Fargate Spotを利用してコストを最適化する。
- **ECR:** `code-server`のカスタムDockerイメージを保存する。

### 3.5. セキュリティ・その他

- **Secrets Manager:** GitHub Appの認証情報など、機密情報を安全に保管する。
- **IAM:** 各AWSリソースに最小権限の原則に基づいたIAMロールを割り当てる。
- **CloudWatch:** Lambda関数とECSタスクのログ、メトリクスを収集・監視する。

## 4. CI/CDパイプライン (GitHub Actions)

### 4.1. フロントエンド

1.  **Trigger:** `main`ブランチへのpush。
2.  **Build:** `npm install`, `npm run build`でReactアプリをビルド。
3.  **Deploy:** ビルド成果物をS3バケットに同期する。
4.  **Invalidate:** CloudFrontのキャッシュを無効化する。

### 4.2. バックエンド (Lambda)

1.  **Trigger:** `main`ブランチへのpush。
2.  **Test:** `pytest`でユニットテストを実行。
3.  **Package:** `sam package`または`cdk deploy`でLambda関数をパッケージング＆デプロイ。

### 4.3. コンテナ (ECR)

1.  **Trigger:** `container`ディレクトリ内の変更があった場合の`main`ブランチへのpush。
2.  **Build:** `docker build`でコンテナイメージをビルド。
3.  **Push:** ビルドしたイメージをECRにプッシュする。

## 5. コスト最適化・運用設計

### 5.1. コスト最適化戦略

#### 5.1.1. Fargate Spot利用

```typescript
// Fargate Spot設定（開発環境用）
const spotTaskDefinition = new FargateTaskDefinition(this, 'SpotTaskDefinition', {
  family: 'devflow-code-server-spot',
  cpu: 1024,
  memoryLimitMiB: 2048,
  taskRole,
  executionRole
});

// ECS Service with Spot capacity
const ecsService = new FargateService(this, 'DevFlowService', {
  cluster,
  taskDefinition: spotTaskDefinition,
  desiredCount: 0, // オンデマンド起動
  capacityProviderStrategies: [
    {
      capacityProvider: 'FARGATE_SPOT',
      weight: 100
    }
  ],
  enableExecuteCommand: true // デバッグ用
});
```

#### 5.1.2. 自動スケールダウン

```typescript
// Lambda関数（セッション自動クリーンアップ）
const cleanupFunction = new Function(this, 'SessionCleanupFunction', {
  runtime: Runtime.PYTHON_3_11,
  handler: 'cleanup.lambda_handler',
  code: Code.fromAsset('lambda/cleanup'),
  timeout: Duration.minutes(15),
  environment: {
    DYNAMODB_TABLE: sessionsTable.tableName,
    ECS_CLUSTER: cluster.clusterArn
  }
});

// 定期実行（1時間ごと）
new Rule(this, 'CleanupRule', {
  schedule: Schedule.rate(Duration.hours(1)),
  targets: [new LambdaFunction(cleanupFunction)]
});
```

### 5.2. 監視ダッシュボード

```typescript
// CloudWatchダッシュボード
const dashboard = new Dashboard(this, 'DevFlowDashboard', {
  dashboardName: 'DevFlow-Engine-Metrics'
});

dashboard.addWidgets(
  new GraphWidget({
    title: 'Lambda Invocations',
    left: [startSessionFunction.metricInvocations()],
    right: [startSessionFunction.metricErrors()]
  }),
  new GraphWidget({
    title: 'ECS Tasks',
    left: [ecsTaskCount],
    right: [new Metric({
      namespace: 'AWS/ECS',
      metricName: 'CPUUtilization',
      dimensionsMap: {
        ClusterName: cluster.clusterName
      }
    })]
  }),
  new SingleValueWidget({
    title: 'Active Sessions',
    metrics: [new Metric({
      namespace: 'AWS/DynamoDB',
      metricName: 'ItemCount',
      dimensionsMap: {
        TableName: sessionsTable.tableName
      }
    })]
  })
);
```

### 5.3. バックアップ・災害復旧

```typescript
// DynamoDBポイントインタイム復旧有効化（上記で設定済み）
// pointInTimeRecovery: true

// S3バケットのクロスリージョンレプリケーション
const backupBucket = new Bucket(this, 'BackupBucket', {
  bucketName: 'devflow-engine-backup',
  versioned: true,
  lifecycleRules: [
    {
      enabled: true,
      expiration: Duration.days(90),
      noncurrentVersionExpiration: Duration.days(30)
    }
  ]
});

// アーカイブ用Lambda
const archiveFunction = new Function(this, 'ArchiveFunction', {
  runtime: Runtime.PYTHON_3_11,
  handler: 'archive.lambda_handler',
  code: Code.fromAsset('lambda/archive'),
  timeout: Duration.minutes(15)
});

// 週次アーカイブ実行
new Rule(this, 'ArchiveRule', {
  schedule: Schedule.cron({
    weekDay: '1', // 月曜日
    hour: '2',    // 2:00 AM
    minute: '0'
  }),
  targets: [new LambdaFunction(archiveFunction)]
});
```

## 6. デプロイメント手順

### 6.1. 初回デプロイ

```bash
# 1. 依存関係のインストール
cd infrastructure
npm install

# 2. AWS CDK Bootstrap
npx cdk bootstrap

# 3. 環境変数の設定
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=123456789012

# 4. シークレットの事前作成
aws secretsmanager create-secret \
  --name "devflow/github-app" \
  --description "GitHub App credentials" \
  --secret-string '{"app_id":"12345","private_key":"-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----","installation_id":"67890"}'

aws secretsmanager create-secret \
  --name "devflow/claude-api-key" \
  --description "Claude API key" \
  --secret-string '{"api_key":"sk-ant-..."}'

# 5. インフラストラクチャデプロイ
npx cdk deploy --all --require-approval never

# 6. フロントエンドビルド・デプロイ
cd ../frontend
npm install
npm run build
aws s3 sync dist/ s3://devflow-engine-frontend --delete

# 7. コンテナイメージビルド・プッシュ
cd ../container
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker build -t devflow-code-server .
docker tag devflow-code-server:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/devflow-code-server:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/devflow-code-server:latest
```

### 6.2. 本番環境設定

```typescript
// 本番環境用スタック
export class DevFlowProductionStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // 本番環境用設定
    const prodConfig = {
      domainName: 'devflow.company.com',
      certificateArn: 'arn:aws:acm:us-east-1:123456789012:certificate/...',
      enableAutoScaling: true,
      enableMultiAZ: true,
      enableSpot: false, // 本番では通常のFargateを使用
      logRetentionDays: RetentionDays.ONE_MONTH,
      enableBackup: true,
      enableMonitoring: true
    };

    // ... rest of production configuration
  }
}
```