# 要件確認質問 — CarHogo

ビジョン文書・テクニカル環境文書を読み込み、以下の点について確認が必要です。
各質問の `[Answer]:` タグの後に回答を記入してください。

---

## Question 1
**【最重要・技術チャレンジ】Amazon Nova 2 Sonic のセッション管理アーキテクチャ**

Nova 2 Sonic はリアルタイム双方向 WebSocket 接続が必要です。
ブラウザから直接接続する場合と、Lambda を中継する場合でアーキテクチャが大きく変わります。

A) ブラウザから直接 Nova 2 Sonic に接続する（WebSocket を Bedrock エンドポイントへ直接確立。Cognito Identity Pool の AWS Credentials を使用して SigV4 署名を行う）
B) Lambda を WebSocket ブリッジとして使用する（API Gateway WebSocket → Lambda → Nova 2 Sonic。ブラウザは API Gateway に接続し、Lambda が Bedrock への接続を管理する）
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 2
**Amazon Nova 2 Sonic の ap-northeast-1（東京）リージョンでの可用性**

テクニカル環境文書では「ap-northeast-1 での可用性要確認」とあります。

A) 確認済み — ap-northeast-1 で利用可能
B) 未確認 — 確認作業が必要
C) 利用不可 — 別リージョン（us-east-1 等）を使用する必要がある
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 3
**音声入出力デバイス（ブラウザアプリ）**

Nova 2 Sonic との音声会話で使用するオーディオ I/O の構成を教えてください。

A) ブラウザのマイク（getUserMedia API）とスピーカー/ヘッドフォンを使用
B) 外部オーディオデバイス（車内スピーカー等）への出力が必要
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 4
**Amazon Bedrock（Claude 3 Haiku）のアクセス状況**

LATEアクション の遅刻メッセージ生成に必要です。テクニカル環境文書では「アクセス申請必要」とあります。

A) アクセス申請済み・承認済み（利用可能）
B) アクセス申請済み・承認待ち
C) まだ申請していない
D) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 5
**Google API（Maps Directions API・Calendar API）の準備状況**

LATEアクションで ETA 計算と予定取得に使用します。

A) 両 API とも準備完了（API キーあり、Calendar 認証設定済み）
B) Google Cloud プロジェクトは作成済みだが API 有効化・キー取得が必要
C) Google Cloud プロジェクトの作成から必要
D) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 6
**Google Calendar の認証方式**

ドライバーの予定（出社時間・場所）を取得するための認証方式を教えてください。

A) OAuth 2.0（ユーザーが Google アカウントでブラウザ認証）
B) サービスアカウント（バックエンド Lambda から直接アクセス）
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 7
**Amazon SNS の SMS 送信環境**

LATEアクションで SMS を送信します。SNS の設定状況を教えてください。

A) 本番環境（実際の電話番号に SMS 送信可能）
B) SNS サンドボックス環境（検証済み電話番号のみに送信可能）
C) SMS 機能は後回し（開発初期はスキップ）
D) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 8
**ブラウザアプリのデプロイ先**

CarHogo ブラウザアプリ（React）のホスティング方法を教えてください。

A) Amazon S3 + CloudFront（静的サイトホスティング）
B) ローカル開発サーバー（開発中は localhost のみ）
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 9
**Pixel Watch の開発・テスト環境**

Wear OS アプリの開発とテストに使用できる環境を教えてください。

A) Pixel Watch 実機でテスト可能
B) Android Studio の Wear OS エミュレータを使用
C) 両方使用可能（エミュレータで開発・実機で最終確認）
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 10
**開発環境のセットアップ状況**

以下のツールのセットアップ状況を教えてください（当てはまるものを選択）。

A) すべてセットアップ済み（AWS CLI・CDK・Node.js 20.x・Android Studio・Kotlin・Git）
B) AWS 系は準備済みだが Android 開発環境のセットアップが必要
C) 多くのセットアップが必要（ほぼゼロから）
D) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 11: Security Extensions
セキュリティ拡張ルールをこのプロジェクトに適用しますか？

A) はい — すべてのセキュリティルールをブロッキング制約として適用（本番グレードアプリケーションに推奨）
B) いいえ — すべてのセキュリティルールをスキップ（PoC・プロトタイプ・実験的プロジェクトに適切）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 12: Property-Based Testing Extension
プロパティベーステスト（PBT）ルールをこのプロジェクトに適用しますか？

A) はい — すべての PBT ルールをブロッキング制約として適用（ビジネスロジック・データ変換・シリアライズ・ステートフルコンポーネントを持つプロジェクトに推奨）
B) 部分適用 — 純粋関数とシリアライズの往復テストにのみ PBT ルールを適用（限られたアルゴリズム的複雑さのプロジェクトに適切）
C) いいえ — すべての PBT ルールをスキップ（シンプルな CRUD アプリ・UI のみのプロジェクト・ビジネスロジックのない薄い統合レイヤーに適切）
X) Other (please describe after [Answer]: tag below)

[Answer]: C
