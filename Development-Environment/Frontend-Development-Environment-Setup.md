# 前端开发环境搭建

## 1. 环境目标

前端开发环境的核心目标是：稳定管理 Node.js 版本、锁定包管理器、统一代码质量工具、跑通本地开发/构建/测试/调试流程。

| 层级 | 常用工具 | 作用 |
|------|----------|------|
| 系统基础 | Homebrew、Git、VS Code、Chrome | 安装工具、版本管理、编辑器、浏览器调试 |
| 运行时 | Node.js、nvm | 管理 JavaScript 运行环境 |
| 包管理 | npm、pnpm、Yarn、Corepack | 安装依赖、锁定包管理器版本 |
| 构建工具 | Vite、Webpack、Rollup、Turbopack | 开发服务器、打包、热更新、生产构建 |
| 语言工具 | TypeScript、tsc | 类型检查、编译配置 |
| 代码质量 | ESLint、Prettier、Stylelint、lint-staged、Husky | 代码检查、格式化、提交前校验 |
| 测试 | Vitest、Jest、Playwright、Testing Library | 单元测试、组件测试、端到端测试 |
| 调试 | Chrome DevTools、React DevTools、Vue Devtools | 性能分析、组件状态、网络和渲染排查 |
| 发布 | npm scripts、CI、Docker、静态托管平台 | 构建、产物检查、部署 |

推荐搭建顺序：

```text
Homebrew / Git / VS Code / Chrome
        |
        v
nvm + Node.js LTS
        |
        v
Corepack + npm / pnpm / Yarn
        |
        v
Vite / Framework CLI
        |
        v
TypeScript / ESLint / Prettier / Test
```

## 2. 基础工具

### 2.1 Homebrew

macOS 推荐通过 Homebrew 安装基础工具：

```bash
brew install git node
brew install --cask visual-studio-code google-chrome
```

如果已经使用 nvm 管理 Node.js，不建议再依赖 Homebrew 的 `node` 做项目开发。可以只通过 Homebrew 安装 Git、编辑器、浏览器等工具。

常用命令：

```bash
brew update
brew upgrade
brew search <name>
brew info <name>
brew install <name>
brew install --cask <app>
brew cleanup
brew doctor
```

### 2.2 Git

基础配置：

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

常用命令：

```bash
git clone <repo-url>
git status
git add .
git commit -m "message"
git pull
git push
git branch
git checkout -b feature/name
git log --oneline --graph --decorate
```

建议为前端项目准备 `.gitignore`：

```gitignore
node_modules/
dist/
build/
.turbo/
.vite/
.next/
.nuxt/
coverage/
.env.local
.DS_Store
```

### 2.3 VS Code

推荐插件：

| 插件 | 作用 |
|------|------|
| ESLint | 显示并自动修复 lint 问题 |
| Prettier | 格式化代码 |
| Stylelint | CSS / Less / Sass 规则检查 |
| EditorConfig | 统一缩进、换行、编码 |
| GitLens | Git 历史和 blame |
| npm Intellisense | npm 包导入补全 |
| Path Intellisense | 路径补全 |
| Error Lens | 行内显示错误 |
| React Developer Tools / Vue Devtools | 浏览器侧组件调试 |

推荐 `.vscode/settings.json`：

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "eslint.validate": ["javascript", "javascriptreact", "typescript", "typescriptreact"],
  "prettier.requireConfig": true,
  "files.eol": "\n"
}
```

## 3. Node.js 与 nvm

前端项目应避免直接依赖系统 Node。推荐使用 nvm 管理多个 Node.js 版本。

安装 nvm 可参考本知识库的 macOS 基础环境文档。安装完成后检查：

```bash
nvm --version
```

安装 Node.js LTS：

```bash
nvm install --lts
nvm use --lts
nvm alias default 'lts/*'
```

安装指定版本：

```bash
nvm install 22
nvm use 22
nvm alias default 22
```

查看当前版本：

```bash
node -v
npm -v
nvm current
nvm ls
```

项目级版本文件 `.nvmrc`：

```text
22
```

进入项目后：

```bash
nvm use
```

如果本地没有该版本：

```bash
nvm install
```

## 4. 包管理器

### 4.1 npm

npm 随 Node.js 一起安装，适合大多数项目和新手环境。

常用命令：

```bash
# 安装依赖
npm install

# 安装生产依赖
npm install axios

# 安装开发依赖
npm install -D typescript eslint prettier

# 删除依赖
npm uninstall axios

# 运行脚本
npm run dev
npm run build
npm run test

