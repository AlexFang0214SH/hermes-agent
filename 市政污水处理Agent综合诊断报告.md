# 市政污水处理 Agent 基于 Hermes Agent 框架的综合诊断报告

---

## 第一部分：项目背景与总体定位

### 1.1 Hermes Agent 框架概述

Hermes 是一个**个人 AI 代理框架**，核心设计哲学为"**核心窄、边缘宽**"——核心保持最小化，所有扩展能力通过 Skill、Plugin、MCP Server 等边缘机制实现。

| 维度 | 说明 |
|---|---|
| **运行模式** | CLI 交互、消息网关（20+ 平台）、TUI 终端 UI、Electron 桌面应用、ACP 编辑器集成 |
| **Agent 核心** | `AIAgent` 类（`run_agent.py`），同步对话循环：prompt构建 → LLM调用 → 工具执行 → 结果注入 |
| **扩展机制** | Skill（技能）、Plugin（插件）、MCP Server（模型上下文协议）、Context Files（上下文文件）、Hook（钩子） |
| **核心原则** | Prompt 缓存神圣不可侵犯；核心是窄腰，能力在边缘；不新增 `HERMES_*` 环境变量 |

### 1.2 目标项目定位

**目标：** 基于 Hermes Agent 框架，开发一套面向**市政污水处理领域**的专业 AI Agent。

**核心约束：**
- 不修改 Hermes 核心代码
- 全部通过配置 + Skill + Plugin + MCP 实现
- 充分利用用户自身的市政水务领域知识

**总体架构：**

```
┌──────────────────────────────────────────────────────────────────┐
│                     市政污水处理 Agent                            │
│                                                                  │
│  ┌─ 身份层 ─────────── SOUL.md（水务专家人设）                   │
│  ├─ 知识层 ─────────── RAG 知识库 + Context Files + Skill        │
│  ├─ 计算层 ─────────── ProcSim 工艺仿真引擎                      │
│  ├─ 编排层 ─────────── Dify 工作流 + Kanban + Cron              │
│  ├─ 交互层 ─────────── 企业微信/钉钉/CLI/Dashboard              │
│  └─ 记忆层 ─────────── Memory Provider + Session DB             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 第二部分：开发路径规划

### 2.1 五阶段开发计划

| 阶段 | 内容 | 工作量 |
|:---:|---|:---:|
| **Phase 1** | 领域身份定义（SOUL.md）+ 核心 Skill 编写 | 1-2 天 |
| **Phase 2** | MCP Server 开发（数据源接入）+ RAG 知识库对接 | 3-5 天 |
| **Phase 3** | Plugin 深度集成 + ProcSim 仿真引擎对接 | 5-7 天 |
| **Phase 4** | 工作流编排（Dify + Kanban + Cron）| 3-5 天 |
| **Phase 5** | 多平台分发 + 用户权限 + 运营优化 | 持续 |

### 2.2 推荐技术选型

| 组件 | 选型 | 理由 |
|---|---|---|
| 领域身份 | SOUL.md | Hermes 原生机制，零代码 |
| 领域技能 | Skill 文件 | 指导 Agent 何时、如何执行领域任务 |
| 知识检索 | MCP Server + Memory Provider | RAG 知识库动态注入 |
| 工艺仿真 | ProcSim MCP Server + Plugin | 计算引擎工具化 |
| 流程编排 | Dify（固定流程）+ Kanban（灵活任务） | 互补分工 |
| 定时任务 | Hermes Cron | 原生调度能力 |
| 消息分发 | Gateway 平台适配器 | 已有企业微信/钉钉适配器 |
| 用户管理 | 自建用户管理层 | Hermes 无多租户能力 |

---

## 第三部分：用户注册与权限管理诊断

### 3.1 现状诊断

Hermes 作为**个人代理**定位，**不具备完整的用户管理系统**。现有访问控制机制：

| 机制 | 位置 | 能力 | 局限 |
|---|---|---|---|
| **白名单** | `gateway/authz_mixin.py` | `ALLOWED_USERS` 环境变量按平台过滤 | 无注册、无角色 |
| **DM 配对** | `gateway/pairing.py` | 8字符一次性码 + 1小时过期 | 仅用于首次绑定 |
| **Dashboard OAuth** | `hermes_cli/dashboard_auth/` | OAuth + 密码登录，Session 管理 | 身份来自外部 IDP |
| **斜杠命令权限** | `gateway/slash_access.py` | 管理员白名单 + 用户命令列表 | 粗粒度 |
| **ACP 权限** | `acp_adapter/permissions.py` | 桥接宿主编辑器权限 | 仅编辑器场景 |

### 3.2 缺失能力

| 缺失 | 说明 | 市政场景需求 |
|---|---|---|
| 用户注册 | 无注册/登录流程 | 多厂区多角色需要独立账户 |
| 多租户 | 无数据隔离 | 不同厂区数据需隔离 |
| RBAC | 无角色/权限矩阵 | 厂长/工艺员/操作员/维修员权限不同 |
| 审计日志 | 无操作审计 | 合规要求记录谁做了什么 |
| 组织架构 | 无部门/层级 | 集团→分公司→厂区→班组 |

### 3.3 建议方案

在 Hermes 之上构建**独立用户管理层**：

```
┌─ 自建用户管理层 ──────────────────────────┐
│  用户注册/登录 │ RBAC │ 数据隔离 │ 审计  │
└──────────────┬────────────────────────────┘
               │ 鉴权后桥接
