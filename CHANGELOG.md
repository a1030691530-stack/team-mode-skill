# Team-Mode Changelog

## [4.0.0] - 2026-04-21

### Added
- `.omc/` 运行时状态管理系统 (state/handoffs, state/checkpoints, state/circuit_breaker.json, state/pipeline_state.json, state/task_board.json, logs/, plans/, research/, notepad.md, project-memory.json)
- HANDOFF v4.0 类型化协议 (protocols/handoff-v4.md) — JSON Schema结构化交接
- 熔断器强制执行规则 (protocols/circuit-breaker.md) — open/half-open/closed三状态机
- 检查点自动写入 (protocols/checkpoints.md) — 每步完成后自动保存快照
- 输入输出护栏 Guardrails (protocols/guardrails.md) — SOFT_FAIL/HARD_FAIL分级
- Orchestrator Agent (agents/orchestrator.md) — 全局任务协调员
- 版本历史 (CHANGELOG.md)

### Changed
- 16个Agent定义文件增加 YAML Frontmatter (role/goal/constraints/competencies/discipline)
- Critic 增加终止保证协议 (硬性2轮限制+强制仲裁)
- Explore 增加按需服务调用规范
- Document-Specialist 增加前置咨询规范

### Source
- 学习10个GitHub高星开源项目: everything-claude-code(82K★), superpowers(89K★), CrewAI(43K★), AG2/AutoGen(32K★), LangGraph(126K★), ClawTeam, MetaGPT, OpenAI Agents SDK(19K★), Google ADK(16.8K★), oh-my-claudecode

## [3.0.0] - 2026-04-17

### Added
- 初始版本: 14 Agent角色, 4条路径(Fast/Standard/Full/Deploy)
- HANDOFF v3.0 混合模式协议
- 3维动态路由判定矩阵
- 4层容错 (重试/熔断/降级/超时)
- 上下文预算全员约束
- 客观Checklist质量门
- 8条安全回环If-Then规则
