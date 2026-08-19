---
name: knockout-go
description: 基于 woocoo knockout 应用架构的 Go 开发实践。用于开发基于 knockout 架构的服务、处理多租户、Ent 缓存、Casbin 权限控制或构建 woocoo 应用。
source: https://github.com/woocoos/knockout-go
license: MIT
---

# Knockout-Go 开发准则

基于 woocoo 框架的 knockout-go 应用 SDK 开发准则。提供多租户、缓存、权限控制和服务集成的最佳实践。

## 1. 项目概述

**knockout-go** 是基于 [woocoo](https://github.com/tsingsun/woocoo) 框架的 Go 应用 SDK 和工具包。提供:

- 应用初始化和组件构建 (`pkg/koapp`)
- 多租户和身份上下文管理 (`pkg/identity`)
- 基于 Casbin 的权限控制 (`pkg/authz`)
- Ent 缓存驱动扩展 (`ent/clientx`)
- 分页工具 (`pkg/pagination`)
- 速率限制中间件 (`pkg/middleware/ratelimiter`)

## 2. 技术栈

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

## 3. 代码风格

### 注释规范

- 结构体字段注释放在字段上方，不在行尾
- 不使用装饰线（`// ===`、`// ---`）作为分节符
- 注释使用中文时, 必须使用半角标点

```go
// 正确: 字段注释在上方
type Options struct {
    // Limit 窗口内允许的最大请求数
    Limit uint
    // Rate 限流窗口时长
    Rate time.Duration
}

// 错误: 注释在行尾
type Options struct {
    Limit uint      // 窗口内允许的最大请求数
    Rate  time.Duration  // 限流窗口时长
}
```

## 5. 多租户模式

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

## 6. Ent 缓存配置

**规则:** 使用 `entcache` 进行数据库查询缓存,通过 `BuildEntCacheDriver` 配置。

```go
// 带 Ent 缓存的应用初始化
func initApp() *woocoo.App {
    app := koapp.New()

    // 构建带缓存的 Ent 组件
    drivers := koapp.BuildEntComponents(app.AppConfiguration())
    // drivers 是 map[string]dialect.Driver,按 schema 名称索引
    return app
}

// 配置示例 (app.yaml)
// store:
//   portal:
//     driverName: mysql
//     dsn: "user:pass@tcp(localhost:3306)/portal"
//   entcache:
//     ttl: 5m
//     gcInterval: 10m
```

**多租户缓存隔离:**

```yaml
# app.yaml - 按租户隔离缓存
entcache:
  isolate: true  # 每个租户获得独立的缓存命名空间
```

## 7. Casbin 权限控制

**规则:** 使用 ARN (Amazon Resource Name) 格式进行资源标识。

```go
// 资源 ARN 格式: app:domain:resource:tenant_id:action
// 示例: portal:production:order:123:read

// 格式化资源 ARN 前缀
prefix := authz.FormatArnPrefix("portal", "production", "order")
// 结果: "portal:production:order:"

// 替换 ARN 中的 tenant_id 占位符
arn := authz.ReplaceTenantID("app:resource:tenant_id:read", 123)
// 结果: "app:resource:123:read"

// 操作类型
const (
    ActionTypeRead   = "read"
    ActionTypeWrite  = "write"
    ActionTypeSchema = "schema"
)
```

## 8. 应用初始化模式

**规则:** 使用 `koapp.New()` 进行应用引导,自动初始化组件。

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

## 9. API 客户端模式

**规则:** 使用 SDK 模式进行 API 调用,禁止直接 HTTP 调用。

```go
// 正确: 使用带拦截器的 SDK
type SDK struct {
    client *http.Client
    Auth   *Auth
    Msg    *Msg
    // ...
}

func NewSDK(cnf *conf.Configuration) (*SDK, error) {
    sdk := &SDK{
        client: &http.Client{},
    }
    // 应用配置
    sdk.Auth = NewAuth()
    sdk.Auth.Apply(sdk, cnf.Sub("auth"))
    return sdk, nil
}

// 正确: 使用拦截器传递租户上下文
func TenantIDInterceptor(req *http.Request) error {
    if tid, ok := identity.TenantIDLoadFromContext(req.Context()); ok {
        req.Header.Set(identity.TenantHeaderKey, strconv.Itoa(tid))
    }
    return nil
}

// 错误: 直接 HTTP 调用,不使用 SDK
resp, err := http.Get("http://auth/api/user") // 不要这样做
```

## 10. 速率限制

**规则:** 使用 `ratelimiter` 中间件保护 API,分布式环境优先使用 Redis。

```go
import "github.com/woocoos/knockout-go/pkg/middleware/ratelimiter"

// 基于 Redis 的速率限制器 (分布式)
limiter := ratelimiter.NewRedisRateLimiter(rdb, &ratelimiter.Config{
    Limit:  100,        // 每个时间窗口的请求数
    Window: time.Minute,
})

// 内存速率限制器 (单实例)
limiter := ratelimiter.NewMemoryRateLimiter(&ratelimiter.Config{
    Limit:  100,
    Window: time.Minute,
})

// 在 Gin 路由中使用
r := gin.New()
r.Use(ratelimiter.Middleware(limiter))
```

## 11. 错误处理

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

## 12. 测试约定

**规则:** 使用项目的测试目录结构,优先使用表驱动测试,断言使用 `github.com/stretchr/testify`。

**验证公开行为:** 测试应验证公开行为和功能正确性,而非断言私有字段的内部状态。

```go
// 错误: 断言内部状态
func TestAddField(t *testing.T) {
    d := NewRegistry()
    d.AddField("code", 1)
    assert.Nil(t, d.fieldsByCode)  // 不要这样做
}

// 正确: 验证公开行为
func TestAddField(t *testing.T) {
    d := NewRegistry()
    d.AddField("code", 1)
    field, ok := d.GetFieldByCode(1)
    assert.True(t, ok)
    assert.Equal(t, "code", field.Name)
}
```

**使用 TestSuite:** 继承 `suite.Suite` 后使用其断言方法,不需要额外导入 `assert`。

```go
import "github.com/stretchr/testify/suite"

type UserSuite struct {
    suite.Suite
    db *ent.Client
}

func (s *UserSuite) TestCreate() {
    user, err := CreateUser("test@example.com")
    s.NoError(err)           // 使用 suite 的断言方法
    s.NotNil(user)
    s.Equal("test@example.com", user.Email)
}

func (s *UserSuite) TestNotFound() {
    _, err := GetUser(999)
    s.Error(err)
    s.ErrorIs(err, ErrNotFound)
}

func TestUserSuite(t *testing.T) {
    suite.Run(t, new(UserSuite))
}
```

```
test/
├── testdata/        # 测试数据
│   └── casbin.yaml
└── integration/     # 集成测试
```

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// 表驱动测试模式
func TestTenantIDFromContext(t *testing.T) {
    tests := []struct {
        name    string
        ctx     context.Context
        want    int
        wantErr error
    }{
        {
            name:    "有效的租户 ID",
            ctx:     identity.WithTenantID(context.Background(), 123),
            want:    123,
            wantErr: nil,
        },
        {
            name:    "缺少租户 ID",
            ctx:     context.Background(),
            want:    0,
            wantErr: identity.ErrMisTenantID,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := identity.TenantIDFromContext(tt.ctx)
            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
            } else {
                assert.NoError(t, err)
            }
            assert.Equal(t, tt.want, got)
        })
    }
}
```

## 13. 关键文件参考

| 文件 | 用途 |
|------|------|
| `pkg/koapp/app.go` | 应用初始化和组件构建 |
| `pkg/identity/context.go` | 多租户上下文工具 |
| `pkg/authz/authz.go` | 权限 ARN 工具 |
| `ent/clientx/driver.go` | Ent 缓存驱动构建器 |
| `api/auth.go` | 认证服务 API 客户端 |
| `pkg/middleware/ratelimiter/` | 速率限制实现 |

## 14. 常见反模式

| 反模式 | 正确做法 |
|--------|----------|
| 全局租户变量 | 使用 `identity.WithTenantID(ctx, id)` |
| 直接创建 HTTP 客户端 | 使用带拦截器的 `api.NewSDK()` |
| 手动设置 Ent 驱动 | 使用 `koapp.BuildEntComponents()` |
| 查询中缺少租户上下文 | 始终从 context 派生租户 |
| 生产环境使用本地速率限制器 | 分布式环境使用 Redis 限制器 |