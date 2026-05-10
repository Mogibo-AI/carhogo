# ユニット・オブ・ワーク定義 — CarHogo

## リポジトリ構成

- **構成**: モノレポ（1リポジトリ）
- **ルートディレクトリ**: `/Users/takamasa/carhogo`

## ディレクトリ構造

```
carhogo/                          # ワークスペースルート（モノレポ）
├── cdk/                          # Unit 1: CDK インフラ
├── backend/                      # Unit 3: バックエンド Lambda
│   ├── biometric-analyzer/       # BiometricAnalyzerLambda
│   ├── action-executor/          # ActionExecutorLambda
│   └── shared/                   # 共有モジュール
├── watch/                        # Unit 2: Pixel Watch Wear OS アプリ
├── browser/                      # Unit 4: ブラウザ React アプリ
│
└── aidlc-docs/                   # AI-DLC ドキュメント（アプリコードなし）
```

---

## Unit 1: CDK インフラ

| 項目 | 内容 |
|------|------|
| **ディレクトリ** | `cdk/` |
| **言語/ランタイム** | TypeScript 5.x / AWS CDK v2 |
| **主要コンポーネント** | CarhogoStack |
| **責務** | 全 AWS リソースをコードとして定義・デプロイ |

**定義する AWS リソース:**
- AWS IoT Core（Thing・Certificate・Policy・Rules Engine・Device Shadow）
- Lambda × 2（BiometricAnalyzerLambda・ActionExecutorLambda）
- DynamoDB（シングルテーブル設計・TTL・GSI）
- Cognito（User Pool・Identity Pool）
- S3 + CloudFront（React アプリホスティング）
- Amazon SNS
- EventBridge Scheduler（ActionExecutorLambda 定期実行）
- AWS Secrets Manager（Google API キー管理）
- IAM ロール（Lambda 最小権限）

**主要ストーリー:** US-03（ブラウザ接続インフラ）、US-05〜14（全アクションのインフラ基盤）

---

## Unit 2: Pixel Watch Wear OS アプリ

| 項目 | 内容 |
|------|------|
| **ディレクトリ** | `watch/` |
| **言語/ランタイム** | Kotlin 1.9.x / Wear OS API 33 |
| **主要コンポーネント** | AppState, WatchConfigLoader, CertificateLoader, BiometricCollector, MqttManager, MonitoringService, MainActivity |
| **責務** | 心拍数・HRV 収集→ MQTT 送信・開始/終了 UI |

**実装内容:**
- Health Services API（ExerciseClient）による心拍数・HRV 収集
- AWS IoT SDK による MQTT over TLS 接続・データ送信
- X.509 証明書認証・指数バックオフ再接続（証明書は `injectCertificates` Gradle タスクで `cdk/output/certs/` から自動注入）
- Foreground Service でバックグラウンド継続動作
- 単一 Activity（開始ボタン・終了ボタン・ステータス/BPM/HRV 表示）
- アンビエントモード対応

**主要ストーリー:** US-01（モニタリング開始）、US-02（モニタリング停止）

---

## Unit 3: backend（Lambda × 2 + shared）

| 項目 | 内容 |
|------|------|
| **ディレクトリ** | `backend/` |
| **言語/ランタイム** | TypeScript 5.x / Node.js 20.x |
| **主要コンポーネント** | BiometricAnalyzerLambda, ActionExecutorLambda, shared modules |
| **責務** | 生体情報解析・アクション実行・外部 API 連携・Device Shadow 更新 |

**backend/ サブ構成:**
```
backend/
├── biometric-analyzer/
│   ├── src/
│   │   ├── handler.ts            # Lambda エントリーポイント（IoT Rule トリガー）
│   │   ├── biometricValidator.ts
│   │   ├── sleepDetector.ts
│   │   ├── angerDetector.ts
│   │   └── heartRateRepository.ts
│   ├── package.json
│   └── tsconfig.json
├── action-executor/
│   ├── src/
│   │   ├── handler.ts            # Lambda エントリーポイント（EventBridge トリガー）
│   │   ├── calendarService.ts
│   │   ├── etaService.ts
│   │   ├── messageGenerator.ts
│   │   └── smsService.ts
│   ├── package.json
│   └── tsconfig.json
└── shared/
    ├── shadowPublisher.ts
    ├── actionLogRepository.ts
    ├── types.ts
    └── logger.ts
```

**主要ストーリー:**
- US-05/08（SLEEP/ANGER パターン検知）— biometric-analyzer
- US-11〜13（LATE 検知・メッセージ生成・SMS 送信）— action-executor

---

## Unit 4: ブラウザ React アプリ

| 項目 | 内容 |
|------|------|
| **ディレクトリ** | `browser/` |
| **言語/ランタイム** | TypeScript 5.x / React 18 / Vite 5 |
| **主要コンポーネント** | AuthService, IoTShadowService, NovaSessionService, Hooks × 3, Components × 7 |
| **責務** | 生体情報リアルタイム表示・アクション状態管理・Nova 2 Sonic 音声会話 UI |

**browser/ サブ構成:**
```
browser/
├── src/
│   ├── services/
│   │   ├── AuthService.ts
│   │   ├── IoTShadowService.ts
│   │   └── NovaSessionService.ts
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useIoTShadow.ts
│   │   └── useNovaSession.ts
│   ├── components/
│   │   ├── App.tsx
│   │   ├── LoginPage.tsx
│   │   ├── Dashboard.tsx
│   │   ├── BiometricDisplay.tsx
│   │   ├── ActionStatusDisplay.tsx
│   │   ├── NovaConversationUI.tsx
│   │   └── ActionHistoryList.tsx
│   └── main.tsx
├── package.json
├── tsconfig.json
└── vite.config.ts
```

**主要ストーリー:** US-03（生体情報表示）、US-04（アクション状態表示）、US-05〜14（各アクション UI）

---

## 推奨開発順序

```
Phase 1: Unit 1 (CDK)
  └─ 全 AWS リソースをデプロイ。IoT 証明書を cdk/output/certs/ に出力・Cognito・DynamoDB を確定。

Phase 2: Unit 2 (Pixel Watch) + Unit 3 (backend) — 並行開発可
  ├─ Unit 2: watch/ - injectCertificates + injectIoTConfig Gradle タスクで自動設定して実装
  └─ Unit 3: backend/ - Lambda コードを実装（Unit 1 の DynamoDB/IoT Rule を使用）

Phase 3: Unit 4 (Browser)
  └─ browser/ - Cognito・IoT Endpoint 確定後に NovaSessionService 等を実装
```

---

**関連ドキュメント**:
- [ユニット依存関係](./unit-of-work-dependency.md) — ユニット間の依存関係・ビルド順序・証明書管理フロー
- [ユニット × ストーリー マッピング](./unit-of-work-story-map.md) — 各ユニットが担当するユーザーストーリー
- [アプリケーション設計 統合サマリー](./application-design.md) — システム全体のアーキテクチャ概要
