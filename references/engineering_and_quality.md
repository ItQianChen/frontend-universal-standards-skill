# 前端工程化与五重代码质量卡点体系 (Engineering & Quality Standards)

## 1. 现代构建与路径别名标准 (Vite / Webpack)

### 1.1 Vite 路径别名配置
在 `vite.config.ts` 中配置基于 Node 原生路径的安全别名：
```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'node:path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
})
```
同时在 `tsconfig.json`（或 `tsconfig.app.json`）中显式配对：
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

---

## 2. 五重全闭环质量卡点体系

```text
  [ 本地开发写入 ]
         │
         ├──> ESLint 9/10 (基于 Flat Config 扁平配置)
         ├──> Stylelint (样式规则与 Recess 排序)
         └──> Prettier (自动化代码格式化排版)
         │
  [ Git Commit 触发 ]
         │
         ├──> Husky pre-commit ──> lint-staged (仅对暂存区变动文件执行自愈)
         └──> Husky commit-msg ──> commitlint (校验 Conventional Commit 标题)
```

### 2.1 ESLint 9+ 扁平化配置标准 (`eslint.config.js`)
ESLint 9 及后续版本已全面废弃旧版 `.eslintrc.*`，强制采用统一数组式导出（Flat Config）：
```javascript
// eslint.config.js
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import pluginVue from 'eslint-plugin-vue'
import prettierConfig from 'eslint-config-prettier'

export default [
  // 忽略规则
  { ignores: ['dist/**', 'node_modules/**', 'unpackage/**'] },
  // JS 推荐规则
  js.configs.recommended,
  // TypeScript 规则
  ...tseslint.configs.recommended,
  // Vue 3 推荐规则
  ...pluginVue.configs['flat/recommended'],
  // 关掉与 Prettier 冲突的格式规则
  prettierConfig,
  // 自定义业务级覆写
  {
    rules: {
      'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
      'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off',
      '@typescript-eslint/no-explicit-any': 'warn',
      'vue/multi-word-component-names': 'off',
    },
  },
]
```

### 2.2 Stylelint 与 Recess 属性排序
在 `.stylelintrc.cjs` 中配置属性书写顺序：
```javascript
module.exports = {
  extends: [
    'stylelint-config-standard-scss',
    'stylelint-config-standard-vue/scss',
    'stylelint-config-recess-order', // 严格强制 CSS 属性书写顺序
    'stylelint-config-prettier',
  ],
}
```
* **Recess 排序准则**：
  1. **定位属性**：`position`, `top`, `right`, `bottom`, `left`, `z-index`
  2. **盒模型与尺寸**：`display`, `flex`, `width`, `height`, `margin`, `padding`, `border`
  3. **排版与文字**：`font-size`, `font-family`, `line-height`, `color`, `text-align`
  4. **视觉与背景**：`background`, `box-shadow`, `border-radius`, `opacity`
  5. **动效与变换**：`transition`, `transform`, `animation`

### 2.3 Prettier 标准配置 (`.prettierrc.json`)
```json
{
  "$schema": "https://json.schemastore.org/prettierrc",
  "semi": false,
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### 2.4 Commitlint 提交信息规范 (`commitlint.config.cjs`)
```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',     // 新功能 (feature)
        'fix',      // 修复 bug
        'docs',     // 文档修改
        'style',    // 代码格式调整 (不影响逻辑)
        'refactor', // 代码重构 (既不是新功能也不是 bug 修复)
        'perf',     // 性能优化
        'test',     // 测试相关
        'chore',    // 构建过程或辅助工具变动
        'revert',   // 撤销提交
        'build',    // 构建依赖或配置打包
      ],
    ],
  },
}
```

### 2.5 Husky 与 lint-staged 增量校验
在 `package.json` 中配置：
```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx,vue}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss,vue}": [
      "stylelint --fix"
    ]
  }
}
```

---

## 3. 多环境变量体系与配置隔离

* `.env.development`：本地开发联调（`VITE_APP_BASE_API = '/dev-api'`）。
* `.env.production`：生产构建发布（`VITE_APP_BASE_API = '/prod-api'`）。
* `.env.test`：自动化测试或测试服发布。
* `.env.*.local`：个人本地私有配置，**必须加入 `.gitignore`**，严禁提交到公共仓库。

---

## 4. 工程卫生与反模式清理清单

1. **模板垃圾清理（Template Purge）**：
   * 脚手架生成后，必须立刻删除 `HelloWorld.vue`、示例静态图片；
2. **零死代码原则**：
   * 严禁在源码仓库中提交 `old/`、`practive/`、`test-temp/` 等临时或历史遗留目录；
   * 彻底清理整段注释掉的 API、Store 与路由，废弃代码由 Git 历史版本记录。
3. **敏感凭证防泄漏**：
   * 严禁在代码中写死内网测试 IP（如 `192.168.1.100`）、私钥或未脱敏密码。