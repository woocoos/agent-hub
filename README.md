# Agent Hub 🤖

> 为 WooCoos 项目精心打造的 AI 开发资源中心,让 Go 开发更高效、更智能

## 📖 项目简介

Agent Hub 是一个面向 WooCoos 项目的 AI 辅助开发资源集合站。我们收集、整理并持续维护针对 Go 语言的最佳实践资源,包括但不限于:

- **Skills** - AI 开发技能包与工作流模板
- **Rules** - 代码规范、审查规则与质量检查清单
- **Prompts** - 高效的 AI 提示词模板与对话策略
- **Workflows** - 自动化开发流程与 CI/CD 集成方案
- **Tools** - 实用的开发工具链与辅助脚本

通过本项目的资源指引,开发者可以更高效地利用 AI 工具进行 Go 项目开发,提升代码质量,减少重复劳动。

## 🚀 快速开始

### 前置要求

- Go 1.24+
- 你偏好的 AI 开发工具(如 Cursor、GitHub Copilot、Qwen Code 等)
- Git

### 使用方式

1. **克隆本项目**
   ```bash
   git clone https://github.com/woocoos/agent-hub.git
   cd agent-hub
   ```

2. **浏览资源目录**
   ```
   agent-hub/
   ├── skills/          # AI 开发技能包
   ├── rules/           # 代码规范与审查规则
   ├── prompts/         # 提示词模板库
   ├── workflows/       # 自动化工作流
   └── tools/           # 实用工具集
   ```

3. **按需引入到你的项目**
   - 参考对应目录的 `README.md` 获取详细使用说明
   - 复制所需资源到你的项目配置目录
   - 根据你的项目特点进行定制化调整

## 📚 资源导航

### 🛠️ Skills (技能包)

| 技能名称 | 描述 | 适用场景 |
|---------|------|---------|
| [karpathy-guidelines](skills/andrej-karpathy/SKILL.md) | 减少 LLM 编码常见错误的行为准则 | 编写、审查或重构代码时避免过度复杂化 |
| [knockout-go(TODO)](skills/knockout-go/SKILL.md) | Knockout Go 最佳实践 | 将Knockout的特点梳理 |

> 💡 欢迎贡献你的 Go 开发技能包!

#### karpathy-guidelines

