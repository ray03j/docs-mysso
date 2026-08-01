# feature/idp-client-service/clients — Spec

## 目次

- [`idp-client/spec/requests/clients_spec.rb`](#idp-clientspecrequestsclients_specrb)
- [`idp-client/spec/models/client_spec.rb`](#idp-clientspecmodelsclient_specrb)

---

### `idp-client/spec/requests/clients_spec.rb`

```ruby
require 'spec_helper'

RSpec.describe 'Clients', type: :request do
  let(:internal_api_key) { ENV.fetch('INTERNAL_API_KEY') }
  let(:headers) { { 'X-Internal-API-Key' => internal_api_key } }

  describe 'POST /api/v1/clients' do
    let(:valid_params) do
      {
        client: {
          name: 'Test RP',
          redirect_uris: ['http://localhost:5174/callback'],
          allowed_scopes: %w[openid profile]
        }
      }
    end

    let(:invalid_params) do
      {
        client: {
          name: '',
          redirect_uris: ['not-a-url'],
          allowed_scopes: %w[openid invalid_scope]
        }
      }
    end

    context '有効なパラメータの場合' do
      it 'クライアントを作成する' do
        expect do
          post '/api/v1/clients', params: valid_params, headers: headers
        end.to change(Client, :count).by(1)
      end

      it '201 Created を返す' do
        post '/api/v1/clients', params: valid_params, headers: headers
        expect(response).to have_http_status(:created)
      end

      it 'client_id と平文 client_secret を返す' do
        post '/api/v1/clients', params: valid_params, headers: headers
        json = response.parsed_body
        expect(json).to include('client_id', 'client_secret')
        expect(json['client_secret']).to be_present
      end

      it 'client_secret_hash が bcrypt で保存される' do
        post '/api/v1/clients', params: valid_params, headers: headers
        client = Client.order(:created_at).last
        expect(client.client_secret_hash).to start_with('$2a$')
      end

      it 'レスポンスに client_secret_hash が含まれない' do
        post '/api/v1/clients', params: valid_params, headers: headers
        json = response.parsed_body
        expect(json).not_to include('client_secret_hash')
      end
    end

    context '無効なパラメータの場合' do
      it 'クライアントを作成しない' do
        expect do
          post '/api/v1/clients', params: invalid_params, headers: headers
        end.not_to change(Client, :count)
      end

      it '422 Unprocessable Content を返す' do
        post '/api/v1/clients', params: invalid_params, headers: headers
        expect(response).to have_http_status(:unprocessable_content)
      end

      it 'エラーメッセージを返す' do
        post '/api/v1/clients', params: invalid_params, headers: headers
        json = response.parsed_body
        expect(json['errors']).to be_present
      end
    end

    context '内部APIキーが不正または未指定の場合' do
      it '不正なキーでは 401 Unauthorized を返す' do
        post '/api/v1/clients',
             params: valid_params,
             headers: { 'X-Internal-API-Key' => 'wrong-key' }
        expect(response).to have_http_status(:unauthorized)
      end

      it 'ヘッダー未指定でも 401 Unauthorized を返す' do
        post '/api/v1/clients', params: valid_params
        expect(response).to have_http_status(:unauthorized)
      end

      it 'クライアントを作成しない' do
        expect do
          post '/api/v1/clients', params: valid_params
        end.not_to change(Client, :count)
      end
    end
  end

  describe 'GET /api/v1/clients' do
    let!(:client) { create(:client) }

    it '200 OK を返す' do
      get '/api/v1/clients', headers: headers
      expect(response).to have_http_status(:ok)
    end

    it '登録済みクライアントの一覧を返す' do
      get '/api/v1/clients', headers: headers
      json = response.parsed_body
      expect(json['clients'].length).to eq(1)
      expect(json['clients'].first['client_id']).to eq(client.client_id)
    end

    it '一覧に client_secret_hash が含まれない' do
      get '/api/v1/clients', headers: headers
      json = response.parsed_body
      expect(json['clients'].first).not_to include('client_secret_hash', 'client_secret')
    end

    it '内部 API キーが不正な場合は 401 を返す' do
      get '/api/v1/clients', headers: { 'X-Internal-API-Key' => 'wrong-key' }
      expect(response).to have_http_status(:unauthorized)
    end

    it 'ヘッダー未指定の場合は 401 を返す' do
      get '/api/v1/clients'
      expect(response).to have_http_status(:unauthorized)
    end
  end

  describe 'GET /api/v1/clients/:client_id' do
    let!(:client) { create(:client) }

    it '200 OK を返す' do
      get "/api/v1/clients/#{client.client_id}", headers: headers
      expect(response).to have_http_status(:ok)
    end

    it 'クライアント詳細を返す' do
      get "/api/v1/clients/#{client.client_id}", headers: headers
      json = response.parsed_body
      expect(json['client_id']).to eq(client.client_id)
      expect(json['name']).to eq(client.name)
      expect(json['redirect_uris']).to eq(client.redirect_uris)
      expect(json['allowed_scopes']).to eq(client.allowed_scopes)
    end

    it '詳細に client_secret_hash が含まれない' do
      get "/api/v1/clients/#{client.client_id}", headers: headers
      json = response.parsed_body
      expect(json).not_to include('client_secret_hash', 'client_secret')
    end

    it '存在しない client_id の場合は 404 を返す' do
      get '/api/v1/clients/non-existent-client-id', headers: headers
      expect(response).to have_http_status(:not_found)
    end

    it '内部 API キーが不正な場合は 401 を返す' do
      get "/api/v1/clients/#{client.client_id}",
          headers: { 'X-Internal-API-Key' => 'wrong-key' }
      expect(response).to have_http_status(:unauthorized)
    end

    it 'ヘッダー未指定の場合は 401 を返す' do
      get "/api/v1/clients/#{client.client_id}"
      expect(response).to have_http_status(:unauthorized)
    end
  end
end
```

#### なぜこの設定か

- **`change(Client, :count).by(1)`**：DB レベルでレコードが増加したことを確認。単にレスポンスが成功しただけではなく、永続化を検証する。
- **`start_with('$2a$')`**：`bcrypt` ハッシュの標準的なプレフィックスを確認。平文保存されていないことを簡潔に保証する。
- **無効なパラメータの検証**：不正なリダイレクトURI・サポート外スコープ・空の名前に対して 422 エラーが返ることを確認。バリデーションがモデル層と API 層の両方で機能することを検証する。
- **一覧・詳細の機密情報除外**：`client_secret` および `client_secret_hash` が一覧・詳細に含まれないことを確認。セキュリティ要件を満たす。
- **`show` は `client_id` 指定で取得**：`/authorize` エンドポイントから内部 API として呼ばれるため、`client_id` による検索と `redirect_uris` / `allowed_scopes` の返却が正しく動作することを検証する。
- **全リクエストに `X-Internal-API-Key` ヘッダーを付与**：全エンドポイントが内部APIキー必須のため、`headers` をトップレベルの `let` で共通化して付与する。各エンドポイントで「不正なキー」「ヘッダー未指定」が 401 となることも検証し、認証なし公開のリグレッションを防ぐ。

### `idp-client/spec/models/client_spec.rb`

```ruby
require 'spec_helper'

RSpec.describe Client, type: :model do
  describe 'バリデーション' do
    it '全必須項目が揃っている場合は有効' do
      client = build(:client)
      expect(client).to be_valid
    end

    it 'client_id が必須' do
      client = build(:client, client_id: nil)
      expect(client).not_to be_valid
      expect(client.errors[:client_id]).to be_present
    end

    it 'client_id は一意である' do
      duplicate_id = SecureRandom.uuid
      create(:client, client_id: duplicate_id)
      client = build(:client, client_id: duplicate_id)
      expect(client).not_to be_valid
      expect(client.errors[:client_id]).to be_present
    end

    it 'name が必須' do
      client = build(:client, name: nil)
      expect(client).not_to be_valid
    end

    it 'redirect_uris に無効な URL を許可しない' do
      client = build(:client, redirect_uris: ['not-a-url'])
      expect(client).not_to be_valid
      expect(client.errors[:redirect_uris]).to be_present
    end

    it 'allowed_scopes にサポート外スコープを許可しない' do
      client = build(:client, allowed_scopes: ['openid', 'admin'])
      expect(client).not_to be_valid
      expect(client.errors[:allowed_scopes]).to be_present
    end

    it 'client_secret が保存時に client_secret_hash へハッシュ化される' do
      client = Client.new(
        client_id: SecureRandom.uuid,
        name: 'Test',
        redirect_uris: ['http://localhost/callback'],
        allowed_scopes: ['openid']
      )
      client.client_secret = 'raw_secret'
      client.save!
      expect(client.client_secret_hash).to start_with('$2a$')
      expect(client.client_secret_valid?('raw_secret')).to be true
      expect(client.client_secret_valid?('wrong_secret')).to be false
    end
  end

  describe '配列変換' do
    it 'redirect_uris を配列として読み書きできる' do
      client = create(:client, redirect_uris: ['http://a/callback', 'http://b/callback'])
      expect(client.redirect_uris).to eq(['http://a/callback', 'http://b/callback'])
    end

    it 'allowed_scopes を配列として読み書きできる' do
      client = create(:client, allowed_scopes: ['openid', 'email'])
      expect(client.allowed_scopes).to eq(['openid', 'email'])
    end
  end
end
```

---

実装コードの詳細は [`feature.md`](feature.md) を参照。
