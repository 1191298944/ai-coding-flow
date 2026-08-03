---
name: ruoyi-dev
description: 若依（RuoYi）前后端分离开发规范。当工作区为 IPD/VOC/SCM（ipd-studio、ipd-web、voc-studio、voc-web、scm-studio、scm-web 等若依 *-studio/*-web）或用户提到若依、RuoYi、Java17、Element Plus、vxe-table、Flowable 时使用。
---

# 若依开发规范（ruoyi-dev）

> **适用范围**：IPD / VOC / SCM 及明确为若依架构的 `*-studio` / `*-web`。  
> **禁止**用于 AIWA（`micro-app-platform*`）或 `airflow-job`。  
> 跨系统偏好（招呼、中文、最小改动）见 `personal.mdc`；本 Skill 管若依技术栈与 CRLF。

## 技术栈

- 后端：Java 17+、Spring Boot 3.3.5、Spring Security 6.x、MyBatis-Plus 3.5.5+、Flowable 7.0.1、MySQL 8.2+
- 前端：Vue 3 Composition API、Element Plus 2.7.x+、vxe-table 4.5.x、Vite 5/6、Node 18/20
- 响应：`R<T>`；OpenAPI 3；权限基于 Vue3 Composition API

## 行尾序列（CRLF，仅若依）

- 所有文本文件强制 CRLF（`\r\n`）
- 禁止对已是 CRLF 的内容再做 `\n` → `\r\n`（会变成 `\r\r\n`）
- 整文件落盘：先归一化为 LF，再**只转换一次**为 CRLF
- 优先局部替换（StrReplace），避免整文件重写引入换行异常

**自检**：写完后文件不应出现隔行空行；行数不应接近「逻辑行数 × 2」。

## 代码风格

- Java：阿里巴巴手册；类名大驼峰；DB 下划线
- Vue：`<script setup>` + Composition API
- 表名前缀按业务（`ipd_` / `voc_` / `scm_`），避开 Flowable 的 `act_` / `flw_`
- 基础字段：`create_time, update_time, create_by, update_by, del_flag`
- 建表：`CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci`

## 后端方法单一职责

- 一个方法只做一件事；多步骤拆 `private` 方法
- 禁止三层及以上 `if-else` 嵌套；用卫语句
- 禁止用注释分块代替拆方法
- Controller 只做路由 + 参数转换；业务在 Service

**自检**：用一句话说清这个方法在做什么——说不清 → 必须拆分。

## 代码模板

需要生成表单 / 列表查询 / Controller-Service-Mapper-Entity 时，读取并遵循同目录 [`references/templates.md`](references/templates.md)。

## 生效信号

- 仅在若依仓库生效；未把 CRLF/Java/Vue 规范带进 AIWA/Airflow
- 新代码符合 `R<T>`、卫语句拆分、CRLF
- 模板生成时引用了 `references/templates.md`
