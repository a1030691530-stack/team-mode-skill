# 检查点自动写入 v4.0

## 自动写入规则

每个 Agent 完成后，主窗口必须：

1. 写入 `.omc/state/checkpoints/{step_name}_{YYYYMMDD_HHMMSS}.json`:
```json
{
  "step": "executor",
  "pipeline": "standard",
  "status": "done",
  "timestamp": "2026-04-21T10:15:00Z",
  "handoff_summary": "代码实现完成，build通过",
  "files_modified": ["src/a.cs", "src/b.cs"],
  "next_step": "code-reviewer",
  "elapsed_seconds": 300
}
```

2. 追加到 `.omc/logs/execution_log.jsonl`:
```json
{"ts":"2026-04-21T10:15:00Z","event":"step_complete","step":"executor","status":"done","pipeline":"standard","elapsed":300}
```

3. 更新 `.omc/state/pipeline_state.json` 中的步骤状态

## 恢复协议

- 新会话启动时检查 `.omc/state/pipeline_state.json`
- 如果 `status == "in-progress"` 或 `"interrupted"`：
  - 读取最近检查点
  - 询问用户: "检测到中断的流程，是否从 {step} 恢复?"
  - 用户确认 → 从下一步继续；用户拒绝 → 重置状态

## 错误日志写入契约

`.omc/logs/error_log.jsonl` 由**主窗口独占写入**，Agent 不直接写入该文件。

### 触发场景

| 触发条件 | 日志条目格式 |
|----------|-------------|
| Agent HANDOFF STATUS=failed | `{"ts":"ISO8601","event":"agent_failed","agent":"X","pipeline":"Y","step":"Z","error_summary":"..."}` |
| Agent 超时 | `{"ts":"ISO8601","event":"agent_timeout","agent":"X","step":"Y","timeout_seconds":N}` |
| 熔断器打开 | `{"ts":"ISO8601","event":"circuit_open","agent":"X","tier":"Y","failure_count":N}` |
| half-open 重试失败 | `{"ts":"ISO8601","event":"half_open_fail","agent":"X","retry_count":N}` |
| 构建失败 | `{"ts":"ISO8601","event":"build_failed","pipeline":"Y","exit_code":N}` |
| 验证 FAIL | `{"ts":"ISO8601","event":"validation_failed","pipeline":"Y","checklist_failures":N}` |

### Agent 职责

Agent 通过 HANDOFF 的 SUMMARY 和 KNOWN RISKS 报告错误，由主窗口提取并写入 error_log。Agent 不得直接操作 error_log.jsonl 文件。