┌──────────────▼────────────────────────────┐
│  Hermes Gateway（现有访问控制）             │
│  白名单 + 配对 + 斜杠命令权限              │
└───────────────────────────────────────────┘
```

---

## 第四部分：RAG 标准规范知识库接入

### 4.1 五条接入路径对比

| 路径 | 机制 | 开发量 | 实时性 | 推荐度 |
|---|---|---|---|---|
| **R1. MCP Server** | RAG 封装为 MCP 服务器，Agent 工具调用 | 低 | 按需查询 | ★★★★★ |
| **R2. Memory Provider** | 实现 `MemoryProvider` ABC，`prefetch()` 自动注入 | 中 | 每轮自动 | ★★★★ |
| **R3. Skill** | Skill 指导 Agent 何时查阅知识库 | 最低 | 按需触发 | ★★★ |
| **R4. Context Files** | 将关键规范写入 AGENTS.md 等工作目录文件 | 最低 | 始终注入 | ★★ |
| **R5. Context Engine** | 实现 `ContextEngine` ABC 自定义压缩策略 | 高 | 自适应 | ★★ |

### 4.2 推荐组合方案（三层知识注入）

| 层级 | 路径 | 注入内容 | 时机 |
|---|---|---|---|
| **静态层** | Context Files (AGENTS.md) | 核心标准摘要（GB18918 一级A标准限值等） | 始终在系统提示中 |
| **动态层** | Memory Provider (prefetch) | 与当前对话相关的标准规范段落 | 每轮自动注入 |
| **按需层** | MCP Server (rag_query 工具) | 深度检索完整标准文档 | Agent 主动调用 |

### 4.3 MCP Server 工具设计

| 工具名 | 功能 |
|---|---|
| `rag_query` | 语义检索标准规范文档 |
| `rag_get_document` | 获取完整标准文档内容 |
| `rag_list_standards` | 列出可用标准规范目录 |
| `rag_compare` | 对比多个标准的差异 |

---

## 第五部分：工作流兼容性诊断

### 5.1 Hermes 五大工作流机制

| 机制 | 类型 | 核心能力 | 适用场景 |
|---|---|---|---|
| **Kanban 看板** | 多代理编排 | 状态机、DAG 依赖、自动分解、Goal 模式、人工参与、重试熔断 | 复杂多步骤任务 |
| **delegate_task** | RPC 子代理 | 同步调用、工具集隔离、最多3并发 | 独立子任务委派 |
| **Cron 调度** | 定时触发 | cron 表达式、链式工作流（`context_from`） | 周期性任务 |
| **Batch Runner** | 并行批处理 | 多输入并行、检查点恢复 | 批量处理 |
| **Teams Pipeline** | 状态机流水线 | 已实现的完整流水线案例 | 固定步骤流水线 |

### 5.2 v2 预留字段

Kanban DB schema 中已预留 `workflow_template_id` 和 `current_step_key` 字段，为未来原生工作流模板支持做准备。

### 5.3 市政污水场景映射

| 场景 | 推荐机制 | 说明 |
|---|---|---|
| 水质异常诊断 | Kanban（DAG 编排） | 数据收集 → 仿真分析 → 原因排查 → 方案制定 |
| 多厂区并行巡检 | Kanban + delegate_task | 主任务分解出多个子任务并行 |
| 设备维修审批 | Kanban（人工参与 block/unblock） | Agent 提交 → 人工审批 → Agent 继续 |
| 每日运行报告 | Cron | 每日定时自动生成 |
| 应急预案制定 | Kanban（Goal 模式） | 设定目标，Agent 自主分解步骤 |
| 合规审查 | Dify 工作流（固定步骤） | 标准化流程，步骤固定 |

### 5.4 现有机制不足

| 缺失能力 | 说明 |
|---|---|
| BPMN 可视化编排 | 无可视化流程设计器 |
| 可复用工作流模板 | 无模板库（v2 预留但未实现） |
| 事件驱动触发 | 无 SCADA 告警 → 自动触发 Agent 的机制 |
| 超时自动升级 | 无任务超时自动通知上级 |

---

## 第六部分：Dify 工作流引擎集成

### 6.1 两者定位对比

| 维度 | Hermes Agent | Dify |
|---|---|---|
| 核心定位 | 个人 AI 代理（终端/消息/桌面） | LLM 应用开发平台（工作流/RAG/Agent） |
| 工作流 | Kanban 任务板 + 子代理委派 | 可视化 DAG 工作流引擎 |
| 知识管理 | Context Files + Memory Provider + MCP | 内置 RAG（向量库 + 分段检索） |
| 多平台 | 20+ 消息平台网关 | Web App / API / 嵌入式 |

**核心结论：** 两者互补而非替代。

### 6.2 三种集成架构

| 架构 | 主控方 | 实现路径 | 推荐度 |
|---|---|---|---|
| **A: Hermes 调用 Dify** | Hermes | MCP Server 封装 Dify API → `mcp_dify_run_workflow` 等工具 | ★★★★★ |
| **B: Dify 调用 Hermes** | Dify | Dify LLM 节点对接 Hermes OpenAI 兼容 API Server（端口 8642） | ★★★★ |
| **C: 并列协作** | 混合 | Webhook 双向桥接 / 共享数据库 / 消息平台中继 | ★★★ |

### 6.3 推荐混合架构分工

| 职责 | 执行方 | 原因 |
|---|---|---|
| 标准化流程（固定步骤） | Dify | 可视化编辑、易于非技术人员维护 |
| 灵活对话/推理 | Hermes | 强大的多轮对话 + 工具调用 |
| RAG 知识检索 | Dify 或 Hermes MCP | Dify 内置向量库更成熟 |
| 终端操作/文件处理 | Hermes | Dify 没有终端和文件系统能力 |
| 多平台消息分发 | Hermes Gateway | Dify 只有 Web App |
| 定时任务 | Hermes Cron | Dify 没有内置调度 |

### 6.4 关键风险

| 风险 | 缓解 |
|---|---|
| 双重 LLM 调用成本 | Dify 用便宜模型跑固定流程，Hermes 用强模型做推理 |
| 延迟叠加 | Dify 工作流设置超时，MCP 配置合理 timeout |
| Prompt 缓存失效 | Dify 结果注入 volatile 层，不影响 stable 和 context 层 |

---

## 第七部分：ProcSim 工艺仿真引擎集成

### 7.1 ProcSim 定位

ProcSim 是**领域计算引擎**，核心价值是执行数值仿真计算并返回结构化结果。集成的核心问题是：**让 Agent 像调用计算器一样调用 ProcSim。**

### 7.2 六条集成路径

| 路径 | 机制 | ProcSim 接口 | 开发量 | 推荐度 |
|---|---|---|---|---|
| **P1. MCP Server** | 封装为 MCP 服务器 | Python SDK / HTTP | 中 | ★★★★★ |
| **P2. Hermes Plugin** | `ctx.register_tool()` | Python SDK | 中 | ★★★★ |
| **P3. Skill + execute_code** | Skill 指导写代码调用 | Python SDK | 低 | ★★★ |
| **P4. Skill + terminal** | Skill 指导 CLI 调用 | CLI 可执行文件 | 低 | ★★★ |
| **P5. Webhook 推送** | ProcSim 异步推送结果 | HTTP 回调 | 低 | ★★ |
| **P6. API Server** | 外部系统通过 Hermes API 触发 | HTTP 客户端 | 低 | ★★ |

### 7.3 推荐三层组合架构

| 层级 | 作用 | 内容 |
|---|---|---|
| **Skill 层** | 告诉 Agent "什么时候该仿真" | 领域知识：何时仿真、参数怎么填、结果怎么解读 |
| **Plugin 层** | 深度集成到 Agent 运行时 | `register_tool` + `register_hook`（pre_llm_call 自动注入仿真状态） |
| **MCP 层** | 暴露具体计算能力 | `run_simulation`、`sensitivity_analysis`、`optimize_dosing` 等工具 |

### 7.4 MCP 工具设计

| 工具 | 功能 | 输入 | 输出 |
|---|---|---|---|
| `run_simulation` | 执行工艺仿真 | 工艺类型、进水水质、运行参数 | 出水水质、能耗、污泥产量 |
| `sensitivity_analysis` | 灵敏度分析 | 基准工况、变量范围 | 变量影响排名 |
| `optimize_dosing` | 药剂投加优化 | 目标出水、药剂种类 | 最优投加量、预估费用 |
| `predict_alarm` | 预警模拟 | 当前工况 + 异常场景 | 未来水质趋势 |
| `compare_scenarios` | 方案对比 | 多组运行方案 | 各方案出水/能耗/成本 |
| `calibrate_model` | 模型校准 | 实测数据、待校准参数 | 校准后参数 + 拟合度 |

### 7.5 长时仿真处理

| 仿真时长 | 策略 |
|---|---|
| < 5 秒 | 同步调用 |
| 5-30 秒 | MCP 调用 + 进度反馈 |
| 30 秒-5 分钟 | 异步调用 + Cron 轮询结果 |
| > 5 分钟 | Webhook 推送模式 |

### 7.6 Schema 设计要点

- 参数使用领域术语（COD/NH3-N/TP/MLSS）
- 工艺类型明确枚举（A2O/SBR/MBR/氧化沟）
- 结果结构化返回 JSON + 自动对比 GB18918 标准
- 包含误差范围和置信度

---

## 第八部分：全系统联动架构

### 8.1 完整系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户交互层                                   │
│   企业微信 ←── Gateway ──→ 钉钉                                      │
│   CLI/TUI ←── Gateway ──→ Dashboard                                 │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                    Hermes Agent 核心                                 │
│                                                                     │
│  ┌─ 身份 ──────── SOUL.md（市政水务专家人设）                       │
│  ├─ 记忆 ──────── Memory Provider（会话记忆 + 知识记忆）            │
│  ├─ 上下文 ────── Context Files（核心标准摘要，始终注入）            │
│  ├─ 技能 ──────── Skill（水质诊断/工艺优化/合规审查...）            │
│  ├─ 钩子 ──────── Hook（pre_llm_call 自动注入仿真/知识上下文）      │
│  ├─ 调度 ──────── Cron（每日报告/定时巡检/定期校准）                │
│  └─ 编排 ──────── Kanban（复杂多步骤任务 DAG 编排）                 │
└──────┬────────────┬─────────────┬──────────────┬────────────────────┘
       │            │             │              │
┌──────▼──────┐ ┌──▼──────────┐ ┌▼───────────┐ ┌▼──────────────┐
│ MCP Server  │ │ MCP Server  │ │ MCP Server │ │ Dify 工作流   │
│ (RAG知识库) │ │ (ProcSim)   │ │ (SCADA)   │ │ (固定流程)    │
│             │ │             │ │            │ │               │
│ rag_query   │ │ run_sim     │ │ scada_read │ │ 合规审核流程  │
│ rag_get_doc │ │ optimize    │ │ scada_ctrl │ │ 报告生成流程  │
│ rag_compare │ │ predict     │ │ scada_alarm│ │ 应急响应流程  │
└─────────────┘ └─────────────┘ └────────────┘ └───────────────┘
```

