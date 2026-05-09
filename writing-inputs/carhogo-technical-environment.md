# テクニカル環境ドキュメント: CarHogo

## プロジェクト技術サマリー

- **プロジェクト名**: CarHogo
- **プロジェクトタイプ**: グリーンフィールド（新規開発）
- **主要ランタイム環境**: クラウド
- **クラウドプロバイダー**: AWS
- **ターゲットデプロイモデル**: サーバーレス（Lambda + IoT Core）
- **チーム規模**: 小規模
- **チームの経験**: JavaScript/TypeScript・Kotlin・AWS CDK・React の実務経験あり。Amazon Nova 2 Sonicは初使用。

---

## プログラミング言語

### 必須言語

| 言語 | バージョン | 用途 | 選定理由 |
|------|----------|------|---------|
| TypeScript | 5.x | Lambdaバックエンド・AWS CDKインフラ・ブラウザアプリ（React） | 型安全性・チームの習熟度。フロントエンドとバックエンドで言語を統一できる。 |
| JavaScript (Node.js) | 20.x | Lambda関数ランタイム | AWSサーバーレスの標準ランタイム |
| Kotlin | 1.9.x | Wear OSアプリ（Pixel Watch） | Android/Wear OS開発の標準言語 |

### 許可言語

| 言語 | 使用条件 |
|------|---------|
| Python | Amazon Nova 2 Sonic統合でPython SDKが必要な場合のみ |

### 禁止言語

| 言語 | 理由 | 代替 |
|------|------|------|
| Java | Wear OSでのJVMコールドスタートのオーバーヘッド。Kotlinで代替可能 | Kotlin |
| PHP | チームの習熟度なし | TypeScript |

---

## フレームワークとライブラリ

### 必須フレームワーク

| フレームワーク/ライブラリ | バージョン | ドメイン | 選定理由 |
|----------------------|----------|---------|---------|
| AWS CDK | 2.x | インフラストラクチャ as Code | AWSリソースのコード定義。TypeScript CDK。 |
| React | 18.x | ブラウザアプリUI | チームの習熟度。コンポーネントベースのUI構築。 |
| Vite | 5.x | フロントエンドビルドツール | 高速な開発サーバーとビルド |
| Tailwind CSS | 3.x | UIスタイリング | ユーティリティファーストCSS。迅速なUI開発。 |
| AWS SDK v3 | 3.x | AWSサービス呼び出し | Lambda内からのDynamoDB・Bedrock・IoT操作 |
| amazon-cognito-identity-js | 6.x | Cognito認証（ブラウザ） | Amplifyなしでの直接Cognito認証 |
| mqtt.js | 5.x | IoT WebSocket接続（ブラウザ） | SigV4署名付きMQTT over WSS |
| Jest | 29.x | ユニットテスト（バックエンド） | Node.jsの標準テストランナー |
| Vitest | 1.x | ユニットテスト（フロントエンド） | Viteとの統合。Jestと互換性のあるAPI。 |

### 推奨ライブラリ

必要な機能が求められる場合に使用します。事前に追加しないでください。

| ライブラリ | 用途 | 使用タイミング |
|----------|------|-------------|
| axios | HTTPクライアント | Google Maps・Google Calendar APIへの外部HTTP呼び出し |
| recharts | グラフ描画 | ブラウザアプリでのECG波形・心拍数グラフ表示 |
| three.js | 3Dアニメーション | SLEEP・ANGERアクションの3Dエフェクト実装 |

### 禁止ライブラリ

| ライブラリ | 理由 | 代替 |
|----------|------|------|
| Paho Android Service | Android 12以降で非対応 | Paho MqttAsyncClient（直接使用） |
| Moment.js | 非推奨。バンドルサイズが大きい | date-fns またはネイティブJS |
| Lodash（フル） | バンドルサイズが大きい | ネイティブJS |

---

## クラウド環境

### クラウドプロバイダー

- **主要プロバイダー**: AWS
- **アカウント構成**: シングルAWSアカウント
- **リージョン**: ap-northeast-1（東京）。他リージョンは使用しない。

### サービス許可リスト

