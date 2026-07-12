# feature/idp-user-service

## 目的

`idp-user` サービスにユーザー認証に必要な基盤を整え、サインアップ API とログイン検証 API を単一ブランチで実装する。後続の `idp-auth-service` から呼ばれる内部認証 API の提供を完了させる。

## ブランチ名

`feature/idp-user-service`

## 親ブランチ

`feature/database-schema`（`main` へのマージ完了後）

---

## 作成・変更するファイル

### 1. 共通設定・モデル（init 相当）

#### `idp-user/Gemfile`

```ruby
# 既存の gem に追加
gem 'bcrypt', '~> 3.1'
```

##### なぜこの設定か

- **`bcrypt '~> 3.1'`**：パスワードの安全なハッシュ化に必須。アダプティブハッシュ化アルゴリズムであり、コスト係数を調整して計算機の性能向上に対応できる。Rails エコシステムで最も広く採用されている。
- **マイクロサービス間共通化**：`idp-user` だけでなく、将来の `idp-client` の `client_secret` 検証でも同様の `bcrypt` ロジックを使い回せる。

#### `idp-user/config/routes.rb`

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :users, only: [:create]
      namespace :auth do
        post 'verify', to: 'verify#create'
      end
    end
  end
end
```

##### なぜこの設定か

- **`namespace :api` + `namespace :v1`**：API バージョニングの標準的な構成。将来の v2 追加時に互換性を保ちながら移行できる。内部マイクロサービスとして他サービスから呼ばれるため、明確なバージョン管理が重要。
- **`resources :users, only: [:create]`**：サインアップのみを提供。ユーザー情報の取得・更新・削除は `idp-auth` や管理画面経由で行うため、最小権限の原則に従う。
- **`post 'verify'`**：ログイン検証は「状態を変更しないが機密情報を扱う」操作のため、GET ではなく POST を採用。パスワードをクエリパラメータに含めず、HTTP ボディでの安全な送信を保証する。

#### `idp-user/app/models/user.rb`

```ruby
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true
  validates :name, presence: true, length: { maximum: 100 }
end
```

##### なぜこの設定か

- **`has_secure_password`**：Rails 標準の bcrypt によるパスワードハッシュ化機能。`password` / `password_confirmation` 仮想属性、`password` の `presence` バリデーション、認証メソッド `authenticate` を自動提供する。長年のセキュリティレビューと実績があり、独自実装では生じがちな「隠れた二重ハッシュ化」や「バリデーションとの競合」を回避できる。
- **`password_digest` カラムへの自動保存**：平文パスワードは永続化せず、保存時に自動的に bcrypt ハッシュが生成される。`before_validation` による自前のコールバックが不要になり、ハッシュ化忘れのリスクが減る。
- **`name` の必須化と長さ制限**：ユーザー表示名を必須とし、100文字を上限に制限。`feature/database-schema` では `name` は `null: false` と定義しており、モデル層の `presence: true` とDB層の `NOT NULL` 制約で二重化している。DB ストレージの無駄遣いと表示崩れを防ぐため長さも制限する。
- **`PasswordService` との併用**：`has_secure_password` を使うこととサービス層への切り出しは排他ではない。`User` モデルは標準機能に任せ、他ドメイン（将来の `Client` 等）や CLI・バッチからの再利用には `User::PasswordService` を使い分ける。必要に応じて `authenticate` メソッドをオーバーライドし、内部で `User::PasswordService.verify?` を呼び出すこともできる。

---

### 2. サービス層（auth 相当）

#### `idp-user/app/services/user/password_service.rb`

```ruby
require 'bcrypt'

class User
  class PasswordService
    def self.hash(password)
      BCrypt::Password.create(password)
    end

    def self.verify?(password, password_digest)
      return false if password.blank? || password_digest.blank?

      BCrypt::Password.new(password_digest) == password
    end
  end
end
```

##### なぜこの設定か

- **クラスメソッドとして定義**：状態を持たない純粋関数の集合。インスタンス化の必要がないためシンプルに実装する。
- **`verify?` のガード節**：`password` または `password_digest` が空の場合は即座に `false` を返し、`BCrypt::Errors::InvalidHash` を防ぐ。真偽値を返すメソッド名は Ruby の慣習に従い `?` で終わる。
- **`has_secure_password` との共存**：`User` モデルは `has_secure_password` にお任せするが、bcrypt のラッパー処理を `PasswordService` に集約しておくことで、ハッシュ化アルゴリズムの変更や他ドメインからの再利用が容易になる。

#### `idp-user/app/services/user/authentication_service.rb`

```ruby
class User
  class AuthenticationService
    def self.authenticate(email, password)
      user = User.find_by(email: email)
      return nil unless user

      return user if PasswordService.verify?(password, user.password_digest)

      nil
    end
  end
