# ユニット × ストーリー マッピング — CarHogo

## マッピング一覧

| ストーリー ID | タイトル | Unit 1: CDK | Unit 2: Watch | Unit 3: backend | Unit 4: Browser |
|------------|---------|:-----------:|:-------------:|:---------------:|:---------------:|
| US-01 | Pixel Watch でモニタリングを開始する | ✓ インフラ基盤 | ✓ 主実装 | — | — |
| US-02 | Pixel Watch でモニタリングを停止する | — | ✓ 主実装 | — | — |
| US-03 | ブラウザでリアルタイム生体情報を確認する | ✓ IoT Shadow | — | ✓ データ提供 | ✓ 主実装 |
| US-04 | ブラウザでシステムのアクション状態を確認する | ✓ IoT Shadow | — | ✓ Shadow 更新 | ✓ 主実装 |
| US-05 | 居眠り検知により音声会話が自動的に始まる | ✓ IoT Rule | — | ✓ 主実装（検知） | ✓ Nova UI 起動 |
| US-06 | 音声会話によって覚醒状態を維持する | — | — | — | ✓ 主実装（Nova） |
| US-07 | 心拍数の回復により音声会話が終了する | — | — | ✓ Shadow クリア | ✓ 主実装（Nova） |
| US-08 | 怒り検知により音声会話が自動的に始まる | ✓ IoT Rule | — | ✓ 主実装（検知） | ✓ Nova UI 起動 |
| US-09 | 共感的な音声会話によって感情が落ち着く | — | — | — | ✓ 主実装（Nova） |
| US-10 | 心拍数の安定により音声会話が終了する | — | — | ✓ Shadow クリア | ✓ 主実装（Nova） |
| US-11 | 遅刻が自動検知され謝罪メッセージが生成される | ✓ EventBridge | — | ✓ 主実装（LATE） | ✓ LATE UI 表示 |
| US-12 | Nova 2 Sonic がメッセージを読み上げて確認する | — | — | ✓ Shadow 更新 | ✓ 主実装（Nova） |
| US-13 | ドライバーの音声承認でSMSが自動送信される | — | — | ✓ 主実装（SNS） | ✓ 承認 UI |
| US-14 | ブラウザアプリで送信済みメッセージを確認する | — | — | ✓ ActionLog 保存 | ✓ 主実装（履歴） |

---

## ユニット別 ストーリー担当まとめ

### Unit 1: CDK インフラ
**役割**: 全ストーリーのインフラ基盤を提供  
**直接関連ストーリー**: US-01（IoT Thing）、US-03/04（Device Shadow）、US-05/08（IoT Rules Engine）、US-11（EventBridge Scheduler）

### Unit 2: Pixel Watch Wear OS
**役割**: ドライバーとの物理的インタフェース（データ収集・操作）  
**主担当ストーリー**: US-01（開始）、US-02（停止）

### Unit 3: backend（Lambda × 2 + shared）
**役割**: ビジネスロジックの中核（検知・判定・外部API・SMS）  
**主担当ストーリー**: US-05/08（SLEEP/ANGER 検知）、US-07/10（Shadow クリア）、US-11（LATE 検知・メッセージ生成）、US-13（SMS 送信）、US-14（ActionLog 保存）

**backend 内の担当分担:**
| Lambda | 担当ストーリー |
|--------|-------------|
| BiometricAnalyzerLambda | US-03/04（生体データ処理・Shadow 更新）、US-05/08（SLEEP/ANGER 検知・Shadow トリガー）、US-07/10（回復検知・Shadow クリア） |
| ActionExecutorLambda | US-11（LATE 検知・Bedrock メッセージ生成）、US-12（Shadow 更新）、US-13（SNS SMS 送信）、US-14（ActionLog 書き込み） |

### Unit 4: ブラウザ React アプリ
**役割**: ドライバー向け視覚インターフェース + Nova 2 Sonic 音声会話  
**主担当ストーリー**: US-03/04（表示）、US-05〜10（Nova 2 Sonic SLEEP/ANGER 会話 UI）、US-11〜14（LATE アクション UI・承認・履歴）

---

## 全14ストーリーの完了条件

| ユニット | 完了時に実現されるストーリー |
|---------|--------------------------|
| Unit 1 完了後 | インフラ基盤（全ストーリーの前提） |
| Unit 2 完了後 | US-01・US-02（Pixel Watch 操作） |
| Unit 3 完了後 | US-05・US-07・US-08・US-10・US-11・US-13・US-14（バックエンド処理） |
| Unit 4 完了後 | US-03・US-04・US-06・US-09・US-12（ブラウザ表示・Nova 会話）および全ストーリーの E2E 完了 |
