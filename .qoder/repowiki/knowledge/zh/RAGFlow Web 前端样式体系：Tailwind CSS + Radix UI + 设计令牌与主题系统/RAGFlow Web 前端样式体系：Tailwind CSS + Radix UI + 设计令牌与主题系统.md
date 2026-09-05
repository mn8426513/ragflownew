---
kind: frontend_style
name: RAGFlow Web 前端样式体系：Tailwind CSS + Radix UI + 设计令牌与主题系统
category: frontend_style
scope:
    - '**'
source_files:
    - web/tailwind.config.js
    - web/tailwind.css
    - web/src/theme/theme.ts
    - web/src/theme/vars.less
    - web/src/less/variable.less
    - web/src/less/mixins.less
    - web/src/global.less
    - web/src/components/ui/button.tsx
    - web/src/components/ui/input.tsx
    - web/src/components/theme-provider.tsx
    - web/package.json
---

## 1. 采用的样式系统与技术栈

RAGFlow 的 Web 前端（位于 `web/`）采用 **Vite + React 18** 构建，样式体系以 **Tailwind CSS v3** 为核心原子化框架，配合以下工具链：
- **CSS 变量驱动的设计令牌**：所有颜色、字体、圆角等通过 `:root` 与 `.dark` 下的 CSS 自定义属性集中声明（见 `web/tailwind.css`），并通过 `tailwind.config.js` 的 `extend.colors` 映射为 Tailwind 语义化 token（如 `text-primary`、`bg-card`、`accent-primary`、`state-success/error/warning`）。
- **Radix UI 无样式基础组件**：`@radix-ui/*` 提供可访问性原语（Dialog、Select、Tabs、Tooltip、Accordion 等），由项目自行用 Tailwind 类名实现视觉样式，位于 `web/src/components/ui/`。
- **class-variance-authority (CVA)**：用于定义组件变体族（如 Button 的 `default/secondary/accent/danger/outline/ghost/link/static` 等 variant 与 `xs/sm/default/lg/xl/icon-*` 尺寸），在 `button.tsx` 中集中管理。
- **tailwind-merge + clsx**：通过 `cn()` 工具函数（来自 `@/lib/utils`）合并 className，避免冲突；`clsx` 用于条件拼接。
- **Ant Design 仅作为图标库使用**：`@ant-design/icons` 引入，但 Antd 的主题配置（`theme.ts`）仅保留少量 legacy 变量（`primary-color`、`border-radius-base`、`menu-dark-bg`），并非主 UI 框架。
- **Less 辅助层**：`src/less/` 下维护全局变量（`variable.less`）、mixin（`mixins.less`，如 `.tableCell()`、`.chunkText()`、`.commonNode()`）以及 Inter 字体加载（`inter.less`、`global.less`），用于历史遗留或复杂布局场景。
- **Storybook**：基于 `storybook` + `@storybook/addon-styling-webpack` 进行组件级样式文档与预览。

## 2. 关键文件与包

| 文件 | 作用 |
|---|---|
| `web/tailwind.config.js` | Tailwind 主题扩展、断点、动画、插件注册（`tailwindcss-animate`、`@tailwindcss/line-clamp`、`tailwind-scrollbar`、`@tailwindcss/container-queries`、`@tailwindcss/typography`） |
| `web/tailwind.css` | 根样式入口，定义 `:root` / `.dark` 全部设计令牌、基础排版、滚动条样式 |
| `web/src/theme/theme.ts` | Antd 遗留主题覆盖（主色 `#338AFF`、菜单深色背景等） |
| `web/src/theme/vars.less` | Less 变量（`@primary-color`、`@header-height`、`@menu-width` 等） |
| `web/src/less/variable.less` | Less 灰度、紫色、字号等局部变量 |
| `web/src/less/mixins.less` | 表格、文本省略、节点阴影/圆角等 mixin |
| `web/src/global.less` | 字体注入（Inter/InterVariable）、全局滚动条、Excel 预览器样式修复 |
| `web/src/components/ui/*.tsx` | 自研 Radix 封装组件（Button、Input、Dialog、Table、Tabs、Sidebar 等） |
| `web/src/components/theme-provider.tsx` | 主题切换 Provider，通过给 `<html>` 添加 `light`/`dark` class 切换暗色模式，并持久化到 localStorage |
| `web/package.json` | 依赖清单（Tailwind v3、Radix UI、Lucide React、Zustand、i18next、Recharts、Mermaid、X6 画布等） |

