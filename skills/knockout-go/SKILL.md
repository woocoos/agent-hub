---
name: knockout-go
description: 基于 woocoo knockout 应用架构的 Go 开发实践。用于开发基于 knockout 架构的服务、处理多租户、Ent ORM (代码生成/Schema/Mixin/查询/迁移/缓存)、gqlgen GraphQL (entgql 集成/Resolver/Server)、Casbin 权限控制或构建 woocoo 应用。
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

## Ent 代码生成

**规则:** 使用 `entc.Generate` 配合 knockout-go 提供的 `entx` 和 `entcachegen` 扩展进行代码生成。

### 生成入口 (entc.go)

代码生成文件放在 `codegen/entgen/entc.go`, 使用 `//go:build ignore` 标签, 通过 `go run codegen/entgen/entc.go` 执行:

```go
//go:build ignore

package main

import (
    "log"
    "os"

    "entgo.io/contrib/entgql"
    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
    entcachegen "github.com/woocoos/entcache/gen"
    "github.com/woocoos/knockout-go/codegen/entx"
)

func main() {
    ex, err := entgql.NewExtension(
        entx.WithGqlWithTemplates(),
        entgql.WithSchemaGenerator(),
        entgql.WithWhereInputs(true),
        entgql.WithConfigPath("codegen/gqlgen/gqlgen.yaml"),
        entgql.WithSchemaPath("api/graphql/ent.graphql"),
        entgql.WithSchemaHook(entx.ChangeRelayNodeType(), entx.DecimalScalar()),
    )
    if err != nil {
        log.Fatalf("creating entgql extension: %v", err)
    }
    os.MkdirAll("./api/graphql", os.ModePerm)
    opts := []entc.Option{
        entc.Extensions(ex, entx.DecimalExtension{}),
        entx.GlobalID(),
        entx.SimplePagination(),
        entcachegen.QueryCache(),
    }
    err = entc.Generate("./codegen/entgen/schema", &gen.Config{
        Package: "your module path/ent",
        Features: []gen.Feature{
            gen.FeatureVersionedMigration,
            gen.FeatureUpsert,
            gen.FeatureIntercept,
            gen.FeatureSchemaConfig,
        },
        Target: "./ent",
    }, opts...)
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

### knockout-go 扩展说明

| 扩展 | 来源 | 功能 |
|------|------|------|
| `entx.GlobalID()` | knockout-go/codegen/entx | 启用全局 ID, Relay Node 兼容 |
| `entx.SimplePagination()` | knockout-go/codegen/entx | 简化 Relay 分页实现 |
| `entx.DecimalExtension{}` | knockout-go/codegen/entx | Decimal 标量类型的代码生成支持 |
| `entx.WithGqlWithTemplates()` | knockout-go/codegen/entx | 使用自定义 GraphQL 模板 |
| `entx.ChangeRelayNodeType()` | knockout-go/codegen/entx | 调整 Relay Node 类型映射 |
| `entx.DecimalScalar()` | knockout-go/codegen/entx | Decimal 类型的 GraphQL 标量处理 |
| `entcachegen.QueryCache()` | woocoos/entcache/gen | 生成 entcache 查询缓存代码, 织入生成的查询方法中 |

### 必需的 Feature 标志

| Feature | 用途 |
|---------|------|
| `gen.FeatureVersionedMigration` | 版本化迁移, 支持增量迁移文件 |
| `gen.FeatureUpsert` | 启用 Upsert (INSERT ON CONFLICT) 操作 |
| `gen.FeatureIntercept` | 启用拦截器, 用于 TenantMixin 的租户隐私过滤 |
| `gen.FeatureSchemaConfig` | 启用运行时 schema 配置, 支持 `ent.AlternateSchema` 跨库查询 |

### Schema 目录结构

Schema 文件放在 `codegen/entgen/schema/` 目录下(非传统的 `ent/schema/`), 生成目标为 `./ent`:

```
project-root/
├── codegen/
│   └── entgen/
│       ├── entc.go              ← 代码生成入口
│       └── schema/              ← Schema 定义目录
│           ├── order.go
│           ├── util.go          ← 共享类型定义
│           └── ...
├── ent/                         ← 生成代码输出目录
│   ├── client.go
│   ├── runtime.go
│   ├── hook/hook.go
│   ├── intercept/intercept.go
│   ├── migrate/
│   ├── enttest/
│   └── ...
└── ...
```

## Ent Schema 定义

**规则:** 使用 knockout-go 提供的 Mixin 组合实现通用的 ID 策略、审计字段、多租户隔离和软删除。

### Mixin 体系

knockout-go 的 `ent/schemax` 包提供以下 Mixin:

| Mixin | 功能 | 泛型参数 |
|-------|------|----------|
| `schemax.SnowFlakeID{}` | 雪花算法 ID, 替代自增 ID | 无 |
| `schemax.AuditMixin{}` | 审计字段: created_by, created_at, updated_by, updated_at | `Precision` 可选, 毫秒精度 |
| `schemax.NewTenantMixin[Q, C]()` | 多租户隐私拦截, 自动注入租户过滤条件 | `Q`: 拦截器类型, `C`: Client 指针类型 |
| `schemax.NewSoftDeleteMixin[Q, C]()` | 软删除, 删除操作自动转为更新 deleted_at | `Q`: 拦截器类型, `C`: Client 指针类型 |

### Mixin 组合示例

```go
import (
    "github.com/woocoos/knockout-go/ent/schemax"
    gen "your module path/ent"
    "your module path/ent/intercept"
    "your module path/version"
)

