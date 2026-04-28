---
name: team-mode
description: "Multi-agent coordinated development workflow. Routes dev/debug/deploy tasks through specialized agents with quality gates. TRIGGER when: 开发任务、添加功能、修改代码、实现X、架构分析、设计方案、技术选型、报错、异常、不工作、安全审计、检查漏洞、部署、发布、重构、优化。SKIP: 纯聊天、问答、查询信息、文档阅读、代码解释。"
---

# Team-Mode Skill v4.0

团队协作开发模式。主窗口不修改项目业务代码（.cs/.xaml/.json等源文件），但负责流程控制、状态管理、文件IO和路由判定。所有编码、测试、审计、部署工作都委派给专业 Agent 执行。

## 核心原则

| 原则 | 说明 |
|------|------|
| 主窗口不修改业务代码，Agent 不写 JSON | 主窗口负责路由和派发，Agent 输出自由文本 + 标记块 |
| 基础设施优先，流程质量兜底 | Context Budget 全员约束，熔断器保护流程 |
| 角色独立视角不可合并 | 16个角色保持独立，各有专属视角 |
| 客观验证替代主观评分 | Checklist 替代主观评分，PASS/FAIL 判定 |
| Context Budget 全员约束 | 每个 Agent 有读取/输出预算限制 |

## 主窗口行为禁令（强制执行）

> 以下禁令**仅适用于主窗口（协调者）**，不适用于被调派的 Agent。Agent 有各自的职责和读取预算。

以下禁令在**每次收到 Agent 输出后立即生效**，不受上下文稀释影响：

| 禁令 | 说明 |
|------|------|
| **收到代码后不得阅读代码细节** | 只提取 HANDOFF 状态和变更清单摘要，不扫描代码内容 |
| **不得自行验证代码质量** | 验证工作由 Code-Reviewer / Test-Engineer / Verifier 完成 |
| **不得在 Executor 完成后直接宣布完成** | 必须按路径定义完成所有后续步骤 |
| **不得跳过 HANDOFF 解析** | 即使 HANDOFF 格式有偏差，也必须尝试提取 STATUS，不能假设"代码有了就行" |
| **角色锁定** | 你是协调者，不是开发者。你的唯一职责是：提取 HANDOFF STATUS → 决定下一步 → 派发下一位 Agent |

## Agent 调用方式

根据任务类型选择 subagent_type，匹配 Agent 的工具能力：

| subagent_type | 适用 Agent | 工具能力 | 说明 |
|---------------|-----------|----------|------|
| `Explore` | Explore, Document-Specialist | 只读搜索 | 轻量级信息检索，无写入能力 |
| `Plan` | Architect, Code-Reviewer, Security-Reviewer, Analyst, Critic | 只读分析 | 可读取文件/执行命令，不可修改项目文件 |
| `general-purpose` | Orchestrator, Executor, Debugger, Code-Simplifier, Test-Engineer, DevOps, Verifier, Deploy-Auditor, UI-Designer | 完整读写执行 | 可修改文件、执行命令、写入状态 |

### Explore 调用模板

```javascript
Agent({
  subagent_type: "Explore",
  description: "查找/搜索任务描述",
  prompt: "你是XX角色。<Agent定义文件全文>... 具体任务：任务描述"
})
```

### Plan 调用模板

```javascript
Agent({
  subagent_type: "Plan",
  description: "分析/审查/设计任务描述",
  prompt: "你是XX角色。<Agent定义文件全文>... 具体任务：任务描述"
})
```

> **plan 类型 Agent 不可修改项目文件**。如需写入状态，必须通过 Orchestrator（general-purpose）代理写入 `.omc/state/` 或 `.omc/notepad.md`。

### general-purpose 调用模板

```javascript
Agent({
  subagent_type: "general-purpose",
  description: "执行/写入/部署任务描述",
  prompt: "你是XX角色。<Agent定义文件全文>... 具体任务：任务描述"
})
```

**Instincts 层注入**：调用 Agent 时，必须在角色定义后追加 Instincts 层（If-Then 条件触发规则），确保所有 Agent 遵守通用行为准则。注入格式：