## 3. 架构与设计约定

### 设计令牌（Design Tokens）
- 所有视觉值集中在 `tailwind.css` 的 `:root`（亮色）和 `.dark`（暗色）块中，命名遵循业务语义而非物理值：`--bg-base`、`--text-primary`、`--accent-primary`、`--state-success`、`--team-group`、`--metallic` 等。
- `tailwind.config.js` 将这些 CSS 变量映射为 Tailwind 实用类，例如 `text-text-primary`、`bg-accent-primary`、`border-border-default`、`ring-state-error`，使组件代码只引用语义 token，不直接写十六进制色值。
- 支持 alpha 通道：通过 `rgb(var(--xxx) / <alpha-value>)` 语法暴露带透明度的变体（如 `bg-canvas`、`text-primary`、`state-success/5`）。

### 主题与暗色模式
- 暗色模式通过 `darkMode: ['selector']` 启用，由 `ThemeProvider` 在 `<html>` 上切换 `light`/`dark` class，默认主题为 `Dark`。
- 主题状态通过 `localStorage('vite-ui-theme')` 持久化，并提供 `useSwitchToDarkThemeOnMount`、`useSyncThemeFromParams` 等 hook 支持 URL 参数同步。

### 组件样式组织
- 所有可复用 UI 组件放在 `src/components/ui/`，每个组件一个 `.tsx` 文件，使用 CVA 定义变体族（variant/size），通过 `cn(...)` 合并 className。
- 业务组件放在 `src/components/<feature>/` 目录，按功能域划分（canvas、document-preview、message-item、pipeline-operator-tabs 等）。
- 第三方图表/可视化（G2、G6、X6、Mermaid、Recharts）通过各自 npm 包引入，样式由各自库控制，项目仅做容器适配。

### 响应式策略
- 断点在 `tailwind.config.js` 中扩展至 `sm/md/lg/xl/2xl/3xl/4xl`，最大支持 `1980px` 屏幕。
- 使用 `@tailwindcss/container-queries` 支持容器查询。
- 字体通过 `fontFamily.sans` 指向 `var(--font-sans)`，并在 `global.less` 中根据 `font-variation-settings` 能力动态切换到 `InterVariable`。

## 4. 约定与约束

- **禁止在组件中硬编码颜色/尺寸**：应优先使用 Tailwind 语义 token（如 `text-text-primary`、`bg-bg-card`、`rounded-lg`），这些 token 最终映射到 CSS 变量，保证明暗主题一致。
- **按钮样式统一走 CVA 变体**：新增按钮风格应在 `buttonVariants` 中声明 variant，而不是在调用处散落 class。
- **暗色模式通过 class 切换**：组件不应直接判断 `window.matchMedia('(prefers-color-scheme: dark)')`，而应消费 `useIsDarkTheme()` 或依赖 `dark:` 前缀。
- **Less 仅用于兼容与 mixin**：新组件优先使用 Tailwind + TSX；已有 Less mixin（如 `.chunkText()`、`.tableCell()`）用于保持历史页面一致性。
- **Antd 不再作为 UI 框架**：仅保留 `@ant-design/icons` 图标，Antd 主题变量已退化为遗留配置，新组件不应再引入 Antd 组件。
- **滚动条样式统一**：全局滚动条通过 `tailwind.css` 的 `scrollbar-width`/`scrollbar-color` 与 `::-webkit-scrollbar` 伪元素统一控制，组件内不应重复定义。
- **构建产物**：通过 Vite 构建，Tailwind 扫描 `src/pages/**/*.tsx`、`src/components/**/*.tsx`、`src/layouts/**/*.tsx` 生成样式，未使用的类会被 tree-shake。