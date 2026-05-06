# ブランチ依存関係マップ

## Phase 1: 基盤構築（feature/project-setup）

```
main
│
└──► feature/project-setup/init
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

### マージ戦略

1. `feature/project-setup/init` を `main` へマージ
2. 残りの8ブランチは `init` のマージ後、**並行して**開発・レビュー・マージを実施
3. 全ブランチが `main` にマージされた時点で `feature/project-setup` 完了

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
| `feature/project-setup/{サブタスク}` | プロジェクトセットアップの細分化 |
| `fix/{バグ内容}` | バグ修正 |
| `docs/{内容}` | ドキュメント更新 |

## 備考

- Phase 1 の `feature/project-setup` は、本定義書により **8つの子ブランチ** に分割した
- Phase 2 以降のブランチ定義は `docs-mysso/project/phase1-feature-branches.md` の「Phase 2 以降の再定義（案）」を参照