### 8.2 典型端到端场景

**场景：水质异常应急响应**

```
1. SCADA 检测到出水 NH3-N 超标 → Webhook → Hermes Gateway
2. Agent 接收告警，自动触发"水质异常诊断"Skill
3. Agent 调用 mcp_scada_read 获取实时运行数据
4. Agent 调用 mcp_procsim_run_simulation 模拟当前工况
5. Agent 调用 mcp_rag_query 查阅 GB18918 相关条款和应急预案
6. Agent 综合分析，生成诊断报告和处置建议
7. Kanban 创建任务：通知运维人员（delegate_task）
8. Cron 安排 2 小时后复查仿真
9. 结果通过企业微信推送到运维群
```

**场景：每日运行报告自动生成**

```
1. Cron 每日 6:00 触发"每日报告"任务
2. Agent 调用 mcp_scada_read 获取过去 24h 运行数据
3. Agent 调用 mcp_procsim_run_simulation 模拟当日工况预测
4. Agent 调用 mcp_procsim_optimize_dosing 优化药剂投加方案
5. Agent 结合 RAG 知识库的格式规范生成报告
6. 报告通过消息平台推送到厂区管理群
```

---

## 第九部分：实施优先级与工作量估算

| 优先级 | 任务 | 工作量 | 依赖 |
|:---:|---|:---:|---|
| 1 | 编写 SOUL.md 定义水务专家身份 | 0.5 天 | 无 |
| 2 | 编写 3-5 个核心 Skill（水质诊断/工艺优化/合规审查等） | 2-3 天 | 1 |
| 3 | 配置 Context Files（核心标准摘要注入 AGENTS.md） | 0.5 天 | 无 |
| 4 | RAG MCP Server 开发（封装现有 RAG 知识库） | 2-3 天 | 无 |
| 5 | ProcSim MCP Server 开发（封装仿真引擎） | 2-3 天 | 无 |
| 6 | ProcSim Plugin 开发（深度集成 + Hook） | 3-5 天 | 5 |
| 7 | Dify 对接（Hermes API Server + Dify MCP Server） | 1-2 天 | 无 |
| 8 | Cron 定时任务配置（每日报告/定时巡检） | 0.5 天 | 4,5 |
| 9 | 消息平台配置（企业微信/钉钉适配器） | 1 天 | 无 |
| 10 | 用户管理层开发（注册/RBAC/审计） | 5-10 天 | 无 |
| **合计** | | **约 18-31 天** | |

