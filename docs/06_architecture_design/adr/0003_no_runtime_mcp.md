# ADR-0003 产品运行时不引入 MCP

- **Status**: Proposed
- **日期**: 2026-09-14

## 上下文

调查需要工具调用。HolmesGPT 使用 MCP；本仓库 `.mcp.json` 已接 LangChain **文档** MCP。调研报告将「MCP 全量 CRUD」列为不纳入立项。官方 MCP 规范定义 Host / Client / Server 与 tools/resources/prompts，并要求工具调用前的用户同意、将工具描述视为不可信。

## 决策

**不**在 HuntAI 运行时引入 MCP Server 或把 kube/Prom/Loki 经 MCP 暴露。工具 = 进程内一等适配器 + 代码内静态白名单（BR-033/075/080）。

仓库文档 MCP **保持**给编码 Agent 使用；`.mcp.json` 与 `.cursor/mcp.json` 本阶段不改。

## 替代

- 每集群一个 MCP Server：增加传输、OAuth/confused deputy 面，且允许列表变成协议发现（Holmes #2228 类风险）。无 BR 要求。
- 让模型经 MCP 自发现工具：明确拒绝。

## 后果

新增工具必须改代码与 BR，不能只加 MCP 配置。与「普通任务查询不引入 MCP」的提示词一致。
