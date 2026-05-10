# コンポーネントメソッド定義 — CarHogo

> **注意**: 詳細なビジネスロジック（閾値・アルゴリズム・エラーハンドリング詳細）は CONSTRUCTION フェーズの Functional Design で定義します。

**関連ドキュメント**:
- [コンポーネント定義](./components.md) — 全コンポーネントの責務・属性一覧
- [コンポーネント依存関係](./component-dependency.md) — 依存関係マトリクス・通信パターン・Shadow スキーマ・DynamoDB 設計
- [サービスフロー図](./services.md) — SLEEP/ANGER/LATE アクション別のシーケンス図

---

## BiometricAnalyzerLambda（TypeScript）

### Lambda ハンドラー
```typescript
// IoT Rules Engine から呼び出されるエントリーポイント
export const handler = async (event: IoTBiometricEvent): Promise<void>
```

### BiometricValidator
```typescript
// MQTT ペイロードをバリデーションして型付きオブジェクトに変換
validatePayload(rawPayload: unknown): BiometricReading
// 心拍数が有効範囲（0〜300 BPM）か確認
isValidHeartRate(heartRate: number): boolean
```

### SleepDetector
```typescript
// 心拍数低下が一定割合・一定時間継続しているか判定
detectSleep(current: BiometricReading, baseline: number): boolean
```

### AngerDetector
```typescript
// 心拍数スパイクのパターンを判定
detectAnger(current: BiometricReading, recent: BiometricReading[]): boolean
```

### ShadowPublisher
```typescript
// Device Shadow の desired 状態にアクショントリガーを書き込む
publishActionTrigger(actionType: ActionType, deviceId: string): Promise<void>
// Device Shadow のアクション状態をアイドルに戻す
clearActionState(deviceId: string): Promise<void>
```

### HeartRateRepository
```typescript
// 心拍数データを DynamoDB に保存（TTL: 1時間）
saveReading(reading: BiometricReading): Promise<void>
// 直近N分の平均心拍数（ベースライン）を取得
getBaseline(userId: string, minutesBack: number): Promise<number>
// 直近N件の心拍数履歴を取得
getRecentReadings(userId: string, count: number): Promise<BiometricReading[]>
```

### ActionLogRepository
```typescript
// アクション発火ログを DynamoDB に記録（TTL: 30日）
logActionStart(userId: string, actionType: ActionType): Promise<string>  // actionLogId を返す
logActionEnd(actionLogId: string, resolution: ActionResolution): Promise<void>
// 直近10件のアクション履歴を取得
getRecentActions(userId: string, limit: number): Promise<ActionLog[]>
```

---

## ActionExecutorLambda（TypeScript）

### Lambda ハンドラー
```typescript
// EventBridge Scheduler または手動テストトリガーから呼び出されるエントリーポイント
export const handler = async (event: ActionTriggerEvent): Promise<void>
```

### CalendarService
```typescript
// Google Calendar から次の予定（開始時刻・場所フィールド）を取得
getNextMeeting(calendarId: string, accessToken: string): Promise<Meeting | null>
```

### EtaService
```typescript
// Google Maps Directions API で ETA（分）を計算
calculateEtaMinutes(origin: LatLng, destinationAddress: string, apiKey: string): Promise<number>
```

### MessageGenerator
```typescript
// Bedrock Claude 3 Haiku で遅刻メッセージを生成（失敗時はフォールバックテキストを返す）
generateLateMessage(minutesLate: number, meetingTitle: string): Promise<string>
// フォールバックメッセージを返す（Bedrock 未承認または失敗時）
getFallbackMessage(minutesLate: number): string
```

### SmsService
```typescript
// Amazon SNS で SMS を送信
sendSms(phoneNumber: string, message: string): Promise<void>
```

---

## Pixel Watch コンポーネント（Kotlin）

### AppState（Kotlin Singleton object）
```kotlin
// 接続状態（STOPPED / CONNECTING / MONITORING / RECONNECTING / ERROR）
val connectionState: MutableStateFlow<ConnectionState>
// 最新の生体データ
val lastBiometricData: MutableStateFlow<BiometricData?>
// 再接続試行カウント
val retryCount: MutableStateFlow<Int>
// 状態を初期値にリセット
fun reset()
```

### WatchConfigLoader
```kotlin
// assets/config.properties を読み込んで WatchConfig を返す
fun load(context: Context): WatchConfig
```

### CertificateLoader
```kotlin
// res/raw/ の証明書ファイルから SSLContext を生成
fun buildSslContext(context: Context): SSLContext
```

### BiometricCollector
```kotlin
// Health Services でセッションを開始し、生体データを Flow として流す
fun startCollection(): Flow<BiometricData>
// 収集セッションを停止する
suspend fun stopCollection()
```

### MqttManager
```kotlin
// MQTT over TLS 接続を確立する
suspend fun connect(config: WatchConfig, sslContext: SSLContext)
// トピックにメッセージを送信する（QoS 1）
suspend fun publish(topic: String, payload: String)
// 接続を切断する
fun disconnect()
// 再接続バックオフ遅延を計算する（ms）
fun calculateBackoffDelay(retryCount: Int): Long
```

