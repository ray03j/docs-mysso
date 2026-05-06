# feature/project-setup/demo-rp

## 目的

デモ用 Relying Party `demo-rp` の Vue 3 SPA 雛形を作成する。ポート5174で動作。

## ブランチ名

`feature/project-setup/demo-rp`

## 親ブランチ

`feature/project-setup/init`

## 作成するファイル

### `demo-rp/package.json`

```json
{
  "name": "demo-rp",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --host --port 5174",
    "build": "vue-tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .vue,.ts,.tsx --fix",
    "format": "prettier --write ."
  },
  "dependencies": {
    "vue": "^3.4"
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
    "prettier": "^3.2"
  }
}
```

#### なぜこの設定か

- **`vite --host --port 5174`**：`frontend`（5173）とポートが被らないように 5174 を明示。同時起動が必須なため、デフォルトポートのままでは Docker Compose 起動時に競合する。
- **`dependencies` に `vue` のみ**：デモ RP は OIDC 認可フローの動作確認用の最小アプリであり、複雑な状態管理や画面遷移は不要。`pinia`・`vue-router` は Phase 3 で必要になった時点で追加する。
- **その他は `frontend` と同一**：ESM・Vite・TypeScript・品質ツールの構成を統一し、開発者が `frontend` と `demo-rp` を行き来しても文脈切り替えコストを下げる。

### `demo-rp/vite.config.ts`

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    port: 5174,
    host: true,
  },
})
```

#### なぜこの設定か

- **`port: 5174`**：`frontend`（5173）との競合回避。`package.json` の `--port` と合わせて二重指定することで、CLI 引数と設定ファイルのどちらが優先されても確実に 5174 になる。
- **`host: true`**：Docker 内での `0.0.0.0` バインド。`frontend` と同様。
- **proxy なし**：デモ RP は自前で `window.location` リダイレクト（OIDC 認可エンドポイントへ）を行うため、API プロキシが不要。認可コードフローではブラウザが直接認可サーバーにアクセスする。

### `demo-rp/index.html`

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Demo RP</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

### `demo-rp/.env.example`

```env
VITE_IDP_BASE_URL=http://localhost:3000
VITE_CLIENT_ID=demo_client
```

#### なぜこの設定か

- **`VITE_IDP_BASE_URL`**：デモ RP が OIDC Discovery や認可エンドポイントの URL を構築する際の基点。`frontend` の `VITE_API_BASE_URL` と異なり、こちらはブラウザから直接アクセスする URL（`localhost:3000`）を指定。コンテナ間通信ではなく、エンドユーザーのブラウザが使うため `localhost` を採用。
- **`VITE_CLIENT_ID=demo_client`**：OIDC では各 RP は事前に認可サーバーへ client_id を登録する必要がある。デモ用に固定値 `demo_client` を設定し、Phase 3 での認可フロー実装時に `idp-auth` の DB に同じ client_id をシード投入することで、すぐに動作確認できるようにする。

### `demo-rp/src/main.ts`

```typescript
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.mount('#app')
```

### `demo-rp/src/App.vue`

```vue
<template>
  <div id="app">
    <h1>Demo Relying Party</h1>
  </div>
</template>

<script setup lang="ts">
</script>
```

## マージ基準（チェックリスト）

- [ ] `docker compose up demo-rp` で デモRP（ポート5174）が起動する
- [ ] `npm run lint` が実行できる
- [ ] `npm run format` が実行できる

## 備考

認可フロー実装は Phase 3 で行う。本ブランチでは最小構成の Vue アプリのみ作成する。