---

## 第十部分：风险矩阵

| 风险类别 | 具体风险 | 影响 | 概率 | 缓解措施 |
|---|---|---|---|---|
| **成本** | 多系统联动导致多次 LLM 调用 | 高 | 高 | Dify 用便宜模型，Hermes 用强模型；合理设置 max_iterations |
| **性能** | 仿真计算+RAG检索+LLM推理叠加延迟 | 中 | 中 | 异步调用、结果缓存、长时仿真用 Webhook |
| **缓存** | Dify/ProcSim 结果注入破坏 Prompt 缓存 | 高 | 中 | 严格控制注入到 volatile 层（第三层） |
| **稳定性** | ProcSim SDK 崩溃影响 Agent 进程 | 高 | 低 | MCP 进程隔离，Plugin 异常兜底 |
| **安全** | 仿真数据含敏感运营信息 | 高 | 低 | 本地运行优先，避免云端传输 |
| **权限** | Hermes 无原生多租户 | 中 | 高 | 自建用户管理层，Hermes 仅做执行引擎 |
| **维护** | 多组件（RAG+ProcSim+Dify+Hermes）版本兼容 | 中 | 中 | 各组件接口标准化（MCP/API），松耦合 |

---

## 第十一部分：核心结论

| 结论项 | 诊断结果 |
|---|---|
| **Hermes 能否作为市政污水 Agent 框架** | 完全可行，所有扩展均通过边缘机制实现，无需改动核心 |
| **RAG 知识库能否接入** | 可行，私有服务器部署，通用无租户区分，MCP Server 封装 REST API |
| **Dify 能否集成** | 可行，自建部署，Dify 直接调用 ProcSim REST API + 独立 RAG 为 source of truth |
| **ProcSim 能否集成** | 可行，REST API + MCP Server 封装，异步调用 + 结果摘要，支持 Celery 并行 |
| **用户注册/权限管理** | Hermes 无原生能力，MVP 用白名单，长期自建（详见第十二部分） |
| **多用户/多租户支持** | 会话隔离已内置，租户隔离用 Profile 系统，完整 RBAC 需自建 |
| **工作流支持** | Kanban + Cron 覆盖灵活任务，固定流程用 Dify（自建，从零开始） |
| **LLM 模型** | 国产模型（Qwen/DeepSeek/GLM）+ 预留私有化部署，日预算 10,000 元 |
| **Agent 能力边界** | **只读模式**——禁用 terminal/write_file，仅给建议禁止执行，每条回答加免责声明 |
| **部署方式** | Docker Compose（无 K8s），支持公有云/私有云 |
| **预估开发周期** | MVP 30 天（详见第十四部分），完整系统 6-8 周 |
| **最大风险** | 国产模型工具调用能力、多系统联调复杂度、30 天 MVP 时间紧张 |

---

## 第十二部分：多用户 / 多租户能力深度诊断

### 12.1 核心判断

**"个人 AI 代理框架"的定位确实构成了对多用户/多租户场景的结构性限制。** Hermes 的设计意图是"一个人运行多个 Agent 实例"，而非"多个用户共享一个 Agent 服务"。但这个限制是**架构层面可绕过的**，而非不可逾越的。

### 12.2 Hermes 现有的"多用户"机制

#### Profile 系统——进程级隔离，非用户级

Profile 是 Hermes 最重的隔离单元。每个 Profile 拥有独立的：

| 资源 | 隔离方式 |
|---|---|
| `config.yaml` / `.env` | 独立文件，独立配置 |
| `SOUL.md` | 独立人设/系统提示 |
| `state.db` (SQLite) | 独立数据库文件 |
| Sessions（会话历史） | 独立存储 |
| Memory（记忆） | 独立存储 |
| Skills / Cron / Logs | 独立目录 |
| Gateway 进程 | 独立 systemd/s6 服务 |

**关键限制：** Profile 的设计场景是 **"同一个人跑多个 Agent"**（例如一个 coding 助手 + 一个研究助手），**不是** "多个人共享一个 Agent"。Profile 之间没有用户身份系统来路由请求。

