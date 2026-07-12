# Phase 1 ブランチ計画ドキュメント — feature/idp-user-service

本ディレクトリは、`docs-mysso/project/phase1-feature-branches.md` に定義された **Phase 1 基盤構築** のうち、`feature/idp-user-service` をさらに細かい粒度の feature ブランチに分解した計画をまとめたものです。

## 方針

- `feature/idp-user-service` を **3つの子ブランチ** に分割
- 各ブランチは独立してレビュー・マージ可能な単位とする
- `init` ブランチは `feature/database-schema` 完了後にマージ基盤とする

### なぜこの構成か

- **3 分割の理由**：`feature/idp-user-service` 単体では「サインアップAPI・ログイン検証API・パスワードサービス・認証サービス・テスト」が含まれ、1 つの PR で差分が大きくなる。API エンドポイントとビジネスロジックを分離することで、各ドメインの実装を専門的にレビューできる。
- **`init` を親とする理由**：全ブランチが共通して「`bcrypt` gem の追加」と「ルーティング設定」を前提とするため。
- **並行マージ戦略**：`init` 以降の 2 ブランチ（`users` / `auth-verify`）は互いに独立して実装できる（`users` はサインアップ、`auth-verify` はログイン検証）。並行してレビュー・マージできる。

## ブランチ一覧

| ブランチ名 | 内容 | 依存 |
|---|---|---|
| `feature/idp-user-service/init` | `bcrypt` gem 追加・ルーティング設定・モデル強化 | `feature/database-schema` |
| `feature/idp-user-service/users` | サインアップ API（`UsersController`）・FactoryBot | `init` |
| `feature/idp-user-service/auth-verify` | ログイン検証 API（`VerifyController`）・サービス層 | `init` |

## マージ順序

```
feature/database-schema （完了後）
    │
    ▼
feature/idp-user-service/init
    │
    ├──► feature/idp-user-service/users
    └──► feature/idp-user-service/auth-verify
```

`init` を先行してマージし、残り 2 ブランチは並行して開発・レビュー・マージを行う。
全ブランチが `main` にマージされた時点で、`feature/idp-user-service` のチェックリストを満たす。

## Phase 1 全体の依存関係

```
feature/project-setup
    │
    ▼
feature/database-schema
    │
    ├──► feature/idp-user-service （上記3ブランチの集合）
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

- [`feature-idp-user-service-init.md`](feature-idp-user-service-init.md)
- [`feature-idp-user-service-users.md`](feature-idp-user-service-users.md)
- [`feature-idp-user-service-auth.md`](feature-idp-user-service-auth.md)
