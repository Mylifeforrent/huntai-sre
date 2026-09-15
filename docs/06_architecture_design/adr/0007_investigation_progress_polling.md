# ADR-0007 调查进度一期用 GET 轮询

- **Status**: Proposed
- **日期**: 2026-09-14

## 上下文

PG-006 需展示 running 时间线。README 提及 SSE 适合调查流式输出。BR-044 禁止出站 IM/邮件/呼叫，故无推送产品。

## 决策

一期以 **GET Investigation** 为进度权威（可短轮询）。SSE 为可选增强，不能替代存库状态。不引入 WebSocket 总线。

## 替代

- 仅 SSE：断线后必须以 GET 对齐，SSE 不能当权威。
- 阻塞式同步 HTTP 跑满 150s：与「关页后服务端继续」（EX-06.5）冲突。

## 后果

前端轮询间隔 `[TBD-FE]`。worker 必须把 ToolCall 尽快落库，轮询才有内容。