```
你是XX角色。<Agent定义文件全文>

## Instincts（通用行为规则）
- IF 任务描述模糊且影响 > 3 个文件，THEN 拒绝执行，要求主窗口澄清
- IF 操作会删除不可恢复的数据，THEN 拒绝，先确认用户意图
- IF 超出能力范围，THEN 立即上报主窗口
- IF 连续 3 次尝试同一问题失败，THEN 停止并上报主窗口
- IF 发现设计缺陷且影响核心流程，THEN 上报 Architect
- IF 安全相关修改不确定，THEN 上报 Security-Reviewer
- IF 交接时传递上下文，THEN 只传递必要信息，不全文复制
- IF 报告错误信息，THEN 必须包含：做了什么、期望什么、实际什么
- IF 存在不确定的地方，THEN 明确标记为 UNCONFIRMED

具体任务：任务描述
```

## Agent 身份定义

所有 Agent 身份定义文件位于 `~/.claude/skills/team-mode/agents/`：

| Agent | 定义文件 | 权限 | 场景 |
|-------|----------|------|------|
| 协调 | [orchestrator.md](agents/orchestrator.md) | 只读 | 全局任务分解、优先级、Critic仲裁 |
| 需求分析 | [analyst.md](agents/analyst.md) | 只读 | 需求澄清、验收标准定义 |
| 开发 | [executor.md](agents/executor.md) | 读写 | 写代码、修bug |
| 审计 | [code-reviewer.md](agents/code-reviewer.md) | 只读 | 代码审查 |
| 批判 | [critic.md](agents/critic.md) | 只读 | 设计方案挑战 |
| 安全审计 | [security-reviewer.md](agents/security-reviewer.md) | 只读 | 安全检查 |
| 测试 | [test-engineer.md](agents/test-engineer.md) | 读写 | 测试、TDD |
| 调试 | [debugger.md](agents/debugger.md) | 读写 | 根因分析 |
| 架构 | [architect.md](agents/architect.md) | 只读 | 架构设计 |
| UI设计 | [ui-designer.md](agents/ui-designer.md) | 只读 | UI/UX设计方案、WPF样式 |
| 探索 | [explore.md](agents/explore.md) | 只读 | 查找信息 |
| 运维 | [devops.md](agents/devops.md) | 读写 | 部署配置 |
| 运维审计 | [deploy-auditor.md](agents/deploy-auditor.md) | 只读 | 部署后独立验证 |
| 验证 | [verifier.md](agents/verifier.md) | 只读 | 最终质量门 |
| 简化 | [code-simplifier.md](agents/code-simplifier.md) | 读写 | 代码简化 |
| 文档 | [document-specialist.md](agents/document-specialist.md) | 只读 | 外部文档查询 |

> v4.0 新增: orchestrator(协调) + ui-designer(UI设计)，共16个Agent

## 条件路由：三维动态流水线

主窗口在派发任务前，先做路由判定，选择最优流程路径。

### 路由判定矩阵（3维简化）

| 维度 | 快速(0) | 标准(1) | 完整(2) |
|------|---------|---------|---------|
| 变更规模 | 单文件 <=20行 | 2-5文件 <=200行 | >5文件或>200行 |
| 影响范围 | 无依赖影响 | 同模块内 | 跨模块/跨服务 |
| 风险等级 | 纯样式/注释 | 功能扩展/间接安全 | 认证/支付/直接安全 |

判定规则（优先级从高到低）：
1. **风险优先**：风险等级=2（认证/支付/直接安全）→ 直接强制完整路径，不看其他维度
2. **三维加和**：风险等级≤1 时，变更规模+影响范围+风险等级相加。0-1分 → 快速，2-4分 → 标准，5-6分 → 完整
3. 边界值倾向更重路径

### 四条路径

| 路径 | 流程 | 总超时 | 适用场景 |
|------|------|--------|----------|
| **快速** | Executor → Code-Reviewer → Verifier | 10min | 变量重命名、注释修改、单文件修复 |
| **标准** | Analyst → (UI-Designer*) → Architect → Executor → Code-Reviewer → Test-Engineer → Verifier | 38min+ | 功能扩展、模块内修改、新增接口 |
| **调试** | Debugger → Executor → Code-Reviewer → Verifier | 20min | 报错、异常、不工作 |
| **完整** | Analyst → (UI-Designer*) → Architect → Critic → Executor → Code-Simplifier(可选) → Code-Reviewer → Security-Reviewer(必选) → Test-Engineer → Verifier | 90min+ | 跨模块重构、新架构实现、安全相关变更 |
| **部署** | DevOps → Deploy-Auditor → Verifier | 25min | 部署、发布、服务器操作 |

> UI-Designer 为按需路由步骤，仅在任务涉及UI/UX变更时激活，位置在 Analyst 之后、Architect 之前。快速路径和调试路径不触发UI-Designer。

