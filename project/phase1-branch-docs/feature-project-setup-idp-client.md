# feature/project-setup/idp-client

## 目的

クライアント管理サービス `idp-client` の Rails API 雛形を作成する。ポート3002で動作。

## ブランチ名

`feature/project-setup/idp-client`

## 親ブランチ

`feature/project-setup/init`

## 作成するファイル

### `idp-client/Gemfile`

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

### `idp-client/config/routes.rb`

```ruby
Rails.application.routes.draw do
  # 後続ブランチで定義
end
```

### `idp-client/config/database.yml`

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
  database: client_db

test:
  <<: *default
  database: client_db_test

production:
  <<: *default
  database: client_db
```

### その他必須ファイル

- `idp-client/Gemfile.lock`
- `idp-client/Rakefile`
- `idp-client/config.ru`
- `idp-client/config/boot.rb`
- `idp-client/config/environment.rb`
- `idp-client/config/environments/development.rb`
- `idp-client/config/environments/test.rb`
- `idp-client/config/environments/production.rb`
- `idp-client/config/initializers/`
- `idp-client/app/controllers/application_controller.rb`
- `idp-client/app/models/application_record.rb`
- `idp-client/db/seeds.rb`
- `idp-client/spec/spec_helper.rb`

## マージ基準（チェックリスト）

- [ ] `docker compose up idp-client` で Rails サーバ（ポート3002）が起動する
- [ ] `bundle exec rspec` が空テストで通る
- [ ] RuboCop が実行できる

## 備考

`idp-auth`, `idp-user` と構成は同一。サービス名とDB名のみ変更。
