## 1. 核心系统与框架
Hermes Agent 采用基于 Python 标准库 `logging` 模块的**集中式日志架构**，通过 `hermes_logging.py` 提供统一的初始化入口。系统不依赖第三方日志框架（如 Loguru），而是通过深度定制 `RotatingFileHandler`、`Formatter` 和 `LogRecordFactory` 来实现结构化输出、安全脱敏和多进程安全。

- **核心组件**：
  - **`hermes_logging.setup_logging()`**：全局唯一的日志配置入口，支持 CLI、Gateway、GUI 和 Cron 等多种运行模式。
  - **`RedactingFormatter`**：自定义格式化器，集成正则表达式引擎，在日志写入前自动脱敏 API Key、Token、密码等敏感信息。
  - **`_ManagedRotatingFileHandler`**：增强型轮转处理器，解决 Windows 下的文件锁定冲突（使用 `concurrent-log-handler`）以及 Linux/NixOS 下的外部轮转（如 logrotate）导致的句柄失效问题。
  - **会话上下文注入**：通过 `_install_session_record_factory()` 全局替换 `LogRecord` 工厂，自动为每条日志注入 `[session_id]` 标签，实现跨线程的会话追踪。

## 2. 关键文件与包结构
- **`hermes_logging.py`**：日志系统的核心实现，包含处理器定义、过滤器逻辑、会话上下文管理及配置读取。
- **`agent/redact.py`**：定义 `RedactingFormatter` 及 `redact_sensitive_text()` 函数，维护超过 30 种敏感信息匹配模式（如 OpenAI `sk-`、GitHub PAT、JWT、DB 连接串等）。
- **`gateway/run.py` / `hermes_cli/main.py`**：主要调用方，分别在网关启动和 CLI 执行时调用 `setup_logging(mode="gateway"/"cli")`。
- **`plugins/observability/langfuse/`**：可观测性插件，通过 Observer Hooks 将日志事件转化为结构化 Trace 发送至 Langfuse，与基础日志系统互补。
- **`docs/observability/README.md`**：定义了 Hermes 的可观测性契约，包括 Session、Turn、API Request 和 Tool Call 的生命周期事件。

## 3. 架构设计与约定
### 3.1 多文件分流策略
系统根据 `mode` 参数将日志分流到不同文件，所有文件均位于 `~/.hermes/logs/`：
- **`agent.log`**：**全量日志**。记录 INFO 及以上级别的所有活动，是主要的调试依据。
- **`errors.log`**：**错误专用**。仅记录 WARNING 及以上级别，用于快速故障排查。
- **`gateway.log`**：**网关专用**（仅 `mode="gateway"`）。通过 `_ComponentFilter` 仅捕获 `gateway.*` 和 `hermes_plugins.*` 命名空间的日志，隔离平台适配层事件。
- **`gui.log`**：**界面专用**（仅 `mode="gui"`）。捕获 `hermes_cli.web_server`、`tui_gateway` 等 Dashboard 相关日志。

### 3.2 会话追踪机制
- **线程隔离**：使用 `threading.local()` 存储 `session_id`。
- **自动注入**：开发者无需在每次 `logger.info()` 中手动传递 session_id。只需在会话开始时调用 `hermes_logging.set_session_id(sid)`，后续该线程产生的所有日志（包括第三方库日志）都会自动带上 `[sid]` 标签。

### 3.3 安全脱敏 (Security by Default)
- **强制脱敏**：`RedactingFormatter` 会拦截所有日志消息。即使开发者不小心记录了包含 `sk-...` 或 `Authorization: Bearer ...` 的字符串，落盘前也会被替换为 `***` 或部分掩码（如 `sk-proj...7890`）。
- **配置开关**：可通过 `security.redact_secrets: false` 或环境变量 `HERMES_REDACT_SECRETS=false` 关闭，但默认开启。

### 3.4 噪声抑制
- 系统自动将 `openai`、`httpx`、`httpcore`、`asyncio` 等高频第三方库的日志级别提升至 `WARNING`，防止底层网络细节淹没业务日志。

## 4. 开发者规范
### 4.1 日志记录最佳实践
- **获取 Logger**：始终使用 `logger = logging.getLogger(__name__)` 以利用命名空间过滤。
- **会话绑定**：在处理用户请求的入口（如 `run_conversation`）立即调用 `set_session_context(session_id)`，结束时调用 `clear_session_context()`。
- **避免敏感数据**：尽管有自动脱敏，仍应避免直接记录完整的 `args` 或 `response` 对象。优先记录摘要信息（如 token 数量、工具名称）。

### 4.2 性能与可靠性
- **异步安全**：日志处理器已针对多进程环境优化。在 Windows 上自动使用 `ConcurrentRotatingFileHandler` 避免 `PermissionError`；在 Linux 上能自动检测并恢复被外部工具轮转的文件句柄。
- **配置优先级**：日志级别和轮转大小可通过 `config.yaml` 中的 `logging` 段配置，但函数调用时的显式参数优先级更高。

### 4.3 可观测性扩展
- 对于需要分布式追踪或高级指标的场景，应使用 `plugins/observability` 提供的 Observer Hooks（如 `pre_api_request`, `post_tool_call`），而不是依赖纯文本日志解析。