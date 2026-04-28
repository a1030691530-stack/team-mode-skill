# 熔断器强制执行规则 v5.0

## 状态文件

`.omc/state/circuit_breaker.json`（per-agent 独立状态）：
```json
{
  "agents": {
    "executor": {
      "tier": "heavy",
      "failures": 0,
      "last_failure": null,
      "cooldown_until": null,
      "cooldown_seconds": 300,
      "status": "closed",
      "recent_strategies": []
    }
  }
}
```

每个 Agent 独立维护一个三状态机。

## 三状态机

| 状态 | 含义 | 行为 |
|------|------|------|
| closed | 正常 | 正常执行 Agent |
| open | 熔断 | 执行降级行为，等待冷却过期 |
| half-open | 试探 | 冷却过期后用简化 prompt 重试，成功→closed，失败→open（冷却时间翻倍） |

## 分级冷却表

不同 Agent 组别使用不同的冷却参数：

| 组别 | 成员 | 初始冷却 | 重试后 | 上限 |
|------|------|---------|--------|------|
| **Heavy** | Executor, Debugger, DevOps | 5min | 15min | 30min |
| **Medium** | Architect, Analyst, Critic, UI-Designer | 3min | 8min | 15min |
| **Light** | Code-Reviewer, Security-Reviewer, Test-Engineer, Verifier, Deploy-Auditor | 2min | 5min | 10min |
| **Infrastructure** | Explore, Document-Specialist, Orchestrator, Code-Simplifier | 1min | 3min | 5min |

冷却时间按 `min(初始冷却 × 2^(重试次数-1), 上限)` 计算。

## 策略去重

每个 Agent 在 `circuit_breaker.json` 中维护 `recent_strategies` 数组（最多 5 条，FIFO 淘汰）。

**判定规则**：
1. 每次派发 Agent 前，主窗口提取本次任务描述的关键词（3-5 个词），归一化（小写、去标点）
2. 与 `recent_strategies` 中最近 3 条进行子串匹配
3. 如匹配到相同策略，判定为**策略重复**
4. 同一策略重复 3 次 → 触发熔断，状态转为 open

**去重写入**：Agent 成功完成步骤后，主窗口将策略关键词追加到 `recent_strategies`。

## 强制执行规则

主窗口在每步开始前必须检查：

1. 读取 `.omc/state/circuit_breaker.json` 中对应 Agent 的状态
2. 如果 `status == "open"` 且 `cooldown_until` 未过期：执行降级行为
3. 如果 `status == "open"` 且已过期：转为 `half-open`，用简化 prompt 尝试执行
4. Agent 失败时：failures++，failures >= 3 → `status="open"`，`cooldown_until=now+cooldown_seconds`
5. 每次派发前检查策略去重，重复 3 次 → 熔断

## 超时强制执行

- 每步启动时记录 `started_at` 和 `timeout_at` 到 pipeline_state.json
- 超时后：保存已有输出 → 触发降级 → 更新 circuit_breaker

## 错误分类

| 错误类型 | 分类 | 自动响应 |
|---------|------|---------|
| 网络超时 | TRANSIENT | 重试（最多2次） |
| Agent 空响应 | MODEL_ERROR | 降低 prompt 复杂度重试 |
| 编译/构建错误 | BUILD | 回 Executor 修复 |
| 安全漏洞 | SECURITY-CRITICAL | 阻塞流程，上报 |
| 需求未覆盖 | COVERAGE-GAP | 回 Executor 补充 |
| 测试失败 | TEST-FAIL | 回 Test-Engineer |
| 超时 | TIMEOUT | 保存已有输出，触发降级 |
| 策略重复 | DUPLICATE_STRATEGY | 触发熔断，冷却后降级 |

## 错误日志写入契约

`.omc/logs/error_log.jsonl` 由**主窗口独占写入**，Agent 不直接写入。

**触发场景**：

| 触发条件 | 日志条目格式 |
|----------|-------------|
| Agent HANDOFF STATUS=failed | `{"ts":"ISO8601","event":"agent_failed","agent":"X","pipeline":"Y","step":"Z","error_summary":"..."}` |
| Agent 超时 | `{"ts":"ISO8601","event":"agent_timeout","agent":"X","step":"Y","timeout_seconds":N}` |
| 熔断器打开 | `{"ts":"ISO8601","event":"circuit_open","agent":"X","tier":"Y","failure_count":N}` |
| half-open 重试失败 | `{"ts":"ISO8601","event":"half_open_fail","agent":"X","retry_count":N}` |
| 构建失败 | `{"ts":"ISO8601","event":"build_failed","pipeline":"Y","exit_code":N}` |
| 验证 FAIL | `{"ts":"ISO8601","event":"validation_failed","pipeline":"Y","checklist_failures":N}` |

**Agent 职责**：通过 HANDOFF 的 SUMMARY 和 KNOWN RISKS 报告错误，由主窗口提取并写入 error_log。