#### 会话级用户隔离——`group_sessions_per_user`

在 Gateway（消息网关）层面，Hermes 有一个非常重要的多用户机制：

```yaml
# config.yaml
group_sessions_per_user: true   # 默认值
```

**工作方式：**

- 当 `true`（默认）：同一群/频道中的不同用户各自拥有**独立的会话历史**。Alice 和 Bob 在同一个 Discord 频道里对话，Hermes 为他们维护两套对话上下文。
- 当 `false`：整个群/频道共享一个会话，所有用户看到同一个对话历史。

**Session Key 构造逻辑：**
```
DM:     agent:main:{platform}:dm:{chat_id}
群组:   agent:main:{platform}:group:{chat_id}:{user_id}   ← 按用户隔离
频道:   agent:main:{platform}:channel:{chat_id}:{user_id} ← 按用户隔离
线程:   agent:main:{platform}:{type}:{chat_id}:{thread_id} ← 默认共享
```

**这是 Hermes 目前最接近"多用户"的能力**，但它是**会话级隔离**，不是数据级或权限级隔离。

#### 访问控制层

| 机制 | 文件 | 能力 | 粒度 |
|---|---|---|---|
| 白名单 | `gateway/authz_mixin.py` | `ALLOWED_USERS` 环境变量，按平台过滤 | 用户级允许/拒绝 |
| DM 配对 | `gateway/pairing.py` | 8字符一次性码，首次绑定 | 一次性身份确认 |
| 斜杠命令权限 | `gateway/slash_access.py` | 管理员白名单 + 用户可用命令列表 | 命令级 |
| Dashboard OAuth | `hermes_cli/dashboard_auth/base.py` | OAuth/密码登录，Session 含 user_id/email/org_id | 身份认证 |
| 平台适配器自管 | 各 adapter 的 `dm_policy`/`group_policy` | allowlist/open/disabled | 平台级 |

### 12.3 缺失能力清单

| 缺失 | 具体表现 |
|---|---|
| **用户注册** | 无注册流程、无用户数据库、无密码存储 |
| **RBAC** | 无角色定义、无权限矩阵、无资源级授权 |
| **多租户数据隔离** | `state.db` 是单文件 SQLite，无行级安全 |
| **组织架构** | 无部门/团队/层级概念 |
| **审计日志** | 无操作审计追踪（谁在何时做了什么） |
| **API 密钥分发给多用户** | `API_SERVER_KEY` 是单一共享密钥 |
| **配额/限流 per user** | 无每用户 token/费用上限 |

### 12.4 "个人代理"定位带来的架构约束

#### 文件系统共享

所有使用同一 Profile 的用户**共享同一文件系统**。Agent 执行的终端命令、文件读写操作，对所有使用该 Profile 的用户都是可见的。

#### 记忆共享

Memory Provider 是 Profile 级别的。所有使用同一 Profile 的用户**共享 Agent 的记忆**——Agent 会把 A 告诉它的偏好记下来，然后在和 B 对话时也使用这些记忆。

#### 单进程模型

每个 Profile 的 Gateway 是**单进程**。虽然可以并发处理多个用户的消息（异步），但所有请求最终通过同一个 Agent 实例处理。高并发场景下可能成为瓶颈。

#### API Server 单一认证

`api_server.py` 暴露的 OpenAI 兼容 API 使用**单一 `API_SERVER_KEY`**——所有调用者共享同一个密钥，没有 per-user 认证。

### 12.5 面向多用户/多租户的应对方案

#### 方案 A：Profile-per-User（每用户一个 Profile）

**适用场景：** 少量用户（5-20 人），每人需要完全独立的 Agent。

```
~/.hermes/
├── profiles/
│   ├── user-alice/       # Alice 独立的 config、memory、sessions
│   ├── user-bob/         # Bob 独立的 config、memory、sessions
│   └── user-charlie/     # Charlie 独立的 config、memory、sessions
```

**优势：**
- 完全隔离（配置、记忆、会话、文件全部独立）
- 零代码修改，纯配置实现
- 每个 Profile 可以有独立的 SOUL.md（不同角色/权限）

**劣势：**
- 每个 Profile 需要独立的 Gateway 进程 → 资源消耗线性增长
- 无统一用户管理界面
- 不适合大规模用户
- 用户需要自己管理 API key（或共享一个 key）

#### 方案 B：共享 Profile + 会话隔离（轻量多用户）

**适用场景：** 多用户共享同一个 Agent 实例，但各自对话独立。

```yaml
# config.yaml
group_sessions_per_user: true   # 默认已开启
```

**现有能力已支持：**
- 每个用户在各平台上自动获得独立会话
- Session Key 按 `user_id` 隔离
- 群聊中不同用户不会看到彼此的对话上下文

**需要补充的：**
- 自建用户认证层（在 API Server 前面加反向代理 + 身份认证）
- 自建 RBAC 层（在消息处理前拦截，按用户角色限制可用命令）

#### 方案 C：自建多租户管理层（完全多租户）

**适用场景：** SaaS 级别的多用户服务，需要完整的用户管理。

```
┌─ 自建管理层 ───────────────────────────────────────────┐
│  用户注册/登录 │ JWT 认证 │ RBAC │ 配额 │ 审计         │
└───────────────┬────────────────────────────────────────┘
                │ 认证后路由
┌───────────────▼────────────────────────────────────────┐
│  Hermes Gateway（执行层）                                │
│  ├─ Profile A: 厂区1的 Agent                            │
│  ├─ Profile B: 厂区2的 Agent                            │
│  └─ Profile C: 集团管理 Agent                            │
└────────────────────────────────────────────────────────┘
```

