# feature/idp-client-service/init

## 目次

- [目的](#目的)
- [ブランチ名](#ブランチ名)
- [親ブランチ](#親ブランチ)
- [作成・変更するファイル](#作成変更するファイル)
  - [`idp-client/config/routes.rb`](#idp-clientconfigroutesrb)
  - [`idp-client/config/application.rb`](#idp-clientconfigapplicationrb)
  - [`idp-client/app/controllers/application_controller.rb`](#idp-clientappcontrollersapplication_controllerrb)
  - [`idp-client/app/models/client.rb`](#idp-clientappmodelsclientrb)
  - [`idp-client/db/seeds.rb`](#idp-clientdbseedsrb)
- [マージ基準（チェックリスト）](#マージ基準チェックリスト)
- [備考](#備考)

---

## 目的

`idp-client` サービスに RP 管理に必要な共通設定（ルーティング、内部API保護の `before_action` 用共通メソッド、モデル強化）を導入し、後続ブランチで API エンドポイントを実装できる基盤を整える。

## ブランチ名

`feature/idp-client-service/init`

## 親ブランチ

`feature/database-schema`（`main` へのマージ完了後）

## 作成・変更するファイル

### `idp-client/config/routes.rb`

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :clients, only: [:create, :index, :show]
      post 'clients/verify', to: 'verify#create'
    end
  end
end
```

#### なぜこの設定か

- **`namespace :api` + `namespace :v1`**：API バージョニングの標準的な構成。将来の v2 追加時に互換性を保ちながら移行できる。`idp-client` は内部マイクロサービスとして他サービスから呼ばれるため、明確なバージョン管理が重要。
- **`resources :clients, only: [:create, :index, :show]`**：Phase 1 では RP クライアントの「登録」「一覧」と、Phase 2 で `/authorize` エンドポイントが使用する「詳細取得」を提供。更新・削除は管理画面の拡張時（Phase 4 以降）に追加する。最小権限の原則に従い、不要なアクションは公開しない。
- **`show` は内部 API としても使用**：`idp-auth` の `/authorize` エンドポイントから `client_id` でクライアント情報を取得し、`redirect_uri` / `scope` の事前検証を行う。`client_secret` は含めず、機密情報の扱いに注意する。
- **`post 'verify'`**：`client_id` / `client_secret` の検証は「状態を変更しないが機密情報を扱う」操作のため、GET ではなく POST を採用。`client_secret` をクエリパラメータに含めないようにし、HTTP ボディでの安全な送信を保証する。

### `idp-client/config/application.rb`

```ruby
require_relative 'boot'
require 'rails/all'

module IdpClient
  class Application < Rails::Application
    config.load_defaults 7.1
    config.api_only = true
    config.time_zone = 'Tokyo'
  end
end
```

#### なぜこの設定か

- **`feature/project-setup` で既に作成済み**：`idp-client/config/application.rb` は `feature/project-setup/idp-client` ブランチで作成されており、本ブランチでは変更なし。`idp-client` は内部マイクロサービスとして原則 API モードを維持し、セッション管理は `idp-auth` サービスで行う。`idp-client` 自体はステートレスな API 応答を提供する。
- **`require_relative 'boot'`**：現在のファイル（`application.rb`）からの相対パスで `boot.rb` を読み込む。`require` は `$LOAD_PATH` を基準に解決するのに対し、`require_relative` は実行中のファイル位置から解決するため、Rails アプリケーションの設定ファイル同士の依存関係を確実に解決できる。gem やライブラリの読み込み順序に依存せず、`config/boot.rb` が正しく読み込まれる。

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

#### メソッドの役割

- **`require_internal_api_key`**：認証チェックの本体。内部APIとして保護したいコントローラが `before_action` として明示的に宣言して使う。リクエストヘッダ `X-Internal-API-Key` の値が環境変数 `INTERNAL_API_KEY` と一致しない場合に 401 Unauthorized を返して処理を中断する。一致した場合は何もせずアクションへ進む。

リクエスト処理の流れは以下の通り。

1. 保護対象コントローラで宣言された `before_action` として `require_internal_api_key` が実行される
2. ヘッダの値と環境変数 `INTERNAL_API_KEY` を比較し、一致すればアクションへ進み、不一致なら 401 を返す

#### なぜこの設定か

- **フック方式（Template Method）を採用しない**：`ApplicationController` に `before_action ..., if: :internal_api?` を設置しサブクラスでオーバーライドさせる方式も検討したが、デフォルトが「認証なし」になるため fail-open となる。新しいコントローラ追加時にオーバーライドを忘れると内部APIが認証なしで公開されるリスクがある。セキュリティ機能は暗黙の継承契約に乗せず、各コントローラで `before_action :require_internal_api_key` を明示的に宣言する Rails 標準の方式とした。
- **`ENV.fetch('INTERNAL_API_KEY')`**：内部APIキーは環境変数から取得。`fetch` を使用して未定義時に即座に例外を発生させ、誤った空文字比較による認証 bypass を防ぐ。
- **`status: :unauthorized`（401）**：認証失敗時は HTTP 401 を返す。内部サービス間通信では即座に異常を検知できるよう、シンプルな JSON エラーを返す。

### `idp-client/app/models/client.rb`

```ruby
class Client < ApplicationRecord
  validates :client_id, presence: true, uniqueness: true
  validates :client_secret_hash, presence: true
  validates :name, presence: true
  validates :redirect_uris, presence: true
  validates :allowed_scopes, presence: true

  def redirect_uris
    (super || '').split(/\r?\n|\r/).map(&:strip).reject(&:blank?)
  end

  def redirect_uris=(uris)
    super(uris.is_a?(Array) ? uris.join("\n") : uris.to_s)
  end

  def allowed_scopes
    (super || '').split.map(&:strip).reject(&:blank?)
  end

  def allowed_scopes=(scopes)
    super(scopes.is_a?(Array) ? scopes.join(' ') : scopes.to_s)
  end
end
```

#### なぜこの設定か

- **`class Client < ApplicationRecord`（1行目）**：Rails の Active Record パターンに従い、`clients` テーブルと1対1でマッピングする。これにより DB 操作をオブジェクト指向で記述でき、マイグレーションとの整合性を自動的に保つ。

- **`validates :client_id, presence: true, uniqueness: true`（2行目）**：`client_id` は OIDC で RP を識別する必須の外部公開識別子。重複は許容できないため、DB レベルの UNIQUE インデックスとセットでモデル層でも保証し、レースコンディションによる重複を早期に検出する。

- **`validates :client_secret_hash, presence: true`（3行目）**：機密情報の平文保存を避け、ハッシュ化後の値を保存する。`feature/database-schema` では `null: false` の制約を設けており、モデル側でも存在検証を行う。

- **`validates :name, presence: true`（4行目）**：管理画面での表示名として必須。`feature/database-schema` でも `null: false` の制約を設けている。

- **`redirect_uris` / `allowed_scopes` の getter/setter オーバーライド**：`feature/database-schema` では `text` 型で改行区切り（`redirect_uris`）またはスペース区切り（`allowed_scopes`）を想定。アプリケーションコードでは配列として扱いたいため、getter で文字列を配列に変換し、setter で配列を文字列に戻す。これにより、API やサービス層で配列操作を自然に行える。

### `idp-client/db/seeds.rb`

```ruby
require 'bcrypt'

# テスト用クライアント（デモ RP 用、development/test のみ）
if Rails.env.development? || Rails.env.test?
  client = Client.find_or_initialize_by(client_id: '550e8400-e29b-41d4-a716-446655440000')
  cost = Rails.env.test? ? BCrypt::Engine::MIN_COST : BCrypt::Engine::DEFAULT_COST
  client.assign_attributes(
    client_secret_hash: BCrypt::Password.create('demo_secret', cost: cost),
    name: 'Demo Relying Party',
    redirect_uris: "http://localhost:5174/callback\nhttp://localhost:5174/",
    allowed_scopes: 'openid profile email'
  )
  client.save!
end
```

#### なぜこの設定か

- **テスト用クライアント**：Phase 3 の認可フロー動作確認時に即座に使用できるデモ RP 用データ。development/test のみ投入する。
- **`client_id`**：`demo-rp/.env.example` で定義した `VITE_CLIENT_ID` と一致させる。
- **`BCrypt::Password.create`**：`bcrypt` gem を使用した安全なハッシュ化。平文の client_secret は保存しない。
- **環境ごとのコスト切り替え**：test 環境では `BCrypt::Engine::MIN_COST` を使用し、テスト実行速度を確保する。production/development では `DEFAULT_COST` を使用し、実環境に近い挙動を確認する。
- **冪等性**：`find_or_initialize_by` を使用し、seed を複数回実行しても重複レコードが作成されない。

## マージ基準（チェックリスト）

- [ ] `idp-client/config/routes.rb` に `/api/v1/clients`（index, create, show）と `/api/v1/clients/verify` が定義されている
- [ ] `rails routes` で上記ルートが確認できる
- [ ] `ApplicationController` に内部API保護用の共通メソッド `require_internal_api_key` が定義されている
- [ ] `Client` モデルに `client_id`, `client_secret_hash`, `name`, `redirect_uris`, `allowed_scopes` のバリデーションが設定されている
- [ ] `Client#redirect_uris` / `Client#allowed_scopes` が配列として読み書きできる
- [ ] `rails db:seed` を実行してテスト用クライアントが投入できる

## 備考

本ブランチでは「RP 管理基盤の前提整備」に留める。API エンドポイントとサービス層の実装は後続ブランチで行う。
