# 要件定義書 — CarHogo

## インテント分析サマリー

| 項目 | 内容 |
|------|------|
| **ユーザーリクエスト** | CarHogoドライビングセーフティサービスのグリーンフィールド新規開発。Pixel WatchとAWSを連携し、居眠り・怒り・遅刻ストレスをAmazon Nova 2 Sonicによるリアルタイム音声会話で解消する。 |
| **リクエストタイプ** | 新規プロジェクト（グリーンフィールド） |
| **スコープ推定** | クロスシステム（Wear OS・AWS バックエンド・ブラウザアプリの複数システム） |
| **複雑度推定** | Complex（Amazon Nova 2 Sonic リアルタイム WebSocket・IoT・Google API・SMS の複数統合点） |

---

## 機能要件

### FR-01: IoT データパイプライン

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-01-01 | Pixel Watch から AWS IoT Core へ MQTT over TLS（ポート8883）でリアルタイム生体データを送信する | 必須 |
| FR-01-02 | AWS IoT Rules Engine でセンサーデータを Lambda へルーティングする | 必須 |
| FR-01-03 | AWS IoT Device Shadow でデバイス状態を管理し、ブラウザアプリとリアルタイム同期する | 必須 |
| FR-01-04 | MQTT トピック命名規則: `carhogo/{device_type}/{device_id}` | 必須 |
| FR-01-05 | センサーからクラウドまでサブ秒レイテンシを達成する | 必須 |

### FR-02: 生体情報解析

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-02-01 | 心拍数を継続モニタリングし、0〜300 BPM 範囲外の値を異常値として除外する | 必須 |
| FR-02-02 | HRV（心拍変動・RMSSD）をリアルタイムにモニタリングする | 必須 |
| FR-02-03 | ドライバーごとのベースライン心拍数を DynamoDB に保持する | 必須 |
| FR-02-04 | SLEEPパターン検知: 心拍数がベースラインから一定割合低下し一定時間継続した場合にトリガー | 必須 |
| FR-02-05 | ANGERパターン検知: 心拍数が短時間で急激にスパイクした場合にトリガー（加速度センサーは初期版では固定値を使用） | 必須 |
| FR-02-06 | 開発者向け手動トリガー機能（SLEEP・ANGER・LATE を任意に発火できるテスト機能） | 必須 |

### FR-03: SLEEPアクション（居眠り防止）

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-03-01 | 居眠り検知トリガーから **5秒以内** に Amazon Nova 2 Sonic による音声会話を開始する | 必須 |
| FR-03-02 | 状況に応じて動的に生成される会話トピック（ニュース・天気・クイズ・雑談など）で会話を開始する | 必須 |
| FR-03-03 | ドライバーの心拍数がベースラインに回復するまで会話を継続する | 必須 |
| FR-03-04 | 会話終了時にブラウザアプリの UI を更新する | 必須 |

### FR-04: ANGERアクション（怒り・ストレス緩和）

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-04-01 | 怒り検知トリガーから **5秒以内** に Amazon Nova 2 Sonic による音声会話を開始する | 必須 |
| FR-04-02 | ドライバーの感情を受け止め・肯定し・注意を転換する共感的な会話スタイルを使用する | 必須 |
| FR-04-03 | ドライバーの心拍数が安定するまで会話を継続する | 必須 |
| FR-04-04 | 会話終了時にブラウザアプリの UI を更新する | 必須 |

### FR-05: LATEアクション（遅刻連絡）

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-05-01 | Google Calendar API（OAuth 2.0 認証）でドライバーの次の予定（開始時刻・場所）を取得する | 必須 |
| FR-05-02 | Google Maps Directions API で現在地から目的地へのETAを継続計算する | 必須 |
| FR-05-03 | ETAが予定開始時刻を超過した時点で自動検知し LATEアクションをトリガーする | 必須 |
| FR-05-04 | Amazon Bedrock（Claude 3 Haiku）でビジネス向け遅刻謝罪メッセージを自動生成する。Bedrock が未承認の場合はフォールバックテキストライブラリを使用する | 必須 |
| FR-05-05 | Nova 2 Sonic が生成メッセージを音声で読み上げ、ドライバーに送信可否を確認する会話を行う | 必須 |
| FR-05-06 | ドライバーが承認した場合、Amazon SNS（サンドボックス環境）で SMS を送信する | 必須 |
| FR-05-07 | ドライバーはスマートフォンに触れることなく全操作を完了できる | 必須 |

**前提制約（LATE）**:
- Google API（Maps・Calendar）は開発中にセットアップを実施する
- Bedrock アクセスは申請済み・承認待ち。承認前はフォールバック実装で動作する
- SMS 送信先は SNS サンドボックス登録済み番号に限定（開発・テスト段階）