| サービス | 承認された用途 | 制約 |
|---------|-------------|------|
| AWS IoT Core | MQTTデータ受信・Device Shadow管理・IoT Rules Engine | MQTT over TLS必須。デバイス証明書認証。 |
| AWS Lambda | センサーデータ解析・アクション実行・ブラウザアプリへのDevice Shadow状態配信 | Node.js 20.x。30秒タイムアウト。256MB。 |
| Amazon DynamoDB | ユーザー設定・アクションログ・心拍数履歴 | オンデマンドキャパシティ。TTL設定必須。 |
| Amazon S3 | 静的アセットの保存 | パブリックアクセスブロック。CORS設定。 |
| Amazon Bedrock | テキスト生成（LATEアクションの遅刻メッセージ生成） | Claude 3 Haiku。ap-northeast-1。アクセス申請必要。 |
| Amazon Nova 2 Sonic | リアルタイム双方向音声会話 | SLEEP・ANGER・LATEアクション全てで使用。ap-northeast-1での可用性要確認。 |
| Amazon Cognito | ブラウザアプリのユーザー認証 | User Pool + Identity Pool。IoT WebSocket接続に必要。 |
| Amazon SNS | SMS送信（LATEアクション） | ドライバーがNova 2 Sonicとの会話で承認した後に送信。 |
| AWS Secrets Manager | 外部APIキーの管理 | Google・SMS APIキーを集約管理。 |
| AWS CDK | インフラストラクチャ as Code | TypeScript CDK v2。全リソースをコードで定義。 |
| Amazon CloudWatch | ログ・メトリクス・アラーム | 全Lambdaから構造化JSONログを出力。 |

### サービス禁止リスト

| サービス | 理由 | 代替 |
|---------|------|------|
| Amazon EC2 | サーバーレスモデルを優先 | Lambda |
| Amazon ECS / Fargate | リクエスト/レスポンスワークロードには過剰 | Lambda |
| Amazon RDS / Aurora | リレーショナルDBは不要。DynamoDBで対応可能 | DynamoDB |
| Amazon ElastiCache | 計算処理はステートレスで高速なためキャッシュレイヤーは不要 | Lambdaメモリ内キャッシュ（必要な場合） |
| Amazon Kinesis | ストリーミング不要。全処理は同期リクエスト/レスポンス | SQS（非同期処理が必要な場合） |
| Amazon Polly | Nova 2 Sonicが全アクションの音声合成を担当するため不要 | Amazon Nova 2 Sonic |
| LINE Messaging API | ビジネス用途にはSMSが適切 | Amazon SNS |

---

## 推奨技術とパターン

### アーキテクチャパターン

**イベント駆動型サーバーレスアーキテクチャ。**

Pixel Watch → AWS IoT Core → Lambda → Device Shadow → ブラウザアプリ という一方向のイベントフローを基本とします。

| 決定事項 | 選択 | 理由 |
|---------|------|------|
| アーキテクチャスタイル | イベント駆動型 | センサーデータの非同期処理に最適 |
| デプロイモデル | サーバーレス（Lambda） | 運用オーバーヘッドの最小化 |
| 状態管理 | IoT Device Shadow | クラウドとブラウザ間のリアルタイム状態同期 |
| 認証 | Cognito User Pool + Identity Pool | IoT WebSocket接続にSigV4署名が必要 |
| SMS送信 | Amazon SNS | AWSネイティブ。追加サービス不要。 |
| 音声会話（全アクション） | Amazon Nova 2 Sonic | SLEEP・ANGER・LATEの全アクションで統一。低遅延なリアルタイム双方向音声会話。 |

### APIデザイン標準

- **IoT通信**: MQTT over TLS（ポート8883）。トピック命名規則: `carhogo/{device_type}/{device_id}`
- **Device Shadow**: AWS IoT Device Shadow標準スキーマ。`desired`/`reported`/`delta`の3状態管理。
- **Lambda内部**: モジュール間は直接関数呼び出し（マイクロサービス分割なし）。
- **エラーハンドリング**: 外部API失敗時はフォールバック処理を必ず実装。エラーはCloudWatch Logsに構造化JSON形式で出力。

### データパターン

- **主要データストア**: DynamoDB（シングルテーブル設計）
- **DynamoDBエンティティ**: ユーザー設定（PK: user_id）・アクションログ（PK: log_id, SK: timestamp）・心拍数履歴（PK: user_id, SK: timestamp）
- **TTL設定**: アクションログ30日・心拍数履歴1時間
- **シークレット管理**: 全APIキーはSecrets Managerで管理。Lambda環境変数にはARNのみ設定。

### ログパターン

全Lambdaハンドラーから構造化JSONログを出力します。

```javascript
// 標準ログ出力
console.log(JSON.stringify({
  level: 'INFO',
  event: 'action_executed',
  trigger_type: 'SLEEP',
  user_id: userId,
  heart_rate: heartRate,
  duration_ms: elapsed,
  timestamp: new Date().toISOString(),
}));

// エラーログ出力
console.error(JSON.stringify({
  level: 'ERROR',
  event: 'bedrock_generation_failed',
  error_message: err.message,
  action_type: 'LATE',
  timestamp: new Date().toISOString(),
}));
```

---

## セキュリティ要件

### 認証と認可