end
```

##### なぜこの設定か

- **戻り値は `User` または `nil`**：認証成功時はユーザー情報を返し、失敗時は `nil` を返す。`idp-auth` から呼び出した際に「認証成功 + ユーザー情報取得」を一気に行える。
- **タイミング攻撃**：現時点では「ユーザーが存在しない場合は即座に `nil` を返す」ため、メールアドレスの存在有無による処理時間の差が生じる。厳密なタイミング攻撃対策は Phase 2 で実装する予定。

---

### 3. コントローラー（users + auth 相当）

#### `idp-user/app/controllers/api/v1/users_controller.rb`

```ruby
module Api
  module V1
    class UsersController < ApplicationController
      def create
        user = User.new(user_params)
        if user.save
          render json: { id: user.id, email: user.email, name: user.name }, status: :created
        else
          render json: { errors: user.errors.full_messages }, status: :unprocessable_entity
        end
      end

      private

      def user_params
        params.require(:user).permit(:email, :password, :name)
      end
    end
  end
end
```

##### なぜこの設定か

- **`params.require(:user)`**：Strong Parameters で `user` キーの存在を必須とし、予期しないパラメータの注入を防ぐ。
- **`status: :created`（201）**：リソース作成成功時は HTTP 201 を返す。RESTful API の慣習に従う。
- **`status: :unprocessable_entity`（422）**：バリデーションエラー時は HTTP 422 を返し、フロントエンドが「内容が不正」を判断できる。
- **レスポンスに `password_digest` を含めない**：セキュリティ上の理由から、ハッシュ化されたパスワードは API レスポンスに絶対に含めない。

#### `idp-user/app/controllers/api/v1/auth/verify_controller.rb`

```ruby
module Api
  module V1
    module Auth
      class VerifyController < ApplicationController
        def create
          user = User::AuthenticationService.authenticate(
            params[:email],
            params[:password]
          )

          if user
            render json: success_response(user), status: :ok
          else
            render json: { code: 'INVALID_CREDENTIALS' }, status: :unauthorized
          end
        end

        private

        def success_response(user)
          {
            user: {
              id: user.id,
              email: user.email,
              name: user.name
            }
          }
        end
      end
    end
  end
end
```

##### なぜこの設定か

- **`params[:email]` / `params[:password]`**：ログイン検証は「単一操作」であるため、フラットな構造を採用。`idp-auth` からの内部API呼び出し時に JSON ボディをシンプルに保てる。
- **`HTTP ステータス + 機械可読な code`**：認証成功時は `200 OK` + ユーザー情報、認証失敗時は `401 Unauthorized` + `code: INVALID_CREDENTIALS` を返す。`idp-auth` 側は HTTP ステータスで判定でき、詳細な理由が必要な場合は `code` を参照できる。
- **`status: :unauthorized`（401）**：認証失敗時は HTTP 401 を返し、RFC 7235 に準拠して「認証が必要である」を明確に伝える。「メールアドレスは存在するがパスワードが違う」場合と「メールアドレスが存在しない」場合を区別しない。

---

### 4. テスト（全統合）

#### `idp-user/spec/factories/users.rb`

```ruby
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { 'password123' }
    name { 'Test User' }
  end
end
```

##### なぜこの設定か

- **`sequence(:email)`**：各ファクトリで一意なメールアドレスを自動生成。`uniqueness` バリデーションがあるため、固定値だと複数の `create(:user)` で衝突する。
- **`password` 属性**：`has_secure_password` が提供する `password` 仮想属性に対応。ファクトリ作成時に平文パスワードを渡すと、保存時に自動的に `password_digest` へハッシュ化される。

#### `idp-user/spec/models/user_spec.rb`

```ruby
require 'rails_helper'

RSpec.describe User, type: :model do
  describe 'バリデーション' do
    it 'email, password, name が必須' do
      user = User.new
      expect(user).not_to be_valid
      expect(user.errors[:email]).to be_present
      expect(user.errors[:password]).to be_present
      expect(user.errors[:name]).to be_present
    end

    it 'email は一意である' do
      create(:user, email: 'test@example.com')
      user = build(:user, email: 'test@example.com')
      expect(user).not_to be_valid
      expect(user.errors[:email]).to be_present
    end

    it 'password が present の場合、保存時に password_digest へハッシュ化される' do
      user = User.new(email: 'test@example.com', password: 'password123', name: 'Test')
      user.save!
      expect(user.password_digest).to start_with('$2a$')
      expect(user.password_digest).not_to eq('password123')
    end
  end
