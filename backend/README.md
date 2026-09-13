# backend — HuntAI SRE 后端

后端源码与后端工程配置。禁止放设计文档或前端代码。

当前阶段（Stage 2）本目录只保留本文件。实现代码于 Stage 11 落地，目录结构按 Stage 6 `docs/06_architecture_design/architecture_spec.md` 创建；该文件不存在时禁止预建模块目录。

依赖与虚拟环境使用 uv（`pyproject.toml` + `uv.lock`）。配置从仓库根 `.env` 加载（禁止 `backend/.env`）。键名见根 `.env.example`。
