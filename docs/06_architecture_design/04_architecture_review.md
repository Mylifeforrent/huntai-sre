# 架构审查

- **Status**: Draft
- **日期**: 2026-09-14
- **角色**: Agent D（架构审查员，不重复设计）
- **审查对象**: `00`–`03`、`architecture_spec.md` / `architecture.md`、`frontend_backend_boundary_spec-v1.0.md`、`adr/0001`–`0007`

---

## 1. 追溯性

| 检查 | 结果 |
|---|---|
| 实体闭集是否等于 Business Model §3 | **通过**。未引入 Incident、ChatSession、上传作业。提示词「文件上传」已映射为 ToolCall 副本。 |
| 一期 20 项是否被二期模块吞没 | **通过**（ADR-0006）。审查未发现把 BR-070 写成一期必做 API。 |
| 工具白名单是否被「通用 Agent」扩大 | **通过**。无 MCP 发现、无 Elastic、无 Playwright。 |
| `[TBD-*]` 是否被假默认填满 | **通过**。token 日限额、`scale_max`、脱敏清单、150s 审批超时均未填数。 |
| 原型缺失是否被用来发明页面 | **通过**。页态来自交互摘要。 |

**瑕疵（不阻断 Draft）**

- A06（pending 时保持 `running`）已写入 **BR-090** 与交互 IX-FB-11。超时秒数仍 `[TBD-BIZ]`。
- 调查读 ACL 已拍板 **BR-055**。

---

## 2. 一致性

| 主题 | 控制面 / 01 | 接口 / 02 | 安全 / 03 | 裁定 |
|---|---|---|---|---|
| 调查终态 | completed / inconclusive | 同；HTTP 不承载终态 | — | 一致 |
| 人批 | 延迟终态 + ApprovalRequest | confirm 控制面命令；推荐校验失败 → rejected | WriteIdentity 无 fallback | 一致；与 BR-085 字面张力已记录 |
| 进度 | worker 投影 ToolCall | GET 权威，SSE 可选 | — | 与 ADR-0007 一致 |
| 开发扫描 | 禁止触发 | 403 | 行级 Finding 过滤在二期 | 一致 |
| 出域 | 图内经脱敏 | `llm_unavailable` 预留 | 无清单不出域 | 一致 |
| 通知 | 无外推 | 轮询 | `ALERT_WEBHOOK_URL` 不用 | 一致 |

**残留不一致（须 Lead 已消解或保持开放）**

- EX-15.3 允许「保持 pending 或转 rejected」；`02` 推荐 `rejected`。可接受，Stage 7 必须二选一。
- 无 LLM 时调查：已拍板 BR-053（只跑工具）。

---

## 3. 幂等、并发、失败恢复、审计、泄露

| 风险 | 是否覆盖 | 缺口 |
|---|---|---|
| webhook 重试 | 有（fingerprint upsert） | ingest 认证算法 TBD |
| 双击调查 / confirm | 有（幂等键假设 + 行锁） | 幂等键是否强制 Header 未冻结 |
| auto 并发 5 | 有（事务计数、不排队） | — |
| worker 崩溃 | 有（不得标 completed） | 多实例队列 TBD |
| 工具失败冒充证据 | 有（失败无 Citation） | — |
| 审计覆盖 MVP-18 | 有 | 审计载荷 schema 未写（Stage 7） |
| 日志/Sentry 泄露 | 有原则 | 未列字段级红名单（待脱敏清单） |
| 深链探测 | 有 TBD | 未拍板 403 vs 404 |
| 写执行重复 | 有（ApprovalRequest 幂等 + 图内禁止写） | — |

未发现「为了上 MCP/复杂工作流而设计」的模块。LangGraph 范围被 ADR-0002 收紧，审查认可。

---

## 4. 不可验证假设与未说明取舍

| 项 | 处理建议 |
|---|---|
| FastAPI / LangGraph / PostgreSQL | 已进 ADR Proposed，未冒充冻结。通过。 |
| 模块化单体 | ADR-0001 有替代。通过。 |
| 150s 不含人批等待 | **最大不可逆解释**。审查要求集成稿保持开放问题 #7，实现前要维护人书面点头（本次会话已口头批准 D2，仍建议 PRD 复述）。 |
| CORS、会话时长、CSRF | 仅 TBD，未假装已选。通过。 |
| 推荐栈来自 huntai-test / SuperBizAgent | 未把 Demo 能力写成已建设。通过。 |

---

## 5. 过度设计检查

| 候选复杂度 | 裁决 |
|---|---|
| MCP Host/Client/Server | 已拒绝 |
| LangGraph interrupt 当审批 | 已拒绝 |
| Kafka / 多租户 / 审批中心 | 未引入 |
| 对象存储 | 保持 TBD，一期库内截断 |
| SSE 必做 | 降为可选 |

无「技术展示」型组件进入推荐路径。

---

## 6. 对 Lead 的必改项 vs 建议

**必改（Draft 内应已满足）**

1. 集成稿必须同时出现 MCP 不采用与 HITL 不采用理由及官方链接 → 见 spec §8 与 `00`。
2. 不得把 `frontend_design_spec` 写成已完成 → README 已标明延后到 Stage 7 之后。

**建议（不阻断本 Draft）**

1. Stage 7 冻结：幂等键、confirm 失败态、深链探测。
2. pending 等待超时秒数仍 `[TBD-BIZ]`。
3. 接受 ADR 时再改 Status，走 `change_log.md`。
4. `frontend_design_spec-v1.0.md` 延后到 Stage 7 API 契约之后。

---

## 7. 审查结论

**可以保留为 Stage 6 后端架构 Draft。** 未发现无来源实体、未发现运行时 MCP、未发现用图执行 kubectl。BR-090 已消解「工具停了但仍 running」的产品语义；剩余风险是 pending 超时秒数与出域清单等 `[TBD-BIZ]` 的 fail-close。

**不批准进入实现（Stage 11）。** 缺数据模型、API 契约、前端设计规范、Gate 2。
