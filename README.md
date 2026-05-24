# Adaptive Multi-Agent

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/version-1.0.0-blue.svg" alt="Version">
</p>

自适应多智能体调度插件 — 根据任务复杂度自动选择最佳 Anthropic 多智能体协作模式，支持智能模式切换、性能持久化和历史学习。

## 5 种协作模式

| 模式 | 复杂度 | 适用场景 |
|------|--------|----------|
| Generator-Verifier | 低 | 代码生成 + 验证 |
| Orchestrator-Subagent | 中 | 任务分解 + 委派 |
| Agent Teams | 中高 | 多角色协作 |
| Message Bus | 高 | 事件驱动通信 |
| Shared State | 极高 | 共享状态协作 |

AMA 会自动评估任务复杂度并选择最优模式，无需手动指定。

## 安装

### 前置条件

- Python 3.10+
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) >= 2.0.0

### 从源码安装

```bash
git clone https://github.com/weksbwrx62862/adaptive-multi-agent.git
cd adaptive-multi-agent
pip install -e .
```

### 依赖

```bash
pip install pyyaml
```

## 使用

```yaml
# hermes_config.yaml
plugins:
  - name: adaptive_multi_agent
    path: ./adaptive-multi-agent
```

启用后，AMA 会自动介入 `post_tool_call` 阶段，根据任务进行评估和调度。

## 提供的工具

| 工具 | 功能 |
|------|------|
| `ama_execute` | 执行多智能体协作任务 |
| `ama_assess` | 评估任务复杂度并推荐模式 |
| `ama_switch_mode` | 手动切换协作模式 |
| `ama_stats` | 查看调度统计和性能指标 |

## 提供的钩子

| 钩子 | 说明 |
|------|------|
| `post_tool_call` | 工具调用后评估与调度 |
| `on_session_start` | 会话启动时恢复历史状态 |
| `on_session_end` | 会话结束时持久化性能数据 |

## 项目结构

```
adaptive-multi-agent/
├── plugin.yaml          # 插件声明
├── engine.py            # 调度引擎
├── graph.py             # 协作拓扑图
├── handlers.py          # 工具处理器
├── persistence.py       # 状态持久化
├── schemas.py           # 数据模式
└── subagent.py          # 子智能体管理
```

## 开发

```bash
git clone https://github.com/weksbwrx62862/adaptive-multi-agent.git
cd adaptive-multi-agent
pip install -e .
# 通过 Hermes 运行时测试
```

## License

MIT