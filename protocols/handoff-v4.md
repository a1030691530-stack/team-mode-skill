# HANDOFF v4.0 协议

## 概述

HANDOFF v4.0 是结构化 JSON 交接协议。**所有 Agent 统一输出 v3.0 纯文本格式**，主窗口在完整路径中自动转换为 v4.0 JSON 存储。

## Agent 输出格式（统一 v3.0）

所有 Agent 只输出 v3.0 格式，主窗口负责存储时转换为 v4.0 JSON：

## 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| schema_version | string | 是 | 协议版本，固定"4.0" |
| from | string | 是 | 发送方Agent名 |
| to | string | 是 | 接收方Agent名 |
| status | string | 是 | complete/failed/blocked |
| timestamp | string | 是 | ISO8601时间戳 |
| summary | string | 是 | 1-2句话总结 |
| key_decisions | array | 是 | 关键决策列表 |
| known_risks | array | 是 | 已知风险 |
| artifacts | array | 否 | 产出文件列表 |
| next_step_expectation | string | 是 | 对下一步的期望。**必须在此字段末尾追加派发规则摘要**，格式：`[DISPATCH: IF complete→{下一Agent}; IF failed→熔断器; IF blocked→终止; IF no_handoff→拒绝]`。例如："按方案实现代码，确保build通过。[DISPATCH: IF complete→Code-Reviewer; IF failed→熔断器; IF blocked→终止; IF no_handoff→拒绝]"。此设计确保派发规则随上下文流动，不被长上下文稀释。**注意**：`{下一Agent}` 名称必须与 SKILL.md Agent 身份定义表（第 50-67 行）中完全一致（含大小写和连字符），如 `Code-Reviewer`、`Test-Engineer`、`Security-Reviewer`。 |
| validation | object | 否 | 质量校验规则 |
| metadata | object | 否 | 元数据（字数、文件数、置信度） |

## v3.0 兼容

v3.0 格式仍可识别，主窗口自动转换为 v4.0 结构：

```
=== HANDOFF BEGIN ===
FROM: architect
TO: executor
STATUS: complete

SUMMARY:
- 技术方案设计完成

KEY DECISIONS:
1. 使用MVVM模式

KNOWN RISKS:
- 速率限制未知

NEXT STEP EXPECTATION:
- 按方案实现代码
=== HANDOFF END ===
```

## 自动处理规则

1. 主窗口提取 STATUS 决定是否继续
2. 主窗口将整个 JSON 写入 `.omc/state/handoffs/{from}_{to}_{timestamp}.json`
3. 下一 Agent 从 `.omc/state/handoffs/` 读取前置条件
4. 主窗口记录 metadata 到 execution_log
5. 下一 Agent 执行前，主窗口校验 next_step_expectation 中的 [DISPATCH:...] 摘要是否与当前 HANDOFF status 匹配，不匹配则触发对应分支。
