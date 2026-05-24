<p align="center">
  <strong>Adaptive Multi-Agent (AMA)</strong>
</p>

<p align="center">
  自适应多智能体调度插件 — 根据任务复杂度自动选择最佳协作模式
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/Hermes-%E2%89%A52.0.0-purple.svg" alt="Hermes">
  <img src="https://img.shields.io/badge/Version-1.0.0-blue.svg" alt="Version">
  <img src="https://img.shields.io/github/last-commit/weksbwrx62862/adaptive-multi-agent.svg" alt="Last Commit">
</p>

---

## 简介

Adaptive Multi-Agent（AMA）是 [Hermes Agent](https://github.com/weksbwrx62862/hermes) 的后端调度插件，核心能力是**根据任务复杂度自动选择最佳多智能体协作模式**。它内置 5 种协作模式，覆盖从简单生成-验证到复杂共享状态协作的全频谱场景；采用 Thompson Sampling + 贝叶斯平滑混合策略进行模式选择，并通过 SQLite 持久化性能数据实现历史学习。AMA 还与 [Model Router](https://github.com/weksbwrx62862/model-router) 双向联动，形成"模型选型 ↔ 模式选型"的闭环反馈。

## 功能矩阵

| 能力 | 说明 |
|------|------|
| **5 种协作模式** | Generator-Verifier / Orchestrator-Subagent / Agent Teams / Message Bus / Shared State |
| **自动复杂度评估** | 7 维特征关键词 + 隐性复杂度信号 + LLM 二次精修（模糊区间触发） |
| **智能模式选择** | Thompson Sampling 采样 + 贝叶斯平滑 + 规则引擎混合策略 |
| **需求澄清** | 多轮 LLM 提问明确模糊需求，再基于澄清结果评分 |
| **失败自动切换** | 执行失败后按错误类型智能升级/降级模式，含冷却期防抖 |
| **熔断保护** | 每模式独立 CircuitBreaker，连续失败自动熔断，半开探测恢复 |
| **性能持久化** | SQLite 存储执行记录与性能指标，支持按时间范围统计 |
| **历史学习** | 指数衰减加权更新性能数据，TS 后验随执行自动演进 |
| **Model Router 联动** | AMA→Router 推送任务权重，Router→AMA 弱模型时降级重型模式 |
| **模式插件扩展** | 实现 ModePlugin 协议即可注册新模式，通过 ModeGraph 声明式定义执行图 |
| **诊断与可观测** | 内置 TS 参数、熔断器状态、Mermaid 流程图生成、执行追踪 |

## 架构图

```
                            ┌─────────────────────┐
                            │     Hermes Agent     │
                            └──────────┬──────────┘
                                       │ delegate_task
                                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    Adaptive Multi-Agent                       │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐  │
│  │   Assessor    │──▶│   Selector   │──▶│  Mode Executor  │  │
│  │ (复杂度评估)  │   │ (模式选择)   │   │  (模式执行)     │  │
│  └──────┬───────┘   └──────┬───────┘   └───────┬─────────┘  │
│         │                  │                    │             │
│    ┌────▼────┐      ┌─────▼──────┐      ┌─────▼──────┐      │
│    │关键词匹配│      │Thompson    │      │ 5种内置模式 │      │
│    │隐性信号  │      │Sampling    │      │ + 插件模式  │      │
│    │LLM精修  │      │贝叶斯平滑  │      │ ModeGraph   │      │
│    └─────────┘      └────────────┘      └─────┬──────┘      │
│                                               │             │
│  ┌──────────────┐   ┌──────────────┐   ┌─────▼──────┐      │
│  │CircuitBreaker│   │ RetryPolicy  │   │Persistence │      │
│  │  (熔断保护)  │   │  (重试策略)  │   │ (SQLite)   │      │
│  └──────────────┘   └──────────────┘   └────────────┘      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Model Router 双向联动                     │   │
│  │  AMA → Router: 推送任务权重 (set_task_weight)         │   │
│  │  Router → AMA: 弱模型降级重型模式                     │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

## 快速开始

### 前置条件

- Python 3.10+
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) >= 2.0.0

### 安装

```bash
git clone https://github.com/weksbwrx62862/adaptive-multi-agent.git
cd adaptive-multi-agent
pip install -e .
```

### 配置

在 Hermes 配置文件中启用插件：

```yaml
# hermes_config.yaml
plugins:
  - name: adaptive_multi_agent
    path: ./adaptive-multi-agent
```

可选的高级配置（写入 `~/.hermes/config.yaml`）：

```yaml
ama:
  allow_mode_switch: true
  switch_threshold:
    max_tokens: 50000
    max_time: 300
  default_mode: auto
  max_concurrent_children: 3
  llm_refine_enabled: true
```

### 最小示例

```python
# 通过 Hermes 工具调用
result = ctx.dispatch_tool("ama_execute", {
    "task": "设计并实现一个用户认证系统，包含注册、登录和Token刷新功能",
    "context": "项目使用 FastAPI + PostgreSQL",
})

# 仅评估不执行
assessment = ctx.dispatch_tool("ama_assess", {
    "task": "分析竞品并生成对比报告",
})
```

## 核心功能详解

### 五种协作模式

| 模式 | 复杂度区间 | 执行流程 | 适用场景 |
|------|-----------|---------|---------|
| **Generator-Verifier** | 1-3 | 生成 → 验证 → 迭代改进（最多5轮） | 代码生成、文本创作、单步验证 |
| **Orchestrator-Subagent** | 4-6 | 分解子任务 → 并行执行 → 综合结果 | 多步骤任务、需要拆分委派 |
| **Agent Teams** | 7-8 | PM规划 → Engineer+Reviewer并行 → 综合 | 多角色协作、质量敏感任务 |
| **Message Bus** | 9 | 规划事件拓扑 → 事件循环处理 → 汇总 | 事件驱动、监控告警、实时处理 |
| **Shared State** | 10 | Explorer→Analyzer→Synthesizer→Validator 迭代收敛 | 跨领域综合分析、需要共享知识 |

### 复杂度评估体系

AMA 采用三层评估机制：

1. **规则引擎**：7 维特征关键词匹配（并行、角色、协作等）+ 隐性复杂度信号（领域/输出/范围/多组件）+ 子任务数量 + 多动作动词检测
2. **LLM 二次精修**：当规则评分落在模糊区间（默认 3.0-7.0）时，触发大模型重新评估，融合语义理解
3. **需求澄清**：可选的多轮 LLM 提问，帮助明确模糊需求后再评分

评分公式：`基础分 1.0 + 显性特征加分 + 隐性信号加分`，上限 10.0。

### 模式选择策略

ModeSelectionEngine 采用 Thompson Sampling + 贝叶斯平滑混合策略：

- **Thompson Sampling**：从 Beta(α, β) 后验分布采样各模式期望成功率，选择采样值最高的模式
- **贝叶斯平滑**：性能评分加入先验（3次成功/5次试验），避免小样本极端值
- **冷启动探索**：无历史数据的模式获得探索加分（0.7 初始分 + 衰减探索奖励）
- **规则兜底**：当所有 TS 采样值低于 0.6 时，回退到历史最优模式

### 失败自动切换

执行失败后，AMA 根据错误类型智能选择切换方向：

- **升级触发**：验证失败、内部错误、JSON 解析错误 → 向高复杂度模式切换
- **降级触发**：上下文溢出、超时 → 向低复杂度模式切换
- **冷却期**：同一切换路径 30 秒内不重复触发，防止抖动
- **中间结果传递**：切换时将前次模式的中间结果注入新模式的上下文

### 熔断保护

每个模式配备独立 CircuitBreaker：

- **CLOSED**：正常状态，允许执行
- **OPEN**：连续 5 次失败后熔断，拒绝执行（60 秒恢复期）
- **HALF_OPEN**：恢复期后允许一次探测请求，成功则复位，失败则继续熔断

### Model Router 联动

AMA 与 Model Router 形成双向闭环：

- **AMA → Router**：执行前推送任务复杂度权重，影响模型选型策略
- **Router → AMA**：当活跃模型质量 ≤ 2 时，自动排除 Agent Teams / Shared State / Message Bus 等重型模式
- **反馈闭环**：执行结果回流 Router，记录模型成功率与 token 消耗

### 提供的工具

| 工具 | 功能 | 必需参数 |
|------|------|---------|
| `ama_execute` | 执行多智能体协作任务 | `task` |
| `ama_assess` | 评估复杂度并推荐模式（不执行） | `task` |
| `ama_switch_mode` | 手动切换会话模式 | `mode` |
| `ama_stats` | 查询调度统计和性能指标 | - |
| `ama_cancel` | 取消正在执行的任务 | `task_id` |
| `ama_clarify` | 多轮需求澄清与评分 | `task` |
| `ama_diagnose` | 诊断内部状态（TS参数/熔断器/执行追踪） | - |

### 提供的钩子

| 钩子 | 说明 |
|------|------|
| `post_tool_call` | 工具调用后监控 token/时间超限，输出告警与建议 |
| `on_session_start` | 会话启动时重置模式覆盖 |
| `on_session_end` | 会话结束时清理结果缓存 |

## 技术栈

```
┌──────────────┬──────────────────────────────────────┐
│ 类别         │ 技术                                 │
├──────────────┼──────────────────────────────────────┤
│ 语言         │ Python 3.10+                         │
│ 插件框架     │ Hermes Agent Plugin API               │
│ 持久化       │ SQLite (WAL mode)                     │
│ 算法         │ Thompson Sampling, 贝叶斯平滑          │
│ 容错         │ Circuit Breaker, 指数退避重试          │
│ 图执行       │ ModeGraph (声明式状态图)               │
│ 扩展机制     │ ModePlugin Protocol                   │
│ 可视化       │ Mermaid 流程图生成                     │
│ 依赖         │ pyyaml                                │
└──────────────┴──────────────────────────────────────┘
```

## 项目结构

```
adaptive-multi-agent/
├── __init__.py          # 插件注册入口，工具与钩子声明
├── plugin.yaml          # 插件元数据声明
├── engine.py            # 调度引擎（评估/选择/执行/切换）
├── graph.py             # 协作拓扑图（ModeGraph 声明式定义）
├── handlers.py          # 工具处理器 + 钩子处理器
├── persistence.py       # 状态持久化（SQLite CRUD + 统计查询）
├── schemas.py           # 工具 JSON Schema 定义
└── subagent.py          # 子智能体管理（配置/注册/熔断/重试/DAG校验）
```

## 开发指南

### 环境搭建

```bash
git clone https://github.com/weksbwrx62862/adaptive-multi-agent.git
cd adaptive-multi-agent
pip install -e .
```

### 扩展新模式

实现 `ModePlugin` 协议即可注册自定义模式：

```python
from adaptive_multi_agent.subagent import ModePlugin, AgentMode
from adaptive_multi_agent.graph import ModeGraph

class MyCustomMode(ModePlugin):
    @property
    def mode(self) -> AgentMode:
        return AgentMode("custom_mode")

    def register_graph(self, graph: ModeGraph) -> None:
        graph.add_node("step_a", handler_a)
        graph.add_node("step_b", handler_b)
        graph.add_edge("step_a", "step_b")

    def validate_config(self) -> bool:
        return True

    def describe(self) -> str:
        return "自定义模式描述"
```

### 代码规范

- 遵循 PEP 8，使用 `from __future__ import annotations` 延迟类型求值
- 数据类使用 `@dataclass`，枚举使用 `Enum`
- 线程安全：共享状态使用 `threading.Lock` 保护
- 日志统一使用 `logging.getLogger("ama.<module>")`

### 测试

通过 Hermes 运行时进行集成测试：

```bash
# 在 Hermes 环境中调用工具验证
hermes tool ama_assess '{"task": "测试任务"}'
hermes tool ama_execute '{"task": "写一个快速排序函数"}'
hermes tool ama_stats '{"detail": true}'
```

## 路线图

- [ ] **v1.1** — 支持 DAG 拓扑排序并行执行子任务（当前为全并行）
- [ ] **v1.1** — 添加记忆层语义检索（embedding 向量化替代关键词匹配）
- [ ] **v1.2** — Web UI 仪表盘（执行历史、模式分布、成功率趋势可视化）
- [ ] **v1.2** — 多 Agent 并发隔离（git worktree 级别的工作目录隔离）
- [ ] **v1.3** — 分布式模式（跨进程/跨机器的 Message Bus 协作）
- [ ] **v1.3** — A/B 测试框架（新模式与基线模式的统计显著性对比）

## FAQ

**Q: AMA 会自动选择模式，我还需要手动指定吗？**
A: 通常不需要。AMA 的 Thompson Sampling 会根据历史表现自动选择最优模式。但在特定场景下（如调试、基准测试），可通过 `ama_switch_mode` 或 `ama_execute` 的 `force_mode` 参数手动指定。

**Q: LLM 二次精修什么时候触发？**
A: 当规则引擎评分落在模糊区间（默认 3.0-7.0）时触发。可通过 `llm_refine_range` 配置调整区间，或通过 `llm_refine_enabled: false` 关闭。

**Q: 模式切换会导致任务从头执行吗？**
A: 不会。切换时 AMA 会将前次模式的中间结果注入新模式的上下文，新模式基于已有成果继续推进。

**Q: 性能数据存储在哪里？**
A: SQLite 数据库，路径为 `~/.hermes/ama_state.db`，包含性能指标表、执行记录表、状态快照表和记忆表。

**Q: 如何与 Model Router 协同工作？**
A: AMA 在执行前向 Router 推送任务复杂度权重（影响模型选型），Router 在模型质量较低时通知 AMA 排除重型模式。两者通过 Python 导入自动联动，无需额外配置。

**Q: CircuitBreaker 熔断后如何恢复？**
A: 熔断后进入 60 秒恢复期，之后自动转为半开状态允许一次探测请求。探测成功则复位为关闭状态，失败则继续熔断。

## Contributing

欢迎贡献代码、报告问题或提出建议！

1. Fork 本仓库
2. 创建特性分支：`git checkout -b agent/AMA-<task-id>-<description>`
3. 提交变更，遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范
4. 推送分支并创建 Pull Request
5. 确保 PR 描述包含：变更摘要、设计决策、测试覆盖

## License

[MIT](LICENSE)

## Security

- 本插件不收集或传输任何用户数据到外部服务
- 所有持久化数据存储在本地 SQLite（`~/.hermes/ama_state.db`）
- 不在日志中输出 API Key 或敏感信息
- 如发现安全漏洞，请通过 GitHub Issues 私密报告

## 致谢

- [Hermes Agent](https://github.com/weksbwrx62862/hermes) — 插件运行时框架
- [Model Router](https://github.com/weksbwrx62862/model-router) — 模型选型联动
- Thompson Sampling 算法参考：*A Tutorial on Thompson Sampling* (Russo et al., 2018)

<p align="center">
  <em>让多智能体协作像呼吸一样自然</em>
</p>