Karpathy 编码行为准则,源自 [Andrej Karpathy 的观察](https://x.com/karpathy/status/2015883857489522876),包含 4 条核心准则减少 LLM 编码常见错误。

**Qwen Code 安装方式:**

```bash
# 全局安装(推荐,所有项目可用)
npx skills add forrestchang/andrej-karpathy-skills@karpathy-guidelines -g -a qwen-code -y

# 项目级安装(仅当前项目)
npx skills add forrestchang/andrej-karpathy-skills@karpathy-guidelines -a qwen-code -y

# 或手动复制(无需 Skills CLI)
mkdir -p ~/.qwen/skills/karpathy-guidelines  # 全局
# 或
mkdir -p .qwen/skills/karpathy-guidelines   # 项目级
cp skills/andrej-karpathy/SKILL.md ~/.qwen/skills/karpathy-guidelines/
```

**核心准则:**
1. **编码前先思考** - 不要假设,不要隐藏困惑,明确权衡
2. **简洁优先** - 用最少的代码解决问题,不要写投机性代码
3. **手术式修改** - 只修改你必须修改的,只清理你自己的代码
4. **目标驱动执行** - 定义成功标准,循环直到验证

**适用场景:**
- 编写、审查或重构代码时避免过度复杂化
- 代码 Review 时检查是否符合行为准则

### 🌟 推荐外部 Skills

以下为社区维护的高质量 Skills,点击查看详情:

| Skill | 描述 | 适用场景 |
|-------|------|---------|
| [samber/cc-skills-golang](#sambercc-skills-golang) | Go 编码最佳实践,涵盖错误处理、并发、泛型等 | Go 后端开发 |
| [uber-go/guide](#uber-go-style-guide) | Uber 官方 Go 编码规范 | 代码规范参考 |
| [golangci-lint](#golangci-lint) | Go lint 聚合工具,100+ linter | CI/CD 代码检查 |

---

#### 📦 Skills 安装指南

**前置:** 安装 Skills CLI
```bash
npm install -g skills
# 或临时使用
npx skills
```

**Qwen Code 目录兼容性:**

| 目录 | Qwen Code 是否识别 |
|------|-------------------|
| `~/.qwen/skills/` | ✅ **个人级(推荐)** |
| `.qwen/skills/` | ✅ **项目级** |
| `.agents/skills/` | ❌ 不识别 |

**安装策略:**
- 使用 `-g` 参数安装到用户全局目录(`~/.qwen/skills/`)
- 指定 `@skill-name` 避免安装全部(42个)
- 使用 `npx skills find [query]` 搜索结果作为筛选依据(安装量 1K+ 为稳定版本)

**更新 Skills:**
```bash
npx skills check   # 检查更新
npx skills update  # 更新所有已安装 skills
```

---

#### samber/cc-skills-golang

Go 编码最佳实践,由 [samber](https://github.com/samber) 维护,涵盖错误处理、并发安全、泛型使用等核心领域。

**精选 Skills(按安装量排序):**

| Skill | 安装量 | 说明 | Qwen Code 安装命令 |
|-------|--------|------|-------------------|
| golang-code-style | 2.6K | 代码风格规范 | `npx skills add samber/cc-skills-golang@golang-code-style -g -a qwen-code -y` |
| golang-performance | 2.6K | 性能优化 | `npx skills add samber/cc-skills-golang@golang-performance -g -a qwen-code -y` |
| golang-error-handling | 2.5K | 错误处理 | `npx skills add samber/cc-skills-golang@golang-error-handling -g -a qwen-code -y` |
| golang-design-patterns | 2.4K | 设计模式 | `npx skills add samber/cc-skills-golang@golang-design-patterns -g -a qwen-code -y` |
| golang-testing | 2.4K | 测试指南 | `npx skills add samber/cc-skills-golang@golang-testing -g -a qwen-code -y` |
| golang-concurrency | 2.4K | 并发编程 | `npx skills add samber/cc-skills-golang@golang-concurrency -g -a qwen-code -y` |

**一键安装全部精选(Qwen Code):**
```bash
for skill in golang-code-style golang-error-handling golang-performance golang-design-patterns golang-testing golang-concurrency; do
  npx skills add samber/cc-skills-golang@$skill -g -a qwen-code -y
done
```

**适用场景:**
- Go 后端开发
- 错误处理规范
- 并发安全代码编写
- 性能优化

---

#### Uber Go Style Guide

Uber 内部 Go 编码规范的开源版本,被视为 Go 社区的权威指南。

**使用方式:**
```bash
# 克隆仓库
git clone https://github.com/uber-go/guide.git

# 在 AI 工具中引用
# Qwen Code: 将 style.md 内容添加到 .qwen/rules/go-style.md
# Cursor: 添加到 .cursorrules
```

**核心内容:**
- Go 惯用法和约定
- 接口设计原则
- 错误处理规范

---

#### golangci-lint

Go 最流行的 lint 聚合工具,内置 100+ linter,可配合 AI 审查使用。

**安装方式:**
```bash
# 安装
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# 在项目中使用
golangci-lint run

# 集成到 CI/CD (GitHub Actions)
# 见: https://golangci-lint.run/usage/install/#github-actions
```

**与 AI 配合:**
- 在 AI 生成代码后运行 lint 检查
- 将 lint 输出作为 AI 审查的反馈
- 配置 `.golangci.yml` 定义项目规范

---

### 📋 Rules (规则集)

| 规则名称 | 描述 | 严格程度 |
|---------|------|---------|
| *(待添加)* | *(待添加)* | *(待添加)* |

> 📝 所有规则均经过 WooCoos 项目实践验证

### 💬 Prompts (提示词)

| 提示词类别 | 描述 | 示例数量 |
|-----------|------|---------|
| *(待添加)* | *(待添加)* | *(待添加)* |

> 🎯 精心调优的提示词可以显著提升 AI 输出质量

### 🔄 Workflows (工作流)

| 工作流名称 | 描述 | 集成难度 |
|-----------|------|---------|
| *(待添加)* | *(待添加)* | *(待添加)* |

> ⚡ 自动化你的重复性开发任务

### 🔧 Tools (工具集)

| 工具名称 | 描述 | 安装方式 |
|---------|------|---------|
| [find-skills](https://github.com/vercel-labs/skills/tree/main/skills/find-skills) | Skills 搜索和安装工具,发现社区高质量 Skills | `npx skills add https://github.com/vercel-labs/skills --skill find-skills -g -a qwen-code -y` |
| [golangci-lint](#golangci-lint) | Go lint 聚合工具,100+ linter | 见下方安装说明 |

> 🛠️ 提升开发效率的实用工具

## 🎯 项目特色

- ✅ **实战导向** - 所有资源均来自 WooCoos 项目的实际开发经验
- ✅ **持续更新** - 跟随 Go 语言生态与 AI 技术发展不断演进
- ✅ **开箱即用** - 提供详细的配置说明与使用示例
- ✅ **模块化设计** - 按需引入,灵活组合
- ✅ **社区驱动** - 欢迎贡献,共同成长

## 🤝 贡献指南

我们非常欢迎社区贡献!无论是新增资源、优化建议还是问题反馈。

### 贡献步骤

1. Fork 本仓库
2. 创建你的特性分支 (`git checkout -b feature/amazing-skill`)
3. 提交你的改动 (`git commit -m 'feat: add amazing skill'`)
4. 推送到分支 (`git push origin feature/amazing-skill`)
5. 提交 Pull Request

### 贡献规范

- 遵循 [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- 为新增资源编写清晰的文档
- 确保资源在 WooCoos 项目中经过验证
- 使用中文注释和文档(方便国内开发者)

## 📄 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件

<p align="center">Made with ❤️ for Go developers</p>
