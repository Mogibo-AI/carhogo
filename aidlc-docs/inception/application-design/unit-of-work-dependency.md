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
| 依存内容 | 責務 | 詳細 |
|---------|------|------|
| IoT エンドポイント | Unit 2 が実装 | CDK デプロイ後に確定。`injectIoTConfig` Gradle タスクで `config.properties` に自動注入 |
| X.509 デバイス証明書 | **Unit 1 が生成・出力 / Unit 2 が注入** | CDK デプロイ時に証明書ファイルを `cdk/output/certs/` に書き出す（Unit 1 責務）。Unit 2 の `injectCertificates` Gradle タスクが `cdk/output/certs/` から `watch/app/src/main/res/raw/` へコピーする（Unit 2 責務） |
| IoT Thing 名・デバイス ID | Unit 2 が実装 | CDK 出力値（CFn Output）から取得 |

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
  $ cd watch && ./gradlew injectCertificates injectIoTConfig assembleDebug
  （injectCertificates タスク: cdk/output/certs/ → watch/app/src/main/res/raw/ へ証明書を自動コピー）
  （injectIoTConfig タスク: IoTEndpoint を config.properties に自動注入）
  ※ Step 1 の CDK デプロイ完了後に実行すること

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
| X.509 証明書の配置 | Unit 1（CDK）+ Unit 2（Pixel Watch） | Unit 1: CDK デプロイ時に `cdk/output/certs/` へ証明書ファイルを出力。Unit 2: `injectCertificates` Gradle タスクで `watch/app/src/main/res/raw/` へ自動コピー |

---

## X.509 証明書管理フロー詳細

証明書管理はUnit 1（CDK）とUnit 2（Pixel Watch）の**共同責務**です。

### Unit 1（CDK）の責務: 証明書の生成と出力

CDK デプロイ時（`npm run deploy`）に以下を実行する:

1. AWS IoT Core で X.509 デバイス証明書を生成
2. 生成した証明書ファイルをローカルの `cdk/output/certs/` ディレクトリに書き出す

```
cdk/output/certs/
├── device-certificate.pem   # デバイス証明書
├── private-key.pem          # 秘密鍵
└── root-ca.pem              # Amazon ルート CA 証明書
```

> `cdk/output/` は `.gitignore` に追加し、証明書をリポジトリに含めない。

### Unit 2（Pixel Watch）の責務: 証明書の注入

`injectCertificates` Gradle カスタムタスクを `watch/build.gradle.kts` に実装する。
`injectIoTConfig`（IoT エンドポイント注入）と同じパターンで実装する。

```kotlin
// watch/build.gradle.kts
tasks.register("injectCertificates") {
    doLast {
        val certsDir = rootProject.file("../cdk/output/certs")
        val rawDir = file("src/main/res/raw")
        rawDir.mkdirs()
        certsDir.listFiles()?.forEach { it.copyTo(File(rawDir, it.name), overwrite = true) }
        println("Certificates injected: ${certsDir.list()?.joinToString()}")
    }
}
// assembleDebug が injectCertificates に依存するよう設定
tasks.named("assembleDebug") { dependsOn("injectCertificates", "injectIoTConfig") }
```

### フロー全体図

```
Unit 1 (CDK)                         Unit 2 (Pixel Watch)
─────────────────────────────────    ─────────────────────────────────
$ npm run deploy
  └─ IoT 証明書を生成
  └─ cdk/output/certs/ に出力  ──►  $ ./gradlew assembleDebug
                                      └─ injectCertificates タスク実行
                                           └─ cdk/output/certs/ を読み込み
                                           └─ res/raw/ へコピー
                                      └─ injectIoTConfig タスク実行
                                      └─ APK ビルド
```
