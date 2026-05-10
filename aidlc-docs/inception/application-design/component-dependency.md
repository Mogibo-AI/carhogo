# コンポーネント依存関係 — CarHogo

## 依存関係マトリクス

| 依存元 → 依存先 | CarhogoStack | BiometricAnalyzerLambda | ActionExecutorLambda | IoT Core / Shadow | DynamoDB | Nova 2 Sonic | Bedrock | SNS | Google API | Cognito | Pixel Watch App |
|----------------|:-----------:|:----------------------:|:-------------------:|:-----------------:|:--------:|:------------:|:-------:|:---:|:----------:|:-------:|:---------------:|
| **CarhogoStack** | — | 定義 | 定義 | 定義 | 定義 | N/A | N/A | 定義 | N/A | 定義 | N/A |
| **BiometricAnalyzerLambda** | N/A | — | N/A | ✓ Write | ✓ R/W | N/A | N/A | N/A | N/A | N/A | N/A |
| **ActionExecutorLambda** | N/A | N/A | — | ✓ Write | ✓ Write | N/A | ✓ | ✓ | ✓ | N/A | N/A |
| **ブラウザアプリ** | N/A | N/A | N/A | ✓ Subscribe | N/A | ✓ WebSocket | N/A | N/A | N/A | ✓ Auth | N/A |
| **Pixel Watch App** | N/A | N/A | N/A | ✓ MQTT Pub | N/A | N/A | N/A | N/A | N/A | N/A | — |

---

## 通信パターン

### 1. MQTT over TLS（Pixel Watch → IoT Core）
```
Pixel Watch (MqttManager)
  ──MQTT/TLS:8883──▶ AWS IoT Core
                       └─ IoT Rule ──▶ BiometricAnalyzerLambda
```
- **プロトコル**: MQTT over TLS（ポート 8883）
- **認証**: X.509 デバイス証明書
- **トピック**: `carhogo/watch/{deviceId}`
- **QoS**: 1（配信保証）

### 2. IoT Device Shadow WebSocket（IoT Core → ブラウザ）
```
BiometricAnalyzerLambda ──desired 更新──▶ IoT Device Shadow
ActionExecutorLambda    ──desired 更新──▶ IoT Device Shadow
                                              │
                                              │ WebSocket delta（SigV4署名）
                                              ▼
                                     ブラウザ（IoTShadowService）
```
- **プロトコル**: WebSocket over TLS（ポート 443）
- **認証**: Cognito Identity Pool の AWS Credentials（SigV4署名）

### 3. Nova 2 Sonic WebSocket（ブラウザ → Nova 2 Sonic）
```
ブラウザ（NovaSessionService）
  ──WebSocket（SigV4署名）──▶ Amazon Nova 2 Sonic（ap-northeast-1）
  ◀──音声ストリーム──────────
  ──────音声ストリーム──────▶
```
- **プロトコル**: WebSocket（Bedrock InvokeModelWithBidirectionalStream）
- **認証**: Cognito Identity Pool Credentials で SigV4 署名
- **セッション**: アクション毎に新規接続・アクション終了時に切断

### 4. Lambda → 外部サービス（HTTPS）
```
ActionExecutorLambda
  ──HTTPS──▶ Google Calendar API
  ──HTTPS──▶ Google Maps Directions API
  ──HTTPS──▶ Amazon Bedrock（Claude 3 Haiku）
  ──SDK──▶ Amazon SNS
```
- **認証（Google）**: OAuth 2.0 アクセストークン（Secrets Manager から取得）
- **認証（AWS）**: Lambda 実行ロール IAM（最小権限）

---

## IoT Device Shadow スキーマ

```json
{
  "state": {
    "desired": {
      "action": "IDLE | SLEEP | ANGER | LATE",
      "lateMessage": "遅刻時のみ：Bedrock 生成メッセージ本文",
      "phoneNumber": "遅刻時のみ：SMS送信先番号"
    },
    "reported": {
      "connectionStatus": "MONITORING | STOPPED",
      "lastHeartRate": 72,
      "lastHrv": 45.2,
      "timestamp": "2026-05-09T00:00:00Z"
    },
    "delta": {
      "action": "SLEEP"
    }
  }
}
```

---

## DynamoDB シングルテーブル設計

| エンティティ | PK | SK | 主要属性 | TTL |
|------------|----|----|---------|-----|
| UserConfig | `USER#{userId}` | `CONFIG` | deviceId, baselineHeartRate, phoneNumber, calendarId | なし |
| HeartRateHistory | `USER#{userId}` | `HR#{timestamp}` | heartRate, hrv, deviceId | 1時間 |
| ActionLog | `USER#{userId}` | `ACTION#{timestamp}` | actionType, startTime, endTime, resolution | 30日 |

**GSI（deviceId 逆引き）:**
- GSI-1: PK = `DEVICE#{deviceId}` → UserConfig の逆引き用

---

**関連ドキュメント**:
- [コンポーネント定義](./components.md) — 全コンポーネントの責務・属性一覧
- [コンポーネントメソッド](./component-methods.md) — 全メソッドシグネチャ・型定義
- [サービスフロー図](./services.md) — SLEEP/ANGER/LATE アクション別のシーケンス図
- [アプリケーション設計 統合サマリー](./application-design.md) — システム全体のアーキテクチャ概要
