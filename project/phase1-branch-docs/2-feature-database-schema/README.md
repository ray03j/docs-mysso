# Phase 1 ブランチ計画ドキュメント — feature/database-schema

本ディレクトリは、`docs-mysso/project/phase1-feature-branches.md` に定義された **Phase 1 基盤構築** のうち、`feature/database-schema` をさらに細かい粒度の feature ブランチに分解した計画をまとめたものです。

## 方針

- `feature/database-schema` を **4つの子ブランチ** に分割
- 各ブランチは独立してレビュー・マージ可能な単位とする
- `init` ブランチは `feature/project-setup` 完了後にマージ基盤とする

### なぜこの構成か

- **4 分割の理由**：`feature/database-schema` 単体では「DB初期化・3サービスのマイグレーション・seeds 投入」が含まれ、1 つの PR で差分が大きくなる。サービスごとに分割することで、各ドメインのスキーマ設計を専門的にレビューできる。
- **`init` を親とする理由**：全ブランチが共通して「PostgreSQL 初期化スクリプトの更新」と「`pg` gem の追加」を前提とするため。
- **並行マージ戦略**：`init` 以降の 3 ブランチ（`idp-user` / `idp-client` / `idp-auth`）は互いに DB レベルで分離しており、並行してレビュー・マージできる。

## ブランチ一覧

| ブランチ名 | 内容 | 依存 |
|---|---|---|
| `feature/database-schema/init` | PostgreSQL 初期化スクリプト・Gemfile 更新 | `feature/project-setup` |
| `feature/database-schema/idp-user` | `users` テーブルマイグレーション・モデル雛形・seeds | `init` |
| `feature/database-schema/idp-client` | `clients` テーブルマイグレーション・モデル雛形・seeds | `init` |
| `feature/database-schema/idp-auth` | `authorization_codes` / `access_tokens` / `refresh_tokens` / `consents` マイグレーション・seeds | `init` |

## マージ順序

```
feature/project-setup （完了後）
    │
    ▼
feature/database-schema/init
    │
    ├──► feature/database-schema/idp-user
    ├──► feature/database-schema/idp-client
    └──► feature/database-schema/idp-auth
```

`init` を先行してマージし、残り 3 ブランチは並行して開発・レビュー・マージを行う。
全ブランチが `main` にマージされた時点で、`feature/database-schema` のチェックリストを満たす。

## Phase 1 全体の依存関係

```
feature/project-setup
    │
    ▼
feature/database-schema （上記4ブランチの集合）
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

## 各ブランチ詳細

詳細は各マークダウンファイルを参照：

- [`feature-database-schema-init.md`](feature-database-schema-init.md)
- [`feature-database-schema-idp-user.md`](feature-database-schema-idp-user.md)
- [`feature-database-schema-idp-client.md`](feature-database-schema-idp-client.md)
- [`feature-database-schema-idp-auth.md`](feature-database-schema-idp-auth.md)