**需要自建的组件：**
1. **用户数据库**：注册、登录、密码管理
2. **JWT/API Key 网关**：认证 + 限流
3. **租户路由器**：根据用户身份路由到对应的 Hermes Profile
4. **权限中间件**：在 `pre_gateway_dispatch` Hook 中实现 RBAC 检查
5. **审计日志**：通过 `pre_tool_call` / `post_tool_call` Hook 记录操作

**Hermes 提供的可利用锚点：**
- `pre_gateway_dispatch` Hook → 实现认证 + 授权拦截
- `pre_tool_call` Hook → 实现工具级权限控制
- Profile 系统 → 实现租户级数据隔离
- Dashboard Auth Provider ABC → 插入自定义认证提供者

#### 方案 D：混合方案（推荐的市政污水场景方案）

针对市政污水处理场景的实际需求（多厂区、多角色、合规审计）：

```
┌─────────────────────────────────────────────────────┐
│                  统一 Web 管理平台                     │
│  用户管理 │ 角色分配 │ 厂区管理 │ 审计看板            │
└──────────┬──────────────────────────────┬────────────┘
           │                              │
    ┌──────▼──────┐              ┌────────▼────────┐
    │ 认证路由层   │              │  Dify 工作流引擎 │
    │ (Nginx/API  │              │  (标准审批流程)   │
    │  Gateway)   │              │                  │
    └──────┬──────┘              └──────────────────┘
           │
    ┌──────▼──────────────────────────────────────────┐
    │          Hermes Agent 集群（按厂区 Profile）      │
    │                                                  │
    │  Profile: plant-east   Profile: plant-west       │
    │  ├─ 该厂区的 SOUL.md    ├─ 该厂区的 SOUL.md      │
    │  ├─ 该厂区的记忆        ├─ 该厂区的记忆           │
    │  ├─ 该厂区的 Cron       ├─ 该厂区的 Cron         │
    │  └─ ProcSim MCP         └─ ProcSim MCP           │
    └──────────────────────────────────────────────────┘
```

**核心设计：**
- **一个厂区 = 一个 Profile**：数据天然隔离
- **角色通过 Web 管理层控制**：Hermes 不感知角色，只执行
- **Web 管理层 → Hermes API Server**：通过 `API_SERVER_KEY` + 自定义 JWT 中间件桥接
- **Dify 编排标准流程**：合规审核、应急响应等固定步骤流程
- **ProcSim MCP 共享**：所有厂区的仿真引擎共享同一个 MCP Server 实例

### 12.6 多用户/多租户结论与建议

| 维度 | 现状 | 评估 |
|---|---|---|
| 多用户会话隔离 | `group_sessions_per_user` 已支持 | **可用**，无需改动 |
| 多用户访问控制 | 白名单 + 配对 + 命令权限 | **基础可用**，但无 RBAC |
| 多租户数据隔离 | Profile 系统 | **可用但笨重**（一租户一 Profile） |
| 用户注册/管理 | 不存在 | **需要自建** |
| 审计追踪 | 不存在 | **需要自建**（可用 Hook 机制） |
| API 多租户 | 单一 Key | **需要自建认证网关** |

**市政污水 Agent 项目的多用户建议：**

1. **短期（验证阶段）**：使用 `group_sessions_per_user: true` + 白名单，让几个关键用户通过企业微信/钉钉直接与 Agent 对话。足够验证产品价值。

2. **中期（试点阶段）**：按厂区建立 Profile，实现数据隔离。在消息平台前面加一层自建的用户认证。

3. **长期（推广阶段）**：自建完整的多租户管理层（用户注册、RBAC、审计），Hermes 退化为纯执行引擎。这是 Hermes "个人代理"定位下的必然路径——它做执行层非常出色，但"多用户服务层"必须在它之上构建。

---

## 第十三部分：基于确认事项的技术决策

### 13.1 ProcSim 仿真引擎确认信息

| 确认项 | 内容 |
|---|---|
| 部署方式 | 独立服务，REST API 接口 |
| 许可证模式 | 支持私有化与云端部署，多计算节点弹性部署，Celery 并行计算 |
| 单次仿真耗时 | 秒级 ~ 分钟级 |
| 仿真结果数据量 | ≤ 500 字段，每字段为时间序列数据（与仿真时长相关） |
| 热启动 | 支持基于上次结果继续仿真 |
| 工艺模型 | ASM1 / ASM2d / ASM3，支持模型类型扩展 |
| 模型校准 | 支持输入实测数据校准，现阶段仅预留接口 |

### 13.2 ProcSim 架构影响与工具设计修正

**确认信息对架构的核心影响：**

- REST API + 独立服务 → **MCP Server 通过 HTTP 调用 ProcSim**，而非 Python SDK 直接嵌入
- Celery 并行 + 弹性多节点 → ProcSim 自身处理并发，Hermes MCP 端只需**提交任务 + 轮询结果**
- 秒级~分钟级耗时 → 需要**异步调用模式**：短仿真同步返回，长仿真提交后轮询
- ≤500 字段 × 时间序列 → 返回数据量可能很大，MCP 层必须做**结果摘要/采样**
- 热启动支持 → MCP 工具需要设计 `continue_simulation(session_id)` 接口
- ASM1/2d/3 + 可扩展 → Schema 中 `process_model` 用枚举但预留扩展空间
- 校准接口仅预留 → MVP 阶段不实现校准工具，仅注册占位

**修正后的 ProcSim MCP 工具设计：**

| 工具 | 调用方式 | 返回策略 |
|---|---|---|
| `procsim_submit` | 提交仿真任务 → 返回 task_id | 秒级仿真直接返回结果；分钟级返回 task_id + 轮询指引 |
| `procsim_result` | 查询 task_id 的结果 | 返回摘要（关键出水指标 + 合规判断）；完整数据存文件 |
| `procsim_continue` | 热启动：基于 task_id 继续仿真 | 同上 |
| `procsim_models` | 查询支持的工艺模型 | 返回 ASM1/ASM2d/ASM3 枚举 |
| `procsim_cancel` | 取消运行中的仿真 | 释放 Celery 资源 |