### MonitoringService
```kotlin
// サービス開始：設定読み込み → MQTT 接続 → 生体データ収集ループ開始
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int
// サービス停止：収集停止 → MQTT 切断 → 状態リセット
fun stopMonitoring()
```

### MainActivity
```kotlin
// 開始ボタンタップ：MonitoringService を Foreground で開始
fun onStartButtonClick()
// 終了ボタンタップ：確認ダイアログを表示
fun onStopButtonClick()
// 確認ダイアログ OK：MonitoringService に停止 Intent を送信
fun confirmStop()
// AppState の Flow を購読して UI を更新
fun observeState()
```

---

## ブラウザ React アプリ（TypeScript）

### AuthService（Singleton）
```typescript
// Cognito User Pool でメール+パスワード認証
signIn(email: string, password: string): Promise<void>
// サインアウト
signOut(): void
// Identity Pool から SigV4 用 AWS Credentials を取得
getAwsCredentials(): Promise<AWSCredentials>
// 現在のセッションユーザー情報を取得
getCurrentUser(): CognitoUser | null
```

### IoTShadowService（Singleton）
```typescript
// SigV4 署名付き WebSocket で IoT Core に接続
connect(credentials: AWSCredentials, endpoint: string): Promise<void>
// Device Shadow の delta を購読（アクション状態変化を検知）
subscribeToShadow(deviceId: string, onDelta: (delta: ShadowDelta) => void): void
// WebSocket 接続を切断
disconnect(): void
```

### NovaSessionService（Singleton）
```typescript
// アクション毎に Nova 2 Sonic WebSocket セッションを開始（SigV4 署名）
startSession(actionType: ActionType, context: SessionContext): Promise<void>
// マイクからキャプチャした音声チャンクを送信
sendAudio(audioChunk: ArrayBuffer): void
// セッションを終了してリソースを解放
endSession(): void
// 音声データ受信コールバック
onAudioReceived: ((audio: ArrayBuffer) => void) | null
// セッション状態変化コールバック
onStateChange: ((state: SessionState) => void) | null
```

### Custom Hooks
```typescript
// 認証状態と操作を提供
function useAuth(): { user: CognitoUser | null; signIn: ...; signOut: ... }
// Device Shadow delta をリアクティブに購読
function useIoTShadow(deviceId: string): ShadowState
// Nova 2 Sonic セッション制御と状態を提供
function useNovaSession(): { sessionState: SessionState; startSession: ...; endSession: ... }
```

---

## Hooks × コンポーネント 対応 Matrix

各 React コンポーネントが使用する Hook と、そのコンポーネントでの具体的な利用目的を示します。

| コンポーネント | `useAuth` | `useIoTShadow` | `useNovaSession` |
|-------------|:---------:|:--------------:|:----------------:|
| `App` | ✓ | — | — |
| `LoginPage` | ✓ | — | — |
| `Dashboard` | — | ✓ | — |
| `BiometricDisplay` | — | ✓ | — |
| `ActionStatusDisplay` | — | ✓ | — |
| `NovaConversationUI` | — | ✓ | ✓ |
| `ActionHistoryList` | — | ✓ | — |

### 各セルの利用目的詳細

#### `App` — useAuth
- `user` の null チェックで認証済み/未認証を判定
- 未認証 → `<LoginPage />` を描画、認証済み → `<Dashboard />` を描画

#### `LoginPage` — useAuth
- `signIn(email, password)` を呼び出してログイン処理を実行
- エラー状態（認証失敗メッセージ）の表示

#### `Dashboard` — useIoTShadow
- `shadowState.actionType`（IDLE / SLEEP / ANGER / LATE）に応じて `<NovaConversationUI />` の表示/非表示を切り替え
- 子コンポーネントへ `shadowState` を props として配布するレイアウト制御

#### `BiometricDisplay` — useIoTShadow
- `shadowState.heartRate`・`shadowState.hrv` をリアルタイムグラフ（recharts）へバインド
- 接続状態（`shadowState.connected`）に応じてローディング表示を切り替え

#### `ActionStatusDisplay` — useIoTShadow
- `shadowState.actionType` をバッジ表示（IDLE / SLEEP中 / ANGER中 / LATE中）
- `shadowState.actionStartedAt` でアクション継続時間を計算・表示

#### `NovaConversationUI` — useIoTShadow + useNovaSession
- `useIoTShadow`: `shadowState.actionType` でどのアクション（SLEEP/ANGER/LATE）が発動中かを取得し、Nova 2 Sonic への初期プロンプトを決定
- `useNovaSession`: `startSession(prompt)`・`endSession()` で音声会話を開始/終了。`sessionState`（IDLE / CONNECTING / ACTIVE / ENDING）でマイクボタンの活性化制御

#### `ActionHistoryList` — useIoTShadow
- `shadowState.recentActions`（直近10件のアクションログ）を一覧表示
- 種別・開始時刻・継続時間を整形して表示
