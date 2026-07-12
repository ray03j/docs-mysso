# ブランチ依存関係マップ — feature/idp-user-service

## Phase 1: feature/idp-user-service

```
main
│
└──► feature/database-schema （完了後）
         │
         ▼
    feature/idp-user-service/init
         │
         ├──► feature/idp-user-service/users
         └──► feature/idp-user-service/auth-verify
```

### マージ戦略

1. `feature/database-schema` の全ブランチが `main` にマージされた後、`feature/idp-user-service/init` を `main` から分岐
2. `init` を `main` へマージ（`bcrypt` gem 追加・ルーティング設定・モデル強化）
3. 残りの 2 ブランチは `init` のマージ後、**並行して**開発・レビュー・マージを実施
4. 全ブランチが `main` にマージされた時点で `feature/idp-user-service` 完了

#### なぜこの構成か

- **`init` 先行の理由**：全ブランチが `idp-user/Gemfile` の `bcrypt` 追加と `config/routes.rb` の更新を前提としている。`init` を先にマージしないと、各ブランチで同じ設定を別々に作成し、マージ競合が起きる可能性がある。
- **並行開発の理由**：`users`（サインアップ）と `auth-verify`（ログイン検証）は独立したエンドポイントであり、互いにコード依存がない。直列にすると待ち時間が無駄に長くなる。

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
| `feature/idp-user-service/{サブタスク}` | ユーザー管理サービスの細分化 |
| `fix/{バグ内容}` | バグ修正 |
| `docs/{内容}` | ドキュメント更新 |

## 備考

- `feature/idp-user-service` は本定義書により **3つの子ブランチ** に分割した
- Phase 1 の `feature/project-setup` は `docs-mysso/project/phase1-branch-docs/1-feature-project-setup/` を参照
- Phase 1 の `feature/database-schema` は `docs-mysso/project/phase1-branch-docs/2-feature-database-schema/` を参照
