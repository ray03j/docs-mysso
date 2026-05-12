# feature/database-schema/idp-client

## 目的

`idp-client` サービスの `clients` テーブルをマイグレーションで定義し、テスト用の初期データを `seeds.rb` に投入する。

## ブランチ名

`feature/database-schema/idp-client`

## 親ブランチ

`feature/database-schema/init`

## 作成・変更するファイル

### `idp-client/db/migrate/001_create_clients.rb`

```ruby
class CreateClients < ActiveRecord::Migration[7.1]
  def change
    create_table :clients do |t|
      t.string :client_id, null: false
      t.string :client_secret_hash, null: false
      t.string :name, null: false
      t.text :redirect_uris, null: false
      t.text :allowed_scopes, null: false
      t.timestamps
    end

    add_index :clients, :client_id, unique: true
  end
end
```

#### なぜこの設定か

- **`client_id` の `null: false` + UNIQUE インデックス**：OIDC で RP を識別する必須の外部公開識別子。重複は許容できないため DB レベルで制約する。
- **`client_secret_hash` の `null: false`**：機密情報の平文保存を避け、ハッシュ化後の値を保存する。`feature/idp-client-service` で `bcrypt` ハッシュ化を実装する。
- **`redirect_uris` / `allowed_scopes` の `text` 型**：配列や JSON 形式で複数の URI・スコープを保存するため、可変長の `text` を採用。PostgreSQL では `jsonb` への移行も可能だが、現時点ではシンプルな改行区切り or CSV 形式を想定し `text` で開始。Phase 2 で正規化を検討。
- **`name` の `null: false`**：管理画面での表示名として必須。

### `idp-client/db/seeds.rb`

```ruby
# テスト用クライアント（デモ RP 用）
Client.create!(
  client_id: 'demo_client',
  client_secret_hash: BCrypt::Password.create('demo_secret'),
  name: 'Demo Relying Party',
  redirect_uris: "http://localhost:5174/callback\nhttp://localhost:5174/",
  allowed_scopes: "openid profile email"
)
```

#### なぜこの設定か

- **`demo_client`**：`demo-rp/.env.example` で定義した `VITE_CLIENT_ID=demo_client` と一致させる。Phase 3 の認可フロー動作確認時に即座に使用できる。
- **`redirect_uris` の改行区切り**：複数 URI を 1 カラムに格納する簡易的な方法。`feature/idp-client-service` で配列変換ロジックを実装する。
- **`allowed_scopes`**：`openid`, `profile`, `email` は OIDC Core で定義される標準スコープ。デモ用に全て許可する。

## マージ基準（チェックリスト）

- [ ] `rails db:migrate` を `idp-client/` で実行して `clients` テーブルが作成される
- [ ] `rails db:seed` を `idp-client/` で実行してテスト用クライアントが投入できる
- [ ] `strong_migrations` で警告が出ない
- [ ] `clients` テーブルの `client_id` に UNIQUE 制約が設定されている

## 備考

本ブランチではテーブル定義のみ。RP 登録 API や検証ロジックは `feature/idp-client-service` で実装する。