// 完整 Mixin 组合 (雪花ID + 审计 + 多租户 + 软删除)
func (Order) Mixin() []ent.Mixin {
    return []ent.Mixin{
        schemax.SnowFlakeID{},
        schemax.AuditMixin{Precision: 3},
        schemax.NewTenantMixin[intercept.Query, *gen.Client](
            version.AppCode,
            intercept.NewQuery,
            schemax.WithTenantMixinDomainStorageKey[intercept.Query, *gen.Client]("domain_id"),
        ),
        schemax.NewSoftDeleteMixin[intercept.Query, *gen.Client](intercept.NewQuery),
    }
}

// 简单 Mixin 组合 (审计 + 多租户, 使用默认存储键 tenant_id)
func (Project) Mixin() []ent.Mixin {
    return []ent.Mixin{
        schemax.AuditMixin{},
        schemax.NewTenantMixin[intercept.Query, *gen.Client](
            version.AppCode,
            intercept.NewQuery,
            schemax.WithTenantMixinStorageKey[intercept.Query, *gen.Client]("org_id"),
        ),
    }
}
```

**TenantMixin 泛型参数说明:**
- 第一个泛型参数固定为 `intercept.Query` (由 `FeatureIntercept` 生成)
- 第二个泛型参数为 `*gen.Client` (Ent 生成的 Client 指针类型)
- `version.AppCode` 为应用标识, 用于缓存命名空间
- `WithTenantMixinStorageKey` 指定租户字段的存储列名, 默认为 `tenant_id`
- `WithTenantMixinDomainStorageKey` 指定域级别的租户字段存储列名

### Schema Annotation

```go
func (Order) Annotations() []schema.Annotation {
    return []schema.Annotation{
        // 指定数据库表名
        entsql.Annotation{Table: "order"},
        // 声明租户字段名, 用于 entcache 缓存隔离
        schemax.TenantField("tenant_id"),
        // GraphQL: 生成查询字段
        entgql.QueryField(),
        // GraphQL: 启用 Relay 连接分页
        entgql.RelayConnection(),
        // GraphQL: 暴露创建和更新 mutation
        entgql.Mutations(entgql.MutationCreate(), entgql.MutationUpdate()),
    }
}
```

### 字段类型

#### 自定义 Decimal 字段

使用 `fieldx.Decimal()` 替代 Ent 原生的 `field.Float()`, 提供精确的数值计算:

```go
import "github.com/woocoos/knockout-go/ent/schemax/fieldx"

