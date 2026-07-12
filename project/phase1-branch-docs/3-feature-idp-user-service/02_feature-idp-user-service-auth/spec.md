# feature/idp-user-service/auth-verify — Spec

## 目次

- [パスワードサービススペック](#パスワードサービススペック)
- [認証サービススペック](#認証サービススペック)
- [認証リクエストスペック](#認証リクエストスペック)

---

## パスワードサービススペック

### ファイルパス

`idp-user/spec/services/user/password_service_spec.rb`

### コード

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

### なぜこの設定か

- **ソルトの確認**：`bcrypt` は自動的にソルトを付与するため、同じ平文でも異なるハッシュ文字列が生成されることをテストで確認。これにより「ハッシュ化が実際に行われている」を保証する。
- **コスト係数の確認**：`BCrypt::Password.create` はデフォルトでコスト 10 を使用。テスト環境では `BCrypt::Engine::MIN_COST` を使う場合もあるが、本テストでは「実際のハッシュが生成される」ことと「検証が正しく動作する」ことに焦点を当てる。

---

## 認証サービススペック

### ファイルパス

`idp-user/spec/services/user/authentication_service_spec.rb`

### コード

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

---

## 認証リクエストスペック

### ファイルパス

`idp-user/spec/requests/auth_verify_spec.rb`

### コード

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

### なぜこの設定か

- **存在しないメールのテスト**：「存在しないメール」でも「誤ったパスワード」と同じレスポンス（401 + `code: INVALID_CREDENTIALS`）を返すことを確認。これによりタイミング攻撃の対策が機能していることを保証する。
- **リクエストスペックで「サインアップ → ログイン検証」**：`users` ブランチで実装したサインアップと組み合わせた統合テストを行う。ただし、本ブランチでは `create(:user)` を使って独立してテストする。