### FR-06: Amazon Nova 2 Sonic 音声会話インフラ

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-06-01 | ブラウザから Nova 2 Sonic へ **直接 WebSocket 接続**する（Cognito Identity Pool の AWS Credentials を使用し SigV4 署名） | 必須 |
| FR-06-02 | ブラウザのマイク（getUserMedia API）を音声入力として使用する | 必須 |
| FR-06-03 | ブラウザのスピーカー/ヘッドフォンを音声出力として使用する | 必須 |
| FR-06-04 | ap-northeast-1（東京）リージョンで動作する | 必須 |
| FR-06-05 | SLEEP・ANGER・LATE の全アクションで共通の Nova 2 Sonic セッション管理ロジックを使用する | 必須 |

### FR-07: CarHogo ブラウザアプリ（React）

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-07-01 | リアルタイム生体情報（心拍数・HRV）をブラウザで表示する | 必須 |
| FR-07-02 | システムステータスとアクティブアクション（SLEEP・ANGER・LATE）の状態を表示する | 必須 |
| FR-07-03 | Nova 2 Sonic 音声会話 UI（マイク ON/OFF・会話状態表示）を全アクションで共通提供する | 必須 |
| FR-07-04 | アクション履歴とログを表示する | 必須 |
| FR-07-05 | Amazon S3 + CloudFront で静的サイトとしてホスティングする | 必須 |
| FR-07-06 | Cognito User Pool によるメール+パスワード認証を実装する | 必須 |
| FR-07-07 | IoT Device Shadow の WebSocket 購読によりリアルタイムに状態更新を受信する | 必須 |
| FR-07-08 | 生体情報トリガーから **2秒以内** にブラウザアプリの表示が更新される | 必須 |

### FR-08: Pixel Watch Wear OS アプリ（Kotlin）

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-08-01 | Health Services API で心拍数（BPM）と HRV（RMSSD）をリアルタイム取得する | 必須 |
| FR-08-02 | AWS IoT SDK を使用して MQTT over TLS（ポート8883）で IoT Core へ送信する | 必須 |
| FR-08-03 | 「開始」ボタンでモニタリングを開始し、Foreground Service でバックグラウンド動作を継続する | 必須 |
| FR-08-04 | 「終了」ボタン（確認ダイアログあり）でモニタリングを停止する | 必須 |
| FR-08-05 | ステータス・BPM・HRV をウォッチ画面に表示する | 必須 |
| FR-08-06 | X.509 証明書による MQTT 認証を実装する | 必須 |
| FR-08-07 | 接続断時は指数バックオフで自動再接続する | 必須 |
| FR-08-08 | アンビエントモード対応（画面消灯中もバックグラウンドで動作を継続） | 必須 |

### FR-09: AWS CDK インフラ（TypeScript）

| ID | 要件 | 優先度 |
|----|------|--------|
| FR-09-01 | 全 AWS リソースを TypeScript CDK v2 でコードとして定義する | 必須 |
| FR-09-02 | デプロイターゲット: ap-northeast-1（東京）、シングル AWS アカウント、シングルスタック | 必須 |
| FR-09-03 | IoT Thing・証明書・ポリシー・Rules Engine・Device Shadow を CDK で定義する | 必須 |
| FR-09-04 | Lambda 関数・DynamoDB・S3・CloudFront・Cognito・SNS・Secrets Manager を CDK で定義する | 必須 |
| FR-09-05 | Gradle カスタムタスク（`injectIoTConfig`）で IoT エンドポイントを Pixel Watch ビルドに自動注入する | 必須 |

---

## 非機能要件

### NFR-01: パフォーマンス

| ID | 要件 | 測定方法 |
|----|------|---------|
| NFR-01-01 | SLEEP/ANGERトリガーから Nova 2 Sonic 音声会話開始まで **5秒以内** | CloudWatch Logs による計測 |
| NFR-01-02 | 生体トリガーからブラウザアプリ表示更新まで **2秒以内** | CloudWatch Logs による計測 |
| NFR-01-03 | LATEトリガーから SMS 送信完了まで **10秒以内** | CloudWatch Logs による計測 |
| NFR-01-04 | MQTT センサーデータのクラウドへの到達: サブ秒 | IoT Core ログによる計測 |

### NFR-02: 可用性・信頼性

| ID | 要件 |
|----|------|
| NFR-02-01 | システム稼働率 99.9% 以上 |
| NFR-02-02 | 全 Lambda から CloudWatch への構造化 JSON ログ出力 |
| NFR-02-03 | 外部 API 失敗時のフォールバック処理必須（Bedrock・Google API・SNS） |
| NFR-02-04 | MQTT 接続断時の自動再接続（指数バックオフ） |