fieldx.Decimal("price").Optional().Precision(16, 4).Default(0).Comment("价格")
fieldx.Decimal("amount").Precision(16, 2).Default(0).Optional().Comment("金额")
```

#### 枚举字段绑定自定义 Go 类型

```go
// dicSchemaType 定义枚举的数据库存储类型
var dicSchemaType = map[string]string{
    dialect.MySQL:    "VARCHAR(10)",
    dialect.Postgres: "VARCHAR(10)",
    dialect.SQLite:   "TEXT",
}

field.Enum("side").GoType(types.Side("")).SchemaType(dicSchemaType).Comment("买卖方向")
```

#### 跨数据库时间类型

```go
var timeSchemaType = map[string]string{
    dialect.MySQL:    "TIMESTAMP(3)",
    dialect.Postgres: "TIMESTAMPTZ",
    dialect.SQLite:   "DATETIME",
}

field.Time("transact_time").Default(time.Now).SchemaType(timeSchemaType)
```

#### Int 类型统一映射

```go
var IntSchemaType = map[string]string{
    dialect.MySQL:    "INT",
    dialect.Postgres: "INT",
    dialect.SQLite:   "INTEGER",
}

field.Int("product_id").SchemaType(IntSchemaType).Annotations(entgql.Type("ID"))
```

### Schema Hooks

在 Schema 中定义业务 Hook, 使用 `hook.On()` 限定操作类型:

```go
import (
    "context"
    "entgo.io/ent"
    gen "your module path/ent"
    "your module path/ent/hook"
)

func (Order) Hooks() []ent.Hook {
    return []ent.Hook{
        hook.On(func(next ent.Mutator) ent.Mutator {
            return hook.OrderFunc(func(ctx context.Context, mu *gen.OrderMutation) (gen.Value, error) {
                // 创建时自动推导 position_effect
                ps, _ := mu.PositionEffect()
                if d, ok := mu.Direction(); ok {
                    if ps == "" {
                        switch d {
                        case types.DirectionOpen:
                            mu.SetPositionEffect(types.PositionEffectOpen)
                        case types.DirectionClose:
                            mu.SetPositionEffect(types.PositionEffectClose)
                        }
                    }
                }
                return next.Mutate(ctx, mu)
            })
        }, ent.OpCreate),
    }
}
```

## Ent 客户端初始化

**规则:** 通过 `koapp.BuildEntComponents()` 获取驱动后创建 Ent Client, 必须空白导入 `ent/runtime` 包注册 Hooks 和 Interceptors。

### 基本初始化

```go
import (
    "your module path/ent"
    _ "your module path/ent/runtime"  // 必须: 注册 hooks/interceptors/validators
    _ "github.com/go-sql-driver/mysql" // 数据库驱动
)

func main() {
    app := koapp.New()
    cnf := app.AppConfiguration()

    ents := koapp.BuildEntComponents(cnf)
    drv, ok := ents["oms"]
    if !ok {
        panic("no oms ent driver")
    }
    db := ent.NewClient(ent.Driver(drv))
    if cnf.Development {
        db = db.Debug()
    }
    defer db.Close()
}
```

### AlternateSchema 跨库查询

使用 `ent.AlternateSchema()` 将特定实体映射到不同的数据库 schema:

```go
db := ent.NewClient(ent.Driver(drv), ent.AlternateSchema(ent.SchemaConfig{
    Project:     "deo_business",
    ProjectIsda: "deo_business",
}))
```

**前提:** 代码生成时必须启用 `gen.FeatureSchemaConfig`。配置中的 key 为 Schema 结构体名(如 `Project`), value 为目标数据库名。

### 服务层注入

通过 Option 模式将 Ent Client 注入到服务层:

```go
// 直接注入 Client
func WithDbClient(client *ent.Client) DayEndOption {
    return func(opts *DayEndOptions) {
        opts.DB = client
    }
}

// 注入为缓存层
func WithCache(client *ent.Client) Option {
    return func(o *Options) {
        o.Cache = caching.NewCache(client)
    }
}

// 注入到 GraphQL Resolver
type ServerOption struct {
    Db *ent.Client
}
```

## Ent 查询模式

### 基本查询

```go
// 条件查询
ords, err := db.Order.Query().Where(
    order.TradeDate(trdDate),
    order.ProductID(int(req.ProductId)),
    order.OrdStatusIn(types.UnclearOrdStatus()...),
    order.IsSettled(false),
).All(ctx)

