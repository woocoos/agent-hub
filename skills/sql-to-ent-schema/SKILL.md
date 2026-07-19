---
name: sql-to-ent-schema
description: 从 SQL CREATE TABLE 定义生成 ent schema Go 文件
---

# SQL 转 Ent Schema

根据 SQL 表结构生成 ent schema Go 文件。

## 使用方式

- 指定表名：`生成 xxx 表的 schema`
- 粘贴 SQL：直接贴 CREATE TABLE 语句
- 指定 SQL 文件：`根据 docs/xxx.sql 生成 xxx 的 schema`

## 执行步骤

1. **定位项目结构**：查找当前项目中的 ent schema 目录（通常在 `ent/schema/` 或 `codegen/entgen/schema/`），以及 SQL 文件位置
2. **获取 SQL 定义**：从 SQL 文件读取或接受用户粘贴的 CREATE TABLE
3. **解析表结构**：提取表名、字段、类型、约束、索引
4. **生成 schema 文件**：按以下规则生成 Go 代码
5. **写入文件**：输出到 schema 目录
6. **编译验证**：运行 `go build` 验证

## 命名转换

- SQL 表名 → Go 结构体名：去业务前缀，PascalCase（如 `mat_trade_rule` → `TradeRule`）
- SQL 表名 → 文件名：去前缀，全小写无分隔符（如 `mat_trade_rule` → `traderule.go`）
- 常见前缀：`bas_`、`mat_`、`prd_`、`sys_`、`t_` 等

## SQL 类型 → ent 字段映射

| SQL 类型 | ent 字段 | 备注 |
|----------|----------|------|
| `int` | `field.Int("xxx")` | 如果项目定义了 IntSchemaType 则加 `.SchemaType(IntSchemaType)` |
| `bigint` | `field.Int("xxx")` | |
| `varchar(N)` | `field.String("xxx").MaxLen(N)` | |
| `char(N)` | `field.String("xxx").MaxLen(N)` | 通常带 Default |
| `text` | `field.Text("xxx")` | |
| `decimal(M,N)` | `fieldx.Decimal("xxx").Precision(M, N)` | 需导入 fieldx |
| `float` / `double` | `field.Float("xxx")` | |
| `timestamp` | `field.Time("xxx")` | |
| `date` | `field.Time("xxx")` | |
| `time` | `field.String("xxx")` | MySQL time 类型用 String 存储 |
| `datetime` | `field.Time("xxx")` | |
| `json` | `field.String("xxx")` | 用 String 存储 |
| `boolean` / `tinyint(1)` | `field.Bool("xxx")` | |

## 可空性与默认值规则

**核心原则：根据 SQL 列的约束决定 ent 修饰链。不使用 Nillable。**

| SQL 约束 | ent 修饰链 |
|----------|-----------|
| `NOT NULL`（无 DEFAULT） | 不加 Optional |
| `NOT NULL DEFAULT <value>` | `.Default(<value>)` |
| `NULL` / `DEFAULT NULL` | `.Optional()` |
| `DEFAULT <value>`（无 NOT NULL） | `.Default(<value>).Optional()` |

**注意：所有字段都不加 `.Nillable()`。**

## COMMENT → Comment

SQL 字段的 COMMENT 映射为 `.Comment("...")`，放在修饰链合适位置（通常在 Annotations 之前）。

## 审计字段 → Mixin（条件生成）

**每个 schema 都必须生成 `Mixin()` 方法**，但内容根据审计字段情况决定：

- **四个审计字段全部存在**（`created_by`、`created_at`、`updated_by`、`updated_at`）：使用 `schemax.AuditMixin{}`，审计字段**禁止出现在 Fields() 中**
- **缺少其中任何一个**：`Mixin()` 返回 `nil`，审计字段直接放在 `Fields()` 中

四个全部存在时：
```go
func (Xxx) Mixin() []ent.Mixin {
    return []ent.Mixin{
        schemax.AuditMixin{},
    }
}
```

不满足条件时：
```go
func (Xxx) Mixin() []ent.Mixin {
    return []ent.Mixin{}
}
```

### 租户字段（org_id）→ TenantMixin

**当表中存在 `org_id` 字段时**，需要额外添加 TenantMixin 和对应 Annotation：

Mixin 中添加 TenantMixin：
```go
func (Xxx) Mixin() []ent.Mixin {
    return []ent.Mixin{
        schemax.AuditMixin{},
        schemax.NewTenantMixin[intercept.Query, *gen.Client]("apis-meta", intercept.NewQuery,
            schemax.WithTenantMixinStorageKey[intercept.Query, *gen.Client]("org_id")),
    }
}
```

同时在 schema 的 `Annotations()` 方法中添加 `schemax.TenantField`：
```go
func (Xxx) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Annotation{Table: "table_name"},
        schemax.TenantField("org_id"),
    }
}
```

