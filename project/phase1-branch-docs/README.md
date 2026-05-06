# Phase 1 ブランチ計画ドキュメント

本ディレクトリは、`docs-mysso/project/phase1-feature-branches.md` に定義された **Phase 1 基盤構築** を、さらに細かい粒度の feature ブランチに分解した計画をまとめたものです。

## 方針

- `feature/project-setup` を **8つの子ブランチ** に分割
- 各ブランチは独立してレビュー・マージ可能な単位とする
- 実際の環境構築（ファイル作成）は本ドキュメント作成時点では行わない

## ブランチ一覧

| ブランチ名 | 内容 | 依存 |
|---|---|---|
| `feature/project-setup/init` | プロジェクト雛形・ディレクトリ構造 | なし |
| `feature/project-setup/docker` | Docker Compose 設定 | `init` |
| `feature/project-setup/infra` | 共有インフラ雛形 | `init` |
| `feature/project-setup/idp-auth` | idp-auth Rails API 雛形 | `init` |
| `feature/project-setup/idp-user` | idp-user Rails API 雛形 | `init` |
| `feature/project-setup/idp-client` | idp-client Rails API 雛形 | `init` |
| `feature/project-setup/frontend` | Vue SPA 雛形 | `init` |
| `feature/project-setup/demo-rp` | デモRP雛形 | `init` |
| `feature/project-setup/tools` | Lefthook, RuboCop, ESLint/Prettier 設定 | `init` |

## マージ順序

```
feature/project-setup/init
    │
    ├──► feature/project-setup/docker
    ├──► feature/project-setup/infra
    ├──► feature/project-setup/idp-auth
    ├──► feature/project-setup/idp-user
    ├──► feature/project-setup/idp-client
    ├──► feature/project-setup/frontend
    ├──► feature/project-setup/demo-rp
    └──► feature/project-setup/tools
```

すべてのブランチを `init` から分岐させ、並行して開発・レビュー・マージを行う。
最終的に全ブランチが `main` にマージされた時点で、`feature/project-setup` のチェックリストを満たす。

## Phase 1 全体の依存関係

```
feature/project-setup （上記8ブランチの集合）
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

## 各ブランチ詳細

詳細は各マークダウンファイルを参照：

- [`feature-project-setup-init.md`](feature-project-setup-init.md)
- [`feature-project-setup-docker.md`](feature-project-setup-docker.md)
- [`feature-project-setup-infra.md`](feature-project-setup-infra.md)
- [`feature-project-setup-idp-auth.md`](feature-project-setup-idp-auth.md)
- [`feature-project-setup-idp-user.md`](feature-project-setup-idp-user.md)
- [`feature-project-setup-idp-client.md`](feature-project-setup-idp-client.md)
- [`feature-project-setup-frontend.md`](feature-project-setup-frontend.md)
- [`feature-project-setup-demo-rp.md`](feature-project-setup-demo-rp.md)
- [`feature-project-setup-tools.md`](feature-project-setup-tools.md)
