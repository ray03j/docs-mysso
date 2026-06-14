---
title: PR #19 レビュー対応と UUID 統一実装記録
created: 2024-06-08
tags: [implementation, uuid, database-schema, phase1]
---

# PR #19 レビュー対応と UUID 統一実装記録

## 目次

- [概要](#概要)
- [変更対象ファイル一覧](#変更対象ファイル一覧)
  - [idp-user](#idp-user)
  - [idp-client](#idp-client)
  - [idp-auth](#idp-auth)
  - [demo-rp](#demo-rp)
- [設計ドキュメント更新](#設計ドキュメント更新)
- [補足：UUID 運用のポイント](#補足uuid-運用のポイント)

---

## 概要

`01-pr19レビュー分析と設計整合性.md` に基づき、PR #19 で指摘された 8 件のレビューコメントに対応しました。当初「bigint 統一」が推奨されていましたが、分散システム・OIDC クレームとの整合性を考慮し、**全テーブルの識別子を UUID に統一**することを選定しました。

### 対応方針

| 指摘 | 当初推奨 | 最終選定 | 理由 |
|---|---|---|---|
| seeds.rb の冪等性欠如 | 冪等化 + 環境限定 | 冪等化 + 環境限定 | そのまま実施 |
| モデルバリデーション欠如 | 最低限バリデーション追加 | 最低限バリデーション追加 | そのまま実施 |
| user_id / client_id の型不一致 | bigint 統一 | **UUID 統一** | 分散システム・セキュリティ・拡張性を優先 |

---

## 変更対象ファイル一覧

### idp-user

| ファイル | 変更内容 |
|---|---|
| `db/migrate/001_create_users.rb` | `create_table :users, id: :uuid` + `enable_extension 'pgcrypto'` を追加 |
| `db/schema.rb` | `users` テーブルの主キーを `uuid`（`gen_random_uuid()`）に更新 |
| `db/seeds.rb` | `find_or_initialize_by` + `Rails.env` 判定で冪等化・環境限定 |
| `app/models/user.rb` | `validates :email, presence: true, uniqueness: true` / `validates :password_hash, presence: true` を追加 |

### idp-client

| ファイル | 変更内容 |
|---|---|
| `db/migrate/001_create_clients.rb` | `create_table :clients, id: :uuid` + `t.uuid :client_id` + `enable_extension 'pgcrypto'` |
| `db/schema.rb` | `clients` テーブルの主キー・`client_id` を `uuid` に更新 |
| `db/seeds.rb` | `find_or_initialize_by` + `Rails.env` 判定で冪等化。`client_id` を固定 UUID `550e8400-e29b-41d4-a716-446655440000` に変更 |
| `app/models/client.rb` | `client_id`（presence/uniqueness）、`client_secret_hash`、`name`、`redirect_uris`、`allowed_scopes` の `presence` バリデーション追加 |

### idp-auth

| ファイル | 変更内容 |
|---|---|
| `db/migrate/001_create_authorization_codes.rb` | `user_id` / `client_id` を `t.uuid` に変更 |
| `db/migrate/002_create_access_tokens.rb` | 同上 |
| `db/migrate/003_create_refresh_tokens.rb` | 同上 |
| `db/migrate/004_create_consents.rb` | 同上 |
| `db/schema.rb` | 4 テーブルの `user_id` / `client_id` を `uuid` に更新 |
| `app/models/authorization_code.rb` | 新規作成。`code`（presence/uniqueness）、`user_id`、`client_id`、`expires_at` の `presence` バリデーション |
| `app/models/access_token.rb` | 新規作成。`token`（presence/uniqueness）、`user_id`、`client_id`、`expires_at` の `presence` バリデーション |
| `app/models/refresh_token.rb` | 新規作成。`token`（presence/uniqueness）、`user_id`、`client_id` の `presence` バリデーション |
| `app/models/consent.rb` | 新規作成。`user_id`、`client_id`、`scope` の `presence` バリデーション |

### demo-rp

| ファイル | 変更内容 |
|---|---|
| `.env.example` | `VITE_CLIENT_ID=demo_client` → `VITE_CLIENT_ID=550e8400-e29b-41d4-a716-446655440000` |

---

## 設計ドキュメント更新

| ファイル | 更新箇所 |
|---|---|
| `01-pr19レビュー分析と設計整合性.md` | bigint 推奨 → UUID 統一に変更。各レビューの判定を「設計見直し済み」に更新 |
| `project/design.md` | `clients.client_id`、`authorization_codes` / `access_tokens` / `refresh_tokens` / `consents` の `client_id` を `uuid` に修正 |
| `project/phase1-branch-docs/2-feature-database-schema/feature-database-schema-idp-auth.md` | マイグレーションコードと解説を `uuid` に統一。seeds.rb のサンプル UUID を更新 |
| `project/phase1-branch-docs/2-feature-database-schema/説明_idp-auth-マイグレーション.md` | 「string 型」→「uuid 型」に全面修正。UUID 選択肢を「本プロジェクトの選定」に更新 |

---

## 補足：UUID 運用のポイント

- **PostgreSQL の `pgcrypto` 拡張**を有効化し、`gen_random_uuid()` で主キーを自動生成する
- **固定 UUID**（`550e8400-e29b-41d4-a716-446655440000`）は `demo_client` 専用。本番では各クライアントが独自の UUID を持つ
- **インデックスサイズ**は bigint より大きくなるが、分散システム・OIDC `sub` クレームとの整合性・セキュリティ（推測困難）を優先して採用
- **マイクロサービス間で外部キー制約は貼らない**前提で、`uuid` 型をサービス間の識別子として保持する

---

以上
