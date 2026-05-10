# ユニット分解プラン — CarHogo

## ステータス
- **フェーズ**: Part 2 - Generation COMPLETED

---

## コンテキスト

Application Design で確定したコンポーネント構成:
- **CDK インフラ**: 全 AWS リソース定義（シングルスタック）
- **BiometricAnalyzerLambda**: IoT トリガー・生体情報解析・SLEEP/ANGER 検知
- **ActionExecutorLambda**: スケジューラートリガー・LATE アクション処理（共有モジュール利用）
- **Pixel Watch Wear OS アプリ**: Kotlin / Wear OS
- **ブラウザ React アプリ**: TypeScript / React 18

以下の質問に回答してください。完了したら「完了しました」とお知らせください。

---

## Question 1
**リポジトリ構成**

コードをどのリポジトリ構成で管理しますか？

A) モノレポ（1リポジトリに全コードを配置）— cdk/・backend/・watch/・browser/ を1つのリポジトリで一元管理。依存関係の把握が容易。MVPに最適。
B) マルチリポ（コンポーネントごとに別リポジトリ）— 将来的なチーム分割には適しているが、MVP 段階では管理コストが高い。
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 2
**トップレベルのディレクトリ構造**

モノレポの場合、ルートのディレクトリ構造をどうしますか？

A) 用途別ディレクトリ — `cdk/`・`backend/`・`watch/`・`browser/` の4ディレクトリ。シンプルで分かりやすい。
B) レイヤー別ディレクトリ — `infrastructure/`・`services/`・`clients/` のようにアーキテクチャレイヤーで分類。
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 3
**Lambda 共有モジュールの配置**

BiometricAnalyzerLambda と ActionExecutorLambda が共有する `ShadowPublisher`・`ActionLogRepository`・`types.ts`・`logger.ts` をどこに配置しますか？

A) `backend/shared/` — backend ディレクトリ内の共有モジュールサブディレクトリ。Lambda ユニットから相対 import。
B) `backend/` 直下 — 各 Lambda と同じフラット階層に配置。
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 4
**ユニットの分割方法と開発順序**

Lambda をどのようにユニット（開発単位）に分割しますか？また開発順序はどうしますか？

A) 5ユニット分割 — Unit 1: CDK / Unit 2: Pixel Watch / Unit 3: BiometricAnalyzerLambda / Unit 4: ActionExecutorLambda / Unit 5: Browser。Lambda を個別ユニットとして設計・コード生成する。
B) 4ユニット分割 — Unit 1: CDK / Unit 2: Pixel Watch / Unit 3: backend（BiometricAnalyzer + ActionExecutor を1ユニットとして扱う） / Unit 4: Browser。共有モジュールが多いため1ユニットにまとめる。
C) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Part 2 - Generation（完了）

### 生成アーティファクト
- [x] `unit-of-work.md` — 4ユニット定義・ディレクトリ構造・開発フェーズ
- [x] `unit-of-work-dependency.md` — 依存関係マトリクス・ビルド順序・技術的制約
- [x] `unit-of-work-story-map.md` — 14ストーリー × 4ユニット マッピング・Lambda 別担当

### 完了日時
2026-05-09
