Hermes Agent 采用多语言、多生态的混合依赖管理架构，核心特征是**严格的版本锁定**与**按需加载（Lazy Loading）**，以应对 AI 代理领域复杂的第三方库依赖和潜在的供应链安全风险。

### 1. Python 后端：uv + 精确锁定 + 可选依赖
*   **包管理器**：使用 `uv` 作为高性能包安装器和解析器，配合 `pyproject.toml` 声明依赖。
*   **锁定策略**：所有核心依赖在 `pyproject.toml` 中均使用**精确版本锁定**（如 `openai==2.24.0`），严禁使用范围约束（如 `>=`）。这一策略旨在防止 PyPI 上的恶意投毒（如 Mini Shai-Hulud 蠕虫事件）或破坏性更新意外进入生产环境。`uv.lock` 文件记录了完整的传递性依赖树。
*   **模块化分发**：通过 `[project.optional-dependencies]` 将功能拆分为多个 extras（如 `anthropic`, `web`, `messaging`, `voice`）。
    *   **核心依赖**：仅包含所有会话必需的库（如 `openai`, `httpx`, `pydantic`）。
    *   **懒加载机制**：非核心后端（如特定 LLM 提供商、搜索工具、TTS 引擎）不直接安装在基础环境中，而是通过 `tools/lazy_deps.py` 在首次使用时动态安装。这减小了初始安装体积，并限制了供应链攻击面。
*   **平台特异性**：利用 PEP 508 环境标记（如 `sys_platform == 'win32'`）处理跨平台差异（例如 Windows 下的 `tzdata` 和 `concurrent-log-handler`）。

### 2. JavaScript/TypeScript 前端：npm Workspaces + Overrides
*   **Monorepo 结构**：根目录 `package.json` 配置了 `workspaces`，统一管理 `apps/desktop` (Electron), `web` (Dashboard), `ui-tui` (终端界面) 和 `apps/shared`。
*   **版本控制**：使用 `package-lock.json` 锁定依赖版本。根目录定义了 `overrides` 字段，强制统一某些传递性依赖的版本（如 `lodash`, `yauzl`），以解决安全漏洞或兼容性问题。
*   **构建工具**：各子项目均采用 Vite 进行构建，Electron 应用使用 `electron-builder` 进行打包和多平台分发。

### 3. 系统级分发：Nix Flakes
*   **可重现构建**：通过 `flake.nix` 集成 Nix 包管理器，利用 `pyproject-nix` 和 `uv2nix` 将 Python 依赖转化为 Nix derivations。这确保了在 NixOS 或任何支持 Nix 的系统上，开发环境和生产构建具有比特级的一致性。
*   **多架构支持**：Flake 配置支持 `x86_64-linux`, `aarch64-linux` 和 `aarch64-darwin`。

### 4. 自动化更新与安全
*   **Dependabot 策略**：`.github/dependabot.yml` 仅启用 `github-actions` 的自动更新。对于 Python 和 npm 依赖，**禁用**了常规的自动版本升级 PR，以维护精确锁定的稳定性。仅当检测到 CVE 时，才通过 GitHub 的安全更新机制触发人工审查后的版本变更。
*   **Homebrew**：提供 `packaging/homebrew/hermes-agent.rb` Formula，用于 macOS 用户的系统级安装。

### 开发者规范
1.  **禁止范围依赖**：在 `pyproject.toml` 中添加新依赖时，必须指定确切版本（`==X.Y.Z`）。
2.  **同步锁文件**：修改依赖后，必须运行 `uv lock` 更新 `uv.lock` 并提交。
3.  **懒加载优先**：除非是全局必需的核心库，否则应将新依赖放入对应的 optional extra 中，并确保 `lazy_deps.py` 能正确处理其安装。
4.  **前端依赖统一**：在 Monorepo 中添加 JS 依赖时，需注意 workspaces 之间的版本一致性，必要时在根目录添加 `overrides`。