// 单条查询 + 预加载关联
cond, err := db.OrderCondition.Query().
    Where(ordercondition.IDEQ(req.ConditionID)).
    WithOrder().
    Only(ctx)
```

### 批量更新

```go
err := db.Order.Update().
    SetIsSettled(true).
    Where(
        order.TradeDate(trdDate),
        order.ProductID(int(req.ProductId)),
    ).Exec(ctx)
```

### 原生 SQL 谓词

```go
dataList, err := db.OrderReport.Query().Where(func(s *sql.Selector) {
    s.Where(sql.ExprP(
        "order_id in (select id from `order` where account_id = ? and is_settled = 0)",
        accountID,
    ))
}).All(ctx)
```

### 跳过租户隐私过滤

使用 `schemax.SkipTenantPrivacy(ctx)` 在需要跨租户查询时跳过自动注入的租户条件:

```go
import "github.com/woocoos/knockout-go/ent/schemax"

// 同时跳过租户隐私和缓存
dataList, err := db.OrderReport.Query().Where(
    // ...
).All(schemax.SkipTenantPrivacy(entcache.Skip(ctx)))
```

## Ent 数据库迁移

**规则:** 使用 `//go:build ignore` 标签的独立脚本执行迁移, 支持版本化迁移。

### 迁移脚本

```go
//go:build ignore

package main

import (
    "context"
    "flag"
    "log"

    "your module path/ent"
    "your module path/ent/migrate"
    "github.com/woocoos/knockout-go/codegen/entx"
    _ "github.com/go-sql-driver/mysql"
)

var (
    dsn  = flag.String("dsn", "root:@tcp(localhost:3306)/oms", "")
    name = flag.String("name", "mysql", "driver name")
)

func main() {
    flag.Parse()
    client, err := ent.Open(*name, *dsn)
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()

    err = client.Schema.Create(
        context.Background(),
        migrate.WithDropIndex(true),
        migrate.WithDropColumn(true),
        migrate.WithForeignKeys(false),
        entx.SkipTablesDiffHook("table_name"),
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

**迁移选项说明:**

| 选项 | 作用 |
|------|------|
| `migrate.WithDropIndex(true)` | 允许删除不再需要的索引 |
| `migrate.WithDropColumn(true)` | 允许删除不再需要的列 |
| `migrate.WithForeignKeys(false)` | 禁用外键约束(推荐, 避免分布式环境的外键问题) |
| `entx.SkipTablesDiffHook(...)` | 跳过指定表的 schema diff, 用于不需要自动迁移的表 |

### 测试用迁移

使用 `enttest.Open()` 自动执行迁移, 用于测试环境:

```go
import "your module path/ent/enttest"

client := enttest.Open(t, "mysql", dsn)
defer client.Close()
```

## Ent 缓存 (entcache)

**规则:** 通过 `entcachegen.QueryCache()` 在代码生成时将缓存逻辑织入查询方法, 运行时由 `koapp.BuildEntComponents()` 自动构建缓存驱动。

### 缓存控制

```go
import "github.com/woocoos/entcache"

// 跳过缓存(用于写操作后的即时查询)
ctx := entcache.Skip(ctx)

// 设置缓存引用 key (用于 Node API)
ctx := entcache.WithRefEntryKey(ctx, "Order", id)
```

### 缓存配置

```yaml
# app.yaml
entcache:
  isolate: true       # 按 schema 隔离缓存命名空间
  ttl: 5m             # 缓存 TTL
  gcInterval: 10m     # GC 间隔
  hashQueryTTL: 10m   # 查询哈希 TTL
  keyQueryTTL: 10m    # Key 查询 TTL
