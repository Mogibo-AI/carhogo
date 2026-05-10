# ユニット依存関係 — CarHogo

## 依存関係マトリクス

| 依存元 → 依存先 | Unit 1: CDK | Unit 2: Pixel Watch | Unit 3: backend | Unit 4: Browser |
|----------------|:-----------:|:-------------------:|:---------------:|:---------------:|
| **Unit 1: CDK** | — | 証明書・エンドポイント提供 | Lambda 定義・DynamoDB 提供 | Cognito・IoT Endpoint 提供 |
| **Unit 2: Pixel Watch** | ✓ 必須 | — | IoT Core 経由で間接連携 | 非依存 |
| **Unit 3: backend** | ✓ 必須 | 非依存（IoT Core 経由） | — | Device Shadow 経由で間接連携 |
| **Unit 4: Browser** | ✓ 必須 | 非依存 | Device Shadow 経由で間接連携 | — |

---

## 依存関係の詳細

### Unit 2（Pixel Watch）→ Unit 1（CDK）
| 依存内容 | 詳細 |
|---------|------|
| IoT エンドポイント | CDK デプロイ後に確定。`injectIoTConfig` Gradle タスクで `config.properties` に注入 |
| X.509 デバイス証明書 | CDK で生成した証明書ファイルを `watch/app/src/main/res/raw/` に手動配置 |
| IoT Thing 名・デバイス ID | CDK 出力値（CFn Output）から取得 |

### Unit 3（backend）→ Unit 1（CDK）
| 依存内容 | 詳細 |
|---------|------|
| Lambda 環境変数 | CDK がデプロイ時に設定（DYNAMODB_TABLE・IOT_ENDPOINT・SECRETS_ARN 等） |
| DynamoDB テーブル名 | CDK 出力値から取得。Lambda 内でテーブル名を参照 |
| IoT Rules Engine | CDK で定義した Rule が BiometricAnalyzerLambda を自動トリガー |
| Secrets Manager ARN | Google API キー・Bedrock アクセス設定の ARN を環境変数で受け取る |

### Unit 4（Browser）→ Unit 1（CDK）
| 依存内容 | 詳細 |
|---------|------|
| Cognito User Pool ID / App Client ID | CDK 出力値。`browser/src/config.ts` に設定 |
| Cognito Identity Pool ID | CDK 出力値。IoT WebSocket 接続用 Credentials 取得に使用 |
| IoT エンドポイント | CDK 出力値。IoTShadowService の WebSocket 接続先 |
| Nova 2 Sonic リージョン | `ap-northeast-1`（固定値。CDK 依存なし） |

---

## ビルド・デプロイ順序

```
Step 1: CDK デプロイ（Unit 1）
  $ cd cdk && npm run deploy
  → CloudFormation Outputs:
    - IoTEndpoint
    - CognitoUserPoolId
    - CognitoIdentityPoolId
    - DynamoDBTableName
    - BiometricAnalyzerLambdaArn
    - ActionExecutorLambdaArn

Step 2a: Pixel Watch ビルド（Unit 2）— CDK デプロイ後に実行
  $ cd watch && ./gradlew assembleDebug
  （injectIoTConfig タスクが IoTEndpoint を config.properties に自動注入）
  → IoT 証明書ファイルを手動で res/raw/ に配置が必要

Step 2b: backend デプロイ（Unit 3）— Step 1 と並行可
  $ cd backend && npm run deploy
  （CDK が Lambda ZIP を自動デプロイ済みのため、コード変更時のみ再デプロイ）

Step 3: Browser ビルド（Unit 4）— Step 1 後に実行
  $ cd browser && npm run build
  （CDK 出力値を browser/src/config.ts に設定後にビルド）
  → S3 にアップロード・CloudFront キャッシュ無効化
```

---

## 技術的制約事項

| 制約 | 影響ユニット | 対応方針 |
|------|-----------|---------|
| Amazon Bedrock 未承認 | Unit 3（ActionExecutorLambda） | フォールバックテキストで動作。承認後に `MessageGenerator` を切り替え |
| Google API 未準備 | Unit 3（ActionExecutorLambda） | `CalendarService`・`EtaService` をスタブ実装で開発。API キー取得後に差し替え |
| SNS サンドボックス | Unit 3（ActionExecutorLambda） | 登録済み電話番号のみ SMS 送信可 |
| X.509 証明書の手動配置 | Unit 2（Pixel Watch） | CDK デプロイ後に証明書ファイルを手動で配置する手順をドキュメント化 |