### NFR-03: セキュリティ（基本セキュリティ・拡張拡張なし）

| ID | 要件 |
|----|------|
| NFR-03-01 | MQTT 通信は TLS 1.2 以上（X.509 デバイス証明書認証） |
| NFR-03-02 | ブラウザ認証: Cognito User Pool（メール+パスワード） |
| NFR-03-03 | AWS 認証情報: Cognito Identity Pool で SigV4 署名用 Credentials 取得 |
| NFR-03-04 | Lambda IAM ロールは最小権限 |
| NFR-03-05 | 全 API キー・証明書は AWS Secrets Manager で管理（ソースコードへのハードコード禁止） |
| NFR-03-06 | DynamoDB: AWS 管理 KMS キーで暗号化 |
| NFR-03-07 | S3: パブリックアクセスブロック・SSE-S3 暗号化 |

### NFR-04: テスト

| ID | 要件 |
|----|------|
| NFR-04-01 | 生体情報判定ロジックのユニットテスト（Jest/Vitest）: 80% 以上カバレッジ |
| NFR-04-02 | IoT→Lambda→Device Shadow フローの統合テスト |
| NFR-04-03 | 全3アクション（SLEEP・ANGER・LATE）の E2E 手動テスト |
| NFR-04-04 | PBT 拡張: 適用しない（シンプルさを優先） |

**テストツール**: Jest（バックエンド）・Vitest（フロントエンド）・Kotest（Wear OS）・mosquitto_pub（IoT 統合テスト）

### NFR-05: 開発・運用制約

| ID | 制約 | 対応方針 |
|----|------|---------|
| NFR-05-01 | 開発環境はほぼゼロからセットアップが必要 | Build and Test フェーズで詳細なセットアップ手順を提供する |
| NFR-05-02 | Amazon Bedrock（Claude 3 Haiku）は申請済み・承認待ち | フォールバックテキストライブラリをコードに実装し、承認後に切り替え可能にする |
| NFR-05-03 | Google API（Maps・Calendar）は開発中にセットアップ | Lambda 内でスタブ/モックを用意し、API キー取得後に差し替え可能にする |
| NFR-05-04 | SNS は サンドボックス環境（検証済み番号のみ SMS 送信可能） | 開発・テスト中は登録済み番号への送信のみで検証する |
| NFR-05-05 | デバイス: Pixel Watch 実機でテスト | 実機接続の手順を Build and Test フェーズで提供する |

---

## 統合ポイント一覧

| サービス | 用途 | 状態 |
|---------|------|------|
| Amazon Nova 2 Sonic | SLEEP・ANGER・LATE 全アクションの音声会話 | ap-northeast-1 確認済み |
| Amazon Bedrock（Claude 3 Haiku） | LATE アクションの遅刻メッセージ生成 | 申請済み・承認待ち |
| AWS IoT Core | MQTT データ受信・Device Shadow 管理 | 利用可能 |
| Amazon DynamoDB | ユーザー設定・アクションログ・心拍数履歴 | 利用可能 |
| Amazon Cognito | ブラウザ認証・IoT WebSocket 認証情報 | 利用可能 |
| Amazon SNS | SMS 自動送信（ドライバー承認後） | サンドボックス環境 |
| Amazon S3 + CloudFront | React アプリの静的ホスティング | 利用可能 |
| AWS Secrets Manager | Google API キー・証明書管理 | 利用可能 |
| AWS Lambda | センサー解析・アクション実行 | Node.js 20.x |
| AWS CDK v2 | 全インフラのコード定義 | TypeScript |
| Google Maps Directions API | ETA 計算 | 開発中にセットアップ |
| Google Calendar API | 予定取得（OAuth 2.0） | 開発中にセットアップ |

---

## MVPの成功基準

- [ ] SLEEP・ANGER・LATE の3アクションが生体信号または手動トリガーで正常に発動する
- [ ] SLEEP/ANGER トリガーから5秒以内に Nova 2 Sonic 音声会話が開始する
- [ ] LATE トリガー時に Nova 2 Sonic が Bedrock 生成メッセージを読み上げ、承認後 SMS を送信する
- [ ] 生体トリガーから2秒以内にブラウザアプリが更新される
- [ ] システムが安定して継続動作する

---

## 拡張機能設定

| 拡張機能 | 状態 | 決定根拠 |
|---------|------|---------|
| Security Baseline | **無効** | PoC・プロトタイプフェーズ優先 |
| Property-Based Testing | **無効** | シンプルさを優先 |
