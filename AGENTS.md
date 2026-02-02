# Repository Guidelines

## 项目结构与模块组织
- `src/` 是主代码目录，主要包含 `components/`、`layouts/`、`pages/`、`styles/`、`utils/`。
- 内容在 `src/content/`，文章位于 `src/content/posts/`（用 `pnpm new-post <slug>` 创建）。
- 静态资源放在 `public/`（原样发布）与 `src/assets/`（由 Astro 打包）。
- 根目录包含关键配置，如 `astro.config.mjs`、`tailwind.config.cjs`、`biome.json`、`tsconfig.json`。
- 脚本放在 `scripts/`（常用 `scripts/new-post.js`）。

## 构建、测试与开发命令
- `pnpm install`：安装依赖（要求 pnpm；Node.js >= 20）。
- `pnpm dev`：启动本地开发服务器 `http://localhost:4321`。
- `pnpm build`：构建站点并运行 Pagefind 索引。
- `pnpm preview`：本地预览生产构建。
- `pnpm check`：Astro 进行类型与内容检查。
- `pnpm type-check`：运行 `tsc`（不生成文件）。
- `pnpm format`：使用 Biome 格式化 `src/`。
- `pnpm lint`：Biome 检查并尽量自动修复。

## 编码风格与命名约定
- 遵循 `src/` 中既有的 TypeScript、Astro、Svelte 用法。
- 缩进按项目默认（配置多为 2 空格；格式由 Biome 统一）。
- 文章文件名与 slug 建议使用小写短横线（kebab-case）。
- 提交前运行 `pnpm format`。

## 测试指南
- 目前没有独立测试框架。
- 以 `pnpm check` 与 `pnpm type-check` 作为主要验证步骤。

## 提交与拉取请求指南
- 使用 Conventional Commits（如 `feat: add tag filter`、`fix: correct nav link`）。
- PR 保持单一目的，避免混合无关改动。
- 重大改动先开 issue 或讨论。
- UI 变更需附截图与清晰说明。
- 提交前请运行：
  - `pnpm check`
  - `pnpm format`

## 配置提示
- 站点元信息主要在 `src/config.ts`。
- 部署设置在 `astro.config.mjs`，主题与样式在 `tailwind.config.cjs` 与 `src/styles/`。