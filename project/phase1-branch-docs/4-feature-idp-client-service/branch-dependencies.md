# feature/idp-client-service ブランチ依存関係

## 目次

- [子ブランチ間の依存](#子ブランチ間の依存)
- [Phase 1 全体の依存関係](#phase-1-全体の依存関係)
- [外部サービス連携](#外部サービス連携)

---

## 子ブランチ間の依存

```
feature/database-schema （完了後）
    │
    ▼
feature/idp-client-service/init
    │
    ├──► feature/idp-client-service/clients
    └──► feature/idp-client-service/verify
```

| ブランチ | 親ブランチ | 理由 |
|---|---|---|
| `feature/idp-client-service/init` | `feature/database-schema` | `clients` テーブル・モデル雛形・初期seedが必要 |
| `feature/idp-client-service/clients` | `feature/idp-client-service/init` | ルーティング・内部API保護の共通メソッド・モデル強化が必要 |
| `feature/idp-client-service/verify` | `feature/idp-client-service/init` | ルーティング・内部API保護の共通メソッド・モデル強化が必要 |

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
    ├──► feature/idp-client-service
    │         │
    │         ▼
    ├──► feature/idp-auth-service
    │         │
    │         ▼
    └──► feature/frontend-login
```

## 外部サービス連携

| 呼び出し元 | 呼び出し先 | 用途 |
|---|---|---|
| `idp-auth` | `idp-client` | `GET /api/v1/clients/:client_id` で `client_id` / `redirect_uri` / `scope` の事前検証（`/authorize` で使用） |
| `idp-auth` | `idp-client` | `POST /api/v1/clients/verify` で `client_id` / `client_secret` / `redirect_uri` 検証（`/token` で使用） |

> 内部APIは `X-Internal-API-Key` ヘッダーで認証し、インターネット公開しない。
