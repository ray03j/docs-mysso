# ブランチ依存関係マップ — feature/database-schema

## Phase 1: feature/database-schema

```
main
│
└──► feature/project-setup （完了後）
         │
         ▼
    feature/database-schema/init
         │
         ├──► feature/database-schema/idp-user
         ├──► feature/database-schema/idp-client
         └──► feature/database-schema/idp-auth
```

### マージ戦略

1. `feature/project-setup` の全ブランチが `main` にマージされた後、`feature/database-schema/init` を `main` から分岐
2. `init` を `main` へマージ（`pg` gem 追加・PostgreSQL 初期化スクリプト更新）
3. 残りの 3 ブランチは `init` のマージ後、**並行して**開発・レビュー・マージを実施
4. 全ブランチが `main` にマージされた時点で `feature/database-schema` 完了

#### なぜこの構成か

- **`init` 先行の理由**：全ブランチが `infrastructure/postgresql/init/01_init.sql` の更新を前提としている。`init` を先にマージしないと、各ブランチで同じ SQL を別々に作成し、マージ競合が起きる可能性がある。
- **並行開発の理由**：3 ブランチ間に DB レベルでの依存がない（`idp-user` の `users` テーブルと `idp-auth` の `authorization_codes` テーブルは別 DB）。直列にすると待ち時間が無駄に長くなる。

## Phase 1 全体（Phase 1 完了まで）

```
main
│
└──► feature/project-setup
         │
         ▼
    feature/database-schema
         │
         ├──► feature/idp-user-service
         │         │
         │         ▼
         ├──► feature/idp-client-service
         │         │
         │         ▼
         ├──► feature/idp-auth-service
         │         │
         │         ▼
         └──► feature/frontend-login
```

## ブランチ命名規則

| パターン | 用途 |
|---|---|
| `feature/{機能名}` | 新機能開発 |
| `feature/database-schema/{サブタスク}` | データベーススキーマ構築の細分化 |
| `fix/{バグ内容}` | バグ修正 |
| `docs/{内容}` | ドキュメント更新 |

## 備考

- `feature/database-schema` は本定義書により **4つの子ブランチ** に分割した
- Phase 1 の `feature/project-setup` は `docs-mysso/project/phase1-branch-docs/1-feature-project-setup/` を参照
