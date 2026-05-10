# アプリケーション設計プラン — CarHogo

## ステータス
- **フェーズ**: 質問回答待ち

---

## 設計コンテキスト

要件定義・ユーザーストーリーから特定されたコンポーネント群に基づき、以下の設計判断が必要です。
各質問の `[Answer]:` タグの後に回答を記入し、完了したら「完了しました」とお知らせください。

---

## Question 1
**Lambda 関数の構成**

バックエンド Lambda をどのように構成しますか？

A) アクション別Lambda（計4関数）— SleepLambda・AngerLambda・LateLambda・BiometricAnalyzerLambda。各アクションが独立して動作し、互いに影響を与えない。
B) 統合Lambda（計2関数）— BiometricAnalyzerLambda（解析・ルーティング担当）+ ActionExecutorLambda（全アクション実行担当）。シンプルな構成。
C) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 2
**CDK スタック構成**

AWS CDK のスタック構成をどうしますか？

A) シングルスタック — 全 AWS リソース（IoT・Lambda・DynamoDB・Cognito・S3・SNS 等）を 1 つの CDK スタックで管理。シンプルで管理しやすい。
B) マルチスタック — インフラ層（IoT・DynamoDB）とアプリ層（Lambda・Cognito・S3）を分離。変更の影響範囲を局所化できる。
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 3
**Pixel Watch アプリの Activity 構成**

Wear OS アプリの Activity / 画面構成をどうしますか？

A) 単一 Activity（MainActivity のみ）— 開始ボタン・終了ボタン・ステータス表示・BPM/HRV をすべて 1 画面で表示。MVPスコープに最適。
B) 複数 Activity — メイン画面と設定画面などを分離。
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 4
**ブラウザ React アプリのサービス層設計**

React アプリのデータ取得・状態管理のアーキテクチャをどうしますか？

A) Hooks のみ — カスタム Hooks（useIoTShadow・useNovaSession 等）で直接 AWS SDK を呼び出す。シンプルで軽量。
B) Singleton サービス + Hooks — IoTService・NovaSessionService などの Singleton クラスで AWS 操作をラップし、Hooks はそれを呼び出す。テストしやすく、ロジックが集約される。
C) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 5
**Nova 2 Sonic セッション管理方針**

SLEEP・ANGER・LATE の各アクションで Nova 2 Sonic セッションをどのように管理しますか？

A) アクション毎に新規接続 — アクションがトリガーされるたびに新規 WebSocket セッションを確立し、アクション終了時に切断。実装がシンプル。
B) 常時接続（セッション再利用）— アプリ起動時に接続を確立し、全アクションで同一セッションを再利用。レイテンシが低いが、接続管理が複雑。
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 生成する設計アーティファクト（回答後）

- [x] `aidlc-docs/inception/application-design/components.md` — 全コンポーネント定義
- [x] `aidlc-docs/inception/application-design/component-methods.md` — メソッドシグネチャ定義
- [x] `aidlc-docs/inception/application-design/services.md` — サービス層・フロー図
- [x] `aidlc-docs/inception/application-design/component-dependency.md` — 依存関係マトリクス
- [x] `aidlc-docs/inception/application-design/application-design.md` — 統合サマリー
