# feature/idp-client-service/verify — Spec

## 目次

- [`idp-client/spec/requests/verify_spec.rb`](#idp-clientspecrequestsverify_specrb)
- [`idp-client/spec/services/client_verification_service_spec.rb`](#idp-clientspecservicesclient_verification_service_specrb)

---

### `idp-client/spec/requests/verify_spec.rb`

```ruby
require 'spec_helper'

RSpec.describe 'Verify', type: :request do
  let(:internal_api_key) { ENV.fetch('INTERNAL_API_KEY', 'test_internal_api_key') }
  let!(:client) { create(:client, client_secret: 'correct_secret') }

  describe 'POST /api/v1/clients/verify' do
    context '内部APIキーが正しい場合' do
      let(:headers) { { 'X-Internal-API-Key' => internal_api_key } }

      context '有効な client_id / client_secret の場合' do
        it '200 OK を返す' do
          post '/api/v1/clients/verify',
               params: { client_id: client.client_id, client_secret: 'correct_secret' },
               headers: headers
          expect(response).to have_http_status(:ok)
        end

        it 'valid: true とクライアント情報を返す' do
          post '/api/v1/clients/verify',
               params: { client_id: client.client_id, client_secret: 'correct_secret' },
               headers: headers
          json = response.parsed_body
          expect(json['valid']).to be true
          expect(json['client_id']).to eq(client.client_id)
          expect(json['redirect_uris']).to eq(client.redirect_uris)
          expect(json['allowed_scopes']).to eq(client.allowed_scopes)
        end

        it 'client_secret_hash を含めない' do
          post '/api/v1/clients/verify',
               params: { client_id: client.client_id, client_secret: 'correct_secret' },
               headers: headers
          json = response.parsed_body
          expect(json).not_to include('client_secret_hash')
        end
      end

      context 'redirect_uri が指定された場合' do
        it '登録済みの redirect_uri なら 200 OK を返す' do
          post '/api/v1/clients/verify',
               params: {
                 client_id: client.client_id,
                 client_secret: 'correct_secret',
                 redirect_uri: client.redirect_uris.first
               },
               headers: headers
          expect(response).to have_http_status(:ok)
        end

        it '未登録の redirect_uri なら 401 Unauthorized を返す' do
          post '/api/v1/clients/verify',
               params: {
                 client_id: client.client_id,
                 client_secret: 'correct_secret',
                 redirect_uri: 'http://evil.example/callback'
               },
               headers: headers
          expect(response).to have_http_status(:unauthorized)
        end
      end

      context '無効な client_secret の場合' do
        it '401 Unauthorized を返す' do
          post '/api/v1/clients/verify',
               params: { client_id: client.client_id, client_secret: 'wrong_secret' },
               headers: headers
          expect(response).to have_http_status(:unauthorized)
        end

        it 'valid: false を返す' do
          post '/api/v1/clients/verify',
               params: { client_id: client.client_id, client_secret: 'wrong_secret' },
               headers: headers
          json = response.parsed_body
          expect(json['valid']).to be false
        end
      end

      context '存在しない client_id の場合' do
        it '401 Unauthorized を返す' do
          post '/api/v1/clients/verify',
               params: { client_id: 'non-existent', client_secret: 'correct_secret' },
               headers: headers
          expect(response).to have_http_status(:unauthorized)
        end
      end
    end

    context '内部APIキーが不正または未指定の場合' do
      it '401 Unauthorized を返す' do
        post '/api/v1/clients/verify',
             params: { client_id: client.client_id, client_secret: 'correct_secret' },
             headers: { 'X-Internal-API-Key' => 'wrong-key' }
        expect(response).to have_http_status(:unauthorized)
      end

      it 'ヘッダー未指定でも 401 Unauthorized を返す' do
        post '/api/v1/clients/verify',
             params: { client_id: client.client_id, client_secret: 'correct_secret' }
        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

#### なぜこの設定か

- **内部APIキー認証の検証**：正しいキーでアクセス可能、不正・未指定のキーで 401 となることを確認。マイクロサービス間通信の保護を担保する。
- **正誤両パターンの検証**：有効な組み合わせで成功、無効なシークレット・存在しないクライアントで失敗することを確認。
- **`redirect_uri` 検証の検証**：登録済み URI で成功、未登録 URI で 401 となることを確認。
- **レスポンスの機密情報除外**：`client_secret_hash` が内部APIの応答にも含まれないことを確認。

### `idp-client/spec/services/client_verification_service_spec.rb`

```ruby
require 'spec_helper'

RSpec.describe ClientVerificationService do
  let!(:client) { create(:client, client_secret: 'correct_secret') }

  describe '.verify' do
    context '有効な client_id / client_secret の場合' do
      it 'valid: true を返す' do
        result = described_class.verify(client_id: client.client_id, client_secret: 'correct_secret')
        expect(result[:valid]).to be true
        expect(result[:client]).to eq(client)
      end
    end

    context '無効な client_secret の場合' do
      it 'valid: false を返す' do
        result = described_class.verify(client_id: client.client_id, client_secret: 'wrong_secret')
        expect(result[:valid]).to be false
        expect(result[:error]).to eq('invalid client_secret')
      end
    end

    context '存在しない client_id の場合' do
      it 'valid: false を返す' do
        result = described_class.verify(client_id: 'non-existent', client_secret: 'correct_secret')
        expect(result[:valid]).to be false
        expect(result[:error]).to eq('client not found')
      end
    end

    context 'client_id が空の場合' do
      it 'valid: false を返す' do
        result = described_class.verify(client_id: '', client_secret: 'correct_secret')
        expect(result[:valid]).to be false
        expect(result[:error]).to eq('client_id is required')
      end
    end

    context 'client_secret が空の場合' do
      it 'valid: false を返す' do
        result = described_class.verify(client_id: client.client_id, client_secret: '')
        expect(result[:valid]).to be false
        expect(result[:error]).to eq('client_secret is required')
      end
    end

    context 'redirect_uri が指定された場合' do
      it '登録済みの redirect_uri なら valid: true を返す' do
        result = described_class.verify(
          client_id: client.client_id,
          client_secret: 'correct_secret',
          redirect_uri: client.redirect_uris.first
        )
        expect(result[:valid]).to be true
      end

      it '未登録の redirect_uri なら valid: false を返す' do
        result = described_class.verify(
          client_id: client.client_id,
          client_secret: 'correct_secret',
          redirect_uri: 'http://evil.example/callback'
        )
        expect(result[:valid]).to be false
        expect(result[:error]).to eq('invalid redirect_uri')
      end
    end
  end
end
```

#### なぜこの設定か

- **サービス層の単体テスト**：コントローラを介さず、検証ロジック自体を直接テスト。エッジケース（空文字、存在しないクライアント、無効なシークレット、未登録 redirect_uri）を網羅する。
- **早期リターンの検証**：空文字チェックが正しく動作し、不要な DB クエリを発生させないことを確認。

---

実装コードの詳細は [`feature.md`](feature.md) を参照。
