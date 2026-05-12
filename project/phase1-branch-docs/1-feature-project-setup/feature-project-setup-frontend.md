# feature/project-setup/frontend

## 目的

統合フロントエンド `idp-portal` の Vue 3 SPA 雛形を作成する。ポート5173で動作。

## ブランチ名

`feature/project-setup/frontend`

## 親ブランチ

`feature/project-setup/init`

## 作成するファイル

### `frontend/package.json`

```json
{
  "name": "idp-portal",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --host",
    "build": "vue-tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .vue,.ts,.tsx --fix",
    "format": "oxfmt ."
  },
  "packageManager": "pnpm@9.0.0",
  "dependencies": {
    "vue": "^3.4",
    "pinia": "^2.1",
    "vue-router": "^4.2"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0",
    "typescript": "^5.3",
    "vite": "^5.0",
    "vue-tsc": "^1.8",
    "eslint": "^8.57",
    "eslint-plugin-vue": "^9.24",
    "@typescript-eslint/eslint-plugin": "^7.5",
    "@typescript-eslint/parser": "^7.5",
    "oxfmt": "^0.24"
  }
}
```

#### なぜこの設定か

- **`"type": "module"`**：ES Modules（`import`/`export`）をネイティブに使用し、Vite の高速 HMR（Hot Module Replacement）を最大限活かす。CommonJS では Tree Shaking が不完全になり、バンドルサイズが肥大化する。
- **`vite --host`**：Docker コンテナ内で開発サーバを起動する際、デフォルトでは localhost のみにバインドされてホスト側ブラウザからアクセスできない。`--host`（`0.0.0.0` バインド）を付けることで、ポートフォワーディングされたホスト側から `http://localhost:5173` で到達できる。
- **`vue`/`pinia`/`vue-router`**：Vue 3 エコシステムの標準的な組み合わせ。Pinia は Vuex 後継で型推論が強く、`vue-router` は SPA 内の画面遷移に必須。雛形段階で入れておくことで、`feature/frontend-login` で即座にルーティング・状態管理を実装できる。
- **`vue-tsc && vite build`**：ビルド前に TypeScript コンパイルチェックを実行し、型エラーを実行時ではなくビルド時に検出。型安全なデプロイを担保する。
- **`eslint`/`oxfmt` を devDependencies に入れる**：フロントエンドの品質担保をそのプロジェクト内で完結させ、グローバルインストールを強制しない。CI でも `pnpm install --frozen-lockfile` で同じバージョンが入る。Oxfmt は Prettier 互換で実行速度が圧倒的に速く、Vue ファイルの JS/TS ブロックもフォーマット可能。

### `frontend/vite.config.ts`

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    port: 5173,
    host: true,
    proxy: {
      '/api': {
        target: 'http://idp-auth:3000',
        changeOrigin: true,
      },
    },
  },
})
```

#### なぜこの設定か

- **`port: 5173`**：Vite 開発サーバのデフォルトポート。特別な理由がない限りデフォルトを維持することで、開発者がドキュメントを見ずにアクセスできる。
- **`host: true`**：`package.json` の `--host` と同様、Docker 内で `0.0.0.0` にバインドし、ホスト側ブラウザからのアクセスを許可。
- **`proxy: '/api'`**：CORS 問題を回避するため、開発時はフロントエンドの `/api/*` リクエストを `idp-auth`（ポート3000）に転送。フロントエンドコード内では相対パス `/api/...` を使えるため、本番 URL と開発 URL の切り替えが不要になる。`changeOrigin: true` は、仮想ホストやオリジンヘッダの改変が必要な場合に備えて有効化している（現時点では必須ではないが、将来的な拡張に備える）。

### `frontend/index.html`

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>IDP Portal</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

### `frontend/src/main.ts`

```typescript
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.mount('#app')
```

### `frontend/src/App.vue`

```vue
<template>
  <div id="app">
    <h1>IDP Portal</h1>
  </div>
</template>

<script setup lang="ts">
</script>
```

## マージ基準（チェックリスト）

- [ ] `docker compose up frontend` で Vite 開発サーバ（ポート5173）が起動する
- [ ] `pnpm lint` が実行できる
- [ ] `pnpm format` が実行できる

## 備考

ルーター・Pinia・APIクライアントは `feature/frontend-login` で導入する。本ブランチでは最小構成の Vue アプリのみ作成する。
