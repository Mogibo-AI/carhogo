# 用語集 — CarHogo

本プロジェクトで使用する専門用語の定義集です。
非技術者・初読者がドキュメントを読む際の参照資料として使用してください。

---

## CarHogo 固有用語

| 用語 | 定義 |
|------|------|
| **SLEEP アクション** | 居眠りパターン検知時に自動発動するアクション。Nova 2 Sonic が音声会話を開始し、ドライバーの覚醒状態を回復させる |
| **ANGER アクション** | 怒り・ストレスパターン検知時に自動発動するアクション。Nova 2 Sonic が共感的な音声会話で感情を沈静化させる |
| **LATE アクション** | 遅刻確定時に自動発動するアクション。Bedrock でメッセージ生成 → Nova 2 Sonic が読み上げ → ドライバー承認後に SNS で SMS 送信 |
| **ベースライン心拍数** | ドライバー個人の「平常時の心拍数」。SLEEP/ANGER 検知の基準値として DynamoDB に保持される |
| **T0** | イベントタイムラインの計測基準点。SLEEP/ANGER では「Pixel Watch が SLEEP/ANGER のトリガーとなる生体データを送信した瞬間」（実際のパターン判定は BiometricAnalyzerLambda が T0+<1s 以降に実施）、LATE では「EventBridge が Lambda を起動した瞬間」（詳細は requirements.md「イベントタイムライン定義」参照） |

---

## 生体情報・医療用語

| 用語 | 正式名称 | 定義 |
|------|---------|------|
| **BPM** | Beats Per Minute | 心拍数の単位。1分間の心拍回数 |
| **HRV** | Heart Rate Variability（心拍変動） | 心拍と心拍の間隔のばらつき。値が低いほどストレス・疲労状態を示す指標 |
| **RMSSD** | Root Mean Square of Successive Differences | HRV の計算方法の一つ。連続する心拍間隔の差の二乗平均平方根。自律神経系の状態を反映する |

---

## AWS サービス

| 用語 | 定義 |
|------|------|
| **AWS CDK** | Cloud Development Kit。TypeScript 等のプログラミング言語で AWS インフラをコードとして定義・デプロイするフレームワーク |
| **Amazon Bedrock** | AWS の生成 AI サービス。Claude（Anthropic）等の大規模言語モデル（LLM）を API 経由で利用できる。CarHogo では Claude 3 Haiku で遅刻メッセージを生成する |
| **Amazon Cognito** | AWS の認証・認可サービス。User Pool（メール認証）と Identity Pool（AWS サービスへの一時的なアクセス権限付与）の2種類がある |
| **Identity Pool** | Cognito の機能の一つ。認証済みユーザーに対して、AWS サービスへの一時的な認証情報（IAM Credentials）を発行する。CarHogo では Nova 2 Sonic への WebSocket 接続に使用する |
| **Amazon Nova 2 Sonic** | AWS の超低遅延リアルタイム音声会話 AI。双方向の WebSocket 接続でブラウザと直接通信し、自然な音声会話を実現する |
| **Amazon DynamoDB** | AWS のフルマネージド NoSQL データベース。CarHogo ではシングルテーブル設計でユーザー設定・心拍数履歴・アクションログを管理する |
| **Amazon SNS** | Simple Notification Service。SMS・メール・プッシュ通知の送信サービス。CarHogo では LATE アクションで遅刻 SMS を送信する |
| **Amazon CloudFront** | AWS のコンテンツ配信ネットワーク（CDN）。S3 に保存した React アプリを世界中に高速配信する |
| **AWS IoT Core** | IoT デバイスと AWS を接続するマネージドサービス。MQTT ブローカーとして機能し、Pixel Watch からのデータを受信する |
| **Device Shadow** | AWS IoT Core の機能。デバイスの「仮想的な状態コピー」をクラウドに保持する。デバイスがオフラインでも状態を参照・更新できる。CarHogo ではアクション状態（SLEEP中・ANGER中など）をブラウザと同期するために使用する |
| **IoT Rules Engine** | AWS IoT Core の機能。MQTT トピックに届いたメッセージを条件に応じて Lambda 等へルーティングするルールを定義する |
| **EventBridge Scheduler** | AWS のスケジューラーサービス。定期的（例：1分毎）に Lambda 関数を起動する。CarHogo では LATE アクションの ETA 監視に使用する |
| **AWS Lambda** | AWS のサーバーレス関数実行環境。コードのみを記述し、インフラ管理が不要。CarHogo では Node.js 20.x で実装する |
| **GSI** | Global Secondary Index。DynamoDB のインデックス機能。プライマリキー以外の属性でデータを高速検索するために使用する |
| **TTL** | Time To Live。DynamoDB のデータ自動削除機能。指定した期限が過ぎたレコードを自動削除する。CarHogo では心拍数履歴データの古いレコード削除に使用する |

