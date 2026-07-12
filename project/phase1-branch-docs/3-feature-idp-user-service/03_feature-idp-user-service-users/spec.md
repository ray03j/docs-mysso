# feature/idp-user-service/users — Spec

## 目次

- [`idp-user/spec/models/user_spec.rb`](#idp-userspecmodelsuser_specrb)
- [`idp-user/spec/requests/users_spec.rb`](#idp-userspecrequestsusers_specrb)
- [補足：bcrypt 検証の責務分担](#補足bcrypt-検証の責務分担)
- [補足：`valid_params` と `invalid_params`](#補足valid_params-と-invalid_params)

---

### `idp-user/spec/models/user_spec.rb`

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

### `idp-user/spec/requests/users_spec.rb`

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

#### なぜこの設定か

- **`change(User, :count).by(1)`**：DB レベルでレコードが増加したことを確認。単にレスポンスが成功しただけではなく、永続化を検証する。
- **`start_with('$2a$')`**：`bcrypt` ハッシュの標準的なプレフィックスを確認。`BCrypt::Password.new` で検証する方法もあるが、prefix チェックで「平文保存されていない」ことを簡潔に保証する。
- **重複メールのテスト**：DB レベルの UNIQUE 制約とモデルバリデーションの両方が機能することを確認。`create(:user)` は FactoryBot を使用し、テストデータの構築を DRY に保つ。

## 補足：bcrypt 検証の責務分担

リクエストスペックで `start_with('$2a$')` に留めているのは、**bcrypt の詳細な挙動はサービス層・モデル層のテストで既にカバーしている**ためです。

| レイヤー | テスト対象 | 検証内容 |
|---|---|---|
| サービス層 | `User::PasswordService` | bcrypt のハッシュ化・検証・ソルトの有無 |
| モデル層 | `User` | 保存時のハッシュ化、email の一意性、必須項目 |
| リクエスト層 | `POST /api/v1/users` | エンドポイントを通じた永続化、レスポンスの漏洩防止、bcrypt 形式での保存 |

リクエストスペックの責務は「エンドポイントを通じて正しくユーザーが作成され、平文パスワードが漏れていないこと」を確認することに留め、より詳細な暗号化の正確性は下位レイヤーに委ねています。

## 補足：`valid_params` と `invalid_params`

リクエストスペックでは、以下の 2 つの `let` でパラメータを定義しています。

| 名前 | 用途 | 内容 |
|---|---|---|
| `valid_params` | 正常系テスト | `email`, `password`, `name` がすべて有効なユーザー作成用パラメータ |
| `invalid_params` | 異常系テスト | `email`・`name` が空文字、`password` が短すぎるなど、バリデーションに引っかかるパラメータ |

- `valid_params` は「ユーザーが正常に作成されること」「正しいレスポンスが返ること」を検証する際に使用します。
- `invalid_params` は「バリデーションエラー時にユーザーが作成されないこと」「`422 Unprocessable Entity` が返ること」「エラーメッセージが含まれること」を検証する際に使用します。
- なお、`重複したメールアドレスの場合` も失敗を期待する**異常系（DB 状態違反系）**ですが、ここでは `valid_params` を使っています。失敗の原因は「パラメータが不正」ではなく「DB に既に同じ email のユーザーが存在する状態」だからです。
- 両方とも `user:` というネストされたキーで囲んでいるのは、Rails の Strong Parameters と同じ構造に合わせるためです。