`org_id` 字段本身在 `Fields()` 中定义为：
```go
field.Int(schemax.FieldTenantID).StorageKey("org_id").Immutable().Comment("业务组织id").SchemaType(IntSchemaType)
```

需要额外导入：
- `"github.com/woocoos/knockout-go/ent/schemax"`
- `gen "t.qeelyn.com/pb/apis-meta/ent"` — 项目 ent 包，别名 `gen`
- `"t.qeelyn.com/pb/apis-meta/ent/intercept"`

TenantMixin 与 AuditMixin 同时存在时，两者都放在 Mixin() 返回中。

### org_id 索引字段

当索引包含 `org_id` 字段时，使用 `schemax.FieldTenantID` 替代字符串 `"org_id"`：

```go
// 正确
index.Fields(schemax.FieldTenantID, "dictionary_type_id", "code").Unique()

// 错误
index.Fields("org_id", "dictionary_type_id", "code").Unique()
```

## 特殊字段规则

### description 字段
```go
field.String("description").MaxLen(255).Optional().Comment("描述")
```

### 排序字段（list_order / display_order / sort_order）
**不加** SkipWhereInput，保持可查询。

### 路径字段（path / tree_path）
**不加** SkipWhereInput，保持可查询。

### JSON 字段

JSON 类型使用 `field.String` 存储：

```go
field.String("tag").Optional().Comment("标签")
```

- 使用 `field.String` 而非 `field.JSON`，JSON 内容以字符串形式存储
- 可空性规则与其他字段一致：SQL 允许 NULL 则加 `.Optional()`，有 DEFAULT 则加 `.Default(...).Optional()`

### DECIMAL 字段
- 使用 `fieldx.Decimal("xxx").Precision(M, N)`（来自 `github.com/woocoos/knockout-go/ent/schemax/fieldx`）
- 可空时加 `.Optional()`，不加 `.Nillable()`
- Default 用整数 `.Default(0)`，不要 `.Default(0.0000000)`
- 如果项目未引入 fieldx，则回退到 `field.Float().SchemaType()`

## 索引生成（必须）

**每个 schema 必须包含 `Indexes()` 方法**，即使没有索引也必须生成返回 `nil` 的空方法。

从 SQL 的 `KEY` / `INDEX` 定义生成：

```go
func (Xxx) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("field_name"),
        index.Fields("field1", "field2"),
        index.Fields("f1", "f2", "f3").Unique(),
    }
}
```

没有非主键索引时：

```go
func (Xxx) Indexes() []ent.Index {
    return []ent.Index{}
}
```

- 跳过 PRIMARY KEY
- 跳过 FULLTEXT KEY（ent 不支持）

## Edges（关联关系）

**当 SQL 表中存在明显外键字段时，必须生成完整的 edge 定义，而非留 TODO。**

### 识别外键

- 字段名以 `_id` 结尾（如 `margin_config_id`、`indicator_category_id`）且类型为 int/bigint
- 排除审计字段（`created_by`、`updated_by`）和普通业务 ID（如 `org_id`、`account_id`）
- 如果不确定是否为外键，检查项目中是否存在对应的目标 schema

### Edge 方向规则

- **子表（持有外键的一方）**：用 `edge.From` 指向父表
- **父表（被引用的一方）**：用 `edge.To` 指向子表（一对多）

### 实现步骤

1. 从外键字段名推断目标实体：`margin_config_id` → `MarginConfig`
2. 检查目标 schema 是否已存在于项目中
3. 如果存在，生成完整的 edge 定义并在目标 schema 中添加反向 edge
4. 如果不存在，生成空 edge 并加注释说明

### 完整示例

子表（持有外键 `margin_config_id`）：
```go
func (MarginLevel) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("config", MarginConfig.Type).Ref("levels").Unique().Field("margin_config_id").Required(),
    }
}
```

父表（反向一对多）：
```go
func (MarginConfig) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("levels", MarginLevel.Type).Comment("保证金等级"),
    }
}
```

### 注意事项

- `edge.From` 必须配合 `Ref()` 指向父表的 `edge.To` 名称
- 一对一关系：子表 `edge.From(...).Unique()`，父表 `edge.To(...).Unique()`
- 外键字段如果是 NOT NULL，edge 加 `.Required()`
- 外键字段如果是 nullable，edge 不加 `.Required()`
- 生成 edge 后需要同时更新目标 schema 的 Edges() 方法

## 完整文件模板

**方法顺序：Annotations → Mixin → Fields → Edges → Indexes**（Indexes 始终放在最后）

