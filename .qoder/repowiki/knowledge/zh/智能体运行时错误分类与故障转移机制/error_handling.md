Hermes Agent 采用了一套高度结构化、基于模式匹配的**错误分类与故障转移（Failover）系统**，专门用于处理多模态 AI 代理在复杂网络环境和异构 LLM 提供商（OpenAI, Anthropic, OpenRouter, Ollama 等）交互中的异常。

### 1. 核心架构：集中式错误分类器
系统的核心是 `agent/error_classifier.py`，它取代了分散在各处的字符串匹配逻辑，提供了一个统一的 `classify_api_error` 管道。该管道将原始异常转换为结构化的 `ClassifiedError` 对象，包含：
- **失败原因 (`FailoverReason`)**：如 `auth`（认证失败）、`billing`（余额耗尽）、`rate_limit`（限流）、`context_overflow`（上下文溢出）、`content_policy_blocked`（内容安全拦截）等。
- **恢复建议**：布尔标志位指示是否应重试 (`retryable`)、压缩上下文 (`should_compress`)、轮换凭证 (`should_rotate_credential`) 或切换到备用模型 (`should_fallback`)。

### 2. 分类流水线 (Priority-Ordered Pipeline)
分类器按照优先级顺序执行以下步骤，确保最具体的错误被优先捕获：
1. **提供商特异性模式**：识别 Anthropic 的 thinking block 签名失效、长上下文层级限制、llama.cpp 的语法解析错误等。
2. **HTTP 状态码细化**：结合状态码（401, 402, 403, 429, 5xx）与消息体内容进行二次判断。例如，402 可能是计费耗尽，也可能是瞬时限流（通过检测 "try again" 等关键词区分）。
3. **错误代码匹配**：解析响应体中的 `error.code` 或 `error.type`。
4. **消息模式匹配**：在没有状态码的情况下，通过正则/子串匹配识别计费、限流、上下文溢出或认证错误。
5. **传输层启发式**：识别 SSL/TLS 瞬时错误、服务器断开连接（结合会话大小判断是否为上下文溢出）以及通用的超时/连接错误。

### 3. 关键错误类型与处理策略
- **认证与授权 (`auth`)**：401/403 错误通常标记为不可直接重试 (`retryable=False`)，而是触发凭证轮换或提供商回退。
- **计费与限流 (`billing` vs `rate_limit`)**：系统严格区分永久性余额耗尽（需切换账号/提供商）和瞬时限流（需指数退避重试）。
- **上下文管理 (`context_overflow`)**：当检测到上下文过长或负载过大（413/特定 400 错误）时，标记 `should_compress=True`，触发代理内部的上下文压缩逻辑而非简单报错。
- **内容安全拦截 (`content_policy_blocked`)**：识别提供商的安全过滤器拒绝（如 OpenAI 的 `content_filter`），标记为不可重试，避免浪费 Token 重复发送被拒提示词。
- **传输与超时 (`timeout`)**：包括 SSL 警报、连接重置等，通常标记为可重试，并配合 `agent/retry_utils.py` 中的抖动退避算法防止雪崩效应。

### 4. 开发者规范
- **禁止硬编码字符串匹配**：在处理 API 异常时，必须使用 `classify_api_error` 函数，严禁在业务逻辑中直接 `if "rate limit" in str(e)`。
- **遵循恢复标志**：调用方（如 `conversation_loop.py`）应根据 `ClassifiedError` 的标志位决定下一步动作，而不是仅依赖 `retryable` 布尔值。
- **自定义错误扩展**：若新增提供商特异性错误，应在 `error_classifier.py` 中补充对应的模式列表（如 `_BILLING_PATTERNS`），并更新分类流水线。
- **异常透传**：底层适配器（如 `agent/transports/`）应尽量保留原始异常链，以便分类器提取状态码和响应体。