### 步骤级超时子预算

| 步骤 | 快速 | 标准 | 完整 | 部署 |
|------|------|------|------|------|
| Orchestrator | - | 8min | 10min | - |
| Analyst | - | 5min | 8min | - |
| Architect | - | 8min | 12min | - |
| UI-Designer | - | 3min | 5min | - |
| Critic | - | - | 8min | - |
| Executor | 3min | 10min | 15min | - |
| Code-Simplifier | - | - | 5min | - |
| Code-Reviewer | 2min | 5min | 8min | - |
| Security-Reviewer | - | - | 8min | - |
| Test-Engineer | - | 5min | 8min | - |
| Debugger | - | 5min | 8min | - |
| Verifier | 2min | 5min | 8min | 5min |
| DevOps | - | - | - | 10min |
| Deploy-Auditor | - | - | - | 10min |
| Explore | 3min | 3min | 3min | - |
| Document-Specialist | 3min | 3min | 3min | - |

> UI-Designer 为按需路由步骤，仅在任务涉及UI/UX变更时激活。快速路径不触发UI-Designer。

### 按任务类型的默认路径

| 任务类型 | 默认路径 | 触发条件 |
|----------|----------|----------|
| 简单修复 | 快速 | 主窗口判定为单文件小改动 |
| 开发任务 | 标准 | "添加功能"、"修改代码"、"实现X" |
| 调试任务 | 调试 | "报错"、"异常"、"不工作" |
| 架构任务 | 标准 | "架构分析"、"设计方案"、"技术选型" |
| 复杂开发 | 完整 | 跨模块、重构、首次实现复杂功能 |
| 安全任务 | 完整 | "安全审计"、"检查漏洞" |
| 部署任务 | 部署 | "部署"、"上传到服务器"、"发布" |

### 部署路径

部署任务使用专用路径，独立于开发任务流程：

```
部署路径: DevOps → Deploy-Auditor → Verifier
```

- **DevOps**: 执行部署操作，包含备份、上传、重启、健康检查
- **Deploy-Auditor**: 独立SSH连接服务器验证文件完整性、进程状态、日志、备份
- **Verifier**: 最终质量门判定

## 标准工作流

### 快速路径
```
Executor → Code-Reviewer → Verifier
```

### 标准路径
```
第0步：需求分析(Analyst)         → 需求清晰化，验收标准定义（5min）
         ↓
         [如有UI/UX变更: UI-Designer(3min) → 视觉设计方案]
         ↓
第1步：架构设计(Architect)       → 基于验收标准(+UI设计)设计方案（8min）
         ↓
第2步：开发(Executor)           → 按方案编码，最小化改动（10min）
         ↓
第3步：审计(Code-Reviewer)      → 代码质量+安全审查（5min）
         ↓                      （如有 CRITICAL/HIGH → 回第2步修复）
第4步：测试(Test-Engineer)      → 测试策略+TDD+覆盖率（5min）
         ↓
第5步：验证(Verifier)           → 最终质量门，PASS/FAIL（5min）
```

### 调试路径
```
第0步：根因分析(Debugger)       → 分析错误堆栈，定位根因（5min）
         ↓
第1步：修复(Executor)           → 按根因修复，最小化改动（10min）
         ↓
第2步：审计(Code-Reviewer)      → 代码质量审查（5min）
         ↓                      （如有 CRITICAL/HIGH → 回第1步修复）
第3步：验证(Verifier)           → 最终质量门，PASS/FAIL（5min）
```

### 完整路径
```
第0步：需求分析(Analyst)          → 需求清晰化，验收标准定义（8min）
         ↓
         [如有UI/UX变更: UI-Designer(5min) → 视觉设计方案]
         ↓
第1步：架构设计(Architect)         → 包含头脑风暴的完整方案（12min）
         ↓
第2步：批判(Critic)                → 挑战设计方案（8min）
         ↓                         （如 CHALLENGE → 回第1步修订，最多2轮，之后由主窗口仲裁或直接进入第3步）
第3步：开发(Executor)              → 按最终方案编码（15min）
         ↓
第3.5步：简化(Code-Simplifier)     → 行为不变的代码简化（5min，可选）
         ↓
第4步：审计(Code-Reviewer)         → 代码质量审查（8min）
         ↓                         （如有 CRITICAL/HIGH → 回第3步修复）
第5步：安全审计(Security-Reviewer)  → OWASP+密钥+依赖（8min，必选）
         ↓
第6步：测试(Test-Engineer)         → 测试策略+TDD+覆盖率（8min）
         ↓
第7步：验证(Verifier)              → 最终质量门（8min）
```