end
```

#### `idp-user/spec/services/user/password_service_spec.rb`

```ruby
require 'rails_helper'

RSpec.describe User::PasswordService do
  describe '.hash' do
    it 'bcrypt ハッシュを生成する' do
      hash = described_class.hash('password123')
      expect(hash).to start_with('$2a$')
    end

    it '同じ平文でも異なるハッシュを生成する（ソルト付き）' do
      hash1 = described_class.hash('password123')
      hash2 = described_class.hash('password123')
      expect(hash1).not_to eq(hash2)
    end
  end

  describe '.verify?' do
    let(:password_digest) { BCrypt::Password.create('password123') }

    it '正しいパスワードで true を返す' do
      expect(described_class.verify?('password123', password_digest)).to be true
    end

    it '誤ったパスワードで false を返す' do
      expect(described_class.verify?('wrongpassword', password_digest)).to be false
    end

    it '空のパスワードで false を返す' do
      expect(described_class.verify?('', password_digest)).to be false
    end

    it '空のハッシュで false を返す' do
      expect(described_class.verify?('password123', '')).to be false
    end
  end
end
```

#### `idp-user/spec/services/user/authentication_service_spec.rb`

```ruby
require 'rails_helper'

RSpec.describe User::AuthenticationService do
  let!(:user) { create(:user, email: 'test@example.com', password: 'password123') }

  describe '.authenticate' do
    context '正しいメール・パスワードの場合' do
      it 'ユーザーを返す' do
        result = described_class.authenticate('test@example.com', 'password123')
        expect(result).to eq(user)
      end
    end

    context '誤ったパスワードの場合' do
      it 'nil を返す' do
        result = described_class.authenticate('test@example.com', 'wrongpassword')
        expect(result).to be_nil
      end
    end

    context '存在しないメールの場合' do
      it 'nil を返す' do
        result = described_class.authenticate('nonexistent@example.com', 'password123')
        expect(result).to be_nil
      end
    end
  end
end
```

#### `idp-user/spec/requests/users_spec.rb`

```ruby
require 'rails_helper'

RSpec.describe 'Users', type: :request do
  describe 'POST /api/v1/users' do
    let(:valid_params) do
      {
        user: {
          email: 'test@example.com',
          password: 'password123',
          name: 'Test User'
        }
      }
    end

    let(:invalid_params) do
      {
        user: {
          email: '',
          password: 'short',
          name: ''
        }
      }
    end

    context '有効なパラメータの場合' do
      it 'ユーザーを作成する' do
        expect {
          post '/api/v1/users', params: valid_params
        }.to change(User, :count).by(1)
      end

      it '201 Created を返す' do
        post '/api/v1/users', params: valid_params
        expect(response).to have_http_status(:created)
      end

      it '作成されたユーザーの情報を返す（password_digest 除く）' do
        post '/api/v1/users', params: valid_params
        json = JSON.parse(response.body)
        expect(json).to include('email', 'name', 'id')
        expect(json).not_to include('password_digest')
      end

      it 'password_digest が bcrypt で保存される' do
        post '/api/v1/users', params: valid_params
        user = User.find_by(email: 'test@example.com')
        expect(user.password_digest).to start_with('$2a$') # bcrypt prefix
      end
    end

    context '無効なパラメータの場合' do
      it 'ユーザーを作成しない' do
        expect {
          post '/api/v1/users', params: invalid_params
        }.not_to change(User, :count)
      end

      it '422 Unprocessable Entity を返す' do
        post '/api/v1/users', params: invalid_params
        expect(response).to have_http_status(:unprocessable_entity)
      end

      it 'エラーメッセージを返す' do
        post '/api/v1/users', params: invalid_params
        json = JSON.parse(response.body)
        expect(json['errors']).to be_present
      end
    end

    context '重複したメールアドレスの場合' do
      before { create(:user, email: 'test@example.com') }

      it 'ユーザーを作成しない' do
        expect {
          post '/api/v1/users', params: valid_params
        }.not_to change(User, :count)
      end

      it '422 を返す' do
        post '/api/v1/users', params: valid_params
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
end
```

#### `idp-user/spec/requests/auth_verify_spec.rb`

```ruby
require 'rails_helper'

