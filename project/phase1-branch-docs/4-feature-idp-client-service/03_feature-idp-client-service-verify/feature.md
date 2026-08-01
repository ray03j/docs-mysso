# feature/idp-client-service/verify

## 目次

- [目的](#目的)
- [ブランチ名](#ブランチ名)
- [親ブランチ](#親ブランチ)
- [作成・変更するファイル](#作成変更するファイル)
  - [`idp-client/config/routes.rb`](#idp-clientconfigroutesrb)
  - [`idp-client/app/controllers/api/v1/verify_controller.rb`](#idp-clientappcontrollersapiv1verify_controllerrb)
  - [`idp-client/app/services/client_verification_service.rb`](#idp-clientappservicesclient_verification_servicerb)
  - [`idp-client/app/controllers/application_controller.rb`](#idp-clientappcontrollersapplication_controllerrb)
- [ClientVerificationService の説明](#clientverificationservice-の説明)
  - [役割](#役割)
  - [インターフェース](#インターフェース)
  - [処理フロー](#処理フロー)
  - [戻り値](#戻り値)
  - [エラー一覧](#エラー一覧)
- [マージ基準（チェックリスト）](#マージ基準チェックリスト)
- [備考](#備考)

---

## 目的

`idp-client` サービスの内部検証 API（`POST /api/v1/clients/verify`）を実装し、`idp-auth` サービスから呼び出される `client_id` / `client_secret` / `redirect_uri` 検証機能を提供する。

## ブランチ名

`feature/idp-client-service/verify`

## 親ブランチ

`feature/idp-client-service/clients`

> `Client` モデルと `client_secret_hash` の検証メソッドが必要。

## 作成・変更するファイル

### `idp-client/config/routes.rb`

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :clients, only: %i[create index show]
      post 'clients/verify', to: 'verify#create'
    end
  end
end
```

#### なぜこの設定か

- **`resources :clients`**：`POST /api/v1/clients`、`GET /api/v1/clients`、`GET /api/v1/clients/:id` を提供する。
- **`post 'clients/verify'`**：`idp-auth` の `/token` エンドポイントから呼ばれる内部検証 API。`VerifyController#create` にルーティングする。

### `idp-client/app/controllers/api/v1/verify_controller.rb`

```ruby
module Api
  module V1
    class VerifyController < ApplicationController
      before_action :require_internal_api_key

      def create
        result = ClientVerificationService.verify(
          client_id: params[:client_id],
          client_secret: params[:client_secret],
          redirect_uri: params[:redirect_uri]
        )

        return render_unauthorized(result[:error]) unless result[:valid]

        render json: verify_response(result[:client])
      end

      private

      def render_unauthorized(error)
        render json: { valid: false, error: error }, status: :unauthorized
      end

      def verify_response(client)
        {
          valid: true,
          client_id: client.client_id,
          name: client.name,
          redirect_uris: client.redirect_uris,
          allowed_scopes: client.allowed_scopes
        }
      end
    end
  end
end
```

#### なぜこの設定か

- **`before_action :require_internal_api_key`**：全アクションに内部APIキー認証を明示的に適用。`X-Internal-API-Key` ヘッダーが不正または未指定の場合、全てのアクションに先立って 401 を返す。
- **`ClientVerificationService.verify` への委譲**：コントローラは HTTP リクエストの受け渡しに専念し、検証ロジックはサービス層に委譲。
- **`redirect_uri` の検証**：`/token` エンドポイントから `redirect_uri` が渡された場合、登録済み `redirect_uris` に含まれるか検証する。
- **`render_unauthorized` の抽出**：エラーレスポンス生成を private メソッドに切り出し、コントローラアクションの可読性を向上させる。
- **`status: :unauthorized`（401）**：検証失敗時は HTTP 401 を返す。`idp-auth` 側で「client 認証失敗」として処理できるよう、明確なステータスコードを使用する。
- **成功時に client 情報を返す**：`idp-auth` が認可フローで必要とする `redirect_uris` と `allowed_scopes` を含め、内部サービス間通信の往復を減らす。

### `idp-client/app/services/client_verification_service.rb`

```ruby
class ClientVerificationService
  def self.verify(client_id:, client_secret:, redirect_uri: nil)
    return { valid: false, error: 'client_id is required' } if client_id.blank?
    return { valid: false, error: 'client_secret is required' } if client_secret.blank?

    client = Client.find_by(client_id: client_id)
    return { valid: false, error: 'client not found' } unless client
    return { valid: false, error: 'invalid client_secret' } unless client.client_secret_valid?(client_secret)

    if redirect_uri.present? && client.redirect_uris.exclude?(redirect_uri)
      return { valid: false, error: 'invalid redirect_uri' }
    end

    { valid: true, client: client }
  end
end
```

#### なぜこの設定か

- **`Client` クラスとの同名衝突を回避**：`Client::ClientVerificationService` とすると `Client` がモデルクラスと名前空間モジュールの両方を兼ねることになり、Ruby で `TypeError` が発生する。そのためトップレベルの `ClientVerificationService` とする。
- **`find_by(client_id:)`**：外部公開識別子である `client_id` で検索。DB 主キーは UUID だが、内部API呼び出し元は `client_id` を知っているため、それをキーにする。
- **`client.client_secret_valid?`**：`Client` モデルが提供する bcrypt 検証メソッド。タイミング攻撃対策として、bcrypt の比較は一定時間で行われる。平文の比較や独自ハッシュ実装を避け、安全な検証を行う。
- **存在しない client と無効な secret で同じエラーメッセージにしない**：現状は区別しているが、必要に応じて「client not found」と「invalid client_secret」を統一することで、ID 列挙攻撃を防ぐことも可能。Phase 1 では内部APIのため簡潔なメッセージを返す。
- **`redirect_uri` の検証**：`/token` エンドポイントから渡された `redirect_uri` が登録済みか検証する。Phase 1 では必須ではなく、指定された場合のみ検証する。
- **バリデーションの前段階チェック**：空文字や nil の早期リターンで、不要な DB アクセスを防ぐ。

### `idp-client/app/controllers/application_controller.rb`

```ruby
class ApplicationController < ActionController::API
  private

  # 内部APIとして公開するコントローラで before_action として明示的に宣言する
  # （宣言漏れによる認証なし公開を防ぐため、デフォルト適用のフックは設けない）
  def require_internal_api_key
    return if request.headers['X-Internal-API-Key'] == ENV.fetch('INTERNAL_API_KEY')

    render json: { error: 'Unauthorized' }, status: :unauthorized
  end
end
```

#### なぜこの設定か

- `VerifyController` で `before_action :require_internal_api_key` を宣言することで、認証が発動する。`ApplicationController` 自体は `feature/idp-client-service/init` で作成済み。

## ClientVerificationService の説明

### 役割

`POST /api/v1/clients/verify` の検証ロジックを担うサービスクラス。`VerifyController` から HTTP パラメータをそのまま受け取り、`client_id` / `client_secret` / `redirect_uri` の組み合わせが有効かを判定して結果ハッシュを返す。コントローラは結果を見て HTTP レスポンスを組み立てるだけで、検証ルールは全てこのクラスに集約される。

### インターフェース

```ruby
ClientVerificationService.verify(client_id:, client_secret:, redirect_uri: nil) # => Hash
```

| 引数 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `client_id` | String | ✓ | RP の外部公開識別子。DB 主キー（UUID）ではなくこちらをキーに検索する |
| `client_secret` | String | ✓ | RP のシークレット（平文）。ハッシュとの照合は `Client#client_secret_valid?` に委譲する |
| `redirect_uri` | String | - | 指定された場合のみ、登録済み `redirect_uris` と照合する |

### 処理フロー

1. `client_id` が空なら `client_id is required` で失敗（DB アクセス前に早期リターン）
2. `client_secret` が空なら `client_secret is required` で失敗（同上）
3. `Client.find_by(client_id:)` で検索し、存在しなければ `client not found` で失敗
4. `client.client_secret_valid?(client_secret)` で bcrypt ハッシュと照合し、不一致なら `invalid client_secret` で失敗
5. `redirect_uri` が指定されていれば `client.redirect_uris` に含まれるか確認し、含まれなければ `invalid redirect_uri` で失敗
6. 全て通過すれば `{ valid: true, client: client }` を返す

### 戻り値

例外は投げず、常に結果ハッシュを返す。コントローラ側は `result[:valid]` のみで分岐できる。

| 結果 | ハッシュ | `VerifyController` の処理 |
| --- | --- | --- |
| 成功 | `{ valid: true, client: <Client> }` | 200 OK + クライアント情報（`client_id` / `name` / `redirect_uris` / `allowed_scopes`） |
| 失敗 | `{ valid: false, error: <String> }` | 401 Unauthorized + `{ valid: false, error: ... }` |

### エラー一覧

| `error` | 発生条件 |
| --- | --- |
| `client_id is required` | `client_id` が nil または空文字 |
| `client_secret is required` | `client_secret` が nil または空文字 |
| `client not found` | 指定された `client_id` のレコードが存在しない |
| `invalid client_secret` | シークレットが bcrypt ハッシュと不一致 |
| `invalid redirect_uri` | 指定された `redirect_uri` が登録済み `redirect_uris` に含まれない |

## マージ基準（チェックリスト）

- [ ] 内部検証 API で `client_id` / `client_secret` の組み合わせを正しく検証できる
- [ ] `redirect_uri` が指定された場合、登録済みリダイレクト URI と一致するか検証できる
- [ ] 無効な組み合わせに対して 401 Unauthorized を返す
- [ ] `X-Internal-API-Key` ヘッダーが不正または未指定の場合、401 Unauthorized を返す
- [ ] トップレベルの `ClientVerificationService` の単体テストが全て通る
- [ ] `verify_spec.rb` が全て通る
- [ ] 内部API用 `X-Internal-API-Key` ヘッダー認証の `before_action` が `VerifyController` で宣言されている

## 備考

本ブランチでは内部検証 API とサービス層のみを実装する。RP 登録・一覧 API は `feature/idp-client-service/clients` で行う。

> `client_id` / `client_secret` の検証（client authentication）は `/token` エンドポイントで使用する。`/authorize` エンドポイントでは `client_secret` を検証せず、`GET /api/v1/clients/:client_id` 等で `client_id` / `redirect_uri` / `scope` のみを事前検証する。

テストコードの詳細は [`spec.md`](spec.md) を参照。
