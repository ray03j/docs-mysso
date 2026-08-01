# feature/idp-client-service/clients

## 目次

- [目的](#目的)
- [ブランチ名](#ブランチ名)
- [親ブランチ](#親ブランチ)
- [作成・変更するファイル](#作成変更するファイル)
  - [`idp-client/app/controllers/api/v1/clients_controller.rb`](#idp-clientappcontrollersapiv1clients_controllerrb)
  - [`idp-client/app/models/client.rb`](#idp-clientappmodelsclientrb)
  - [`idp-client/spec/factories/clients.rb`](#idp-clientspecfactoriesclientsrb)
- [マージ基準（チェックリスト）](#マージ基準チェックリスト)
- [備考](#備考)

---

## 目的

`idp-client` サービスの RP 登録・一覧 API（`POST /api/v1/clients`, `GET /api/v1/clients`）を実装し、連携アプリ（RP）の管理機能を提供する。

## ブランチ名

`feature/idp-client-service/clients`

## 親ブランチ

`feature/idp-client-service/init`

## 作成・変更するファイル

### `idp-client/app/controllers/api/v1/clients_controller.rb`

```ruby
module Api
  module V1
    class ClientsController < ApplicationController
      before_action :require_internal_api_key

      def index
        render json: { clients: Client.all.map { |client| client_response(client) } }
      end

      def create
        client = Client.new(client_params)
        client.client_id = SecureRandom.uuid
        client.client_secret = SecureRandom.urlsafe_base64(32)

        if client.save
          render json: client_response(client).merge(client_secret: client.client_secret), status: :created
        else
          render json: { errors: client.errors.full_messages }, status: :unprocessable_content
        end
      end

      def show
        client = Client.find_by(client_id: params[:id])
        return head :not_found unless client

        render json: client_response(client)
      end

      private

      def client_params
        params.require(:client).permit(:name, redirect_uris: [], allowed_scopes: [])
      end

      def client_response(client)
        {
          client_id: client.client_id,
          name: client.name,
          redirect_uris: client.redirect_uris,
          allowed_scopes: client.allowed_scopes,
          created_at: client.created_at,
          updated_at: client.updated_at
        }
      end
    end
  end
end
```
#### 関数説明

- `index`：登録済みRPクライアントの一覧を返す。内部APIキー認証の保護対象。
- `create`：新規RPクライアントを作成。`client_id` にUUID、`client_secret` にランダム文字列を生成し、201 Created で平文 `client_secret` を含むレスポンスを返す。バリデーションエラー時は422 Unprocessable Contentを返す。内部APIキー認証の保護対象。
- `show`：`client_id` で指定されたRPクライアントの詳細を返す。`idp-auth` の `/authorize` から内部APIとして呼ばれる。
- `client_params`：Strong Parameters で `name`、`redirect_uris`、`allowed_scopes` のみを許可し、内部値のマスアサインメントを防ぐ。
- `client_response`：クライアント情報をJSON構造化する共通ヘルパー。`index` / `create` / `show` でDRYに共通化。

#### なぜこの設定か

- **`SecureRandom.uuid`**：`client_id` は OIDC で RP を識別する外部公開識別子。衝突しにくい UUID v4 を採用し、予測困難な識別子とする。
- **`SecureRandom.urlsafe_base64(32)`**：`client_secret` は 256bit のランダム文字列を Base64URL エンコード。OAuth 2.0 で推奨される十分なエントロピーを確保する。
- **`client_secret` は作成時のみ返却**：`client_secret` は平文で一度しか取得できない。以降は `client_secret_hash` のみを保持するため、紛失時は再発行が必要。これは OAuth 2.0 のクライアントシークレット管理の標準的な運用方針である。
- **`status: :created`（201）**：リソース作成成功時は HTTP 201 を返す。RESTful API の慣習に従い、クライアントが「作成完了」を正確に認識できるようにする。
- **`status: :unprocessable_content`（422）**：バリデーションエラー時は HTTP 422 を返す。Rack 3 準拠のシンボル名を使用し、エラー内容は `full_messages` で配列形式で返す。
- **`show` は `client_id` で検索**：DB 主キーではなく OIDC の外部公開識別子である `client_id` で検索する。`idp-auth` の `/authorize` エンドポイントから内部 API として呼ばれる。
- **全アクションを内部 API 保護対象とする**：`before_action :require_internal_api_key` を `only:` 指定なしで宣言し、`create` / `index` / `show` すべてに `X-Internal-API-Key` ヘッダー認証を適用する。`create` は `client_secret` の払い出し、`index` は登録クライアントの一覧返却を行うため、認証なし公開は重大な情報露出・不正発行につながる（PR #21 レビュー指摘による修正。当初は `index` / `create` を管理画面用として保護対象外とする設計だったが、管理画面未実装の間は無認証公開となるため全アクション必須に変更）。管理画面実装時に管理者向け認証への切り替えを再検討する。判定用のフックメソッドは設けない（宣言漏れによる認証なし公開を防ぐため）。
- **レスポンスに `client_secret_hash` を含めない**：セキュリティ上の理由から、ハッシュ化されたクライアントシークレットは一覧・詳細 API レスポンスに含めない。
- **`client_response` ヘルパー**：クライアント情報の JSON 構造化を private メソッドに抽出し、`index` / `create` / `show` で共通化。DRY かつ可読性を高める。
- **`permit(:name, redirect_uris: [], allowed_scopes: [])`**：Strong Parameters で配列パラメータをホワイトリスト化。`client_id` や `client_secret_hash` などの内部値をマスアサインメントで変更されるリスクを防ぐ。

### `idp-client/app/models/client.rb`

```ruby
require 'bcrypt'

class Client < ApplicationRecord
  validates :client_id, presence: true, uniqueness: true
  validates :client_secret_hash, presence: true
  validates :name, presence: true
  validates :redirect_uris, presence: true
  validates :allowed_scopes, presence: true

  validate :redirect_uris_must_be_valid_urls
  validate :allowed_scopes_must_be_supported

  attr_reader :client_secret

  def client_secret=(value)
    return if value.blank?

    @client_secret = value
    self.client_secret_hash = BCrypt::Password.create(value)
  end

  def client_secret_valid?(secret)
    return false if client_secret_hash.blank? || secret.blank?

    BCrypt::Password.new(client_secret_hash) == secret
  end

  def redirect_uris
    (super || '').tr("\r", "\n").lines.map(&:strip).compact_blank
  end

  def redirect_uris=(uris)
    super(uris.is_a?(Array) ? uris.join("\n") : uris.to_s)
  end

  def allowed_scopes
    (super || '').split.map(&:strip).compact_blank
  end

  def allowed_scopes=(scopes)
    super(scopes.is_a?(Array) ? scopes.join(' ') : scopes.to_s)
  end

  private

  def redirect_uris_must_be_valid_urls
    redirect_uris.each do |uri|
      parsed = URI.parse(uri)
      errors.add(:redirect_uris, "#{uri} is not a valid URL") unless parsed.is_a?(URI::HTTP) || parsed.is_a?(URI::HTTPS)
    rescue URI::InvalidURIError
      errors.add(:redirect_uris, "#{uri} is not a valid URL")
    end
  end

  def allowed_scopes_must_be_supported
    supported = %w[openid profile email]
    invalid = allowed_scopes - supported
    errors.add(:allowed_scopes, "#{invalid.join(', ')} are not supported") if invalid.any?
  end
end
```
#### 関数説明

- `client_secret=`：平文の `client_secret` を受け取り、bcrypt で `client_secret_hash` にハッシュ化して保存。平文は永続化せず、作成時のレスポンスでのみ返却する。
- `client_secret_valid?`：bcrypt を使用して `client_secret` を検証。`client_secret_hash` または引数が空の場合は `false` を返す。

- `redirect_uris`：DBに保存された改行区切り文字列を配列に変換して返す。
- `redirect_uris=`：配列を改行区切り文字列に変換してDBに保存。

- `allowed_scopes`：DBに保存された空白区切り文字列を配列に変換して返す。
- `allowed_scopes=`：配列を空白区切り文字列に変換してDBに保存。

- `redirect_uris_must_be_valid_urls`：カスタムバリデーション。各 `redirect_uri` が HTTP/HTTPS スキームの有効なURLであることを検証。
- `allowed_scopes_must_be_supported`：カスタムバリデーション。`allowed_scopes` が `openid`、`profile`、`email` のみであることを検証。

#### なぜこの設定か

- **`client_secret=`**：平文の `client_secret` を受け取り、bcrypt で `client_secret_hash` にハッシュ化して保存。平文は永続化せず、作成時の一時的なレスポンスでのみ返却する。
- **`client_secret_valid?`**：bcrypt を使用して `client_secret` を検証。タイミング攻撃対策として比較は一定時間で行われる。メソッド名を predicate（`?` 終わり）にすることで、 RuboCop の命名規約に沿い、boolean 戻り値を明示する。
- **`has_secure_password` を使わない理由**：`has_secure_password :client_secret` は `client_secret_digest` カラムを前提とする。既存スキーマの `client_secret_hash` カラム名と一致させるため、bcrypt ハッシュ化を自前で実装する。

- **`redirect_uris_must_be_valid_urls`**：OIDC では `redirect_uri` は有効な URL でなければならない。`URI.parse` で検証し、HTTP/HTTPS スキームのみを許可。これにより、任意のスキームや不正な文字列によるリダイレクト先改ざんを防ぐ。

- **`redirect_uris` の分割に正規表現を使わない理由**：`split(/\r?\n|\r/)` の代わりに、`tr("\r", "\n")` で改行コードを `\n` に正規化してから `lines` で分割する。`\r\n` は一時的に `\n\n` となるが、`compact_blank` で空行が除去されるため結果は等価。文字列操作のみで完結し、意図が読み取りやすい。

- **`allowed_scopes_must_be_supported`**：Phase 1 でサポートするスコープは `openid`, `profile`, `email` に限定。サポート外スコープの登録を防ぎ、認可フローでの想定外の動作を未然に防ぐ。

### `idp-client/spec/factories/clients.rb`

```ruby
FactoryBot.define do
  factory :client do
    sequence(:client_id) { SecureRandom.uuid }
    name { 'Test Relying Party' }
    redirect_uris { ['http://localhost:5174/callback'] }
    allowed_scopes { ['openid', 'profile'] }
    client_secret { 'test_client_secret' }
  end
end
```

#### なぜこの設定か

- **`sequence(:client_id)`**：各ファクトリで一意な `client_id` を自動生成。`uniqueness` バリデーションがあるため、固定値だと複数の `create(:client)` で衝突する。
- **`SecureRandom.uuid`**：`client_id` は DB 上 UUID 型のため、有効な UUID 文字列を生成する。無効な文字列を渡すと DB 書き込み時に型変換エラーとなる。
- **`client_secret` 属性**：`Client#client_secret=` が提供する仮想属性に対応。ファクトリ作成時に平文シークレットを渡すと、保存時に自動的に `client_secret_hash` へハッシュ化される。

## マージ基準（チェックリスト）

- [ ] RP クライアント登録 API で `client_id`, `client_secret`, `redirect_uris`, `allowed_scopes` が保存できる
- [ ] `redirect_uris` と `allowed_scopes` のバリデーションが正しく機能する
- [ ] RP クライアント一覧 API で登録済みクライアントが取得できる
- [ ] RP クライアント詳細 API（`GET /api/v1/clients/:client_id`）で `client_id` / `redirect_uris` / `allowed_scopes` が取得できる
- [ ] 全エンドポイント（`POST /api/v1/clients` / `GET /api/v1/clients` / `GET /api/v1/clients/:client_id`）で `X-Internal-API-Key` ヘッダー認証が必要
- [ ] レスポンスに `client_secret_hash` / `client_secret` が含まれない
- [ ] `clients_spec.rb` が全て通る
- [ ] `client_spec.rb` が全て通る
- [ ] `db/seeds.rb` が正常に動作し、テスト用クライアント初期データが投入できる（冪等・development/test 限定）

## 備考

本ブランチでは RP 登録・一覧 API とモデルテストのみを実装する。内部検証 API は `feature/idp-client-service/verify` で行う。

> Phase 2 では `GET /api/v1/clients/:client_id`（詳細取得）を追加し、`/authorize` エンドポイントでの `client_id` / `redirect_uri` / `scope` の事前検証に使用する。`/authorize` では `client_secret` は検証せず、検証は `/token` エンドポイントで `POST /api/v1/clients/verify` を使って行う。

テストコードの詳細は [`spec.md`](spec.md) を参照。
