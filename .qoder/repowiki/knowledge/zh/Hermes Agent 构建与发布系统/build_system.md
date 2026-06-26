## 1. 核心构建体系

Hermes Agent 采用 **多语言混合构建架构**，以 Python (`uv`) 为核心后端，Node.js (`npm`) 为前端及 TUI 界面，并辅以 Nix 和 Docker 实现跨平台分发。

*   **Python 后端**: 使用 `uv` 作为包管理器和构建工具。依赖管理遵循**严格精确锁定策略**（Exact Pinning），所有核心依赖在 `pyproject.toml` 中均使用 `==X.Y.Z` 格式，并通过 `uv.lock` 确保全环境一致性。这种策略旨在最小化供应链攻击面（如应对 PyPI 恶意包注入）。
*   **前端与 TUI**: 采用 Monorepo 结构，通过 `package.json` 的 `workspaces` 管理 `web` (React/Vite)、`ui-tui` (Ink/React) 和 `apps/desktop` (Electron/Tauri)。构建产物在打包前通过 `npm run build` 预编译并嵌入 Python 包中。
*   **Nix Flake**: 提供声明式开发环境与打包方案。利用 `uv2nix` 将 `uv.lock` 转换为 Nix derivations，支持生成包含不同可选依赖组（如 `messaging`, `full`）的定制化包。

## 2. 容器化与部署

*   **Docker 镜像**: 基于 `debian:13` (trixie) 构建，采用多阶段构建优化层缓存。
    *   **进程监管**: 引入 `s6-overlay` 替代传统的 `tini`，实现对主进程、Dashboard 及多 Profile 网关的动态监管与僵尸进程回收。
    *   **安全隔离**: 运行时默认切换至非 root 用户 (`hermes`, UID 10000)，并通过 `/opt/data` 卷挂载实现数据持久化与代码只读隔离。
    *   **多架构支持**: CI 流程分别在原生 `amd64` 和 `arm64`  runners 上构建，最终合并为多架构 Manifest List。
*   **安装脚本**: 提供跨平台的 `install.sh` (Linux/macOS) 和 `install.ps1` (Windows)，支持自动检测环境并配置 `uv` 虚拟环境。

## 3. CI/CD 流水线 (GitHub Actions)

*   **测试矩阵**: 采用分片并行执行策略 (`scripts/run_tests_parallel.py`)，将测试文件按历史耗时动态分配到 6 个并行 Job 中，显著缩短反馈周期。
*   **质量门禁**: 
    *   **Linting**: 结合 `ruff` (强制规则 PLW1514) 和 `ty` (类型检查) 进行增量 diff 审查。
    *   **安全检查**: 集成 `osv-scanner` 和自定义的 `supply-chain-audit` 工作流，监控依赖漏洞。
*   **发布流程**: 采用 **CalVer** (日历版本) 命名规范。`scripts/release.py` 自动化处理 Changelog 生成、版本号同步（`pyproject.toml`, `acp_registry/agent.json`）及 GitHub Release 创建。

## 4. 开发者规范

*   **依赖更新**: 修改 `pyproject.toml` 后必须运行 `uv lock` 重新生成锁文件。禁止在核心依赖中使用版本范围（如 `>=1.0`）。
*   **前端构建**: 提交代码前需确保 `web` 和 `ui-tui` 的构建产物已更新，或通过 CI 验证构建完整性。
*   **Docker 开发**: 本地构建时可传入 `HERMES_GIT_SHA` 参数以确保运行时版本信息准确。