---

## プロトコル・セキュリティ標準

| 用語 | 定義 |
|------|------|
| **MQTT** | Message Queuing Telemetry Transport。IoT デバイス向けの軽量メッセージングプロトコル。Pixel Watch と AWS IoT Core 間の通信に使用する |
| **TLS** | Transport Layer Security。通信を暗号化するプロトコル。MQTT over TLS（ポート 8883）で Pixel Watch と IoT Core を安全に接続する |
| **X.509 証明書** | デジタル証明書の標準規格。Pixel Watch を AWS IoT Core に接続する際のデバイス認証に使用する。AWS CDK でデプロイ時に生成される |
| **OAuth 2.0** | 認可フレームワークの標準規格。Google Calendar API へのアクセスに使用する。ユーザーが Google アカウントで CarHogo にカレンダーへのアクセスを許可する仕組み |
| **SigV4** | AWS Signature Version 4。AWS サービスへのリクエストに付与する署名方式。Nova 2 Sonic への WebSocket 接続で使用する |
| **WebSocket** | ブラウザとサーバー間で双方向通信を可能にするプロトコル。HTTP と異なり、一度接続するとリアルタイムにデータを送受信し続けられる。Nova 2 Sonic との音声会話に使用する |

---

## 競合システム・自動車技術

| 用語 | 定義 |
|------|------|
| **EyeSight** | スバルが提供する運転支援技術。ステレオカメラを使用した前方衝突警告・自動ブレーキ・車線逸脱警告などを統合した安全システム |
| **PCS** | Pre-Collision System（プリクラッシュセーフティシステム）。トヨタが提供する衝突回避支援機能。自動ブレーキ・警告音などを統合する |
| **DMS** | Driver Monitoring System（ドライバーモニタリングシステム）。車載カメラでドライバーの顔・視線を撮影し、眠気・脇見運転を検知して警告する技術 |
| **Autopilot** | テスラが提供する高度運転支援機能。車両の速度・操舵を自動制御する。同社の上位機能として FSD がある |
| **FSD** | Full Self-Driving（フルセルフドライビング）。テスラの高度自動運転機能。Autopilot の上位機能として位置付けられる |

---

## OS・フレームワーク

| 用語 | 定義 |
|------|------|
| **Wear OS** | Google のスマートウォッチ向け OS。Pixel Watch はこの OS で動作する |
| **Health Services API** | Wear OS が提供する生体情報取得 API。心拍数・HRV・加速度センサー等のデータをリアルタイムに取得できる |
| **Foreground Service** | Android/Wear OS のバックグラウンド処理の仕組み。画面がオフの状態でもアプリを継続動作させる。CarHogo では画面消灯中も心拍数の収集・送信を継続するために使用する |

---

## 開発手法・テスト用語

| 用語 | 定義 |
|------|------|
| **MVP** | Minimum Viable Product（実用最小限の製品）。最低限の機能で動作する最初のバージョン。CarHogo の MVP は SLEEP・ANGER・LATE の3アクションが E2E で動作することを目標とする |
| **PoC** | Proof of Concept（概念実証）。新しいアイデアや技術が実現可能かを検証するための小規模な試作・実装。MVP より前段階の検証フェーズを指すことが多い |
| **Gherkin** | BDD（振る舞い駆動開発）でテストシナリオを記述するための形式的な言語。`Given（前提条件）/ When（操作）/ Then（期待結果）` の構文でユーザーストーリーの受け入れ基準を記述する |
| **ETA** | Estimated Time of Arrival（到着予想時刻）。CarHogo の LATE アクションで Google Maps Directions API を使って計算する |
