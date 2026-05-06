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
    "format": "prettier --write ."
  },
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
    "prettier": "^3.2"
  }
}
```

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
- [ ] `npm run lint` が実行できる
- [ ] `npm run format` が実行できる

## 備考

ルーター・Pinia・APIクライアントは `feature/frontend-login` で導入する。本ブランチでは最小構成の Vue アプリのみ作成する。