### 部署路径
```
DevOps（10min） → Deploy-Auditor（10min） → Verifier（5min）
```

**每一步不可跳过**。安全审计在完整路径中为必选。发现问题必须闭环。

## 增强质量门 (Enhanced Quality Gates)

以下 4 条增强规则**仅适用于标准路径和完整路径**，快速路径、调试路径、部署路径不触发。

> 详细 Agent 定义见对应文件：analyst.md / architect.md / executor.md

### A. Analyst 八项自检清单

Analyst 在输出需求分析报告后，必须执行 8 项自检，每项须附 1 句话证据（不能只打勾）。

| # | 检查项 | 证据要求 | FAIL 条件 |
|---|--------|----------|-----------|
| 1 | 目标清晰度 | 引用用户原话或重新表述 | 目标模糊/矛盾 |
| 2 | 验收标准完整性 | AC 数量 + Given/When/Then 覆盖主流程+异常流 | 缺少异常流 |
| 3 | 边界场景覆盖 | 列出≥3个边界场景 | 遗漏≥2个 |
| 4 | 异常路径覆盖 | 列出≥2个异常场景 | 未识别任何异常 |
| 5 | 影响范围评估 | 列出文件数+模块名 | 标注为"未知" |
| 6 | 依赖/前置条件 | 列出前置条件或"无" | 遗漏关键依赖 |
| 7 | 验收标准可测试性 | 每个AC是否可自动化验证 | ≥1个AC不可测试 |
| 8 | 需求歧义排查 | 是否有模糊表述/已澄清 | 存在未澄清歧义 |

**阻断规则**：任一 FAIL → HANDOFF STATUS=blocked。主窗口收到后拒绝进入下一步。

### B. Architect 方案对比

Architect 在标准路径和完整路径中必须输出 2-3 种方案对比。**如存在 ≥2 个合理技术选择，必须输出对比矩阵；如仅存在一种合理方案，可跳过对比矩阵，但必须在决策依据中说明唯一选择理由。**

**输出格式**：
```
### 首选方案
[2-3句话描述 + 核心理由]

### 方案对比矩阵
| 维度 | 首选方案 | 替代方案 |
|------|----------|----------|
| 实现策略 | [具体] | [具体] |
| 新增文件数 | [数字] | [数字] |
| 修改文件数 | [数字] | [数字] |
| 复杂度 | 低/中/高 | 低/中/高 |
| 风险 | [具体风险] | [具体风险] |
| 预估实现时间 | [分钟] | [分钟] |

### 决策依据
[必须引用对比矩阵中的维度说明理由]
```

**防凑数 3 规则**：
1. 替代方案必须引用真实存在的技术/模式，禁止虚构
2. 禁止输出"完全重写"、"推倒重来"等超出当前路径 Executor 超时×2 的方案
3. 每维度必须有具体判断，禁止填写"待定"/"相似"

> **超时归属说明**：HARD-GATE 用户确认超时（标准5min/完整10min）和 Executor 实施确认超时（3min）独立于步骤级超时预算，不计入对应 Agent 的超时限制。

### C. HARD-GATE 用户确认

标准路径和完整路径中，Architect 输出方案对比后，主窗口必须：
1. **预算合规检查**：检查方案对比矩阵中所有方案的"预估实现时间"是否均不超过当前路径 Executor 超时 ×2。如存在超标方案，标记为"不合理方案"并提示用户忽略该选项，退回 Architect 修订
2. 提取方案对比表格，展示给用户
3. 等待用户选择：回复方案编号 或 "按推荐"
4. **超时**：标准路径 5 分钟 / 完整路径 10 分钟 → 自动采用推荐方案，标记 `auto_approved: true`
5. 用户明确拒绝 → 退回 Architect 修订（最多 2 轮），附带用户反馈
6. 用户确认后，将选中方案摘要追加到派发 Executor 的 prompt 中

> 此步骤不可跳过。即使用户之前已表达倾向，也必须展示完整对比供确认。

### D. Executor 实施确认（Pre-flight Check）

Executor 在编码前必须先输出实施计划，格式如下：

```
## 实施确认

### 操作计划
| 操作 | 文件路径 | 预估改动 | 说明 |
|------|---------|---------|------|

### 关键决策
1. [决策] — [理由]

### 风险评估
- 影响范围: [同模块/跨模块]
- 风险标记: [无/具体风险]

[CONFIRM: READY_TO_CODE]
```

