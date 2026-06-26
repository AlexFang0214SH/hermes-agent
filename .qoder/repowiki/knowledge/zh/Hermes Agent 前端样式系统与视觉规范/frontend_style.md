## 1. 核心系统与技术栈

Hermes Agent 的前端视觉风格建立在 **Tailwind CSS v4**、**shadcn/ui** 组件模式以及自定义的 **Nous Design System (DS)** 之上。项目采用多形态 UI 架构（Web Dashboard、Desktop Electron App、TUI），各形态共享底层的设计令牌（Design Tokens）和主题逻辑，但针对平台特性进行了适配。

*   **CSS 框架**: Tailwind CSS v4 (使用 `@tailwindcss/vite` 插件)。
*   **组件库基础**: shadcn/ui (New York 风格)，基于 Radix UI primitives。
*   **设计系统**: `@nous-research/ui` (内部 DS 包)，提供字体、全局变量及核心视觉透镜（Lens）。
*   **工具函数**: `clsx` + `tailwind-merge` (通过 `cn()` 函数处理类名合并与冲突)。
*   **动画与交互**: `motion` (Framer Motion), `gsap`, `tw-shimmer`。
*   **图标**: `lucide-react`, `@tabler/icons-react`, `@vscode/codicons` (桌面端)。

## 2. 关键文件与目录结构

### Web Dashboard (`web/`)
*   `web/src/index.css`: 全局样式入口。导入 Tailwind、Nous DS 字体与全局变量。定义了 LENS_0 (Hermes Teal) 的默认调色板，并映射了 shadcn 兼容的 CSS 变量（如 `--color-card`, `--color-primary`）。
*   `web/src/themes/presets.ts`: 内置主题定义（Hermes Teal, Midnight, Ember, Mono, Cyberpunk, Rosé, Nous Blue）。每个主题包含调色板、排版（字体族、字号）和布局（圆角、密度）配置。
*   `web/src/themes/context.tsx`: 主题上下文提供者，负责将主题令牌注入 CSS 变量。

### Desktop App (`apps/desktop/`)
*   `apps/desktop/src/styles.css`: 桌面端全局样式。定义了复杂的 `--dt-*` (Desktop Theme) 变量体系，包括背景、前景、卡片、边框、阴影（`--shadow-nous`）等。支持深色/浅色模式切换。
*   `apps/desktop/components.json`: shadcn/ui 配置文件，指定样式为 "new-york"，基础色为 "neutral"。
*   `apps/desktop/src/themes/presets.ts`: 桌面端内置主题（Nous, Midnight, Ember, Mono, Cyberpunk, Slate）。与 Web 端主题名称部分对应，但配色更贴近原生应用体验。
*   `apps/desktop/src/components/ui/button.tsx`: 按钮组件实现，使用 `class-variance-authority` (CVA) 定义变体（default, destructive, outline, ghost, text 等）和尺寸。

### TUI (`ui-tui/`)
*   基于 `hermes-ink` 包，采用终端友好的字符渲染风格，不依赖传统 CSS。

## 3. 架构与设计约定

### 设计令牌 (Design Tokens)
项目采用 CSS 变量作为设计令牌的载体，实现了主题的动态切换：
*   **Web 端**: 使用 `--background`, `--midground`, `--foreground` 等语义化变量。通过 `@theme inline` 在 Tailwind v4 中注册颜色映射。
*   **桌面端**: 使用 `--dt-background`, `--dt-foreground`, `--dt-card` 等前缀变量。引入了 `--radius-scalar` 来控制全局圆角比例，以及 `--spacing-mul` 控制间距密度。

### 主题系统 (Theming)
*   **多主题支持**: 用户可在设置中切换预设主题。主题不仅改变颜色，还改变字体（如 Ember 使用 Spectral  serif 字体，Cyberpunk 使用 Share Tech Mono）和圆角大小。
*   **Lens 概念**: Web 端引入了 "Lens" 概念（如 LENS_0 Teal, LENS_5I Nous Blue），通过 CSS `mix-blend-mode` (如 `difference`) 实现复杂的视觉叠加效果。
*   **深色模式**: 桌面端通过 `:root.dark` 类切换深色模式，重新定义混合比例（`--theme-mix-*`）和中性色基准。

### 组件样式规范
*   **Shadcn 兼容**: 尽管使用了自定义 DS，但保留了 shadcn 的类名习惯（如 `bg-card`, `text-muted-foreground`），通过在 `index.css` / `styles.css` 中映射变量实现兼容。
*   **CVA 变体**: 所有交互式组件（Button, Badge, Input 等）均使用 CVA 定义变体，确保状态（hover, focus, disabled）的一致性。
*   **滚动条定制**: 定义了 `.scrollbar-dt` 和 `.dt-portal-scrollbar` 类，统一美化 Webkit 和 Firefox 的滚动条样式，使其融入主题色。

## 4. 开发者规则

1.  **类名合并**: 始终使用 `cn()` 函数合并类名，以确保 Tailwind 类的正确覆盖和条件渲染。
2.  **主题扩展**: 新增主题需在 `themes/presets.ts` 中定义，并确保与后端 `_BUILTIN_DASHBOARD_THEMES` 同步（Web 端）。
3.  **颜色使用**: 避免硬编码颜色值。优先使用语义化 CSS 变量（如 `var(--dt-primary)`）或 Tailwind 映射类（如 `bg-primary`）。
4.  **字体加载**: 新主题若使用外部字体，需在 `typography.fontUrl` 中提供 Google Fonts 链接，或在本地 `@font-face` 中声明。
5.  **响应式策略**: Web 端针对移动端（`max-width: 768px`）有特定的布局调整（如解除 `overflow: hidden` 允许滚动）。
6.  **无障碍**: 遵循 shadcn/ui 的无障碍标准，确保焦点状态（`focus-visible:ring`）清晰可见。桌面端全局移除了默认焦点环，但为特定输入框保留了自定义辉光效果。