# 查看过期依赖
npm outdated

# 更新依赖
npm update

# 查看依赖树
npm ls

# 清理缓存
npm cache verify
```

创建项目：

```bash
npm create vite@latest
```

安全检查：

```bash
npm audit
npm audit fix
npm audit --audit-level=moderate
```

`npm audit fix --force` 可能升级大版本依赖，执行前要先看变更范围。

### 4.2 pnpm

pnpm 的优势是安装速度快、磁盘占用低、依赖结构更严格，适合中大型项目和 monorepo。

推荐通过 Corepack 启用：

```bash
npm install --global corepack@latest
corepack enable pnpm
```

在项目中锁定 pnpm 版本：

```bash
corepack use pnpm@latest
```

常用命令：

```bash
# 安装依赖
pnpm install

# 添加生产依赖
pnpm add axios

# 添加开发依赖
pnpm add -D typescript eslint prettier

# 删除依赖
pnpm remove axios

# 运行脚本
pnpm dev
pnpm build
pnpm test

# 查看依赖
pnpm list

# 更新依赖
pnpm update

# 查看 store 路径
pnpm store path

# 清理 store
pnpm store prune
```

创建项目：

```bash
pnpm create vite
```

pnpm workspace 文件：

```yaml
# pnpm-workspace.yaml
packages:
  - apps/*
  - packages/*
```

### 4.3 Yarn

Yarn 现代版本推荐通过 Corepack 管理，适合已有 Yarn 技术栈或需要 Yarn workspace / Plug'n'Play 的项目。

安装 Corepack：

```bash
npm install -g corepack
```

初始化 Yarn 项目：

```bash
yarn init -2
```

更新到稳定版本：

```bash
yarn set version stable
yarn install
```

常用命令：

```bash
yarn install
yarn add axios
yarn add -D typescript eslint prettier
yarn remove axios
yarn dev
yarn build
yarn test
yarn workspaces list
```

### 4.4 如何选择包管理器

| 场景 | 建议 |
|------|------|
| 简单项目、教程、脚手架默认 | npm |
| 团队项目、中大型项目、monorepo | pnpm |
| 既有项目已经使用 Yarn | Yarn |
| 需要和团队完全一致 | 按项目锁文件和 `packageManager` 字段 |

不要在同一个项目里混用多个锁文件：

| 包管理器 | 锁文件 |
|----------|--------|
| npm | `package-lock.json` |
| pnpm | `pnpm-lock.yaml` |
| Yarn | `yarn.lock` |

## 5. package.json

`package.json` 是前端项目的入口配置。

典型结构：

```json
{
  "name": "frontend-app",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@10.0.0",
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier . --write",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {},
  "devDependencies": {}
}
```

关键字段：

| 字段 | 作用 |
|------|------|
| `private` | 防止误发布到 npm |
| `type` | 指定 ESM / CommonJS 模块模式 |
| `packageManager` | 锁定团队使用的包管理器和版本 |
| `engines` | 声明 Node.js 版本要求 |
| `scripts` | 统一开发、构建、测试、检查命令 |

## 6. Vite 开发环境

Vite 是现代前端项目常用的开发服务器和构建工具。它提供快速冷启动、HMR、生产构建、插件系统和框架模板。

创建项目：

```bash
npm create vite@latest
pnpm create vite
yarn create vite
```

指定模板：

```bash
npm create vite@latest my-react-app -- --template react-ts
pnpm create vite my-vue-app --template vue-ts
```

常用脚本：

```bash
npm run dev
npm run build
npm run preview
```

Vite 默认开发端口通常是：

```text
http://localhost:5173
```

常见 `vite.config.ts`：

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    open: true,
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true
      }
    }
  }
})
```

常见命令：

```bash
# 查看 Vite CLI 帮助
npx vite --help

# 指定端口
npx vite --port 3000

# 构建生产产物
npx vite build

# 本地预览 dist
npx vite preview
```

## 7. TypeScript

安装：

```bash
pnpm add -D typescript
```

初始化配置：

```bash
pnpm exec tsc --init
```

常用命令：

```bash
# 类型检查，不输出文件
pnpm exec tsc --noEmit

# 构建项目引用
pnpm exec tsc -b

# 查看 TypeScript 版本
pnpm exec tsc -v
```

常见 `tsconfig.json` 关注点：

| 配置 | 建议 |
|------|------|
| `strict` | 开启 |
| `noImplicitAny` | 开启 |
| `moduleResolution` | 根据构建工具选择，Vite 项目常用 `bundler` |
| `baseUrl` / `paths` | 用于路径别名 |
| `noEmit` | 仅类型检查时开启 |

路径别名需要同时配置 TypeScript 和构建工具。

## 8. ESLint 与 Prettier

### 8.1 ESLint

ESLint 用于发现代码问题、统一规则、配合编辑器自动修复。

安装：

```bash
pnpm add -D eslint @eslint/js typescript-eslint
```

常用命令：

```bash
pnpm exec eslint .
pnpm exec eslint . --fix
```

脚本：

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  }
}
```

### 8.2 Prettier

Prettier 负责格式化，不负责代码质量判断。

安装：

```bash
pnpm add -D prettier
```

`.prettierrc` 示例：

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "none",
  "printWidth": 100
}
```

常用命令：

```bash
pnpm exec prettier . --check
pnpm exec prettier . --write
```

脚本：

```json
{
  "scripts": {
    "format": "prettier . --write",
    "format:check": "prettier . --check"
  }
}
```

### 8.3 EditorConfig

`.editorconfig`：

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true
```

## 9. Git Hooks

Husky + lint-staged 可以在提交前只检查本次改动的文件。

安装：

```bash
pnpm add -D husky lint-staged
pnpm exec husky init
```

`package.json`：

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{css,scss,less,json,md}": ["prettier --write"]
  }
}
```

`.husky/pre-commit`：

```bash
pnpm exec lint-staged
```

注意：Git Hook 是本地质量门禁，CI 仍然需要重新执行 lint、test、build。

## 10. 测试工具

### 10.1 Vitest

Vitest 适合 Vite 项目做单元测试和轻量组件测试。

安装：

```bash
pnpm add -D vitest jsdom @testing-library/react @testing-library/jest-dom
```

脚本：

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "coverage": "vitest run --coverage"
  }
}
```

常用命令：

```bash
pnpm test
pnpm test -- --watch
pnpm coverage
```

### 10.2 Playwright

Playwright 用于端到端测试。

安装：

```bash
pnpm create playwright
```

常用命令：

```bash
pnpm exec playwright test
pnpm exec playwright test --ui
pnpm exec playwright show-report
pnpm exec playwright codegen http://localhost:5173
```

## 11. 浏览器调试工具

Chrome DevTools 常用面板：

| 面板 | 用途 |
|------|------|
| Elements | DOM、CSS、布局、样式覆盖 |
| Console | 日志、表达式调试、错误定位 |
| Sources | 断点、源码映射、运行时调试 |
| Network | 请求、缓存、瀑布流、接口耗时 |
| Performance | 主线程、长任务、渲染性能 |
| Memory | 堆快照、内存泄漏 |
| Application | localStorage、IndexedDB、Cookie、Service Worker |
| Lighthouse | 性能、可访问性、SEO 粗略评估 |

常用技巧：

```text
Network 勾选 Disable cache
Performance 录制首屏加载和交互
Sources 使用 conditional breakpoint
Application 清理站点数据排查缓存问题
Console 使用 copy(...) 拷贝对象
```

React / Vue 项目建议安装对应浏览器插件：

- React Developer Tools
- Vue Devtools

## 12. 环境变量

Vite 项目常见环境文件：

```text
.env
.env.local
.env.development
.env.production
```

Vite 暴露给客户端的变量默认需要 `VITE_` 前缀：

```env
VITE_API_BASE_URL=https://api.example.com
```

代码中使用：

```ts
const baseURL = import.meta.env.VITE_API_BASE_URL
```

建议：

- `.env` 可以提交默认值。
- `.env.local` 放本机私密配置，不提交。
- 不要把密钥、Token、私有证书放到前端环境变量里。
- 前端环境变量会进入客户端产物，不能当服务端秘密使用。

## 13. 常用脚手架

| 技术栈 | 常用命令 |
|--------|----------|
| Vite 通用 | `npm create vite@latest` |
| React + Vite | `pnpm create vite my-app --template react-ts` |
| Vue + Vite | `pnpm create vite my-app --template vue-ts` |
| Next.js | `npx create-next-app@latest` |
| Nuxt | `npx nuxi init my-app` |
| SvelteKit | `npx sv create my-app` |
| Astro | `npm create astro@latest` |

选择原则：

| 场景 | 建议 |
|------|------|
| 普通 SPA / 后台管理 | Vite + React/Vue |
| SSR / SSG / 全栈 React | Next.js |
| SSR / SSG / 全栈 Vue | Nuxt |
| 内容站点 / 多框架孤岛架构 | Astro |
| 组件库 | Vite library mode / tsup / Rollup |

## 14. Monorepo 工具

中大型前端项目常见 monorepo 组合：

```text
pnpm workspace
        |
        v
Turborepo / Nx
        |
        v
apps/* + packages/*
```

目录示例：

```text
repo/
  apps/
    web/
    admin/
  packages/
    ui/
    config-eslint/
    config-ts/
  pnpm-workspace.yaml
  package.json
```

pnpm workspace 常用命令：

```bash
# 安装所有 workspace 依赖
pnpm install

# 给某个包添加依赖
pnpm --filter web add axios

# 只运行某个包脚本
pnpm --filter web dev

# 运行所有包的 build
pnpm -r build

# 按依赖拓扑运行
pnpm -r --sort build
```

## 15. 构建与发布检查

本地提交前建议跑：

```bash
pnpm lint
pnpm test
pnpm build
```

生产构建后检查：

```bash
pnpm preview
```

常见构建产物：

| 工具 | 默认产物目录 |
|------|--------------|
| Vite | `dist/` |
| Next.js | `.next/` |
| Nuxt | `.output/` / `.nuxt/` |
| Astro | `dist/` |

CI 推荐步骤：

```text
checkout
setup node
enable corepack
install dependencies with lockfile
lint
typecheck
test
build
upload artifact / deploy
```

pnpm CI 示例命令：

```bash
corepack enable pnpm
pnpm install --frozen-lockfile
pnpm lint
pnpm test
pnpm build
```

## 16. 依赖安全与版本治理

基础检查：

```bash
npm audit
pnpm audit
```

查看过期依赖：

```bash
npm outdated
pnpm outdated
```

更新依赖建议：

| 类型 | 策略 |
|------|------|
| patch | 可以较频繁更新 |
| minor | 看 changelog，配合测试 |
| major | 单独分支升级，评估破坏性变更 |
| 安全漏洞 | 优先修复，但避免无脑 `--force` |

建议提交锁文件，并在 CI 使用 frozen lockfile，保证安装结果可复现。

## 17. 常见问题

### 17.1 node 版本不对

检查：

```bash
which node
node -v
nvm current
cat .nvmrc
```

修复：

```bash
nvm install
nvm use
```

### 17.2 包管理器混用导致依赖异常

检查项目里是否同时存在多个锁文件：

```bash
ls package-lock.json pnpm-lock.yaml yarn.lock
```

保留团队约定的锁文件，删除其他锁文件后重新安装依赖。

### 17.3 node_modules 异常

常见处理：

```bash
rm -rf node_modules
pnpm install
```

如果是 npm：

```bash
rm -rf node_modules package-lock.json
npm install
```

删除锁文件会改变依赖解析结果，团队项目不要随意这么做。

### 17.4 端口被占用

查看端口：

```bash
lsof -i :5173
```

结束进程：

```bash
kill -9 <pid>
```

或换端口：

```bash
pnpm dev -- --port 3000
```

### 17.5 浏览器缓存导致调试不生效

处理顺序：

```text
打开 DevTools
Network 勾选 Disable cache
Hard Reload
Application 清理 Storage
重启 dev server
```

## 18. 推荐项目基线

一个新前端项目建议至少具备：

```text
.nvmrc
packageManager
lockfile
TypeScript
ESLint
Prettier
EditorConfig
lint / typecheck / test / build scripts
README
.env.example
CI
```

示例脚本：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier . --write",
    "format:check": "prettier . --check",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

## 19. 参考资料

- Node.js: https://nodejs.org/
- npm Docs: https://docs.npmjs.com/
- pnpm: https://pnpm.io/
- Yarn: https://yarnpkg.com/
- Vite: https://vite.dev/
- TypeScript: https://www.typescriptlang.org/
- ESLint: https://eslint.org/
- Prettier: https://prettier.io/
- Vitest: https://vitest.dev/
- Playwright: https://playwright.dev/

## 20. 总结

前端环境搭建的关键不是安装很多工具，而是把版本和流程固定下来：

| 问题 | 推荐做法 |
|------|----------|
| Node 版本不一致 | `.nvmrc` + nvm |
| 包管理器不一致 | `packageManager` + Corepack |
| 依赖结果不可复现 | 提交锁文件 + frozen lockfile |
| 代码风格不一致 | ESLint + Prettier + EditorConfig |
| 提交质量不稳定 | lint-staged + Husky + CI |
| 本地能跑线上失败 | 本地执行 `lint / typecheck / test / build` |