RSpec.describe 'Auth Verify', type: :request do
  let!(:user) { create(:user, email: 'test@example.com', password: 'password123') }

  describe 'POST /api/v1/auth/verify' do
    context '正しい認証情報の場合' do
      it '200 OK を返す' do
        post '/api/v1/auth/verify', params: { email: 'test@example.com', password: 'password123' }
        expect(response).to have_http_status(:ok)
      end

      it 'ユーザー情報を返す（password_digest 除く）' do
        post '/api/v1/auth/verify', params: { email: 'test@example.com', password: 'password123' }
        json = response.parsed_body
        expect(json['user']).to include('id', 'email', 'name')
        expect(json['user']).not_to include('password_digest')
      end
    end

    context '誤ったパスワードの場合' do
      it '401 Unauthorized を返す' do
        post '/api/v1/auth/verify', params: { email: 'test@example.com', password: 'wrongpassword' }
        expect(response).to have_http_status(:unauthorized)
      end

      it 'code: INVALID_CREDENTIALS を返す' do
        post '/api/v1/auth/verify', params: { email: 'test@example.com', password: 'wrongpassword' }
        json = response.parsed_body
        expect(json['code']).to eq('INVALID_CREDENTIALS')
      end
    end

    context '存在しないメールの場合' do
      it '401 Unauthorized を返す' do
        post '/api/v1/auth/verify', params: { email: 'nonexistent@example.com', password: 'password123' }
        expect(response).to have_http_status(:unauthorized)
      end

      it 'code: INVALID_CREDENTIALS を返す' do
        post '/api/v1/auth/verify', params: { email: 'nonexistent@example.com', password: 'password123' }
        json = response.parsed_body
        expect(json['code']).to eq('INVALID_CREDENTIALS')
      end
    end
  end
end
```

---

## 実装順序

以下の順序で実装することで、各層の依存関係を自然に解決する。

1. **`Gemfile` に `bcrypt` を追加し `bundle install`**
2. **`User` モデルに `has_secure_password` とバリデーションを実装**
3. **`PasswordService` を実装**（`AuthenticationService` や将来の他ドメインで再利用するため）
4. **`AuthenticationService` を実装**（`PasswordService` と `User` モデルが依存）
5. **`routes.rb` を設定**
6. **`UsersController` を実装**
7. **`Auth::VerifyController` を実装**（`AuthenticationService` が依存）
8. **ファクトリ・モデルスペックを実装**
9. **サービススペックを実装**
10. **リクエストスペックを実装**（統合テスト）

---

## マージ基準（チェックリスト）

### 基盤
- [ ] `idp-user/Gemfile` に `bcrypt` が含まれている
- [ ] `bundle install` が成功する
- [ ] `idp-user/config/routes.rb` に `/api/v1/users` と `/api/v1/auth/verify` が定義されている
- [ ] `rails routes` で上記ルートが確認できる

### モデル・サービス
- [ ] `User` モデルのバリデーション（`email` の `presence` / `uniqueness`、`name` の `presence` / `length`）が正しく機能する
- [ ] `password` が present の場合、`has_secure_password` によって `password_digest` にハッシュ化される
- [ ] `PasswordService` の単体テストが通る（bcrypt cost 10以上、ソルト付き）
- [ ] `AuthenticationService` の単体テストが通る

### サインアップ API
- [ ] サインアップ API でユーザーが作成される
- [ ] `password_digest` が bcrypt で保存される（`$2a$` prefix）
- [ ] レスポンスに `password_digest` が含まれない
- [ ] 重複メールアドレスで 422 エラーが返る
- [ ] 無効なパラメータで 422 エラーが返る

### ログイン検証 API
- [ ] 正しいメール・パスワードに対して 200 OK + ユーザー情報を返す
- [ ] 誤ったパスワードを拒否する（401 + `code: INVALID_CREDENTIALS`）
- [ ] 存在しないメールも拒否する（401 + `code: INVALID_CREDENTIALS`）
- [ ] レスポンスに `password_digest` が含まれない

### テスト全体
- [ ] `user_spec.rb` が全て通る
- [ ] `password_service_spec.rb` が全て通る
- [ ] `authentication_service_spec.rb` が全て通る
- [ ] `users_spec.rb` が全て通る
- [ ] `auth_verify_spec.rb` が全て通る
- [ ] `db/seeds.rb` が正常に動作し、テスト用ユーザー初期データが投入できる（冪等・development/test 限定）

---

## 備考

- 本ブランチは元々 `feature/idp-user-service/init`、`feature/idp-user-service/users`、`feature/idp-user-service/auth-verify` の3ブランチに分けていたものを統合したもの。
- `idp-auth` からの内部API呼び出しは、後続の `feature/idp-auth-service` ブランチで実装される。本ブランチでは `idp-user` 単体での動作を保証する。
- タイミング攻撃対策（「存在しないダミーユーザーで `BCrypt::Password.new` を実行して処理時間を均一化する」方式）は Phase 2 で実装する予定。
