---
name: golang-fullstack-best-practices
description: Go 全栈最佳实践技能包,包含 89 条规则覆盖 9 个领域:并发安全、架构设计、错误处理、GORM、PostgreSQL 等
source: https://github.com/SAFOELLOH/GOLANG-BEST-PRACTICES-SKILL
license: MIT
---

# GOLANG FULLSTACK BEST PRACTICES SKILL

> 面向 AI Agent 的 Go + GORM + PostgreSQL 代码审查完整解决方案,基于权威来源构建

**v3.0.0**: "终极合并"版本。统一 89 条规则覆盖 9 个专业领域,从 Go 并发到 PostgreSQL 性能优化。

## 📋 概览

本元技能提供完整的 Go 全栈后端服务代码审查解决方案,结合通用 Go 语言最佳实践(v2.1.0)与专业的 GORM & PostgreSQL 专家知识(v1.0.0)。

## 📚 可用技能模块

| 领域 | 规则数 | 重点 |
| :--- | :---: | :--- |
| Concurrency Safety (并发安全) | 12 | Goroutines、channels、竞态条件 |
| Clean Architecture (整洁架构) | 9 | 分层架构、gRPC、依赖倒置 |
| Error Handling (统一错误处理) | 14 | Go 错误链 + Postgres 错误码/重试 |
| Design Patterns (设计模式) | 13 | 代码异味、重构、GoF 模式 |
| Idiomatic Go (地道 Go) | 7 | Go 约定、接口设计、指针 |
| GORM Query Patterns (GORM 查询) | 11 | ORM 查询、事务、context |
| PostgreSQL Syntax (PostgreSQL 语法) | 10 | 原生 SQL、PostgreSQL 特有功能 |
| Query Performance (查询性能) | 7 | N+1 检测、索引、EXPLAIN |
| Migration Safety (迁移安全) | 6 | 零停机 schema 变更、回滚 |

**优先级分布:**
* `20` CRITICAL — 数据丢失、SQL 注入、崩溃、生产环境 bug
* `30` HIGH — 可靠性、性能、架构
* `34` MEDIUM — 代码质量、惯用模式
* `5` ARCHITECTURE — 结构合规

## 🚀 快速开始

### 使用方式
**完整审查:**
`"Review this Go project for all language and database anti-patterns"`

**针对性审查:**
* `"Check for race conditions"` → 并发安全
* `"Audit architecture"` → 整洁架构
* `"Review GORM queries for N+1"` → GORM 模式/查询性能
* `"Debug this 23505 Postgres error"` → 错误处理
* `"Is this migration safe for production?"` → 迁移安全

## 🏗️ 结构
```text
golang-fullstack-best-practices/
├── SKILL.md                    # 元技能协调器 (v3.0.0)
├── metadata.json               # 完整规则清单
│
├── concurrency-safety/         # Go 特有
├── clean-architecture/         # 架构
├── design-patterns/            # 质量与异味
├── idiomatic-go/               # 约定
│
├── gorm-query-patterns/        # GORM ORM
├── postgresql-syntax/          # 原生 SQL
├── query-performance/          # 优化
├── migration-safety/           # DB Schema 变更
│
├── error-handling/             # 统一 Go & DB
├── shared/                     # 规则模板 & 分类
└── references/                 # 深入指南
```

## ✨ 特性
* **端到端覆盖:** 从 Go 语法到 PostgreSQL 索引优化
* **统一路由:** 使用元技能协调实现高精度目标定位
* **基于证据:** 规则引用 8+ 权威来源(Jon Bodner、Martin Fowler、Robert Martin、PostgreSQL 文档等)
* **生产就绪:** 聚焦零停机、安全操作和真实故障调试

## 👤 作者
**saifoelloh**
* GitHub: @saifoelloh
* 仓库: golang-best-practices-skill

---
*Built with ❤️ for bulletproof Go systems.*
