Hermes Agent 采用**分层、多源、插件化**的配置系统，核心围绕 `~/.hermes/config.yaml`（主配置）和 `~/.hermes/.env`（敏感凭证）展开，支持环境变量覆盖、NixOS/Docker 托管模式以及外部密钥管理器（如 Bitwarden）集成。

### 1. 核心架构与加载逻辑

*   **配置层级（优先级从高到低）**：
    1.  **环境变量 (Env Vars)**：直接覆盖 YAML 中的特定键值（如 `HERMES_HOME`, `OPENAI_API_KEY`）。
    2.  **用户配置 (`~/.hermes/config.yaml`)**：主要的 YAML 配置文件，包含模型、工具集、网关平台、终端后端等所有运行时设置。
    3.  **遗留配置 (`~/.hermes/gateway.json`)**：仅作为默认值兜底，`config.yaml` 中的同名键会覆盖它。
    4.  **代码默认值 (`DEFAULT_CONFIG`)**：硬编码在 `hermes_cli/config.py` 中的默认字典。

*   **加载流程**：
    *   **环境初始化**：启动时通过 `hermes_cli/env_loader.py` 加载 `.env` 文件。支持从 Bitwarden Secrets Manager 等外部源动态注入密钥，并自动清洗凭证中的非 ASCII 字符以防止 HTTP 头错误。
    *   **YAML 解析与合并**：`hermes_cli/config.py` 使用 `yaml.safe_load` 读取配置，并通过 `_deep_merge` 将用户配置合并到默认配置中。支持环境变量插值（`${VAR}`）。
    *   **缓存机制**：模块级缓存 (`_LOAD_CONFIG_CACHE`) 基于文件 mtime 和大小，避免重复解析开销。
    *   **容错处理**：若 `config.yaml` 解析失败，系统会备份损坏文件（`.corrupt.<ts>.bak`）并回退到默认配置，同时在 stderr 和日志中发出警告。

### 2. 关键配置文件与模块

*   **`hermes_cli/config.py`**：配置系统的核心。定义了 `DEFAULT_CONFIG`（包含模型、终端、代理、技能等所有默认值），提供 `load_config()`、`save_config()` 和原子写入功能。
    *   **托管模式检测**：通过 `HERMES_MANAGED` 环境变量或 `.managed` 标记文件识别 NixOS/Homebrew 托管安装，禁止在此模式下通过 CLI 修改配置。
*   **`gateway/config.py`**：网关专用配置。定义 `GatewayConfig` 数据类，处理多平台（Telegram, Discord 等）的连接参数、会话重置策略和流式传输设置。它从 `config.yaml` 的 `gateway` 和 `platforms` 节中提取数据。
*   **`hermes_constants.py`**：定义基础路径常量。`get_hermes_home()` 是单点真理，支持 `HERMES_HOME` 环境变量和 ContextVar 覆盖，确保多进程/多线程环境下路径一致性。
*   **`cli-config.yaml.example`**：官方提供的配置模板，详细注释了所有可用选项，包括模型路由、终端后端（Local/SSH/Docker/Modal）、技能配置和 MCP 服务器集成。

### 3. 设计约定与开发者规范

*   **敏感信息管理**：
    *   **严禁**将 API Key 等敏感信息直接写入 `config.yaml`。必须使用 `~/.hermes/.env` 或外部密钥管理器。
    *   `.env` 加载器会自动清洗凭证值中的非 ASCII 字符，并记录警告。
    *    dashboard 和环境写入器设有**黑名单**（如 `LD_PRELOAD`, `PYTHONPATH`, `HERMES_HOME`），防止通过配置界面注入危险环境变量。

*   **配置原子性与安全**：
    *   所有配置写入操作必须使用 `atomic_replace`（先写临时文件再重命名），防止断电或崩溃导致配置文件损坏。
    *   `~/.hermes` 目录及其子目录在非托管模式下默认设置为 `0700` 权限，文件为 `0600`，确保多用户系统下的隐私安全。

*   **扩展性与插件化**：
    *   **MCP 服务器**：通过 `config.yaml` 中的 `mcp_servers` 节配置，支持 stdio 和 HTTP 两种连接方式。
    *   **平台插件**：网关平台配置支持动态发现，插件可通过 `plugin.yaml` 注册新的平台类型，并在 `config.yaml` 中通过平台名称配置。
    *   **技能目录**：支持通过 `skills.external_dirs` 配置外部技能路径，实现跨项目共享。

*   **环境隔离**：
    *   **Profile 支持**：通过 `HERMES_HOME` 指向不同目录实现多 Profile 隔离。`get_hermes_home()` 会自动检测并警告未设置 `HERMES_HOME` 但激活了非默认 Profile 的情况。
    *   **容器感知**：在 Docker/K8s 环境中，`is_container()` 检测会跳过严格的权限设置（chmod），并适配卷挂载路径。

### 4. 常用配置示例

*   **模型切换**：在 `config.yaml` 中设置 `model.provider` 和 `model.default`。
*   **终端后端**：在 `terminal` 节中切换 `backend` 为 `local`, `ssh`, `docker`, `modal` 等，并配置相应的镜像或主机信息。
*   **网关平台**：在 `platforms` 节下配置各聊天平台的 Token 和 Home Channel。

该系统设计强调**安全性**（权限控制、敏感信息隔离）、**鲁棒性**（原子写入、容错回退）和**灵活性**（多源合并、插件扩展），能够适应从本地 CLI 到复杂分布式网关的各种部署场景。