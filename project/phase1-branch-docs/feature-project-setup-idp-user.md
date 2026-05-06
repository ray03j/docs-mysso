# feature/project-setup/idp-user

## 目的

ユーザー管理サービス `idp-user` の Rails API 雛形を作成する。ポート3001で動作。

## ブランチ名

`feature/project-setup/idp-user`

## 親ブランチ

`feature/project-setup/init`

## 作成するファイル

### `idp-user/Gemfile`

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

### `idp-user/config/application.rb`

```ruby
require_relative 'boot'
require 'rails/all'

module IdpUser
  class Application < Rails::Application
    config.load_defaults 7.1
    config.api_only = true
    config.time_zone = 'Tokyo'
  end
end
```

### `idp-user/config/routes.rb`

```ruby
Rails.application.routes.draw do
  # 後続ブランチで定義
end
```

### `idp-user/config/database.yml`

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
  database: user_db

test:
  <<: *default
  database: user_db_test

production:
  <<: *default
  database: user_db
```

### その他必須ファイル

- `idp-user/Gemfile.lock`
- `idp-user/Rakefile`
- `idp-user/config.ru`
- `idp-user/config/boot.rb`
- `idp-user/config/environment.rb`
- `idp-user/config/environments/development.rb`
- `idp-user/config/environments/test.rb`
- `idp-user/config/environments/production.rb`
- `idp-user/config/initializers/`
- `idp-user/app/controllers/application_controller.rb`
- `idp-user/app/models/application_record.rb`
- `idp-user/db/seeds.rb`
- `idp-user/spec/spec_helper.rb`

## マージ基準（チェックリスト）

- [ ] `docker compose up idp-user` で Rails サーバ（ポート3001）が起動する
- [ ] `bundle exec rspec` が空テストで通る
- [ ] RuboCop が実行できる

## 備考

`idp-auth` と構成は同一。サービス名とDB名のみ変更。