- **デバイス認証（Pixel Watch）**: X.509証明書によるMQTT over TLS接続。IoT Policyで接続・Publish・Subscribe権限を最小権限で付与。
- **ブラウザアプリ認証**: Cognito User Pool（メール+パスワード）。Identity PoolでAWS Credentialsを取得しIoT WebSocket接続に使用。
- **Lambda実行権限**: IAMロールによる最小権限。各サービスへのアクセスは必要なリソースのみに限定。

### データ保護

- **転送中の暗号化**: 全通信でTLS 1.2以上を使用。MQTT over TLS・HTTPS・WebSocket over TLS。
- **保存時の暗号化**: DynamoDBはAWS管理KMSキーで暗号化。S3はSSE-S3で暗号化。
- **シークレット管理の禁止事項**:
  - ソースコードへのシークレットのハードコード禁止
  - Lambda環境変数への直接シークレット設定禁止（Secrets Manager ARNのみ可）
  - Gitリポジトリへの証明書・APIキーのコミット禁止

### 入力バリデーション

- **MQTTペイロード**: 受信データのJSONスキーマバリデーション必須。不正なペイロードはログ記録後に破棄。
- **心拍数値**: 0〜300 BPMの範囲外の値は異常値として除外。
- **アクショントリガー**: 許可された値（SLEEP・ANGER・LATE）のみ受け付ける。

---

## テスト要件

### テスト戦略概要

| テスト種別 | 必須 | カバレッジ目標 | ツール |
|----------|------|-------------|------|
| ユニットテスト | 必須 | 判定ロジック80%以上 | Jest（バックエンド）・Vitest（フロントエンド） |
| 統合テスト | 必須 | IoT→Lambda→Shadowフロー | mosquitto_pub + CloudWatch Logs確認 |
| エンドツーエンドテスト | 必須 | 全3アクション100%成功 | 手動テスト |

### ユニットテスト標準

- **必須テスト対象**: 生体情報解析ロジック（BiometricAnalyzer）・遅刻判定ロジック（isLate）
- **モック方針**: 外部依存（DynamoDB・Bedrock・Nova 2 Sonic・IoT）はモック化。内部ビジネスロジックはモック化しない。
- **テスト命名規則**: `describe('BiometricAnalyzer') > it('心拍数が15%低下し3分継続した場合にSLEEPを検知する')`

### 統合テスト標準

- **スコープ**: Pixel Watch（またはmosquitto_pub）→ IoT Core → Lambda → Device Shadow → ブラウザアプリの全フロー
- **環境**: デプロイ済みAWS環境（ap-northeast-1）
- **確認チェックリスト**:
  - [ ] SLEEPトリガー → Nova 2 Sonic会話開始（5秒以内）
  - [ ] ANGERトリガー → Nova 2 Sonic会話開始（5秒以内）
  - [ ] LATEトリガー → Bedrockメッセージ生成 → Nova 2 Sonicによる読み上げと確認会話 → SMS送信（3秒以内）
  - [ ] ブラウザアプリ更新（2秒以内）

---

## サンプルコードガイダンス

### サンプルコードの目的

サンプルコードはプロジェクトの**標準パターン**を確立します。AI-DLCがコードを生成する際は、新しいパターンを発明するのではなく、これらのパターンに従います。

### サンプルコードを提供すべき場面

以下のパターンについてサンプルコードを提供します:

- **Lambdaハンドラーパターン**: IoT Ruleからのイベント受信・処理・エラーハンドリング
- **DynamoDBアクセスパターン**: GetItem・PutItem・Queryの標準的な実装
- **IoT Device Shadow更新パターン**: desired状態の更新方法
- **Bedrock呼び出しパターン**: Claude 3 Haikuへのプロンプト送信とフォールバック処理
- **Nova 2 Sonic統合パターン**: リアルタイム音声会話セッションの開始・管理・終了（SLEEP・ANGER・LATE全アクション共通）
  - [AWS公式サンプルコード](https://github.com/aws-samples/amazon-nova-samples/tree/main/speech-to-speech/amazon-nova-2-sonic/repeatable-patterns/nova-sonic-speaks-first)
- **Wear OS MQTTパターン**: Pixel WatchからのMQTT over TLS接続と送信
- **CDK Constructパターン**: 標準的なAWSリソース定義の方法
- **React IoT Shadowフックパターン**: useIoTShadowフックによるリアルタイム状態購読

### AI-DLCによるサンプルコードの使用方法

コード生成時、AI-DLCは以下を行います:

1. **既存パターンを優先**: サンプルが存在するパターンについては、新しいアプローチを発明しない
2. **確立されたパターンに従う**: サンプルに示された構造・命名・エラーハンドリング・テストパターンを踏襲する
3. **コード生成プランでサンプルを参照**: 各ステップに適用されるサンプルを明示する