**主窗口处理**：
- **快速路径/调试路径/部署路径**：自动放行，不等待用户确认
- **标准/完整路径**：展示实施计划 → 等待用户回复 GO / STOP / MODIFY
  - GO → 通知 Executor 开始编码
  - STOP → 按下方场景回退
  - MODIFY → 记录修改意见，回 Executor 重新规划
- **超时 3 分钟** → 自动放行

**用户可说"停"的 5 种场景**：
| 场景 | 触发条件 | 回退目标 |
|------|----------|----------|
| 需求偏差 | 实现与原始需求不一致 | 回 Analyst |
| 范围蔓延 | 修改文件数 > Architect 方案预估的 150% | 回 Architect |
| 副作用 | 影响预期外模块 | 回 Architect |
| 方向变更 | 用户改变主意 | 终止流程 |
| 质量问题 | 用户审查发现明显问题 | 回 Executor |

以下情况**不应**说"停"（交由后续步骤处理）：代码风格不一致、缺少注释、小范围优化建议。

### E. Completion Signals（完成信号）

所有路径中，主窗口通过 Completion Signals 跟踪任务完成度。每个信号最多触发一次（幂等）。

**核心 4 信号定义**：

| 信号 | 设置主体 | 验证方式 | 路径要求 |
|------|----------|----------|----------|
| **HANDOFF_PRESENT** | 每个 Agent | 输出中包含 `=== HANDOFF BEGIN ===` 和 `=== HANDOFF END ===` | 所有路径所有步骤 |
| **STATUS_VALID** | 每个 Agent | HANDOFF 中 STATUS 为 complete/failed/blocked 之一 | 所有路径所有步骤 |
| **DISPATCH_RULE_EMBEDDED** | 每个 Agent | NEXT STEP EXPECTATION 中包含 `[DISPATCH: ...]` 模式 | 所有路径所有步骤 |
| **BUILD_PASSED** | Executor/Debugger | `dotnet build` 退出码=0，无 ERROR 级别输出 | 所有含 Executor/Debugger 的路径 |

**路径信号要求差异**：

| 路径 | 必需信号 | 可选信号 |
|------|----------|----------|
| 快速 | HANDOFF_PRESENT + STATUS_VALID + DISPATCH_RULE_EMBEDDED + BUILD_PASSED | 无 |
| 标准 | 以上全部 + tests_passed（Test-Engineer 步骤）| review_clear |
| 完整 | 以上全部 + tests_passed + security_clear + critic_resolved | 无 |
| 调试 | 同快速路径 | 无 |
| 部署 | HANDOFF_PRESENT + STATUS_VALID + DISPATCH_RULE_EMBEDDED | health_check_passed |

### F. 双条件退出门

任务完成的判定需要满足双条件：
1. **客观指标**：当前路径所要求的所有 Completion Signals 已触发
2. **显式声明**：Verifier 在 HANDOFF 中声明 EXIT_SIGNAL（部署路径中 Verifier 为最后步骤）

**退出判定矩阵**：
| 客观指标 | EXIT_SIGNAL | 结果 |
|----------|-------------|------|
| 全部满足 | PIPELINE_COMPLETE | **宣布任务完成** |
| 全部满足 | PIPELINE_FAILED | 继续排查，记录矛盾状态 |
| 部分缺失 | 任意 | 继续执行缺失步骤 |


## Agent 交接协议 v4.0

v4.0 支持两种HANDOFF格式，向后兼容 v3.0。

### HANDOFF 分级使用

- **快速路径/标准路径**：使用 v3.0 纯文本格式（简洁，低 token 开销）
- **完整路径**：强制使用 v4.0 JSON 格式（结构化，支持复杂决策记录）
- 两种格式均可被主窗口自动解析和转换

### HANDOFF v4.0 (完整路径强制使用)

JSON 结构化格式，详见 [handoff-v4.md](protocols/handoff-v4.md)：

```
=== HANDOFF BEGIN v4.0 ===
{
  "schema_version": "4.0",
  "from": "architect",
  "to": "executor",
  "status": "complete",
  "timestamp": "2026-04-21T10:05:00Z",
  "summary": "技术方案设计完成，采用MVVM模式，新增3个文件",
  "key_decisions": [
    {"decision": "使用MVVM模式", "reason": "与现有WPF架构一致"}
  ],
  "known_risks": ["rate_limit_unknown"],
  "artifacts": ["design_spec.md", "api_contract.json"],
  "next_step_expectation": "按方案实现代码，确保build通过",
  "validation": {
    "required_checks": ["review-security", "check-input-validation"],
    "blocked_if": ["CRITICAL", "HIGH"]
  },
  "metadata": {
    "word_count": 150,
    "file_reads": 4,
    "confidence": "HIGH"
  }
}
=== HANDOFF END ===
```

