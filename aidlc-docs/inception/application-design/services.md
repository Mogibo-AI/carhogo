# サービス定義・フロー図 — CarHogo

## アーキテクチャ概要

CarHogoは**イベント駆動型サーバーレスアーキテクチャ**を採用します。
基本フロー: `Pixel Watch → IoT Core → Lambda → Device Shadow → ブラウザアプリ`

---

## SLEEPアクション フロー

```
Pixel Watch（BiometricCollector）
  │ MQTT over TLS（心拍数・HRV）
  ▼
AWS IoT Core（IoT Rules Engine）
  │ IoT Rule トリガー
  ▼
BiometricAnalyzerLambda
  ├─ BiometricValidator.validatePayload()
  ├─ HeartRateRepository.saveReading()
  ├─ HeartRateRepository.getBaseline()
  ├─ SleepDetector.detectSleep()  ← SLEEPパターン検知
  ├─ ActionLogRepository.logActionStart()
  └─ ShadowPublisher.publishActionTrigger("SLEEP")
       │
       ▼
  IoT Device Shadow（desired.action = "SLEEP"）
       │ WebSocket delta
       ▼
  ブラウザ（IoTShadowService → useIoTShadow → ActionStatusDisplay）
       │ アクション検知
       ▼
  NovaSessionService.startSession("SLEEP")
       │ WebSocket（SigV4署名）
       ▼
  Amazon Nova 2 Sonic（ap-northeast-1）
       │ 双方向音声会話
       ▼
  ブラウザ（NovaConversationUI）← マイク入力 / スピーカー出力
```

**SLEEPアクション終了:**
```
ブラウザ（心拍数回復を Device Shadow で検知）
  → NovaSessionService.endSession()
  → ActionExecutorLambda（または BiometricAnalyzerLambda）: ShadowPublisher.clearActionState()
  → ActionLogRepository.logActionEnd()
```

---

## ANGERアクション フロー

```
Pixel Watch（BiometricCollector）
  │ MQTT over TLS（心拍数スパイク）
  ▼
AWS IoT Core（IoT Rules Engine）
  │
  ▼
BiometricAnalyzerLambda
  ├─ BiometricValidator.validatePayload()
  ├─ HeartRateRepository.saveReading()
  ├─ HeartRateRepository.getRecentReadings()
  ├─ AngerDetector.detectAnger()  ← ANGERパターン検知
  ├─ ActionLogRepository.logActionStart()
  └─ ShadowPublisher.publishActionTrigger("ANGER")
       │
       ▼
  IoT Device Shadow（desired.action = "ANGER"）
       │ WebSocket delta
       ▼
  ブラウザ → NovaSessionService.startSession("ANGER")
       │
       ▼
  Amazon Nova 2 Sonic（共感的会話モード）
```

---

## LATEアクション フロー

```
EventBridge Scheduler（定期実行）
  │ または 手動テストトリガー
  ▼
ActionExecutorLambda
  ├─ CalendarService.getNextMeeting()
  │    └─ Google Calendar API（OAuth 2.0トークン）
  ├─ EtaService.calculateEtaMinutes()
  │    └─ Google Maps Directions API
  ├─ [ETAが予定開始時刻を超過した場合のみ続行]
  ├─ MessageGenerator.generateLateMessage()
  │    └─ Amazon Bedrock Claude 3 Haiku
  │         └─ [失敗時] getFallbackMessage()
  ├─ ActionLogRepository.logActionStart()
  └─ ShadowPublisher.publishActionTrigger("LATE", { message, phoneNumber })
       │
       ▼
  IoT Device Shadow（desired.action = "LATE", desired.lateMessage = "..."）
       │ WebSocket delta
       ▼
  ブラウザ → NovaSessionService.startSession("LATE", { lateMessage })
       │ Nova 2 Sonic がメッセージ読み上げ＋送信確認
       ▼
  ドライバーが音声で「はい、送って」と応答
       │
       ▼
  ブラウザ → [HTTP POST to Lambda] または [IoT Shadow 経由]
       ▼
  ActionExecutorLambda
  ├─ SmsService.sendSms(phoneNumber, message)
  │    └─ Amazon SNS（サンドボックス）
  ├─ ShadowPublisher.clearActionState()
  └─ ActionLogRepository.logActionEnd()
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
