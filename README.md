# Adaptive Multi-Agent

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
</p>

自适应多智能体调度插件 — 根据任务复杂度自动选择最佳协作模式。

## 5 种协作模式

| 模式 | 适用场景 |
|------|----------|
| Generator-Verifier | 代码生成 + 验证 |
| Orchestrator-Subagent | 任务分解 + 委派 |
| Agent Teams | 多角色协作 |
| Message Bus | 事件驱动通信 |
| Shared State | 共享状态协作 |

## 快速开始

```yaml
plugins:
  - name: adaptive_multi_agent
    path: ./adaptive-multi-agent
```

## License

MIT