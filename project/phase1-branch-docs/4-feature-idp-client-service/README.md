# Phase 1 ブランチ計画ドキュメント — feature/idp-client-service

## 目次

- [方針](#方針)
  - [なぜこの構成か](#なぜこの構成か)
- [ブランチ一覧](#ブランチ一覧)
- [マージ順序](#マージ順序)
- [Phase 1 全体の依存関係](#phase-1-全体の依存関係)
- [各ブランチ詳細](#各ブランチ詳細)

---

本ディレクトリは、`docs-mysso/project/phase1-feature-branches.md` に定義された **Phase 1 基盤構築** のうち、`feature/idp-client-service` をさらに細かい粒度の feature ブランチに分解した計画をまとめたものです。

## 方針

- `feature/idp-client-service` を **3つの子ブランチ** に分割
- 各ブランチは独立してレビュー・マージ可能な単位とする
- `init` ブランチは `feature/database-schema` 完了後にマージ基盤とする

### なぜこの構成か

- **3 分割の理由**：`feature/idp-client-service` 単体では「RP登録API・一覧API・内部検証API・クライアントシークレット検証サービス・テスト」が含まれ、1 つの PR で差分が大きくなる。管理用 API と内部サービス連携 API を分離することで、各ドメインの実装を専門的にレビューできる。
- **`init` を親とする理由**：全ブランチが共通して「ルーティング設定」と「内部API保護用 `X-Internal-API-Key` 認証の共通メソッド」を前提とするため。
- **並行マージ戦略**：`init` 以降の 2 ブランチ（`clients` / `verify`）は互いに独立して実装できる（`clients` は RP 管理、`verify` は内部検証）。並行してレビュー・マージできる。

## ブランチ一覧

| ブランチ名 | 内容 | 依存 |
|---|---|---|
| `feature/idp-client-service/init` | ルーティング設定・内部API保護の共通メソッド・モデル強化 | `feature/database-schema` |
| `feature/idp-client-service/clients` | RP 登録・一覧・詳細 API（`ClientsController`）・FactoryBot | `init` |
| `feature/idp-client-service/verify` | 内部検証 API（`VerifyController`）・`ClientVerificationService` | `init` |

## マージ順序

```
feature/database-schema （完了後）
    │
    ▼
feature/idp-client-service/init
    │
    ├──► feature/idp-client-service/clients
    └──► feature/idp-client-service/verify
```

`init` を先行してマージし、残り 2 ブランチは並行して開発・レビュー・マージを行う。
全ブランチが `main` にマージされた時点で、`feature/idp-client-service` のチェックリストを満たす。

## Phase 1 全体の依存関係

```
feature/project-setup
    │
    ▼
feature/database-schema
    │
    ├──► feature/idp-user-service
    │         │
    │         ▼
    ├──► feature/idp-client-service （上記3ブランチの集合）
    │         │
    │         ▼
    ├──► feature/idp-auth-service
    │         │
    │         ▼
    └──► feature/frontend-login
```

## 各ブランチ詳細

詳細は各マークダウンファイルを参照：

- [`01_feature-idp-client-service-init.md`](01_feature-idp-client-service-init.md)
- `02_feature-idp-client-service-clients/`：[`feature.md`](02_feature-idp-client-service-clients/feature.md) / [`spec.md`](02_feature-idp-client-service-clients/spec.md)
- `03_feature-idp-client-service-verify/`：[`feature.md`](03_feature-idp-client-service-verify/feature.md) / [`spec.md`](03_feature-idp-client-service-verify/spec.md)