```

## GraphQL (gqlgen + entgql)

**规则:** 使用 entgql 自动生成 ent 实体的 GraphQL schema 和 resolver, 手动编写业务 schema 通过 `extend type` 扩展。

### 代码生成流程

Ent 代码生成 (`entc.go`) 和 gqlgen 代码生成 (`gqlgen.go`) 是两个独立步骤:

1. **先执行 ent 代码生成**: `go run codegen/entgen/entc.go` — 生成 ent schema + `ent.graphql` + ent 的 Go 代码
2. **再执行 gqlgen 代码生成**: `go run codegen/gqlgen/gqlgen.go` — 读取所有 `.graphql` 文件, 生成 resolver 骨架和 model

### gqlgen 配置

配置文件放在 `codegen/gqlgen/gqlgen.yaml`:

```yaml
schema:
  - api/graphql/*.graphql    # 包含 ent.graphql (自动生成) + 手动编写的 schema

exec:
  layout: follow-schema      # 生成文件按 schema 文件名对应
  dir: api/graphql/generated
  package: generated

model:
  filename: api/graphql/model/models_gen.go
  package: model

resolver:
  layout: follow-schema      # resolver 文件按 schema 文件名对应
  dir: api/graphql
  package: graphql

skip_mod_tidy: true
omit_gqlgen_version_in_file_notice: true

# 自动绑定 ent 生成的类型, 避免重复定义
autobind:
  - your module path/ent

models:
  ID:
    model:
      - github.com/99designs/gqlgen/graphql.IntID  # 整型 ID

directives:
  constraint:
    skip_runtime: true
```

### gqlgen 生成入口

```go
//go:build ignore

package main

import (
    "log"
    "os"

    "github.com/99designs/gqlgen/api"
    "github.com/99designs/gqlgen/codegen/config"
    "github.com/99designs/gqlgen/plugin/modelgen"
    "github.com/woocoos/knockout-go/codegen/gqlx"
)

func main() {
    cfg, err := config.LoadConfig("./codegen/gqlgen/gqlgen.yaml")
    if err != nil {
        log.Fatal(err)
    }
    p := modelgen.Plugin{}
    err = api.Generate(cfg,
        api.ReplacePlugin(&p),
        // knockout-go 的 ResolverPlugin, 增强 Relay Node 支持
        api.AddPlugin(gqlx.NewResolverPlugin(
            gqlx.WithRelayNodeEx(),
            gqlx.WithConfig(cfg),
        )),
    )
    if err != nil {
        log.Fatal(err)
    }
}
```

### Schema 文件组织

GraphQL schema 文件放在 `api/graphql/` 目录下, 分为自动生成和手动编写两类:

| 文件 | 类型 | 内容 |
|------|------|------|
| `ent.graphql` | entgql 自动生成 | ent 实体的 type/input/enum/connection/edge, 基础 Query (node/nodes) |
| `query.graphql` | 手动编写 | `extend type Query` 扩展自定义查询 |
| `mutation.graphql` | 手动编写 | `type Mutation` 完整定义 |
| `types.graphql` | 手动编写 | 自定义 scalar/input/type/enum, `extend type` 扩展 ent 类型 |

**协同模式:**
- `ent.graphql` 由 entgql 自动生成, 包含所有 ent schema 对应的 GraphQL 类型和 Relay 规范 (Node, Connection, Edge, PageInfo)
- 手动 schema 使用 `extend type Query` 扩展查询, 使用 `extend type XxxEntity` 为 ent 实体添加计算字段
- `autobind` 配置使 ent 类型自动映射到 GraphQL, 无需手动重复定义

### 自定义 Scalar 和类型映射

```graphql
# knockout-go 提供的自定义 scalar
scalar Decimal @goModel(model: "github.com/woocoos/knockout-go/ent/schemax/typex.Decimal")
scalar MapString @goModel(model: "github.com/woocoos/knockout-go/ent/schemax/typex.MapString")

# 枚举通过 @goModel 绑定到 Go 类型
enum OrderSide @goModel(model: "your module path/types.Side") {
    BUY
    SELL
}

# input 可直接映射到 protobuf 类型
input NewOrderRequest @goModel(model: "your module path/api/omspb.NewOrderRequest") {
    account: String!
    symbol: String!
    # ...
}
```

### Resolver 结构

Resolver 使用 Functional Options 模式构造:

```go
type Resolver struct {
    client     *ent.Client
    KoSdk      *api.SDK
    // 其他服务客户端...
}

type Option func(*Resolver)

func WithEntClient(client *ent.Client) Option {
    return func(r *Resolver) { r.client = client }
}

func NewResolver(opts ...Option) *Resolver {
    r := &Resolver{}
    for _, opt := range opts {
        opt(r)
    }
    return r
}

func NewSchema(resolver *Resolver) graphql.ExecutableSchema {
    return generated.NewExecutableSchema(generated.Config{
        Resolvers: resolver,
    })
}
```

**Resolver 文件分工 (follow-schema layout):**

| 文件 | 说明 |
|------|------|
| `resolver.go` | Resolver 结构体定义 + Option + NewSchema (不自动生成) |
| `query.resolvers.go` | Query resolver (gqlgen 生成骨架, 手动填充逻辑) |
| `mutation.resolvers.go` | Mutation resolver (gqlgen 生成骨架, 手动填充逻辑) |
| `ent.resolvers.go` | ent 实体的 edge/计算字段 resolver |
| `types.resolvers.go` | 自定义类型的字段 resolver |

### GraphQL Server 初始化

使用 woocoo 框架的 `gql.RegisterSchema()` 集成 gqlgen:

```go
import (
    "github.com/tsingsun/woocoo/contrib/gql"
    "github.com/tsingsun/woocoo/web"
    "github.com/woocoos/knockout-go/pkg/middleware"
    "entgo.io/contrib/entgql"
)

func (s *Server) buildWebEngine(cnf *conf.AppConfiguration) {
    s.webSrv = web.New(web.WithConfiguration(cnf.Sub("web")),
        web.WithGracefulStop(),
        gql.RegisterMiddleware(),           // woocoo GraphQL 中间件
        otelweb.RegisterMiddleware(),       // OpenTelemetry
        web.WithMiddlewareNewFunc("authz", authz.Middleware),
        middleware.RegisterTenantID(),      // 租户 ID 注入
        middleware.RegisterTokenSigner(),   // JWT Token 解析
        middleware.RegisterCacheControl(),  // 缓存控制
        ratelimiter.RegisterMiddleware(),   // 限流
    )

    // 创建 resolver 并注册 schema
    s.resolver = NewResolver(WithEntClient(s.Db), ...)
    ss, _ := gql.RegisterSchema(s.webSrv, NewSchema(s.resolver))
    s.gqlSrv = ss[0]

    // 分页中间件
    s.gqlSrv.AroundResponses(middleware.SimplePagination())

    // ent 事务管理: 自动为 mutation 开启事务
    s.gqlSrv.Use(entgql.Transactioner{
        TxOpener: s.Db,
        // 跳过特定 mutation 的事务 (如走 gRPC 远程调用的下单操作)
        SkipTxFunc: entgql.SkipIfHasFields(
            "newOrder", "cancelOrder", "replaceOrder",
        ),
    })
}
```

**关键集成点:**

| 组件 | 来源 | 作用 |
|------|------|------|
| `gql.RegisterMiddleware()` | woocoo/contrib/gql | 注册 GraphQL HTTP handler 中间件 |
| `gql.RegisterSchema()` | woocoo/contrib/gql | 将 ExecutableSchema 注册到 web server |
| `entgql.Transactioner` | entgo/contrib/entgql | 自动事务管理 |
| `middleware.SimplePagination()` | knockout-go/pkg/middleware | Relay 分页元数据处理 |
| `middleware.RegisterTenantID()` | knockout-go/pkg/middleware | 从 JWT 提取租户 ID 注入 context |
| `middleware.RegisterTokenSigner()` | knockout-go/pkg/middleware | JWT Token 解析和验证 |

### 应用入口集成

```go
func main() {
    app := koapp.New()
    cnf := app.AppConfiguration()

    // Ent 初始化 (参见 "Ent 客户端初始化" 章节)
    ents := koapp.BuildEntComponents(cnf)
    db := ent.NewClient(ent.Driver(ents["oms"]))
    defer db.Close()

    // GraphQL Server
    so := graphql.ServerOption{
        Db: db,
        // 注入其他服务客户端...
    }
    so.KoSdk, _ = api.NewSDK(cnf.Sub("kosdk"))
    gqlServer, _ := graphql.NewServer(app, so)

    // 注册到 knockout 应用
    app.RegisterServer(gqlServer)
    app.Run()
}
```

## Casbin 权限控制

**规则:** 使用 ARN (Amazon Resource Name) 格式进行资源标识。

## 应用初始化模式

**规则:** 使用 `koapp.New()` 进行应用引导,其自动读取配置文件并初始化组件。
**规则:** 初始化工作避免放在`Start`方法中。

### 配置文件查找规则

`koapp.New()` 内部通过 woocoo 框架的 `conf` 包定位配置文件,解析顺序:

1. **默认路径:** `<可执行文件所在目录>/etc/app.yaml`
   - 基于 `filepath.Dir(os.Args[0])` 计算,即编译产物(二进制文件)所在目录
2. **环境变量覆盖:** 设置 `WOOCOO_BASEDIR` 可覆盖默认的基础目录
3. **Option 覆盖:** 使用 `koapp.New(woocoo.WithConf(conf.WithBaseDir("/path")))` 指定

### etc 目录放置位置

**规则:** `etc` 目录放在 `cmd/<服务名>/etc/` 下,与 `main.go` 同级。

例如管理端服务放在 `cmd/admin/etc/`,认证服务放在 `cmd/auth/etc/`。

项目目录结构:

```
project-root/
├── cmd/
│   ├── admin/
│   │   ├── main.go
│   │   └── etc/              ← 管理端配置目录
│   │       ├── app.yaml      ← 主配置文件(必须)
│   │       └── rbac_model.conf
│   └── auth/
│       ├── main.go
│       └── etc/              ← 认证服务配置目录
│           └── app.yaml
├── internal/
├── go.mod
└── go.sum
```

**编译与运行:**

```bash
# 编译: 二进制输出到 cmd/admin/ 目录,etc 与之同级
go build -o cmd/admin/admin ./cmd/admin
# 此时 basedir = cmd/admin/, 配置文件 = cmd/admin/etc/app.yaml ✓

# 运行: 从项目根目录执行,二进制在 cmd/admin/ 下
./cmd/admin/admin
```

**开发阶段注意事项:**

`go run` 会将编译产物放在临时目录,导致找不到 `etc`。开发时应使用以下方式之一:

```bash
# 方式1: 设置 WOOCOO_BASEDIR 指向 cmd/<服务名> 目录
WOOCOO_BASEDIR=./cmd/admin go run ./cmd/admin

# 方式2: 先编译再运行
go build -o cmd/admin/admin ./cmd/admin && ./cmd/admin/admin
```

**反模式:** 不要将 `etc` 放在项目根目录,因为 `go build` 的输出目录是 `cmd/<服务名>/`,不是项目根目录。

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

### 配置结构

```yaml
# cmd/<服务名>/etc/app.yaml
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
| `ent/schemax/` | Schema Mixin 体系 (SnowFlakeID, AuditMixin, TenantMixin, SoftDeleteMixin) |
| `ent/schemax/fieldx/` | 自定义字段类型 (Decimal) |
| `codegen/entx/` | Ent 代码生成扩展 (GlobalID, SimplePagination, DecimalScalar) |
| `codegen/gqlx/` | gqlgen 代码生成扩展 (ResolverPlugin, RelayNodeEx) |
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
| 忘记空白导入 `ent/runtime` | 初始化 Ent Client 的 main.go 必须 `_ "module/ent/runtime"` |
| 使用 `field.Float` 存储金额 | 使用 `fieldx.Decimal()` 确保精确计算 |
| 未启用 `FeatureIntercept` 使用 TenantMixin | TenantMixin 依赖拦截器, 必须启用该 Feature |
| Schema 定义在 `ent/schema/` | knockout-go 项目放在 `codegen/entgen/schema/` |
| 手动修改 `ent.graphql` | 该文件由 entgql 自动生成, 修改 ent schema 后重新生成 |
| 手动修改 `*.generated.go` | gqlgen 自动生成的文件不可手动编辑, 修改 resolver 骨架中的逻辑 |
| 先执行 gqlgen 再执行 ent 代码生成 | ent schema 有变更时, 必须先 `entc.go` 生成 `ent.graphql`, 再 `gqlgen.go` 生成 resolver; schema 无变更时无顺序要求 |