### 13.3 SCADA / 实时数据源

| 决策 | 内容 |
|---|---|
| MVP 阶段 | **不集成 SCADA**，仅在 Skill/工具 Schema 中预留接口占位 |
| 数据获取方式 | 用户手动提供运行数据（通过对话或文件上传） |
| 后续规划 | 待 MVP 验证后，根据实际 SCADA 协议（OPC UA / Modbus 等）设计数据管道 |

### 13.4 RAG 知识库确认信息

| 确认项 | 内容 |
|---|---|
| 部署位置 | 私有服务器 |
| 收录标准 | GB18918、CJJ 系列、地方标准，后续定期更新 |
| 文档类型 | 包含非结构化文档（如 PDF） |
| 更新频率 | 年度（标准修订或新标准发布后） |
| 多租户需求 | 无——知识库具有通用性，无需按用户/企业区分 |

**架构影响：** 所有 Profile 共享同一个 RAG MCP Server 实例，Context Files 注入核心标准摘要，年度更新手动触发即可。

### 13.5 Dify 工作流确认信息

| 确认项 | 内容 |
|---|---|
| 部署方式 | 自建部署 |
| 工作流状态 | 从零开始建 |
| 是否需要调用 ProcSim | 是 |
| Source of Truth | 独立 RAG 知识库（非 Dify 内置 RAG） |

**架构影响：**

- Dify 工作流中的仿真节点建议**直接 HTTP 调用 ProcSim REST API**（绕过 Hermes，减少链路延迟）
- Hermes 也通过 MCP 调用 ProcSim，两者共享同一个 ProcSim 服务实例
- Dify 不使用自身 RAG 能力，通过 HTTP 调用独立 RAG 服务（或 Hermes API Server 中转）

### 13.6 LLM 模型选择确认信息

| 确认项 | 内容 |
|---|---|
| 提供商限制 | 必须支持国产模型 + 预留私有化部署能力 |
| 多模型策略 | 不同任务可用不同模型 |
| 日预算 | 10,000 元/天 |

**推荐模型分层策略：**

| 任务类型 | 推荐模型 | 原因 |
|---|---|---|
| 日常对话/简单查询 | Qwen2.5-72B（API） | 成本低，中文能力好 |
| 仿真结果解读/工艺分析 | DeepSeek-V3 或 Qwen-Max | 推理能力强 |
| 合规审查/标准引用 | GLM-4 或 Qwen-Plus | 准确性优先 |
| 私有化备选 | Qwen2.5-72B-Instruct（vLLM） | 数据不出境 |

**预算风险评估：**

| 场景 | 估算消耗 | 是否可行 |
|---|---|---|
| 单用户轻度使用（10 轮对话/天） | ~50-100 元 | 充裕 |
| 10 用户中度使用（50 轮/用户/天） | ~500-1000 元 | 充裕 |
| 50 用户 + 大量仿真（含强模型推理） | ~2000-5000 元 | 可控 |
| 100+ 用户高频使用 | 可能超限 | 需要分层模型策略控制成本 |

**成本控制手段（Hermes 已内置）：**
- `agent/usage_pricing.py` — 按模型计费追踪
- `agent/credits_tracker.py` — 额度管理
- Prompt Caching — 减少重复 token 消耗
- 分层模型策略 — 简单任务用便宜模型

### 13.7 部署环境确认信息

| 确认项 | 内容 |
|---|---|
| 部署环境 | 公有云或私有云 |
| 容器编排 | 无 K8s，使用 Docker Compose |
| 外网访问 | 可访问外网（调用 LLM API），也支持私有化部署 LLM |
| GPU | 待确认 |

**Docker Compose 编排规划：**

```yaml
services:
  hermes-gateway:      # Hermes Agent 主服务
  procsim:             # ProcSim 仿真引擎
  procsim-worker:      # ProcSim Celery Worker
  dify:                # Dify 工作流引擎
  rag-service:         # RAG 知识库服务
  redis:               # Celery Broker + Dify 缓存
  postgres:            # Dify 数据库
  nginx:               # 反向代理 + 认证
```

### 13.8 用户与使用场景确认信息

| 确认项 | 内容 |
|---|---|
| 目标用户 | 厂长 / 工艺工程师 / 运营管理人员 / 设计院技术人员 |
| 使用设备 | PC / 手机 / 平板 |
| 首选界面 | 聊天界面 |
| 应用范围 | 面向行业应用，不局限于某个厂或水务集团 |
| AI 经验 | 用户具有使用 AI 工具的经验 |

**架构影响：** 聊天界面为首选 → 消息平台（企业微信/钉钉）为主入口，Dashboard 为辅。多设备适配需响应式 Web 或消息平台原生支持。

### 13.9 合规与安全确认信息

| 确认项 | 内容 |
|---|---|
| Agent 能力边界 | **仅给出建议，禁止执行** |
| 对话归档 | 预留归档开关，支持持久化保存 |
| 免责声明 | 每个回答中必须包含免责声明 |

**只读安全实现（工具集裁剪）：**

| 禁用工具 | 原因 |
|---|---|
| `terminal` | 禁止执行终端命令 |
| `write_file` / `patch` | 禁止写文件 |
| `browser_*` | 禁止浏览器操作（或限制为只读） |
| `computer_use` | 禁止桌面控制 |
| `process` | 禁止进程管理 |

| 保留工具 | 用途 |
|---|---|
| `read_file` / `search_files` | 只读文件访问 |
| `web_search` / `web_extract` | 信息检索 |
| `session_search` / `memory` | 会话和记忆 |
| `procsim_*` | 仿真工具（通过 MCP） |
| `rag_*` | 知识检索（通过 MCP） |
| `clarify` | 向用户提问确认 |
| `todo` | 任务追踪 |

