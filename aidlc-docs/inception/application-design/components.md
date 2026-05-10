# コンポーネント定義 — CarHogo

## Lambda バックエンド（TypeScript / Node.js 20.x）

### BiometricAnalyzerLambda
| 項目 | 内容 |
|------|------|
| **トリガー** | AWS IoT Rules Engine（MQTT メッセージ受信時） |
| **責務** | 受信した生体データのバリデーション・SLEEP/ANGER パターン検知・Device Shadow 更新・DynamoDB 書き込み |

**内部モジュール:**
| モジュール | 責務 |
|----------|------|
| `BiometricValidator` | MQTT ペイロードのスキーマバリデーション・心拍数値の範囲チェック（0〜300 BPM） |
| `SleepDetector` | 心拍数ベースラインとの差分・継続時間からSLEEPパターンを判定 / 回復判定（ベースライン復帰検知） |
| `AngerDetector` | 心拍数スパイク速度からANGERパターンを判定 / 回復判定（心拍数安定検知） |
| `ShadowPublisher` | IoT Device Shadow の desired 状態を更新してアクションをブラウザへ通知 |
| `HeartRateRepository` | 心拍数履歴を DynamoDB に書き込み・ベースライン値を取得 |
| `ActionLogRepository` | アクション発火ログを DynamoDB に記録 |

---

### ActionExecutorLambda
| 項目 | 内容 |
|------|------|
| **トリガー** | EventBridge Scheduler（定期実行：LATEアクション）／開発者向け手動トリガー（全アクション） |
| **責務** | LATEアクション実行（Google Calendar・Maps・Bedrock・SNS連携）・手動テストトリガー処理・Device Shadow 更新 |

**内部モジュール:**
| モジュール | 責務 |
|----------|------|
| `CalendarService` | Google Calendar API から次の予定（開始時刻・場所）を取得（OAuth 2.0） |
| `EtaService` | Google Maps Directions API で現在地→目的地のETAを計算 |
| `MessageGenerator` | Amazon Bedrock（Claude 3 Haiku）で遅刻謝罪メッセージを生成。Bedrock 未承認時はフォールバックテキスト使用 |
| `SmsService` | Amazon SNS で SMS を送信（サンドボックス環境） |
| `ShadowPublisher` | IoT Device Shadow 更新（BiometricAnalyzerLambda と共有モジュール） |
| `ActionLogRepository` | アクションログ書き込み（BiometricAnalyzerLambda と共有モジュール） |

---

## CDK インフラ（TypeScript CDK v2）

### CarhogoStack（シングルスタック）
| 項目 | 内容 |
|------|------|
| **責務** | 全 AWS リソースをコードとして定義・管理 |

**定義リソース:**
| リソース | 用途 |
|---------|------|
| IoT Thing / Certificate / Policy | Pixel Watch の MQTT 認証 |
| IoT Rules Engine | 受信データを BiometricAnalyzerLambda へルーティング |
| IoT Device Shadow | クラウドとブラウザ間のリアルタイム状態同期 |
| Lambda×2（Node.js 20.x） | BiometricAnalyzerLambda・ActionExecutorLambda |
| DynamoDB（シングルテーブル） | UserConfig・HeartRateHistory・ActionLog |
| Cognito User Pool + Identity Pool | ブラウザ認証・IoT WebSocket 用 AWS Credentials |
| S3 + CloudFront | React アプリ静的ホスティング |
| Amazon SNS | SMS 送信 |
| EventBridge Scheduler | ActionExecutorLambda の定期実行 |
| Secrets Manager | Google API キー・IoT 証明書 ARN 管理 |

---

## Pixel Watch Wear OS アプリ（Kotlin 1.9 / Wear OS API 33）

### AppState
| 項目 | 内容 |
|------|------|
| **種別** | Singleton object（Kotlin） |
| **責務** | アプリ全体の状態を StateFlow で管理（接続状態・最新生体データ・再接続カウント） |

### WatchConfigLoader
| 項目 | 内容 |
|------|------|
| **責務** | `assets/config.properties` から IoT エンドポイント・デバイスID・ユーザーIDを読み込む |

### CertificateLoader
| 項目 | 内容 |
|------|------|
| **責務** | `res/raw/` の X.509 証明書ファイルから SSLContext を構築する |

### BiometricCollector
| 項目 | 内容 |
|------|------|
| **責務** | Health Services API（ExerciseClient）から心拍数・HRV を収集し Flow<BiometricData> として提供する |

### MqttManager
| 項目 | 内容 |
|------|------|
| **責務** | AWS IoT SDK を使用した MQTT over TLS 接続・データ送信・指数バックオフ再接続 |

### MonitoringService
| 項目 | 内容 |
|------|------|
| **種別** | Foreground Service |
| **責務** | BiometricCollector の起動・MqttManager の接続・生体データの継続送信・LifecycleScope 管理 |

### MainActivity
| 項目 | 内容 |
|------|------|
| **種別** | ComponentActivity（単一Activity） |
| **責務** | 開始/終了ボタン UI・ステータス/BPM/HRV 表示・AmbientLifecycleObserver 対応 |

---

## ブラウザ React アプリ（TypeScript / React 18）

### Services（Singleton クラス）

| コンポーネント | 責務 |
|-------------|------|
| `AuthService` | Cognito User Pool 認証・Identity Pool での AWS Credentials 取得・セッション管理 |
| `IoTShadowService` | SigV4 署名付き WebSocket で IoT Device Shadow に接続・delta 購読・状態同期 |
| `NovaSessionService` | Nova 2 Sonic への WebSocket 直接接続・マイク音声送信・スピーカー音声受信・アクション毎の新規セッション管理 |

### Custom Hooks

| フック | 責務 |
|-------|------|
| `useAuth` | AuthService をラップ。認証状態・ログイン/ログアウト操作を提供 |
| `useIoTShadow` | IoTShadowService をラップ。Device Shadow delta をリアクティブに購読 |
| `useNovaSession` | NovaSessionService をラップ。セッション状態・音声入出力制御を提供 |

### React Components

| コンポーネント | 責務 |
|-------------|------|
| `App` | ルートコンポーネント。認証状態に応じて LoginPage または Dashboard を表示 |
| `LoginPage` | Cognito User Pool のメール+パスワード認証画面 |
| `Dashboard` | 全ウィジェットのレイアウトを管理するメイン画面 |
| `BiometricDisplay` | 心拍数・HRV をリアルタイム表示（recharts グラフ含む） |
| `ActionStatusDisplay` | 現在のアクション状態（IDLE / SLEEP / ANGER / LATE）をバッジ表示 |
| `NovaConversationUI` | マイク ON/OFF ボタン・会話状態インジケーター・Nova 2 Sonic 音声入出力 UI |
| `ActionHistoryList` | 直近10件のアクション履歴（種別・開始時刻・継続時間）を一覧表示 |

---

**関連ドキュメント**:
- [コンポーネントメソッド](./component-methods.md) — 全メソッドシグネチャ・型定義
- [コンポーネント依存関係](./component-dependency.md) — 依存関係マトリクス・通信パターン・Shadow スキーマ・DynamoDB 設計
- [サービスフロー図](./services.md) — SLEEP/ANGER/LATE アクション別のシーケンス図
- [アプリケーション設計 統合サマリー](./application-design.md) — システム全体のアーキテクチャ概要
