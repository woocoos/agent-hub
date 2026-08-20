---
name: knockout-go
description: 基于 woocoo knockout 应用架构的 Go 开发实践。用于开发基于 knockout 架构的服务、处理多租户、Ent 缓存、Casbin 权限控制或构建 woocoo 应用。
source: https://github.com/woocoos/knockout-go
license: MIT
---

# Knockout-Go 开发准则

基于 woocoo 框架的 knockout-go 应用 SDK 开发准则。提供多租户、缓存、权限控制和服务集成的最佳实践。

## 项目概述

**knockout-go** 是基于 [woocoo](https://github.com/tsingsun/woocoo) 框架的 Go 应用 SDK 和工具包。提供:

- 应用初始化和组件构建 (`pkg/koapp`)
- 多租户和身份上下文管理 (`pkg/identity`)
- 基于 Casbin 的权限控制 (`pkg/authz`)
- Ent 缓存驱动扩展 (`ent/clientx`)
- 速率限制中间件 (`pkg/middleware/ratelimiter`)

## 技术栈

| 组件 | 技术 |
|------|------|
| 框架 | woocoo |
| ORM | Ent + entcache |
| GraphQL | gqlgen |
| HTTP | Gin |
| 权限控制 | Casbin |
| 可观测性 | OpenTelemetry |
| 缓存 | Redis / 本地 (TinyLFU) |
| ID 生成 | Snowflake |

## 代码规范

### 注释规范

- 代码注释放在上方，不要放在行尾
- 不使用装饰线（`// ===`、`// ---`）作为分节符
- 注释使用中文时, 必须使用半角标点
- 涉及服务地址时,需要使用回环地址localhost,如配置文件中定义服务地址使用`localhost:8080`,而不是`:8080`

## 多租户模式

**规则:** 始终通过 `context.Context` 传递租户上下文,禁止使用全局状态。

**关键上下文函数:**

```go
// 从 context 获取租户 ID
tenantID, err := identity.TenantIDFromContext(ctx)

// 从 context 获取用户 ID
userID, err := identity.UserIDFromContext(ctx)

// 从 context 获取域 ID
domainID, err := identity.DomainIDFromContext(ctx)

// 在 context 中设置租户 ID
ctx := identity.WithTenantID(parentCtx, tenantID)
```

## Ent 配置

**规则:** 使用 `koapp.BuildEntComponents()` 从配置文件构建 Ent 数据库驱动,自动处理 OTel 追踪和缓存层。

### BuildEntComponents 核心功能

`BuildEntComponents(cnf *conf.AppConfiguration) map[string]dialect.Driver` 从配置文件加载所有 store 配置并构建驱动:

- **返回值:** `map[string]dialect.Driver`, 键为 schema 名称, 值为对应的驱动实例
- **OTel 追踪:** 如果配置了 `otel` 节点, 自动为数据库驱动注册 otelsql 包装器
- **Ent 缓存:** 如果配置了 `entcache` 节点, 自动构建带缓存的驱动
- **缓存隔离:** 支持 `entcache.isolate: true` 实现按 schema 的缓存命名空间隔离

### 配置示例

```yaml
# app.yaml
otel:
  endpoint: "localhost:4317"
  service: "my-service"

store:
  # Schema 名称: portal
  portal:
    driverName: mysql
    dsn: "user:pass@tcp(localhost:3306)/portal?parseTime=true&loc=Local"
  
  # Schema 名称: msg
  msg:
    driverName: mysql
    dsn: "user:pass@tcp(localhost:3306)/msg?parseTime=true&loc=Local"
  
  # Schema 配置映射 (可选)
  schemaConfig:
    portal:
      Org: db_portal
      User: db_portal
    msg: db_msg  # 字符串表示所有表使用相同 schema

# Ent 缓存配置
entcache:
  isolate: true       # 按 schema 隔离缓存命名空间
  ttl: 5m             # 缓存 TTL
  gcInterval: 10m     # GC 间隔
  hashQueryTTL: 10m   # 查询哈希 TTL
```

### 使用示例

```go
// 应用初始化
func main() {
    app := koapp.New()
    cnf := app.AppConfiguration()
    
    // 构建所有 Ent 驱动
    drivers := koapp.BuildEntComponents(cnf)
    // drivers["portal"] -> portal schema 的驱动 (带 OTel + 缓存)
    // drivers["msg"] -> msg schema 的驱动 (带 OTel + 缓存)
    
    // 使用驱动创建 Ent Client
    portalClient := ent.NewClient(ent.Driver(drivers["portal"]))
    
    app.Run()
}
```

## Casbin 权限控制

**规则:** 使用 ARN (Amazon Resource Name) 格式进行资源标识。

## 应用初始化模式

**规则:** 使用 `koapp.New()` 进行应用引导,自动初始化组件。
**规则:** 初始化工作避免放在`Start`方法中。

```go
// 正确: 使用 koapp.New()
func main() {
    app := koapp.New()
    // 组件从配置自动初始化
    app.Run()
}

// 错误: 手动创建组件
func main() {
    db := sql.Open(...)      // 不要这样做
    cache := redis.New(...)  // 不要这样做
    // 缺失: otel, snowflake, entcache 等
}
```

**配置结构:**

```yaml
# app.yaml
snowflake:
  node: 1        # ID 生成的唯一节点 ID

otel:
  endpoint: "localhost:4317"
  service: "my-service"

cache:
  redis:
    driverName: "my-cache"
    addr: "localhost:6379"

store:
  portal:
    driverName: mysql
    dsn: "user:pass@tcp(localhost:3306)/portal"
  schemaConfig:
    portal:
      Org: db_portal
      User: db_portal
```

## API 客户端模式

**规则:** 可使用 SDK 访问 Knockout的 API 调用,避免直接 HTTP 调用。

## 速率限制

**规则:** 使用 `ratelimiter` 中间件保护 API,分布式环境优先使用 Redis。

## 错误处理

`pkg/fmterr` 提供基于错误码的结构化错误处理机制，支持 Gin 和 gRPC。

**核心机制:**

| 函数 | 用途 |
|------|------|
| `New(code, err)` | 创建带错误码的错误 |
| `Code(code)` | 仅使用错误码创建错误 |
| `Codef(code, kvs...)` | 创建带元数据的错误 |
| `InitErrorHandler(cfg)` | 初始化错误码映射配置 |
| `WrapperGrpcStatus(err)` | 将错误包装为 gRPC status |

**配置驱动:**

```yaml
# app.yaml - 错误码映射
errorCodeMap:
  1001: "用户不存在"
  1002: "权限不足"
  
errorMap:
  "missing jwt": "认证信息缺失"
```

**gRPC 拦截器:**
- `UnaryServerInterceptor` - 服务端自动转换 `fmterr.Error` 为 gRPC status
- `UnaryClientInterceptorInGin` - 客户端自动转换 gRPC status 为 `gin.Error`

**使用方式:**

```go
// 创建带错误码的错误
err := fmterr.New(1001, errors.New("user not found"))

// 创建带元数据的错误
err := fmterr.Codef(1001, "userId", 123)

// 在 gRPC 服务中返回
return nil, err  // 拦截器自动转换为 gRPC status
```

## 测试约定

**规则:** 使用项目的测试目录结构,优先使用表驱动测试,断言使用 `github.com/stretchr/testify`。

**验证公开行为:** 测试应验证公开行为和功能正确性,而非断言私有字段的内部状态。
**使用 TestSuite:** 继承 `suite.Suite` 后使用其断言方法,不需要额外导入 `assert`。

## 关键文件参考

| 文件 | 用途 |
|------|------|
| `pkg/koapp/app.go` | 应用初始化和组件构建 |
| `pkg/identity/context.go` | 多租户上下文工具 |
| `pkg/authz/authz.go` | 权限 ARN 工具 |
| `ent/clientx/driver.go` | Ent 缓存驱动构建器 |
| `api/auth.go` | 认证服务 API 客户端 |
| `pkg/middleware/ratelimiter/` | 速率限制实现 |

## 常见反模式

| 反模式 | 正确做法 |
|--------|----------|
| 全局租户变量 | 使用 `identity.WithTenantID(ctx, id)` |
| 直接创建 HTTP 客户端 | 使用带拦截器的 `api.NewSDK()` |
| 手动设置 Ent 驱动 | 使用 `koapp.BuildEntComponents()` |
| 查询中缺少租户上下文 | 始终从 context 派生租户 |
| 生产环境使用本地速率限制器 | 分布式环境使用 Redis 限制器 |