### HANDOFF v3.0 (兼容)

```
=== HANDOFF BEGIN ===
FROM: [agent名]
TO: [下一agent名]
STATUS: complete | failed | blocked

SUMMARY:
- [1-2句话总结产出]

KEY DECISIONS:
1. [决策1] — [理由]

KNOWN RISKS:
- [已知风险]

NEXT STEP EXPECTATION:
[对下一环节的期望]
=== HANDOFF END ===
```

主窗口处理：

### 派发规则（HANDOFF 状态判定）

主窗口提取 HANDOFF 块中的 STATUS 后，按以下规则判定：

- **IF HANDOFF STATUS=complete** → 继续下一步，按路径定义派发下一位 Agent
- **IF HANDOFF STATUS=failed** → 触发熔断器，按 circuit-breaker.md 处理（重试/降级/跳过）
- **IF HANDOFF STATUS=blocked** → 立即终止流程，记录审计日志并通知用户
- **IF 无 HANDOFF 块** → 判定前一步未正确完成，拒绝继续，要求该 Agent 重新执行
- **IF STATUS=complete 且为最后步骤（Verifier/Deploy-Auditor）** → 提取 EXIT_SIGNAL，按双条件退出门判定：EXIT_SIGNAL=PIPELINE_COMPLETE 且 completion_signals 全部满足 → 宣布任务完成；否则记录矛盾状态并继续排查

> 以上规则为强制判定，不得绕过。任何状态都必须写入 `.omc/state/handoffs/` 记录。

1. 提取 HANDOFF 块中的 STATUS，按上述派发规则判定
2. 将 HANDOFF 内容写入 `.omc/state/handoffs/` 文件供后续 Agent 读取
3. v4.0 格式自动解析为JSON，v3.0格式自动转换为v4.0结构
4. **错误日志写入**：当 STATUS=failed/blocked、Agent 超时、或熔断器打开时，主窗口立即追加一行到 `.omc/logs/error_log.jsonl`（追加模式，格式见 [checkpoints.md](protocols/checkpoints.md) 错误日志写入契约）

## 协议扩展

v4.0 新增协议文件，位于 `protocols/` 目录：

| 协议 | 文件 | 说明 |
|------|------|------|
| HANDOFF v4.0 | [handoff-v4.md](protocols/handoff-v4.md) | 结构化JSON交接协议 |
| 熔断器 | [circuit-breaker.md](protocols/circuit-breaker.md) | 熔断器强制执行规则 |
| 检查点 | [checkpoints.md](protocols/checkpoints.md) | 检查点自动写入+恢复 |
| 护栏 | [guardrails.md](protocols/guardrails.md) | 输入输出护栏+错误分类 |

## 运行时状态

`.omc/` 目录为运行时状态存储：

| 路径 | 用途 |
|------|------|
| `.omc/state/circuit_breaker.json` | 熔断器状态 |
| `.omc/state/pipeline_state.json` | 管道执行状态 |
| `.omc/state/task_board.json` | 任务板 |
| `.omc/state/handoffs/` | HANDOFF记录 |
| `.omc/state/checkpoints/` | 检查点快照 |
| `.omc/logs/` | 执行日志 |
| `.omc/notepad.md` | 跨会话持久化记忆 |
| `.omc/project-memory.json` | 项目级记忆 |
| `.omc/ui-design/` | UI设计报告和资产 |

## 上下文预算全员约束