**免责声明双保险实现：**
- **SOUL.md 指令**：在系统提示中写明"每条回答末尾必须加免责声明"
- **Gateway Hook 兜底**：`post_llm_call` Hook 自动追加免责声明文本，确保 100% 覆盖

### 13.10 运维与质量保证确认信息

| 确认项 | 内容 |
|---|---|
| 测试集 | 需要 Ground Truth 测试集 |
| 可解释性 | 预留机制 |
| 监控 | 预留功能，作为系统后台指标分析依据 |
| 版本管理 | Agent 的记忆/Skill 需要版本管理 |

### 13.11 组织与项目计划确认信息

| 确认项 | 内容 |
|---|---|
| 团队 | AI vibe Coding 团队 |
| MVP 时间线 | 30 天 |
| Hermes 上游贡献 | 验证可行性之前暂停 |
| 竞品/参考 | 暂无同类项目竞品或参考案例 |

---

## 第十四部分：30 天 MVP 交付计划

### 14.1 MVP 范围裁剪

| 包含 | 不包含（后续迭代） |
|---|---|
| ProcSim 基础仿真调用 | 模型校准（仅预留接口） |
| RAG 标准规范查询 | SCADA 实时数据集成 |
| GB18918 合规判断 | 多厂区 Profile 管理 |
| 企业微信/钉钉单平台 | 多平台同时接入 |
| 免责声明 + 只读约束 | 完整 RBAC 用户管理 |
| 对话归档开关 | 完整审计日志 |
| Ground Truth 基础测试集 | 自动化 CI/CD 测试 |
| 国产模型 API | 私有化 LLM 部署（仅预留接口） |
| Dify 基础工作流（可选） | Dify 全流程编排 |
| Docker Compose 部署 | K8s 编排 |

### 14.2 分阶段交付计划

#### Phase 0: 基础设施搭建（Day 1-3）

| Day | 任务 | 交付物 |
|---|---|---|
| 1 | Docker Compose 编排文件（Hermes + Redis + Nginx） | 可启动的容器环境 |
| 2 | Hermes Profile 创建 + 基础配置 | 水务 Agent Profile |
| 3 | LLM 提供商配置（国产模型 API + 私有化 vLLM 备选） | 可调用的 LLM |

#### Phase 1: 核心身份与知识层（Day 4-8）

| Day | 任务 | 交付物 |
|---|---|---|
| 4-5 | 编写 SOUL.md（水务专家人设 + 免责声明指令 + 只读约束） | 系统身份 |
| 6-7 | 编写 3 个核心 Skill（水质诊断、工艺优化建议、标准规范查询） | 技能文件 |
| 8 | 配置 Context Files（GB18918 核心指标摘要） | 静态知识注入 |

#### Phase 2: ProcSim 集成（Day 9-16）

| Day | 任务 | 交付物 |
|---|---|---|
| 9-10 | ProcSim MCP Server 开发（REST API 封装：submit/result/cancel） | MCP 服务器 |
| 11-12 | 异步调用模式（长仿真提交 → 轮询结果 → 摘要返回） | 异步调用链 |
| 13-14 | 结果摘要逻辑（500字段时间序列 → 关键出水指标 + 合规判断） | 摘要模块 |
| 15-16 | ProcSim MCP 联调测试 + Ground Truth 测试集验证 | 测试报告 |

#### Phase 3: RAG 集成（Day 17-20）

| Day | 任务 | 交付物 |
|---|---|---|
| 17-18 | RAG MCP Server 开发（封装现有 RAG REST API） | MCP 服务器 |
| 19 | RAG + ProcSim 联合查询测试（仿真结果 + 标准对比） | 联调报告 |
| 20 | 免责声明 Hook 实现 + 只读工具集配置 | 安全机制 |

#### Phase 4: 消息平台 + 用户入口（Day 21-25）

| Day | 任务 | 交付物 |
|---|---|---|
| 21-22 | 企业微信/钉钉适配器配置 + 白名单 | 消息通道 |
| 23-24 | Dashboard 部署 + 聊天界面验证 | Web 入口 |
| 25 | 对话归档开关配置 + 持久化验证 | 归档机制 |

#### Phase 5: 测试与打磨（Day 26-30）

| Day | 任务 | 交付物 |
|---|---|---|
| 26-27 | 端到端场景测试（水质异常诊断、工艺优化建议） | 测试报告 |
| 28 | Ground Truth 测试集跑通 + 准确率评估 | 评估报告 |
| 29 | 成本测试（1天真实使用 → Token 消耗 → 费用估算） | 成本报告 |
| 30 | 文档整理 + MVP 演示 | 演示材料 |

---

## 第十五部分：新发现风险

| 风险 | 说明 | 缓解 |
|---|---|---|
| **Dify 直接调 ProcSim 的链路复杂度** | Dify→ProcSim 和 Hermes→ProcSim 两条链路，需统一任务 ID 和状态 | ProcSim 侧实现统一的任务管理 API |
| **免责声明的 LLM 遵从性** | 仅靠 SOUL.md 指令，LLM 可能偶尔遗漏 | 必须加 Hook 兜底 |
| **时间序列数据摘要精度** | 500 字段时间序列 → 摘要可能丢失关键信息 | 提供 `procsim_result` 的 detail_level 参数（summary/medium/full） |
| **30 天 MVP 紧张** | 5 个模块（Hermes+ProcSim+RAG+Dify+消息平台）联调 | 严格按优先级裁剪，Dify 可降级为 Phase 5 可选 |
| **国产模型工具调用能力** | 部分国产模型的 function calling 能力弱于 GPT-4 | MVP 阶段先用 DeepSeek-V3 验证，备选 GLM-4 |
