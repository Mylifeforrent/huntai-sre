# 前后端边界规范 v1.0

- **Status**: Draft（Stage 6；**非**完整 API 契约）
- **日期**: 2026-09-14
- **权威**: Business Model；[`interaction_design_summary.md`](../04_interaction_design/interaction_design_summary.md)；[`02_api_workflow_and_review.md`](./02_api_workflow_and_review.md)
- **非目标**: OpenAPI、字段表、视觉值、前端目录结构（`frontend_design_spec-v1.0.md` 本次不写）。

---

## 1. 谁拥有什么

| 对象 | 拥有者 | 前端可以做 | 前端不可以做 |
|---|---|---|---|
| 登录会话 | 后端 + IdP | 跳转 OIDC / 提交 break-glass | 匿名预览业务数据 |
| Alert 可见性 | 后端 TeamBinding | 展示返回的行 | 靠改 query 看 unscoped |
| Investigation.status | 后端控制面 | 按枚举渲染徽章 | 本地把 inconclusive 画成根因 |
| Citation | 后端铸造 | 打开已成功只读 ToolCall 副本 | 把 RelatedLink / 粘贴文本当证据 |
| 配额与闸 | 后端 | 展示剩余次数与禁用原因 | 前端自己允许第 31 次调查 |
| 扫描触发 | 后端角色 | SRE 点「分析该集群」 | 开发直调（须 403） |
| 写修复（二期） | 后端 ApprovalRequest | 展示 preview、第二次 confirm | 模型或前端直接调写接口 |
| 集群凭证 | 后端 / secret | 显示已配置/未配置 | 上传 kubeconfig、回显密钥 |

渲染在浏览器（Vite SPA 为 README 假设）；服务端只有 FastAPI。前端不持有集群 SA。

---

## 2. 协作原则

1. **深链**: 参数必须是 `cluster_id` + fingerprint。path `[TBD-FE]`。未登录先登录再回跳。未入库固定文案「尚未收到 AM webhook」。不可见 → No Permission，不展示 alertname。
2. **创建调查**: POST 成功返回 id 后跳转 PG-006（IX-FB-01）。确认层文案含日额与只读。连点靠禁用 + 幂等键。无 LLM 时仍创建，横幅见 IX-FB-12。
3. **进度**: 一期 GET 轮询（ADR-0007）。关页不 cancel。〔二期〕人批等待时 L1 用 IX-FB-11，不得显示「正在分析」。
4. **终态**: `completed` 与 `inconclusive` 在徽章、主标题、主内容三处可区分；零 citation 不得像已找到根因。
5. **错误**: 业务失败用页内 Success-Failure；系统 Error 无栈。HTTP 与业务码映射见 `02` §1.3。
6. **权限 UI**: 一期开发隐藏扫描与设置。直链仍须后端 403。永禁能力不渲染置灰入口。
7. **LLM 关闭**: 横幅叠在列表/详情，不隐藏告警。
8. **二期**: 人批只活在所属 Investigation。继续查不得离开 `alert_id`。写结果分区且不可点成 Citation。

---

## 3. 状态同步

前端页态机（Default / Loading / Empty / No Result / Error / No Permission / Editing / Submitting / Success-Failure）必须能由后端资源状态 + HTTP 推出，禁止前端私自发明第四种调查终态。

保留期满删除后，旧调查 URL 走 Empty/Error，不伪造历史 RCA。

---

## 4. 配置与密钥

- 配置来自仓库根 `.env` 注入，前端构建期不得写入真实密钥。
- RelatedLink 只存 URL；host 允许列表 `[TBD-BIZ]`。
- 一期设置页无二期字段（env / write_remediation / scale_max / Loki / WriteIdentity）。

---

## 5. 待 Stage 7 / 前端规范补齐

- 精确 URL 与 JSON schema。
- CSRF、cookie 属性、轮询间隔。
- `frontend_design_spec-v1.0.md`：**等 Stage 7 API 契约之后**再写（维护人 2026-09-14）。
