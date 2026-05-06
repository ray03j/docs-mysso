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

## マージ基準（チェックリスト）

- [ ] `docker compose up idp-auth` で Rails サーバ（ポート3000）が起動する
- [ ] `bundle exec rspec` が空テストで通る
- [ ] RuboCop が実行できる

## 備考

Rails 雛形は `rails new . --api --database=postgresql --skip-test` で生成したものをベースとする。
