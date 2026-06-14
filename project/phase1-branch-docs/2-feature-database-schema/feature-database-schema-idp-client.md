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

### `idp-client/app/models/client.rb`

```ruby
class Client < ApplicationRecord
  validates :client_id, presence: true, uniqueness: true
  validates :client_secret_hash, presence: true
  validates :name, presence: true
  validates :redirect_uris, presence: true
  validates :allowed_scopes, presence: true
end
```

#### なぜこの設定か

- **モデル雛形**：`db:seed` で `Client.create!` を実行するために最低限のモデルクラスが必要。
- **DB 制約に対応する最小限のバリデーション**：`client_id` は `NOT NULL` + `UNIQUE`、その他必須カラムも `NOT NULL` の制約があるため、モデル側で `presence` / `uniqueness` を設定しておく。これにより、DB 例外を発生させず、API としての扱いやすさを向上する。
- **`ApplicationRecord` を継承**：Rails 7.1 の標準的なモデル定義。後続ブランチで `client_secret` 検証メソッドなどを追加する。

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
    allowed_scopes: "openid profile email"
  )
  client.save!
end
```

#### なぜこの設定か

- **テスト用クライアント**：Phase 3 の認可フロー動作確認時に即座に使用できるデモ RP 用データ。development/test のみ投入する。
- **`client_id`**：`demo-rp/.env.example` で定義した `VITE_CLIENT_ID` と一致させる。
- **`BCrypt::Password.create`**：`bcrypt` gem を使用した安全なハッシュ化。平文の client_secret は保存しない。
- **環境ごとのコスト切り替え**：test 環境では `BCrypt::Engine::MIN_COST` を使用し、テスト実行速度を確保する。production/development では `DEFAULT_COST` を使用し、実環境に近い挙動を確認する。
- **`redirect_uris` の改行区切り**：複数 URI を 1 カラムに格納する簡易的な方法。`feature/idp-client-service` で配列変換ロジックを実装する。
- **`allowed_scopes`**：`openid`, `profile`, `email` は OIDC Core で定義される標準スコープ。デモ用に全て許可する。

## マージ基準（チェックリスト）

- [ ] `rails db:migrate` を `idp-client/` で実行して `clients` テーブルが作成される
- [ ] `rails db:seed` を `idp-client/` で実行してテスト用クライアントが投入できる
- [ ] `strong_migrations` で警告が出ない
- [ ] `clients` テーブルの `client_id` に UNIQUE 制約が設定されている

## 備考

本ブランチではテーブル定義のみ。RP 登録 API や検証ロジックは `feature/idp-client-service` で実装する。
