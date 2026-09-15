# ADR-0004 审批账本在控制面，不用 LangGraph HITL 执行写

- **Status**: Proposed
- **日期**: 2026-09-14

## 上下文

二期 BR-080–088：模型只生成 pending WriteAction + preview；有权人第二次 confirm；WriteIdentity 执行；离开 `running` 则 void。BR-040 单次调查墙钟 150s。官方 LangGraph `interrupt()` **无限等待** resume，且中断节点从**开头重跑**，interrupt 前副作用必须幂等。

用户批准的工作假设（D2）已写入 **BR-040 / BR-090**：150s 只计 LLM+工具；pending 期间 Investigation 保持 `running`、不消耗墙钟；超时 `[TBD-BIZ]`。界面 L1：「只读调查已结束，等待确认写操作」。

## 决策

1. `ApprovalRequest` 是 PostgreSQL 领域对象，是审批唯一真相。
2. 写执行只发生在控制面 `ConfirmWriteAction`，使用 WriteIdentity；**图内零写动词**。
3. **不采用** LangGraph HITL 作为账本或写路径。
4. 调查图可在提出 WriteAction 后继续 RCA；若仍有 pending，控制面延迟终态转换（主状态仍为 `running`）。
5. 超时或调查必须终态时：pending → `void`（BR-085），再按 citation 规则标 `completed`/`inconclusive`。

## 替代

- **interrupt 暂停图直到人批**：与「无限等待」和 150s 产品语义难对齐；resume 重跑使写操作极易重复。若未来仅暂停剩余 LLM、写仍在控制面，可再开 ADR，默认仍关闭。
- **调查一结束就 void pending**：严格字面 BR-085 + 150s，人批窗口过短，二期写修复几乎不可用。已用 D2 解释墙钟范围，而非改 BR 条文。

## 后果

PG-006 在 RCA 工具循环结束后仍可能显示 `running` + 人批卡片，直到审批结束。需在 Stage 8/交互补一句产品说明（开放问题）。超时数字未定前不得实现「假默认 1h」。
