---
title: "Google Cloud API GatewayのAIモデルルーティングを試す前に知るべきこと"
emoji: "🚦"
type: "tech"
topics: ["googlecloud", "ai", "llm", "api"]
published: true
---

2026年8月4日、Google Cloud API Gatewayに**AIモデルルーティング**がPublic Previewとして追加されました。OpenAI互換のリクエストを1つのゲートウェイで受け、JSONの`model`値に応じてVertex AI Model Garden上のGemini、Claude、OpenAI GPT系モデルへ振り分ける機能です。

これは「プロンプトを見て最適モデルを自動選択する機能」ではありません。クライアントが指定したモデル名をOpenAPIの静的ルールへ照合するマネージドなルーターです。接続先や形式をアプリから切り離し、認証・クォータ・ログを入口へ集約できます。

:::message
本記事の設定例はGoogle Cloud公式ドキュメントを基に構文を確認していますが、筆者のGoogle Cloudプロジェクトへの実デプロイまでは行っていません。利用時は対象モデルの提供状況と権限を自分のプロジェクトで確認してください。
:::

## 何が変わったか

従来、複数のLLMを使うには、SDKやエンドポイントの違いをアプリ側で吸収するか、プロキシを自分で運用する必要がありました。

今回のモデルルーティングでは、API Gatewayが次の処理を担当します。

1. OpenAI互換の`POST`リクエストを受ける
2. JSON本文の`model`を読む
3. OpenAPI 3.xで定義したルールと照合する
4. 宛先モデルのネイティブ形式へ変換する
5. Vertex AI Model Gardenのエンドポイントへ送る

ルールに一致しなければ`defaultModel`へフォールバックします。クライアントは共通形式のまま`model`だけを切り替えられます。

## 重要なのは「動的選択」ではなく「責務の分離」

Public Preview時点の条件はリクエスト本文のモデル名だけで、価格、障害率、プロンプト内容などによる自動選択ではありません。価値は次の責務をアプリから分離できる点にあります。

- モデル名とVertex AIバックエンドの対応
- プロバイダーごとのペイロード変換
- API Gatewayでの認証・クォータ管理
- Cloud Loggingによるリクエスト状況の確認
- バックエンド変更時の設定集約

たとえば、軽量モデルと高性能モデルの使い分けはアプリ側で判断しつつ、具体的な接続先はゲートウェイ側で管理できます。

## 最小構成を読む

以下は、Geminiを既定値にし、明示的に指定された場合だけClaudeへ送る構成の要点です。モデルIDは公式例のものなので、実際には自分のプロジェクトで利用可能なモデルを確認してください。

```yaml
openapi: 3.0.3
info:
  title: Model Router Example
  version: 1.0.0

x-google-api-management:
  backends:
    gemini:
      address: https://aiplatform.googleapis.com/v1/projects/PROJECT_ID/locations/global/publishers/google/models/gemini-3.5-flash-lite:generateContent
      deadline: 60.0
      pathTranslation: CONSTANT_ADDRESS
    claude:
      address: https://aiplatform.googleapis.com/v1/projects/PROJECT_ID/locations/global/publishers/anthropic/models/claude-opus-4-7:rawPredict
      deadline: 60.0
      pathTranslation: CONSTANT_ADDRESS
  ai:
    models:
      routing:
        routers:
          main-router:
            defaultModel:
              backend: gemini
              targetModel: google/gemini-3.5-flash-lite
            rules:
              - model: claude-opus-4-7
                backend: claude
                targetModel: anthropic/claude-opus-4-7

paths:
  /v1/chat:
    post:
      operationId: chat
      x-google-model-router: main-router
      responses:
        "200":
          description: OK
```

設定の読みどころは4つです。

- `backends`に実際のVertex AIエンドポイントを書く
- `routers`で受信するモデル名と宛先を対応させる
- `defaultModel`で未一致時の宛先を決める
- `POST`操作へ`x-google-model-router`を付ける

ゲートウェイ用サービスアカウントには、対象モデルへアクセスするための`roles/aiplatform.user`が必要です。API設定やゲートウェイを作る利用者側には`roles/apigateway.admin`が必要になります。

