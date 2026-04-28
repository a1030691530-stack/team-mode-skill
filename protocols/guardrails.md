# 输入输出护栏 (Guardrails) v4.0

## 输入护栏 (每个Agent启动前检查)

| 检查项 | 规则 | 失败动作 |
|--------|------|---------|
| 前序HANDOFF状态 | STATUS == "complete" | 拒绝执行，上报 |
| 必要上下文完整 | Analyst验收标准存在(Architect) | 回退到缺失步骤 |
| 路径自动升级 | 涉及认证/支付/安全但路径为"快速" | 升级为完整路径 |
| 变更规模检查 | 预估变更超出当前路径预算 | 升级路径或熔断 |

## 输出护栏 (每个Agent完成后检查)

| 检查项 | 规则 | 失败动作 |
|--------|------|---------|
| HANDOFF完整性 | 必须包含HANDOFF块 | 标记INCOMPLETE，要求补充 |
| 代码build验证 | Executor产出必须build通过 | SOFT_FAIL:修正一次 |
| 审计等级分类 | Code-Reviewer必须有严重等级 | HARD_FAIL:打回重写 |
| 测试通过率 | Test-Engineer必须有通过率数据 | HARD_FAIL:补充测试 |
| 安全回环 | 发现硬编码密钥/SQL注入/XSS | 标记CRITICAL，停止执行 |

## 失败分级

- **SOFT_FAIL**: 格式问题、缺少标记 → 自动修正一次
- **HARD_FAIL**: 内容错误、安全检查不通过 → 打回上一步修复

## 错误分类与恢复

| 错误类型 | 分类 | 恢复路径 |
|---------|------|---------|
| 编译错误 | BUILD | 回Executor修复 |
| 安全漏洞 | SECURITY-CRITICAL | 阻塞流程，上报 |
| 需求未覆盖 | COVERAGE-GAP | 回Executor补充 |
| 测试失败 | TEST-FAIL | 回Test-Engineer |
| 超时 | TIMEOUT | 触发降级流程 |
