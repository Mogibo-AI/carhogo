# ストーリー生成プラン — CarHogo

## ステータス
- **フェーズ**: Part 1 - Planning（質問回答待ち）

---

## Part 1: 計画フェーズ（質問への回答が必要）

以下の質問に回答してください。`[Answer]:` タグの後に選択肢を記入してください。
完了したら「完了しました」とお知らせください。

---

### Question 1
**ストーリーの整理方法（ブレークダウンアプローチ）**

User Stories をどのように整理しますか？

A) ユーザージャーニーベース — ドライバーの体験の流れに沿ってストーリーを整理（乗車→走行→SLEEP検知→会話→解消、など）
B) 機能・アクションベース — SLEEP・ANGER・LATE・インフラ など機能単位で整理
C) ペルソナベース — ドライバー・同乗者・管理者などユーザータイプごとに整理
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 2
**ストーリーの粒度**

1つのユーザーストーリーの粒度（細かさ）はどのくらいが適切ですか？

A) 粗粒度 — 各アクション1〜2ストーリー（例:「ドライバーとして、眠気を検知したときに音声会話が始まるようにしたい」）
B) 中粒度 — 各アクション3〜5ストーリー（トリガー・会話開始・継続・終了などに分割）
C) 細粒度 — 各アクション6ストーリー以上（個々のUI要素やAPIコールレベルで分割）
D) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 3
**ペルソナの範囲**

どのペルソナ（ユーザータイプ）のストーリーを作成しますか？

A) ドライバーのみ（MVP の中心ユーザー）
B) ドライバー＋同乗者/家族（ブラウザ閲覧者も含む）
C) ドライバー＋同乗者/家族＋企業管理者（全ペルソナ）
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 4
**受け入れ基準のフォーマット**

各ストーリーの受け入れ基準をどのフォーマットで記述しますか？

A) シンプルチェックリスト — 箇条書きの確認項目（例: `- [ ] 5秒以内に会話が開始する`）
B) Gherkin形式 — Given/When/Then 形式（例: `Given ドライバーの心拍数が低下 When 3分継続 Then 音声会話が開始する`）
C) Gherkin＋チェックリスト — 主要シナリオは Gherkin、詳細はチェックリストで補足
D) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 5
**Wear OS（Pixel Watch）ストーリーのスコープ**

Pixel Watch アプリのストーリーはどこまで含めますか？

A) セットアップ＋開始/終了ボタン操作のみ（MVPスコープ）
B) セットアップ・操作・ステータス表示・エラー処理も含む（より詳細）
C) Wear OS のストーリーはブラウザアプリのストーリーに統合する（別ストーリーにしない）
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 6
**Nova 2 Sonic 会話の受け入れ基準への含め方**

Nova 2 Sonic の音声会話について受け入れ基準にどのように記述しますか？

A) 会話の開始・終了条件のみ記述する（具体的な発話内容は含めない）
B) 会話トーン・スタイルの指定のみ（例: 「共感的なトーンで」）
C) サンプルダイアログを含める（例: 「CarHogo: 長距離お疲れ様です。今日の夕ご飯は…」）
D) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Part 2: 生成フェーズ（承認後に実行）

以下のステップは **Part 1 の承認後** に実行します。

### Step 1: ペルソナ定義
- [x] `aidlc-docs/inception/user-stories/personas.md` を生成
  - 選択されたペルソナのアーキタイプを定義
  - 各ペルソナの目標・課題・主なインタラクションを記述

### Step 2: ストーリー生成
- [x] `aidlc-docs/inception/user-stories/stories.md` を生成
  - 選択されたブレークダウンアプローチでストーリーを整理
  - INVEST 基準に準拠
  - 各ストーリーに受け入れ基準を記述（選択されたフォーマット）
  - ペルソナとストーリーのマッピングを含める

### Step 3: aidlc-state.md 更新
- [x] User Stories ステージを COMPLETED に更新