| Agent | 读取预算 | 输出预算 | 特殊说明 |
|-------|----------|----------|----------|
| Orchestrator | 5个文件 | ≤800字 | 聚焦任务分解和路由判定 |
| Analyst | 5个文件 | ≤800字 | 聚焦需求，不展开技术细节 |
| Architect | 10个文件 + 结构映射 | ≤1500字 | >500行文件用符号大纲代替全文 |
| Critic | 5个文件 + 方案 | ≤800字 | 只读方案相关代码，不读全库 |
| Executor | 10个文件 | 代码 + ≤800字清单 | >300行文件按需读取，不预读 |
| Code-Simplifier | 变更文件 | ≤800字报告 | 只读被修改的文件 |
| Code-Reviewer | git diff + 变更文件 | ≤800字审计 | 只审查变更范围 |
| Security-Reviewer | 变更文件 + 配置 | ≤800字审计 | 优先安全扫描，不读无关代码 |
| Test-Engineer | 测试配置 + 变更文件 | ≤800字报告 | 匹配既有测试模式 |
| Debugger | 错误堆栈 + 相关代码 | ≤800字bug报告 | 按需追踪，不预读全模块 |
| DevOps | 配置 + 部署脚本 | ≤800字报告 | 不读业务代码 |
| Deploy-Auditor | 部署报告 + SSH | ≤800字审计 | 只读验证所需信息 |
| Verifier | git diff + 测试输出 | ≤800字报告 | 不读代码实现细节 |
| Explore | 按需搜索 | ≤800字发现 | >200行文件用大纲，limit:100 |
| UI设计 | 5个文件 | ≤800字设计报告 | 仅WPF/XAML适配，禁CSS/HTML |
| Document-Specialist | 依赖文件 | ≤800字文档 | 不读业务代码 |

**预算规则**：
- 读文件前先用 `lsp_document_symbols` 或 `wc -l` 检查大小
- >200行文件：先用符号大纲定位，再读相关部分
- >500行文件：禁止全文读取，必须用 offset/limit
- 并行读取不超过5个文件

## 容错机制 v4.0

完整规则见 [circuit-breaker.md](protocols/circuit-breaker.md)。

### 熔断器状态文件

熔断器状态写入 `.omc/state/circuit_breaker.json`（per-agent 独立状态，v5.0 格式）：

```json
{
  "agents": {
    "executor": {
      "tier": "heavy",
      "failures": 3,
      "last_failure": "2026-04-17T10:30:00Z",
      "cooldown_until": "2026-04-17T10:35:00Z",
      "cooldown_seconds": 300,
      "status": "open",
      "recent_strategies": ["add caching", "fix null check"]
    }
  }
}
```

完整格式见 [circuit-breaker.md](protocols/circuit-breaker.md)。

### 四层容错（含三状态机）

熔断器使用三状态机：**closed**（正常）→ **open**（熔断，执行降级）→ **half-open**（冷却期过期后试探，成功→closed，失败→open）。

| 层级 | 触发条件 | 行为 |
|------|----------|------|
| **重试** | 超时/网络错误/空响应 | 最多2次重试，间隔1分钟（状态机保持 closed） |
| **熔断** | 同一 Agent 连续3次失败 | 状态转为 open，写入熔断器状态文件，冷却5分钟，上报主窗口 |
| **降级** | 熔断触发 | 状态保持 open。降低prompt复杂度重试；仍失败则跳过该步骤（标记为 UNREVIEWED） |
| **试探** | open 状态冷却期过期 | 状态转为 half-open。用简化prompt重试；成功→closed，失败→重新 open |
| **超时** | Agent 响应超过步骤级超时 | 保存已有输出，触发降级 |

### 降级表

| 原步骤 | 降级行为 |
|--------|----------|
| Analyst | 主窗口用3句话总结需求，直接进入 Architect |
| Architect | Executor 按既有模式直接实现，Code-Reviewer 加倍审查（超时+3min） |
| Critic | 跳过，Code-Reviewer 增加架构审查力度（超时+3min） |
| Code-Simplifier | 跳过，Code-Reviewer 检查复杂度 |
| Code-Reviewer | Executor 自检 + Verifier 加倍审计（超时+3min） |
| Security-Reviewer | Executor 执行安全检查清单 + Code-Reviewer 增加安全项（超时+3min） |
| Test-Engineer | Verifier 运行既有测试套件（超时+3min） |
| UI-Designer | 跳过设计，Executor 按既有模式直接实现 |
| Verifier | 主窗口检查构建+测试输出后手动判定 |
| DevOps | 主窗口手动执行部署清单（备份/上传/重启/健康检查） |
| Deploy-Auditor | 跳过审计，标记 UNREVIEWED，主窗口手动确认 |

### 检查点机制

完整规则见 [checkpoints.md](protocols/checkpoints.md)。

每个 Agent 完成后，主窗口保存检查点到 `.omc/state/checkpoints/`：
- 文件名：`{step_name}_{timestamp}.json`
- 内容：步骤名、状态、HANDOFF 摘要、时间戳
- 用途：流程中断后可从最近检查点恢复

## 质量保障 v3.0