```go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/dialect/entsql"
    "entgo.io/ent/schema"
    "entgo.io/ent/schema/field"
    // 按需导入：
    // "entgo.io/ent/schema/index"
    // "entgo.io/ent/schema/edge"
    // "github.com/woocoos/knockout-go/ent/schemax"
    // "github.com/woocoos/knockout-go/ent/schemax/fieldx"
    // gen "t.qeelyn.com/pb/apis-meta/ent"
    // "t.qeelyn.com/pb/apis-meta/ent/intercept"
)

type Xxx struct {
    ent.Schema
}

func (Xxx) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Annotation{Table: "table_name"},
        // 如果有 org_id 字段，添加：
        // schemax.TenantField("org_id"),
    }
}

func (Xxx) Mixin() []ent.Mixin {
    // 四个审计字段全部存在时：
    return []ent.Mixin{
        schemax.AuditMixin{},
        // 如果有 org_id 字段，添加：
        // schemax.NewTenantMixin[intercept.Query, *gen.Client]("apis-meta", intercept.NewQuery,
        //     schemax.WithTenantMixinStorageKey[intercept.Query, *gen.Client]("org_id")),
    }
    // 缺少任何一个审计字段时：
    // return []ent.Mixin{}
}

func (Xxx) Fields() []ent.Field {
    return []ent.Field{
        field.Int("id").SchemaType(IntSchemaType),
        // 如果有 org_id 字段：
        // field.Int(schemax.FieldTenantID).StorageKey("org_id").Immutable().Comment("业务组织id").SchemaType(IntSchemaType),
        // ...
    }
}

func (Xxx) Edges() []ent.Edge {
    return []ent.Edge{}
}

func (Xxx) Indexes() []ent.Index {
    return []ent.Index{}
    // 如果索引包含 org_id，使用 schemax.FieldTenantID：
    // index.Fields(schemax.FieldTenantID, "other_field").Unique()
}
```

## 后续操作（schema 生成并编译通过后执行）

### 步骤 A：询问是否执行代码生成

schema 文件写入并通过 `go build` 编译后，**必须询问用户**：

> 是否执行 `go run codegen/entgen/entc.go` 生成 ent 代码？

- 如果用户同意，执行该命令并确认输出无错误，然后进入步骤 B
- 如果用户拒绝，进入步骤 B

### 步骤 B：询问是否创建测试用例

**询问用户**：

> 是否在 `ent_test.go` 中为该 schema 创建测试用例？

- 如果用户同意，执行以下操作
- 如果用户拒绝，跳过此步骤，流程结束

在项目根目录的 `ent_test.go` 中为新生成的 schema 添加测试用例。

**如果 `ent_test.go` 不存在**，新建文件，包含基础的 `getClient` 函数和必要的 import：

```go
package apismeta_test

import (
	"context"
	"testing"

	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql"
	"github.com/stretchr/testify/assert"
	"t.qeelyn.com/pb/apis-meta/ent"

	_ "github.com/go-sql-driver/mysql"
	_ "t.qeelyn.com/pb/apis-meta/ent/runtime"
)

var client *ent.Client

func getClient(migration bool) *ent.Client {
	if client == nil {
		drvori, err := sql.Open(dialect.MySQL, "deo:deo135@$^@tcp(192.168.0.18:3306)/deo_meta?parseTime=true&loc=Asia%2FShanghai")
		if err != nil {
			panic(err)
		}
		client = ent.NewClient(ent.Driver(drvori), ent.Debug())
	}
	return client
}
```

**测试用例模板**：

测试函数命名为 `Test{EntityName}`，查询使用 `Limit(100)` 并按 `id` 倒序排列：

```go
func Test{EntityName}(t *testing.T) {
	client := getClient(false)
	ctx := context.Background()
	result, err := client.{EntityName}.Query().Order(ent.Desc("{entity_field_id}")).Limit(100).All(ctx)
	assert.NoError(t, err)
	assert.NotNil(t, result)
}
```

- `{EntityName}` 替换为生成的 schema 结构体名（如 `TradeRule`、`Product`）
- `{entity_field_id}` 替换为该实体的 ID 字段名，通常为 `id`；需要检查生成的 schema 中是否有对应的 `{entity}FieldID` 常量，如果有则使用该常量包引用（如 `currencyrate.FieldID`），否则直接用字符串 `"id"`
- 如果 schema 有租户字段（org_id），ctx 需要包装：`ctx = schemax.SkipTenantPrivacy(ctx)`，并导入 `"github.com/woocoos/knockout-go/ent/schemax"`
- 将新的测试函数追加到文件末尾，不修改已有的测试函数

## 注意事项

- 不导入未使用的包
- 生成后运行 `go build` 验证编译
- 已有同名文件时提示用户确认是否覆盖
- 先读取项目中已有 schema 文件，保持风格一致（import 顺序、注释风格等）
