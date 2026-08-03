---
name: airflow-job-dev
description: airflow-job（Apache Airflow 工作流）开发规范。当工作区为 airflow-job，或用户提到 Airflow、DAG、调度任务、airflow-job 时使用。非若依架构，Python 技术栈。
---

# Airflow Job 开发规范（airflow-job-dev）

> **仓库**：`airflow-job`（单仓）。  
> **禁止**套用若依 Java/Vue/CRLF 规范。  
> 与仓内 `.cursor/rules/general.mdc` 对齐；部署细节见仓库 `README.md`。

## 技术栈

- Apache Airflow 3.x（见 `requirements.txt`）
- Python DAG + Docker Compose 容器化部署
- 常用依赖：mysql-connector、pandas、requests、playwright、oss2、chinese_calendar

## 目录

| 路径 | 用途 |
|---|---|
| `dags/` | DAG 定义 |
| `config/` | 配置 |
| `utils/` | 公共工具 |
| `sql/` | SQL 脚本 |
| `scripts/` | 运维/辅助脚本 |
| `docker-compose.yml` / `Dockerfile` / `startup.sh` | 容器与启动 |

## 开发约束

1. 回复使用中文（与仓内 general 规则一致）
2. **不要自动 commit**；由用户手动执行（Git 操作走 `git-workflow` Skill）
3. DAG 须可重试、尽量幂等；避免把不可恢复的副作用写进默认重试路径
4. 密钥与连接信息走配置/环境变量，禁止写入仓库明文
5. 换行跟随仓库既有风格，不强制 CRLF
6. 改 DAG 前确认调度影响面（下游依赖、执行时段）

**自检**：本次改动是否把密钥写进代码？DAG 失败重试是否安全？Git 是否在未确认时自动 commit？

## 生效信号

- 对话加载本 Skill 而非 `ruoyi-dev`
- 改动落在 `dags/` / `utils/` / `sql/` 等合理目录
- 未自动 commit；未套用若依规范