## デプロイ後の検証手順

API設定とゲートウェイをデプロイし、状態が`ACTIVE`になってから最終ホスト名を取得します。

```bash
gcloud api-gateway gateways describe GATEWAY_ID \
  --location=GATEWAY_LOCATION \
  --project=PROJECT_ID \
  --format='value(defaultHostname)'
```

次に、明示ルールとフォールバックを別々にテストします。

```bash
curl "https://GATEWAY_URL/v1/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{
    "model": "claude-opus-4-7",
    "messages": [
      {"role": "user", "content": "再帰を一文で説明して"}
    ]
  }'
```

1. 明示した`model`で期待するルールが選ばれるか
2. 未登録のモデル名で`defaultModel`へ送られるか
3. Cloud Loggingのステータスとレイテンシが想定内か
4. 4xxと5xxを区別して監視できるか
5. ストリーミング応答がクライアントで正しく終了するか

標準ログではURL、ステータス、レイテンシ、バックエンドのホスト名を確認できます。モデル別・ルーター別の専用メトリクスは今後追加予定のため、必要なら`httpRequest.latency`からログベース指標を作ります。

## 導入前に確認したい制約

### 同じルーターの宛先は同一ホストに限られる

1つのルーターで参照する全モデルは、`aiplatform.googleapis.com`など同一のホスト名とスキームを共有する必要があります。任意の外部LLM APIを横断する汎用マルチクラウドルーターではありません。

### OpenAPI 3.x専用で、通常ルートと混在できない

OpenAPI 2.0は対象外です。また、同じOpenAPI仕様内でモデルルーティング操作と通常の`x-google-backend`操作を混在させられません。既存APIへ一部だけ追加するより、AI用ゲートウェイを分ける方が安全です。

### 既存ゲートウェイをそのまま切り替えられない

既存ゲートウェイに後から有効化したり、有効なゲートウェイから機能を外したりはできません。変更時は新しいAPI設定とゲートウェイを作り、経路を切り替えます。

### ネットワークとプロトコルに制約がある

VPC Service ControlsとPrivate Service Connectには対応していません。応答側のServer-Sent Eventsは利用できますが、リクエストストリーミング、gRPC、WebSocket、Gemini Liveは対象外です。テキスト中心のOpenAI互換JSONが現在の主用途です。

### `model`を必須として自前でも検証する

Public Preview中は`model`がないリクエストを正しく拒否しない既知の挙動があります。手前の検証層でも`model`の存在と許可リストを確認し、意図しないフォールバックをテストしてください。

## コストを見るときの注意

API GatewayはAPIコール数とデータ転送量に応じて課金されます。公式料金表では月200万コールまでAPIコール料金が無料ですが、別途Vertex AIのモデル利用料金が発生します。

比較時はゲートウェイ料金だけでなく、データ転送、モデル推論、自前プロキシなら計算資源と保守工数まで含めます。

## まとめ

Google Cloud API Gatewayのモデルルーティングは、複数モデルを賢く自動選択する仕組みというより、**OpenAI互換の入口と静的なモデル対応表をマネージド化する機能**です。

向いているのは、Vertex AI Model Garden上の複数モデルを使い、接続先・認証・クォータ・ログを中央管理したいケースです。一方、複数クラウドをまたぐルーティング、VPC Service Controls、プロンプト内容やコストに基づく自動選択が必要なら、現時点では別のプロキシやアプリ側ロジックも検討する必要があります。

まずは検証用の新規ゲートウェイで、明示ルール、フォールバック、権限エラー、ストリーミング、ログの5点を確認してから、本番経路へ組み込むのがよいでしょう。

## 参考リンク

- [A unified API for AI model routing（Google Developers Blog）](https://developers.googleblog.com/en/a-unified-api-for-ai-model-routing/)
- [Overview of model routing（Google Cloud Documentation）](https://docs.cloud.google.com/api-gateway/docs/model-routing-overview)
- [Configure model routing（Google Cloud Documentation）](https://docs.cloud.google.com/api-gateway/docs/model-routing-configure)
- [API Gateway pricing（Google Cloud）](https://cloud.google.com/api-gateway/pricing)
