---
title: PR #19 再レビュー対応と BCrypt コスト最適化
created: 2026-06-10
tags: [review, seeds, bcrypt, performance, phase1]
---

# PR #19 再レビュー対応と BCrypt コスト最適化

## 目次

- [概要](#概要)
- [対応ファイル一覧](#対応ファイル一覧)
- [詳細変更内容](#詳細変更内容)
  - [1. idp-user/db/seeds.rb](#1-idp-userdbseedsrb)
  - [2. idp-client/db/seeds.rb](#2-idp-clientdbseedsrb)
  - [3. feature-database-schema-idp-user.md](#3-feature-database-schema-idp-usermd)
  - [4. feature-database-schema-idp-client.md](#4-feature-database-schema-idp-clientmd)
- [対応方針](#対応方針)

---

## 概要

PR #19 に対する再レビュー（Review ID: 4450913222）の指摘 2 件に対応し、seeds.rb の BCrypt ハッシュ生成コストを環境ごとに最適化しました。

---

## 対応ファイル一覧

| # | ファイル | 変更種別 |
|---|---|---|
| 1 | `my-sso/idp-user/db/seeds.rb` | 修正 |
| 2 | `my-sso/idp-client/db/seeds.rb` | 修正 |
| 3 | `docs-mysso/project/phase1-branch-docs/2-feature-database-schema/feature-database-schema-idp-user.md` | 更新 |
| 4 | `docs-mysso/project/phase1-branch-docs/2-feature-database-schema/feature-database-schema-idp-client.md` | 更新 |

---

## 詳細変更内容

### 1. idp-user/db/seeds.rb

**修正前**

```ruby
require 'bcrypt'

# テスト用ユーザー（development/test のみ）
if Rails.env.development? || Rails.env.test?
  user = User.find_or_initialize_by(email: 'test@example.com')
  user.assign_attributes(
    password_hash: BCrypt::Password.create('password123'),
    name: 'Test User'
  )
  user.save!
end
```

**修正後**

```ruby
require 'bcrypt'

# テスト用ユーザー（development/test のみ）
if Rails.env.development? || Rails.env.test?
  user = User.find_or_initialize_by(email: 'test@example.com')
  cost = Rails.env.test? ? BCrypt::Engine::MIN_COST : BCrypt::Engine::DEFAULT_COST
  user.assign_attributes(
    password_hash: BCrypt::Password.create('password123', cost: cost),
    name: 'Test User'
  )
  user.save!
end
```

**変更点**

- `Rails.env.test?` のときは `BCrypt::Engine::MIN_COST` を使用
- development 環境では `DEFAULT_COST` を維持

---

### 2. idp-client/db/seeds.rb

**修正前**

```ruby
require 'bcrypt'

# テスト用クライアント（デモ RP 用、development/test のみ）
if Rails.env.development? || Rails.env.test?
  client = Client.find_or_initialize_by(client_id: '550e8400-e29b-41d4-a716-446655440000')
  client.assign_attributes(
    client_secret_hash: BCrypt::Password.create('demo_secret'),
    name: 'Demo Relying Party',
    redirect_uris: "http://localhost:5174/callback\nhttp://localhost:5174/",
    allowed_scopes: "openid profile email"
  )
  client.save!
end
```

**修正後**

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

**変更点**

- `Rails.env.test?` のときは `BCrypt::Engine::MIN_COST` を使用
- development 環境では `DEFAULT_COST` を維持

---

### 3. feature-database-schema-idp-user.md

**更新前**

- seeds.rb のコード例に環境によるコスト切り替えなし
- 説明に BCrypt コストについての言及なし

**更新後**

- seeds.rb のコード例を修正後のものに差し替え
- 「環境ごとのコスト切り替え」の説明を追加

> - **環境ごとのコスト切り替え**：test 環境では `BCrypt::Engine::MIN_COST` を使用し、テスト実行速度を確保する。production/development では `DEFAULT_COST` を使用し、実環境に近い挙動を確認する。

---

### 4. feature-database-schema-idp-client.md

**更新前**

- seeds.rb のコード例に環境によるコスト切り替えなし
- 説明に BCrypt コストについての言及なし

**更新後**

- seeds.rb のコード例を修正後のものに差し替え
- 「環境ごとのコスト切り替え」の説明を追加
- seeds.rb の説明を `demo_client` から「テスト用クライアント（デモ RP 用）」へ統一

> - **環境ごとのコスト切り替え**：test 環境では `BCrypt::Engine::MIN_COST` を使用し、テスト実行速度を確保する。production/development では `DEFAULT_COST` を使用し、実環境に近い挙動を確認する。

---

## 対応方針

| 環境 | BCrypt コスト | 用途 |
|---|---|---|
| production / development | `DEFAULT_COST` (10〜12) | 実運用・手動検証用。高セキュリティ |
| test | `MIN_COST` (4) | 自動テスト・CI。スピード優先 |

- test 環境では `db:prepare` や `db:setup` の過程で seeds.rb が頻繁に実行されるため、コスト最小化でボトルネックを回避する
- development では通常コストを維持し、実環境に近いハッシュ化挙動を確認できるようにする
- 設計ドキュメントにも同じ方針を明記し、実装と整合性を確保する

---

**関連ドキュメント**

- `03-pr19再レビュー分析と設計整合性.md` — レビュー指摘の詳細分析
- `01-pr19レビュー分析と設計整合性.md` — 前回のレビュー対応記録
