---
name: aiwa-dev
description: AIWA（雨文 AI 微应用平台）开发规范。当工作区为 micro-app-platform、micro-app-platform-web，或用户提到 AIWA、miniclaw、mini-claw、微应用平台、apps 微应用时使用。非若依架构。
---

# AIWA 开发规范（aiwa-dev）

> **仓库**：前端主站 `micro-app-platform-web`；后端 `micro-app-platform`。  
> **禁止**套用若依 Java/Vue/Element Plus/CRLF 规范。  
> 细节以仓库内 `CLAUDE.md` / `AGENTS.md` 为准；本 Skill 只固化边界与关键约束。

## 仓库与职责

| 路径 | 职责 |
|---|---|
| `micro-app-platform-web` | 产品前端主站（React 18 + TypeScript + Vite + Tailwind） |
| `micro-app-platform` | 后端与平台核心（Python 3.10+ / FastAPI / SQLAlchemy Async） |
| `micro-app-platform/framework/frontend` | 后端仓内**独立前端**（miniclaw / 平台内嵌 UI），与 web 仓分离，勿混改 |
| `micro-app-platform/framework/` | **受保护**核心；改动需 `ALLOW_FRAMEWORK_CHANGES=true` 且更新 `docs/FRAMEWORK_INTERNALS.md` |
| `micro-app-platform/apps/` | 独立微应用（可单独建设前后端） |
| `micro-app-platform/agents/` | LangGraph Agent（含 `mini-claw` 基座） |

## 技术栈要点

- 后端：FastAPI、SQLAlchemy 2 Async、MySQL、Redis；AI 引擎含 Dify / LangGraph / 多 LLM
- 前端主站：React + TS + Zustand + TanStack Query；勿引入若依 Vue/Element Plus 模式
- 数据库：`yuwen_ai_microapp_platform`（简称 aiwa / AIWA / 微应用 / micro-app）
- 换行：跟随仓库既有风格，**不强制 CRLF**

## 开发边界

1. 改前端产品功能 → 优先 `micro-app-platform-web`
2. 改 miniclaw 内嵌 UI → `framework/frontend` 或对应 `apps/*/frontend`，确认后再动
3. 新业务能力优先落在 `apps/<app-key>/`，避免污染 `framework/`
4. Agent 改动落在 `agents/<agent-key>/`，须导出 `create_graph(config)`
5. 数据隔离：MySQL `app_id`；Redis `app:{app_key}:...`；向量库按 app_key 分 collection

**自检**：本次改动是否误进 `framework/`？是否把若依 CRLF/Java 规范带进来了？是 → 停下纠正。

## 常用入口

- 后端：`framework/backend/app/main.py`；加载器 `app_loader.py` / `agent_loader.py`
- 前端主站：`micro-app-platform-web/src/`
- 本地细节与命令：读仓库 `CLAUDE.md`

## 生效信号

- 对话加载本 Skill 而非 `ruoyi-dev`
- framework 改动有意识；apps/agents 边界清晰
- SQL 使用 `yuwen_ai_microapp_platform` 前缀
