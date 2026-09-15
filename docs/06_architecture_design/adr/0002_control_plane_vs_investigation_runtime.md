# ADR-0002 控制面与调查运行时分离

- **Status**: Proposed
- **日期**: 2026-09-14

## 上下文

根 README 将 FastAPI + LangGraph 列为工作假设，并写明「控制面与 Agent 边界 TBD」「禁止把模型当作权限 / 审批 / 重试 / 幂等的决策者」。Business Model 要求 Citation 服务端铸造、配额与 ACL 写死。内部前身 SuperBizAgent 用 LangGraph 做 Plan-Execute，但无登录、无 cited。

## 决策

- **控制面**（非图）：身份、TeamBinding、ingest、Alert、配额与闸、ScanRun、配置、AuditEvent、Citation 铸造、Investigation 主状态、二期 ApprovalRequest 与写执行。
- **调查运行时**（LangGraph 图）：Skill 加载、只读工具循环、成本闸计数、Claim 草案。`thread_id` = Investigation id。
- 图禁止：HTTP 鉴权、扣配额、颁发 Citation id、调用写动词、把 RelatedLink 当证据。

## 替代

- **全部进 LangGraph**：模型或图条件可能成为权限决策者，违反 README 禁令与 BR-031。
- **不用 LangGraph、自研循环**：可行，但与已有栈假设和前身规划环重复；若维护人否定 LangGraph，可在不改控制面的前提下替换 runtime 适配器。

## 后果

调查失败不得把控制面标 `completed`。生产若启用 checkpointer，必须耐久存储（见官方 persistence），且 checkpointer ≠ 审批账本。
