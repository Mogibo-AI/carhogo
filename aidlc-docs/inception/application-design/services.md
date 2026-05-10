# サービス定義・フロー図 — CarHogo

## アーキテクチャ概要

CarHogoは**イベント駆動型サーバーレスアーキテクチャ**を採用します。
基本フロー: `Pixel Watch → IoT Core → Lambda → Device Shadow → ブラウザアプリ`

---

## SLEEPアクション フロー

```mermaid
sequenceDiagram
    autonumber
    participant W as Pixel Watch
    participant IoT as AWS IoT Core
    participant L as BiometricAnalyzerLambda
    participant S as Device Shadow
    participant B as ブラウザ
    participant Nova as Nova 2 Sonic

    Note over W,Nova: SLEEP 検知・音声会話開始
    W->>IoT: MQTT/TLS (心拍数・HRV)
    IoT->>L: IoT Rule トリガー
    L->>L: BiometricValidator.validatePayload()
    L->>L: HeartRateRepository.saveReading()
    L->>L: HeartRateRepository.getBaseline()
    L->>L: SleepDetector.detectSleep() ✓
    L->>L: ActionLogRepository.logActionStart()
    L->>S: ShadowPublisher.publishActionTrigger("SLEEP")
    S-->>B: WebSocket delta (SigV4)
    B->>Nova: NovaSessionService.startSession("SLEEP")
    Nova-->>B: 双方向音声会話 (マイク↔スピーカー)

    Note over W,Nova: SLEEP 終了 (心拍数回復) — Lambda 主導で判定
    W->>IoT: 次の心拍数データ (回復値)
    IoT->>L: IoT Rule トリガー
    L->>S: ShadowPublisher.getCurrentActionState() → "SLEEP"
    L->>L: SleepDetector.detectRecovery() ✓
    L->>S: ShadowPublisher.clearActionState() (action=IDLE)
    L->>L: ActionLogRepository.logActionEnd()
    S-->>B: WebSocket delta (IDLE)
    B->>Nova: NovaSessionService.endSession()
```

**回復判定の責務**: 会話継続中も Pixel Watch は通常通り心拍数データを送信し続け、各データを **BiometricAnalyzerLambda が受信時に Shadow から現在のアクション状態を取得**して判定する。SLEEP/ANGER 状態であれば `detectRecovery()` を呼び出し、回復が確認できれば Shadow を `IDLE` に戻す。ブラウザは Shadow delta を受信して自動的にセッションを終了する。

---

## ANGERアクション フロー

```mermaid
sequenceDiagram
    autonumber
    participant W as Pixel Watch
    participant IoT as AWS IoT Core
    participant L as BiometricAnalyzerLambda
    participant S as Device Shadow
    participant B as ブラウザ
    participant Nova as Nova 2 Sonic

    Note over W,Nova: ANGER 検知・音声会話開始
    W->>IoT: MQTT/TLS (心拍数スパイク)
    IoT->>L: IoT Rule トリガー
    L->>L: BiometricValidator.validatePayload()
    L->>L: HeartRateRepository.saveReading()
    L->>L: HeartRateRepository.getRecentReadings()
    L->>L: AngerDetector.detectAnger() ✓
    L->>L: ActionLogRepository.logActionStart()
    L->>S: ShadowPublisher.publishActionTrigger("ANGER")
    S-->>B: WebSocket delta (SigV4)
    B->>Nova: NovaSessionService.startSession("ANGER")
    Nova-->>B: 双方向音声会話 (共感的トーン)

    Note over W,Nova: ANGER 終了 (心拍数安定) — Lambda 主導で判定
    W->>IoT: 次の心拍数データ
    IoT->>L: IoT Rule トリガー
    L->>S: ShadowPublisher.getCurrentActionState() → "ANGER"
    L->>L: AngerDetector.detectRecovery() ✓
    L->>S: ShadowPublisher.clearActionState() (action=IDLE)
    L->>L: ActionLogRepository.logActionEnd()
    S-->>B: WebSocket delta (IDLE)
    B->>Nova: NovaSessionService.endSession()
```

---

## LATEアクション フロー

```mermaid
sequenceDiagram
    autonumber
    participant Sched as EventBridge Scheduler
    participant L as ActionExecutorLambda
    participant GCal as Google Calendar
    participant GMap as Google Maps
    participant Bed as Amazon Bedrock
    participant S as Device Shadow
    participant B as ブラウザ
    participant Nova as Nova 2 Sonic
    participant SNS as Amazon SNS

    Note over Sched,SNS: LATE 検知・メッセージ生成・読み上げ
    Sched->>L: 定期実行トリガー (または手動テスト)
    L->>GCal: CalendarService.getNextMeeting()
    L->>GMap: EtaService.calculateEtaMinutes()
    Note over L: ETA > 予定開始時刻 ならば続行
    L->>Bed: MessageGenerator.generateLateMessage()
    Note over L,Bed: 失敗時は getFallbackMessage()
    L->>L: ActionLogRepository.logActionStart()
    L->>S: ShadowPublisher.publishActionTrigger("LATE", message, phoneNumber)
    S-->>B: WebSocket delta (lateMessage 含む)
    B->>Nova: NovaSessionService.startSession("LATE", lateMessage)
    Nova-->>B: メッセージ読み上げ + 送信確認

    Note over B,SNS: ドライバー承認後
    B-->>Nova: 「はい、送って」(音声承認)
    B->>L: 送信承認 (HTTP POST または Shadow 経由 / 詳細は Construction で確定)
    L->>SNS: SmsService.sendSms(phoneNumber, message)
    L->>S: ShadowPublisher.clearActionState()
    L->>L: ActionLogRepository.logActionEnd()
    S-->>B: WebSocket delta (IDLE)
```

---

## 共有サービスモジュール（Lambda 間）

以下のモジュールは BiometricAnalyzerLambda と ActionExecutorLambda の両方が使用します。
`backend/shared/` ディレクトリに配置し、各 Lambda が import します。

| モジュール | 配置 |
|----------|------|
| `ShadowPublisher` | `backend/shared/shadowPublisher.ts` |
| `ActionLogRepository` | `backend/shared/actionLogRepository.ts` |
| `types.ts`（共通型定義） | `backend/shared/types.ts` |
| `logger.ts`（構造化ログ） | `backend/shared/logger.ts` |

---

## ブラウザ サービス層

```
React コンポーネント
  └─ Custom Hooks（useAuth / useIoTShadow / useNovaSession）
       └─ Singleton Services（AuthService / IoTShadowService / NovaSessionService）
            └─ AWS SDK v3（Cognito / IoT / Bedrock）
```

**サービス初期化フロー（アプリ起動時）:**
```
1. AuthService.signIn() → Cognito 認証
2. AuthService.getAwsCredentials() → Identity Pool で Credentials 取得
3. IoTShadowService.connect() → IoT Core WebSocket 接続確立
4. IoTShadowService.subscribeToShadow() → Shadow delta 購読開始
（NovaSessionService はアクション毎に startSession() / endSession()）
```

---

**関連ドキュメント**:
- [コンポーネント定義](./components.md) — 全コンポーネントの責務・属性一覧
- [コンポーネントメソッド](./component-methods.md) — 全メソッドシグネチャ・型定義
- [コンポーネント依存関係](./component-dependency.md) — 依存関係マトリクス・通信パターン・Shadow スキーマ
- [アプリケーション設計 統合サマリー](./application-design.md) — システム全体のアーキテクチャ概要
