---
title: RSpec フォーマッタと出力設定
created: 2026-07-10
tags: [rspec, testing, formatter]
---

# RSpec フォーマッタと出力設定

## 目次

- [はじめに](#はじめに)
- [現在の設定](#現在の設定)
- [標準フォーマッタ一覧](#標準フォーマッタ一覧)
- [設定方法](#設定方法)
  - [コマンドライン](#コマンドライン)
  - [`.rspec` ファイル](#rspec-ファイル)
  - [`spec_helper.rb` / `rails_helper.rb`](#spec_helperrb--rails_helperrb)
- [複数フォーマッタの併用](#複数フォーマッタの併用)
- [外部 gem フォーマッタ](#外部-gem-フォーマッタ)
- [おすすめ組み合わせ](#おすすめ組み合わせ)

---

## はじめに

RSpec では、テスト実行結果の表示形式を「フォーマッタ」で切り替えられる。  
`idp-user/.rspec` で `--format documentation` を設定しているため、現在はテスト名が階層表示されている。

## 現在の設定

`idp-user/.rspec`

```text
--format documentation
--color
--require spec_helper
```

実行イメージ：

```text
User
  バリデーション
    email, password, name が必須
    email は一意である
    password が present の場合、保存時にハッシュ化される

3 examples, 0 failures
```

## 標準フォーマッタ一覧

| フォーマッタ | 用途 | 出力例 |
|---|---|---|
| `progress` | デフォルト。進捗をドットで表示 | `...` `3 examples, 0 failures` |
| `documentation` | テスト名を階層表示 | `User / バリデーション / ...` |
| `json` | 機械可読な JSON | `{"examples": [...], "summary": {...}}` |
| `html` | HTML レポート生成 | `report.html` をブラウザで閲覧 |

### 各フォーマッタの実行例

#### progress

```bash
bundle exec rspec spec/models/user_spec.rb --format progress
```

```text
...

3 examples, 0 failures
```

#### documentation

```bash
bundle exec rspec spec/models/user_spec.rb --format documentation
```

```text
User
  バリデーション
    email, password, name が必須
    email は一意である
    password が present の場合、保存時にハッシュ化される

3 examples, 0 failures
```

#### json

```bash
bundle exec rspec spec/models/user_spec.rb --format json
```

```json
{
  "version": "3.13.6",
  "examples": [
    {
      "id": "./spec/models/user_spec.rb[1:1:1]",
      "description": "email, password, name が必須",
      "full_description": "User バリデーション email, password, name が必須",
      "status": "passed",
      "file_path": "./spec/models/user_spec.rb",
      "line_number": 5,
      "run_time": 0.166733601
    }
  ],
  "summary": {
    "duration": 0.200795072,
    "example_count": 3,
    "failure_count": 0,
    "pending_count": 0
  }
}
```

#### html

```bash
bundle exec rspec spec/models/user_spec.rb \
  --format html \
  --out tmp/rspec_report.html
```

`tmp/rspec_report.html` が生成される。ブラウザで開いて閲覧する。

## 設定方法

### コマンドライン

一時的に切り替えたい場合。

```bash
bundle exec rspec spec/models/user_spec.rb --format documentation
bundle exec rspec spec/models/user_spec.rb --format json
```

### `.rspec` ファイル

プロジェクト全体にデフォルトで適用したい場合。  
`idp-user/.rspec` に記述する。

```text
--format documentation
--color
--require spec_helper
```

### `spec_helper.rb` / `rails_helper.rb`

Ruby コードとして設定したい場合。

```ruby
RSpec.configure do |config|
  config.formatter = :documentation
  config.color = true
end
```

## 複数フォーマッタの併用

ターミナルには簡潔な進捗を表示しつつ、HTML レポートも同時に生成する例。

```bash
bundle exec rspec spec/models/user_spec.rb \
  --format progress \
  --format html \
  --out tmp/rspec_report.html
```

`.rspec` での設定例：

```text
--format progress
--format html
--out tmp/rspec_report.html
```

## 外部 gem フォーマッタ

| gem | 特徴 | 導入例 |
|---|---|---|
| `fuubar` | プログレスバー形式 | `Gemfile` に追加 → `--format fuubar` |
| `rspec_junit_formatter` | JUnit XML 出力（CI 向け） | `--format RspecJunitFormatter --out tmp/rspec.xml` |
| `fivemat` | ファイル単位でまとまった出力 | `--format Fivemat` |

### CI 向け JUnit XML 出力例

`Gemfile`

```ruby
group :test do
  gem 'rspec_junit_formatter'
end
```

実行：

```bash
bundle exec rspec \
  --format progress \
  --format RspecJunitFormatter \
  --out tmp/rspec.xml
```

## おすすめ組み合わせ

| 場面 | おすすめ |
|---|---|
| 普段のローカル開発 | `--format documentation` |
| 大量のテスト / CI | `--format progress` + `--format RspecJunitFormatter --out tmp/rspec.xml` |
| レポートを残したい | `--format progress` + `--format html --out tmp/report.html` |
| 実行結果を他ツールで解析 | `--format json` |

---

## 関連ファイル

- `idp-user/.rspec`
- `idp-user/spec/spec_helper.rb`
- `idp-user/spec/rails_helper.rb`
