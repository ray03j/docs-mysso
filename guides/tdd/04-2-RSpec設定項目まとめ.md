---
title: RSpec 設定項目まとめ
created: 2026-07-10
tags: [rspec, testing, configuration]
---

# RSpec 設定項目まとめ

## 目次

- [はじめに](#はじめに)
- [FactoryBot 関連](#factorybot-関連)
- [DB・フィクスチャ関連](#dbフィクスチャ関連)
- [Spec タイプ・推論関連](#spec-タイプ推論関連)
- [エラー表示関連](#エラー表示関連)
- [出力・フォーマット関連](#出力フォーマット関連)
- [実行順序・パフォーマンス関連](#実行順序パフォーマンス関連)
- [モック・スタブ関連](#モックスタブ関連)
- [フック関連](#フック関連)
- [設定例全体](#設定例全体)

---

## はじめに

`spec_helper.rb` / `rails_helper.rb` の `RSpec.configure` ブロックで指定する代表的な設定項目をまとめる。

```ruby
RSpec.configure do |config|
  # 各種設定
end
```

---

## FactoryBot 関連

### `config.include FactoryBot::Syntax::Methods`

FactoryBot のメソッドをテスト内で省略して書けるようにする。

```ruby
config.include FactoryBot::Syntax::Methods
```

有効にすると以下のように書ける。

```ruby
# before
FactoryBot.create(:user)
FactoryBot.build(:user)

# after
create(:user)
build(:user)
attributes_for(:user)
```

---

## DB・フィクスチャ関連

### `config.use_transactional_fixtures = true`

各 example（テスト）の終了時に DB トランザクションをロールバックし、テスト間でデータが残らないようにする。

```ruby
config.use_transactional_fixtures = true
```

| 値 | 意味 |
|---|---|
| `true` | 各テストをトランザクションで囲み、終了時にロールバック（デフォルト） |
| `false` | ロールバックしない。DatabaseCleaner など別途クリーニングが必要 |

### `config.fixture_path = Rails.root.join('spec/fixtures')`

Rails の fixture ファイル（`users.yml` など）を置くパスを指定する。

```ruby
config.fixture_path = Rails.root.join('spec/fixtures')
```

FactoryBot を使う場合は指定不要なことが多い。

### `config.use_instantiated_fixtures = false`

fixture をハッシュではなくインスタンスとして読み込むかどうか。
FactoryBot を使うプロジェクトでは通常 `false` のまま。

---

## Spec タイプ・推論関連

### `config.infer_spec_type_from_file_location!`

ファイルの配置場所から自動的に spec type を推論する。

```ruby
config.infer_spec_type_from_file_location!
```

| 配置場所 | 推論される type |
|---|---|
| `spec/models/` | `type: :model` |
| `spec/requests/` | `type: :request` |
| `spec/controllers/` | `type: :controller` |
| `spec/features/` | `type: :feature` |

有効にすると、`RSpec.describe User, type: :model` の `type` 指定を省略できる。

### `config.infer_base_class_for_anonymous_controllers = false`

匿名コントローラの基底クラスを推論するかどうか。古い設定で、現在はあまり使われない。

---

## エラー表示関連

### `config.filter_rails_from_backtrace!`

エラー発生時の backtrace から、Rails フレームワーク内部のファイルを非表示にする。

```ruby
config.filter_rails_from_backtrace!
```

これにより、自分のコードの問題箇所が探しやすくなる。

### `config.filter_gems_from_backtrace('gem_name')`

指定した Gem の中身を backtrace から非表示にする。

```ruby
config.filter_gems_from_backtrace('factory_bot_rails')
config.filter_gems_from_backtrace('devise')
```

### `config.full_backtrace = true`

backtrace をすべて表示する。デバッグ時に一時的に有効にする。

```ruby
config.full_backtrace = true
```

---

## 出力・フォーマット関連

### `config.formatter = :documentation`

テスト結果の出力形式を指定する。

```ruby
config.formatter = :documentation
```

| 値 | 説明 |
|---|---|
| `:progress` | デフォルト。`.` で進捗を表示 |
| `:documentation` | テスト名を階層表示 |
| `:json` | JSON 形式で出力 |
| `:html` | HTML レポートを生成 |

### `config.color = true`

ターミナル出力に色をつける。

```ruby
config.color = true
```

### 複数フォーマッタの併用

```ruby
config.formatter = :progress
config.add_formatter(:html, 'tmp/rspec_report.html')
```

---

## 実行順序・パフォーマンス関連

### `config.order = :random`

テストの実行順序をランダムにする。

```ruby
config.order = :random
```

テスト間の依存関係を発見しやすくなる。

### `config.seed = 12345`

ランダム実行時のシード値を固定する。再現性が必要な場合に使う。

```ruby
config.seed = 12345
```

### `config.profile_examples = 10`

実行時間が長いテスト上位 10 件を表示する。

```ruby
config.profile_examples = 10
```

### `config.warnings = true`

Ruby の警告を表示する。

```ruby
config.warnings = true
```

---

## モック・スタブ関連

### `config.mock_with :rspec`

モックフレームワークとして RSpec 標準のものを使う。

```ruby
config.mock_with :rspec
```

| 値 | 説明 |
|---|---|
| `:rspec` | RSpec 標準のモック（デフォルト） |
| `:mocha` | Mocha を使う |
| `:flexmock` | FlexMock を使う |
| `:rr` | RR を使う |

### `config.expect_with :rspec`

expectation 構文として RSpec 標準のものを使う。

```ruby
config.expect_with :rspec
```

### `config.verify_partial_doubles = true`

モック化したオブジェクトに存在しないメソッドを呼び出そうとした場合にエラーにする。

```ruby
config.verify_partial_doubles = true
```

タイポやリファクタリング漏れを防ぐのに有効。

---

## フック関連

### `config.before(:each)` / `config.before(:suite)`

全テストの前に実行する処理を設定する。

```ruby
config.before(:each) do
  # 各 example の前に実行
end

config.before(:suite) do
  # テストスイート全体の前に1回実行
end
```

### `config.after(:each)` / `config.after(:suite)`

全テストの後に実行する処理を設定する。

```ruby
config.after(:each) do
  # 各 example の後に実行
end
```

### `config.around(:each)`

各 example の前後に処理を挟む。

```ruby
config.around(:each) do |example|
  puts "before: #{example.description}"
  example.run
  puts "after: #{example.description}"
end
```

---

## 設定例全体

```ruby
RSpec.configure do |config|
  # 環境
  ENV['RAILS_ENV'] ||= 'test'
  require_relative '../config/environment'
  abort('Production mode!') if Rails.env.production?

  # 必須 Gem
  require 'rspec/rails'
  require 'factory_bot_rails'

  # FactoryBot
  config.include FactoryBot::Syntax::Methods

  # DB
  config.use_transactional_fixtures = true

  # Spec type 推論
  config.infer_spec_type_from_file_location!

  # Backtrace
  config.filter_rails_from_backtrace!

  # 出力
  config.formatter = :documentation
  config.color = true

  # 実行順序
  config.order = :random

  # モック
  config.mock_with :rspec
  config.verify_partial_doubles = true
end
```

---

## 関連ファイル

- `idp-user/spec/spec_helper.rb`
- `idp-user/spec/rails_helper.rb`
