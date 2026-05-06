# feature/project-setup/tools

## 目的

プロジェクト全体の品質担保ツール（Lefthook, RuboCop, ESLint, Prettier）を設定する。

## ブランチ名

`feature/project-setup/tools`

## 親ブランチ

`feature/project-setup/init`

## 作成・変更するファイル

### `lefthook.yml`

```yaml
pre-commit:
  commands:
    rubocop:
      glob: "*.{rb,gemfile}"
      run: bundle exec rubocop --force-exclusion {staged_files}
    eslint-frontend:
      root: frontend/
      glob: "*.{ts,vue}"
      run: npx eslint --fix {staged_files}
    eslint-demo-rp:
      root: demo-rp/
      glob: "*.{ts,vue}"
      run: npx eslint --fix {staged_files}

pre-push:
  commands:
    rspec-auth:
      root: idp-auth/
      run: bundle exec rspec
    rspec-user:
      root: idp-user/
      run: bundle exec rspec
    rspec-client:
      root: idp-client/
      run: bundle exec rspec
```

### `.rubocop.yml`

```yaml
AllCops:
  TargetRubyVersion: 3.3
  Exclude:
    - '*/db/schema.rb'
    - '*/vendor/**/*'
    - '*/tmp/**/*'
    - '*/bin/**/*'

Style/Documentation:
  Enabled: false

Metrics/BlockLength:
  Exclude:
    - '*/spec/**/*'
    - '*/config/routes.rb'
```

### `frontend/.eslintrc.cjs`

```javascript
module.exports = {
  root: true,
  env: { browser: true, es2021: true },
  extends: [
    'eslint:recommended',
    'plugin:vue/vue3-recommended',
    'plugin:@typescript-eslint/recommended',
  ],
  parser: 'vue-eslint-parser',
  parserOptions: {
    parser: '@typescript-eslint/parser',
    sourceType: 'module',
  },
  plugins: ['@typescript-eslint'],
  rules: {
    'vue/multi-word-component-names': 'off',
  },
}
```

### `frontend/.prettierrc`

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5"
}
```

### `demo-rp/.eslintrc.cjs`

（`frontend/.eslintrc.cjs` と同様）

### `demo-rp/.prettierrc`

（`frontend/.prettierrc` と同様）

## マージ基準（チェックリスト）

- [ ] RuboCop が各 Rails サービスで実行できる
- [ ] ESLint / Prettier が `frontend/` で実行できる
- [ ] ESLint / Prettier が `demo-rp/` で実行できる
- [ ] `lefthook install` 後、pre-commit フックが動作する

## 備考

Lefthook のインストールは各開発者がローカルで `lefthook install` を実行する必要がある。CI では `lefthook run pre-commit` を実行する想定。
