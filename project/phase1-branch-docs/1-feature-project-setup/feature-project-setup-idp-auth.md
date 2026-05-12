# feature/project-setup/idp-auth

## 目的

認可サービス `idp-auth` の Rails API 雛形を作成する。ポート3000で動作。

## ブランチ名

`feature/project-setup/idp-auth`

## 親ブランチ

`feature/project-setup/init`

## 作成するファイル

### `idp-auth/Gemfile`

```ruby
source 'https://rubygems.org'
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby '3.3.0'

gem 'rails', '~> 7.1'
gem 'pg', '~> 1.5'
gem 'puma', '~> 6.0'
gem 'bcrypt', '~> 3.1'
gem 'strong_migrations'

group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'rubocop', require: false
  gem 'rubocop-rails', require: false
end
```

#### なぜこの設定か

- **`ruby '3.3.0'`**：Ruby 3.3 は現時点での最新安定版であり、パフォーマンス改善（YJIT など）とセキュリティパッチの観点から採用。パッチバージョンまで固定することで、開発者間・CI 間の Ruby バージョン不一致を防ぐ。
- **`rails '~> 7.1'`**：Rails 7.1 は安定版の最新マイナーであり、API モード・Dockerfile 生成・暗号化機能の強化が含まれている。7.0 より新しく、7.2（未リリースや最新）に比べてエコシステムの互換性が高い。
- **`pg '~> 1.5'`**：PostgreSQL 16 と公式アダプタの組み合わせで検証されているバージョン。Rails 7.1 のデフォルト推奨。
- **`puma '~> 6.0'`**：Rails 7.1 のデフォルトサーバー。マルチスレッド・マルチプロセス両方に対応し、コンテナ環境でも広く使われている。
- **`bcrypt '~> 3.1'`**：パスワードハッシュ化に必須。OIDC の認可サーバーではクライアントシークレットやユーザーパスワードの安全な保存に必要。
- **`strong_migrations`**：本番環境での危険なマイグレーション（カラム削除、デフォルト値追加など）を開発段階で検出し、ダウンタイムゼロの方法を促す。マイクロサービス化してもDB停止は許容できないため、初期段階から導入。
- **`rspec-rails` + `factory_bot_rails`**：Rails 標準の Minitest よりも表現力が高く、チーム内で RSpec を採用する方針のため。`factory_bot` はテストデータ構築のボイラープレートを減らす。
- **`rubocop` + `rubocop-rails`**：静的解析でコードスタイルを統一。`require: false` で Rails 起動時のオーバーヘッドを排除し、CLI 実行時のみ読み込む。

### `idp-auth/config/application.rb`

```ruby
require_relative 'boot'
require 'rails/all'

module IdpAuth
  class Application < Rails::Application
    config.load_defaults 7.1
    config.api_only = true
    config.time_zone = 'Tokyo'
  end
end
```

#### なぜこの設定か

- **`config.load_defaults 7.1`**：Rails 7.1 の新規アプリとして動作し、非推奨警告や古い設定の継承を防ぐ。フレームワークの最新のセキュリティ・パフォーマンス設定を自動適用するため。
- **`config.api_only = true`**：ビュー層（ActionView）、セッション Cookie、CSRF トークンなど、API サーバーに不要なミドルウェアを除外し、起動時間とメモリフットプリントを削減。OIDC エンドポイントは JSON 応答のみなので必須。
- **`config.time_zone = 'Tokyo'`**：日本向けサービスであり、ログ・DB タイムスタンプの見直しやバッチ処理のスケジュールを JST 基準で扱えるようにする。`UTC` 保存 + `Tokyo` 表示が Rails のデフォルト動作となる。

### `idp-auth/config/routes.rb`

```ruby
Rails.application.routes.draw do
  # 後続ブランチで定義
end
```

### `idp-auth/config/database.yml`

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: <%= ENV.fetch("DATABASE_HOST") { "postgres" } %>
  username: <%= ENV.fetch("POSTGRES_USER") %>
  password: <%= ENV.fetch("POSTGRES_PASSWORD") %>

development:
  <<: *default
  database: auth_db

test:
  <<: *default
  database: auth_db_test

production:
  <<: *default
  database: auth_db
```

#### なぜこの設定か

- **`host: <%= ENV.fetch("DATABASE_HOST") { "postgres" } %>`**：Docker Compose 内ではサービス名 `postgres` が DNS 解決されるため、デフォルト値を `postgres` にする。ローカルネイティブ開発時は `.env` で `localhost` に上書き可能。
- **`ENV.fetch` で必須項目はデフォルトなし**：`POSTGRES_USER`/`POSTGRES_PASSWORD` は接続に必須であり、未設定時に即座に `KeyError` を発生させることで、「.env を忘れて動かなくてもnilで謎のエラーになる」を防ぐ。
- **`database: auth_db` / `auth_db_test`**：開発・本番で同じ DB 名を使うことで、構成の複雑化を避ける。test はsuffixを付けて分離し、開発データを壊さないようにする。

### その他必須ファイル

- `idp-auth/Gemfile.lock`（`bundle install` 後に生成）
- `idp-auth/Rakefile`
- `idp-auth/config.ru`
- `idp-auth/config/boot.rb`
- `idp-auth/config/environment.rb`
- `idp-auth/config/environments/development.rb`
- `idp-auth/config/environments/test.rb`
- `idp-auth/config/environments/production.rb`
- `idp-auth/config/initializers/`（空ディレクトリ）
- `idp-auth/app/controllers/application_controller.rb`
- `idp-auth/app/models/application_record.rb`
- `idp-auth/db/seeds.rb`
- `idp-auth/spec/spec_helper.rb`

#### なぜこのファイル群か

- **標準 Rails 雛形**：`rails new . --api --database=postgresql --skip-test` で生成されるファイル群をそのまま採用。これらが欠けると `rails server` `rspec` `rubocop` が正常に起動しない。
- **`config/environments/*.rb`**：Rails は起動時に必ず環境設定ファイルを読み込む。API モード用の最小設定（ログ出力、キャッシュ、DB 接続など）を含む。
- **`application_controller.rb` / `application_record.rb`**：全コントローラー・モデルの継承元。雛形段階では空でも存在しないと後続ブランチで継承エラーが発生する。
- **`spec/spec_helper.rb`**：RSpec のグローバル設定ファイル。`rspec-rails` インストール時に生成される標準テンプレートを使用。

## マージ基準（チェックリスト）

- [ ] `docker compose up idp-auth` で Rails サーバ（ポート3000）が起動する
- [ ] `bundle exec rspec` が空テストで通る
- [ ] RuboCop が実行できる

## 備考

Rails 雛形は `rails new . --api --database=postgresql --skip-test` で生成したものをベースとする。