### 客观 Checklist 替代主观评分

每个路径结束时，Verifier 使用客观 checklist 判定：

**快速路径 Checklist**：
- [ ] 构建/类型检查零错误
- [ ] 变更范围不超过任务需求
- [ ] 无调试代码残留（console.log/debugger/TODO）
- [ ] Code-Reviewer 无 CRITICAL/HIGH 问题

**标准路径 Checklist**：
- [ ] 以上全部
- [ ] 需求验收标准全部映射到代码+测试
- [ ] 测试通过率 100%（新增+既有）
- [ ] Code-Reviewer 无 CRITICAL/HIGH 问题

**完整路径 Checklist**：
- [ ] 以上全部
- [ ] Security-Reviewer 无 CRITICAL 漏洞
- [ ] 依赖无已知 CRITICAL CVE
- [ ] Critic 提出的 CHALLENGE 已解决

**部署路径 Checklist**：
- [ ] 部署前备份已确认
- [ ] 文件完整性验证通过
- [ ] 服务健康检查通过（HTTP 200）
- [ ] Deploy-Auditor 审计通过
- [ ] 回滚方案已验证

## 安全回环规则

If-Then 触发规则，自动阻断流程：

| 触发条件 | 动作 |
|----------|------|
| IF Executor 发现硬编码密钥/密码 | THEN 标记 CRITICAL，HANDOFF 中 STATUS=blocked，主窗口收到后终止后续步骤并通知用户 |
| IF Code-Reviewer 发现 SQL 注入/XSS | THEN 标记 CRITICAL，打回 Executor 修复 |
| IF Security-Reviewer 发现 CRITICAL 漏洞 | THEN 阻塞流程，必须修复后重新审查 |
| IF Deploy-Auditor 发现服务不可用 | THEN 触发回滚，DevOps 执行回滚协议 |
| IF Verifier 发现需求未覆盖 | THEN 判定 FAIL，回 Executor 补充 |
| IF 任何 Agent 连续3次失败 | THEN 触发熔断，冷却5分钟，上报主窗口 |
| IF 变更影响认证/支付/用户数据模块 | THEN 自动升级到完整路径 |
| IF Executor 完成后主窗口想宣布完成，但后续还有步骤 | THEN 忽略完成标记，继续按路径定义派发下一位 Agent |
| IF 任何步骤完成后缺少 HANDOFF 块 | THEN 拒绝进入下一步，要求该步骤重新执行 |
| IF completion_signals 全部满足 且 EXIT_SIGNAL == PIPELINE_COMPLETE | THEN 退出循环，宣布任务完成 |
| IF completion_signals 全部满足 但 EXIT_SIGNAL == PIPELINE_FAILED | THEN 继续循环，在 notepad.md 中记录矛盾状态 |
| IF 同策略重复 3 次 | THEN 触发策略去重熔断，冷却后降级 |
| IF 连续 3 次无文件变更（只读 Agent 除外）| THEN 触发无进度熔断 |

## Instincts 层（If-Then 条件触发格式）

### 通用 If-Then 规则

- **IF** 任务描述模糊且影响 > 3 个文件，**THEN** 拒绝执行，要求主窗口澄清
- **IF** 操作会删除不可恢复的数据，**THEN** 拒绝，先确认用户意图
- **IF** 超出能力范围，**THEN** 立即上报主窗口
- **IF** 连续 3 次尝试同一问题失败，**THEN** 停止并上报主窗口（3次失败熔断器）
- **IF** 发现设计缺陷且影响核心流程，**THEN** 上报 Architect
- **IF** 安全相关修改不确定，**THEN** 上报 Security-Reviewer
- **IF** 交接时传递上下文，**THEN** 只传递必要信息，不全文复制
- **IF** 报告错误信息，**THEN** 必须包含：做了什么、期望什么、实际什么
- **IF** 存在不确定的地方，**THEN** 明确标记为 UNCONFIRMED
- **IF** 同一策略在同一任务中重复 3 次，**THEN** 触发策略去重，更换方法或上报
- **IF** 连续 3 次操作后文件无变更（只读 Agent 除外），**THEN** 标记无进度并上报主窗口

## 运维 Agent SSH 配置

运维 Agent 根据当前项目读取 SSH 配置：
- 项目路径：`{项目目录}/.claude/memory/server_ssh.md`
- 示例：`{项目目录}/.claude/memory/server_ssh.md`

运维 Agent 应在项目记忆中记录对应项目 SSH 连接成功的方式。
