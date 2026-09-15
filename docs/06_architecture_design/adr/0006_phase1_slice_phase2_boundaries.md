# ADR-0006 一期可部署切片与二期模块边界

- **Status**: Proposed
- **日期**: 2026-09-14

## 上下文

Business Model §8 一期 20 项验收不因二期改写。§11 二期 8 项独立验收。交互要求一期 UI/API 不得出现二期入口。用户批准架构同时覆盖一期切片与二期边界。

## 决策

- 领域模块现在就划出 `auto_investigate` / `loki_tool` / `remediation`。
- 一期部署关闭这些能力：创建调查只有 `trigger=human`；无 Loki 适配器调用；无 WriteAction API（或返回 capability 关闭）。
- 二期靠配置打开（`write_remediation`、LokiCredential、调度扫描），不拆新服务。

## 替代

- 只设计一期：二期写入时可能推翻 Citation/身份/并发模型。
- 一期实现二期 API 仅前端隐藏：违反「隐藏不是安全边界」与交互 0.3。

## 后果

Stage 11 代码目录按一期模块创建；二期目录可存在但无路由。验收不得用 §11 充 §8。
