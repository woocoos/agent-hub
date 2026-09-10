---
name: knockout-gqlgen-urql
description: GraphQL 接口开发 Skill — 基于 GraphQL Code Generator (client-preset) + urql + @knockout-js/ice-urql 技术栈，处理 *.graphql schema → services gql() 操作 → 生成类型的完整开发流程。
---

# GraphQL 接口开发 Skill — *.graphql → services → generated 全流程

> 适用项目：所有基于 knockout-js 生态的 React Web 项目
> 核心工具链：GraphQL Code Generator (client-preset) + urql + @knockout-js/ice-urql
>
> **自动调用说明**：本 skill 不绑定具体项目，适用于任何使用上述技术栈的项目。
> 使用时自动按当前项目的 `script/gqlgen.ts` 和 `src/services/index.ts` 适配模块与实例配置。

---

## 使用方式

当你需要为项目新增或修改 GraphQL 接口时，按本 skill 操作：
1. 确认目标模块是否已有 schema（查 `script/generated/`）
2. 如需新 schema，先拉取（`pnpm gqlgen:schema-ast`）
3. 在 `src/services/<模块>/` 中编写 `gql()` 操作
4. 在 `src/services/<模块>/enums.ts` 中编写枚举映射（查 §十）
5. 运行 `pnpm gqlgen` 生成类型
6. 从 `@/generated/<模块>` 导入类型使用
7. **已存在的接口不要重复编写**（查项目已有 services 目录）

---

## 一、代码生成管线概览

整个管线分两步，由 `script/gqlgen.ts` 统一配置：

```
┌─────────────────────────────────────────────────────────────┐
│  第一步：pnpm gqlgen:schema-ast                              │
│  从后端 GraphQL 端点拉取 Schema                              │
│  输出 → script/generated/<模块>.graphql                      │
│  （这一步只在后端 Schema 变更时才需要执行）                      │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  第二步：pnpm gqlgen                                         │
│  读取 script/generated/*.graphql 作为 Schema                 │
│  扫描 src/services/<模块>/**/*.ts 中的 gql() 操作             │
│  输出 → src/generated/<模块>/ (graphql.ts + gql.ts + ...)    │
│  （每次修改 service 文件后都要执行）                             │
└─────────────────────────────────────────────────────────────┘
```

### gqlgen.ts 双配置结构

```ts
// script/gqlgen.ts
import { CodegenConfig } from "@graphql-codegen/cli";

// ====== 配置 A：拉取 Schema（--schema-ast 模式） ======
const schemaAstConfig: CodegenConfig = {
  generates: {
    'script/generated/<模块>.graphql': {
      plugins: ['schema-ast'],
      config: { includeDirectives: true },
      schema: {
        '<后端GraphQL端点URL>': {
          headers: {
            "Authorization": `Bearer ${token}`,
            "X-Tenant-ID": `${tid}`,
          }
        },
      }
    },
    // ... 更多模块（按项目实际后端服务添加）
  },
}

// ====== 配置 B：生成客户端类型（默认模式） ======
const config: CodegenConfig = {
  generates: {
    "src/generated/<模块>/": {
      preset: 'client',
      presetConfig: { gqlTagName: 'gql' },
      schema: "script/generated/<模块>.graphql",       // Schema 来源
      documents: "src/services/<模块>/**/*.ts",         // 扫描 gql() 的目录
    },
    // ... 更多模块
  },
  ignoreNoDocuments: true,  // 没有 gql() 的模块不报错
}

// 通过命令行参数切换
export default process.argv.includes('--schema-ast') ? schemaAstConfig : config
```

### 生成的产物结构

每个模块在 `src/generated/<模块>/` 下生成 4 个文件：

```
src/generated/<模块>/
├── graphql.ts           # TypeScript 类型（接口、枚举、输入类型）
├── gql.ts               # 类型安全的 gql() 函数（带重载）
├── fragment-masking.ts  # Fragment 类型遮蔽
└── index.ts             # 统一导出
```

⚠️ **禁止手动修改 `src/generated/` 下的任何文件**，它们由代码生成器维护。

---

## 二、新增接口的标准流程

### Step 1：确认 Schema 是否最新

```bash
# 仅在后端 Schema 有变更时执行
pnpm gqlgen:schema-ast
```

前提：`.env.local` 中需配置 `GQL_SCHEMA_AST_TOKEN`。

### Step 2：在 services 中编写 gql() 操作

在 `src/services/<模块>/` 下的 `.ts` 文件中编写 GraphQL 操作：

```ts
// src/services/<模块>/index.ts（或新建子文件如 detail.ts）

// 1. 从对应生成模块导入 gql 标签
import { gql } from '@/generated/<模块>';

// 2. 从对应生成模块导入 TypeScript 类型
import {
  XxxWhereInput,
  XxxOrder,
  XxxOrderField,
  OrderDirection,
  CreateXxxInput,
  UpdateXxxInput,
} from '@/generated/<模块>/graphql';

// 3. 导入 urql 请求辅助函数
import { KoHeaders, mutation, paging, query } from '@knockout-js/ice-urql/request';

// 4. 如果需要非 default 实例，从当前项目 services/index.ts 导入 instanceName
import { instanceName } from '..';

// ====== 查询定义 ======
const queryXxxList = gql(`
  query <模块><实体>List($first: Int, $orderBy: XxxOrder, $where: XxxWhereInput) {
    xxxs(first: $first, orderBy: $orderBy, where: $where) {
      totalCount
      pageInfo { hasNextPage hasPreviousPage startCursor endCursor }
      edges {
        cursor
        node { id code name state createdAt ... }
      }
    }
  }
`);

// ====== 封装为服务函数 ======
export const getXxxList = async (gather: {
  current?: number;
  pageSize?: number;
  where?: XxxWhereInput;
  orderBy?: XxxOrder;
}) => {
  const result = await paging(queryXxxList, {
    first: gather.pageSize || 20,
    where: gather.where,
    orderBy: gather.orderBy ?? {
      direction: OrderDirection.Desc,
      field: XxxOrderField.CreatedAt,
    },
  }, gather.current || 1, {
    instanceName: instanceName.XXX,     // 非 default 实例时必须指定（从当前项目 instanceName 查找）
    fetchOptions: { headers: KoHeaders.noCache },
  });
  return result.data?.xxxs;
};
```

### Step 3：运行代码生成

```bash
pnpm gqlgen
```

这会自动：
1. 读取 `script/generated/<模块>.graphql` 作为 Schema
2. 扫描 `src/services/<模块>/**/*.ts` 中所有 `gql()` 调用
3. 生成类型安全的代码到 `src/generated/<模块>/`

### Step 4：在页面/组件中导入使用

```ts
// 导入生成的类型
import { Xxx, XxxWhereInput } from '@/generated/<模块>/graphql';

// 导入服务函数
import { getXxxList, mutCreateXxx } from '@/services/<模块>';
```

### Step 5（可选）：监听模式开发

```bash
pnpm gqlgen:watch
```

监听 `src/services/` 下文件变化，自动重新生成。

---

## 三、项目模块映射（适配指南）

> **项目适配**：每个项目的模块映射不同，开发前先查看当前项目的以下文件确认配置：
> - `script/gqlgen.ts` — 查看有哪些模块的 Schema 和 documents 配置
> - `src/services/index.ts` — 查看 `instanceName` 注册了哪些后端实例
> - `script/generated/` — 查看已拉取的 Schema 文件
> - `src/generated/` — 查看已生成的类型目录

### 模块映射关系

每个项目的模块遵循统一的目录结构约定：

```
<模块>.graphql (Schema)          → script/generated/<模块>.graphql
src/services/<模块>/**/*.ts      → gql() 操作定义
src/generated/<模块>/            → 生成的 TypeScript 类型
```

### instanceName 注册表

> **项目适配**：每个项目在 `src/services/index.ts` 中定义自己的 `instanceName`。
> 编写服务函数时，从当前项目的 `instanceName` 中查找对应后端服务。

```ts
// src/services/index.ts — 按项目实际后端服务配置
export const instanceName = {
  // 示例结构，具体值因项目而异
  DEFAULT: 'default',
  // XXX: 'xxx-service',
  // ...
};
```

### 如何确认模块对应的 urql 实例

1. 查看 `src/services/index.ts` 中的 `instanceName` 导出
2. 查看 `app.tsx` 中 `urqlConfig` 的实例注册
3. 如果服务函数不传 `instanceName`，则使用 `default` 实例

---

## 四、urql 实例与 default 的关系

- **每个项目有一个 `default` urql 实例**，指向该项目的主后端端点
- `default` 实例在 `app.tsx` 的 `urqlConfig` 中配置了完整的 token 刷新、租户头、i18n、错误处理
- **其他实例**通常只需配置 `url`，不需要完整 auth 配置
- 调用 `query()`/`paging()`/`mutation()` 时：
  - 如果不传 `instanceName`，使用 `default` 实例
  - 如果传了 `instanceName`，使用对应的实例

```ts
// 使用 default 实例（不传 instanceName）
const result = await query(doc, vars);

// 使用指定实例
const result = await query(doc, vars, {
  instanceName: instanceName.XXX,  // 从当前项目 instanceName 查找
  fetchOptions: { headers: KoHeaders.noCache },
});
```

---

## 五、已有接口检查

> ⚠️ 新增接口前先检查当前项目 `src/services/` 目录中是否已存在对应操作，**不要重复创建已存在的操作**。

检查方法：

```bash
# 扫描当前项目所有 gql() 操作
grep -r "gql(" src/services/ --include="*.ts" -l

# 查看已生成的类型目录
ls src/generated/

# 查看已拉取的 Schema
ls script/generated/
```

---

## 六、gql() 操作命名约定

### 查询命名

```graphql
# 格式：{service}{Entity}{Action}
query userOrderList($first: Int, ...) { ... }
query userConditionOrderList(...) { ... }
query bizTypes($first: Int, ...) { ... }
query tradeTypes($first: Int, ...) { ... }
```

### Mutation 命名

```graphql
# 格式：{service}{Action} 或 {action}{Entity}
mutation userCancelOrder($req: CancelOrderRequest) { ... }
mutation createBizType($input: CreateBizTypeInput!) { ... }
mutation updateBizType($id: ID!, $input: UpdateBizTypeInput!) { ... }
mutation deleteBizType($id: ID!) { ... }
```

### 服务函数命名

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 列表查询 | `get` + 实体 + `List` | `getOrderList`、`getBizTypeList` |
| 详情查询 | `get` + 实体 + `Info` | `getOrderInfo`、`getBizTypeInfo` |
| 创建 | `mut` + `Create` + 实体 | `mutCreateBizType`、`mutNewOrder` |
| 更新 | `mut` + `Update` + 实体 | `mutUpdateBizType`、`mutUpdatePricingConfig` |
| 删除 | `mut` + `Delete` + 实体 | `mutDeleteBizType` |
| 特殊操作 | `mut` + 动作 | `mutCancelOrder`、`mutCommitOrder` |
| 枚举映射 | `Enum` + 实体 + 字段 | `EnumOrderOrdStatus`、`EnumBizTypeBizTypeState` |

---

## 七、三种 urql 辅助函数详解

### paging() — 列表分页查询

```ts
import { paging, KoHeaders } from '@knockout-js/ice-urql/request';

const result = await paging(
  queryDocument,                    // gql() 定义的查询文档
  {
    first: pageSize || 20,          // 每页条数
    where: whereInput,              // 查询条件（生成的 WhereInput 类型）
    orderBy: orderByInput,          // 排序（可选）
  },
  currentPage || 1,                 // 当前页码（从 1 开始）
  {
    instanceName: instanceName.XXX, // urql 实例名（可选，不传用 default）
    fetchOptions: { headers: KoHeaders.noCache },  // 请求头（可选）
  }
);

// 返回结构
result.data?.xxxs  // → { totalCount, pageInfo, edges: [{ cursor, node }] }
```

### query() — 单实体查询

```ts
import { query, KoHeaders } from '@knockout-js/ice-urql/request';
import { gid } from '@knockout-js/api';

const result = await query(
  queryDocument,                    // gql() 定义的查询文档
  { gid: gid('EntityName', id) },  // 变量（GID 格式：`EntityName:id`）
  {
    instanceName: instanceName.XXX,
    fetchOptions: { headers: KoHeaders.noCache },
  }
);

// 类型守卫
if (result.data?.node?.__typename === 'EntityName') {
  return result.data.node;
}
```

### mutation() — 写操作

```ts
import { mutation } from '@knockout-js/ice-urql/request';

const result = await mutation(
  mutationDocument,                 // gql() 定义的 mutation 文档
  { input: inputData },            // 变量
  {
    instanceName: instanceName.XXX,
  }
);

return result.data?.createXxx;     // 或 updateXxx / deleteXxx
```

---

## 八、新增模块的完整步骤

当需要对接一个**全新的后端服务**（当前项目还没有对应的 schema/generated/services）时：

### 1. 在 `script/gqlgen.ts` 中添加配置

```ts
// === schemaAstConfig 中添加 ===
'script/generated/new-service.graphql': {
  plugins: ['schema-ast'],
  config: { includeDirectives: true },
  schema: {
    '<后端GraphQL端点URL>': {
      headers: {
        "Authorization": `Bearer ${token}`,
        "X-Tenant-ID": `${tid}`,
      }
    },
  }
},

// === config 中添加 ===
"src/generated/new-service/": {
  preset: 'client',
  presetConfig: { gqlTagName: 'gql' },
  schema: "script/generated/new-service.graphql",
  documents: "src/services/new-service/**/*.ts",
},
```

### 2. 在 `src/services/index.ts` 中注册 instanceName

```ts
export const instanceName = {
  // ... 已有实例
  NEWSERVICE: 'new-service',  // 新增
};
```

### 3. 创建 service 目录和文件

```bash
mkdir -p src/services/new-service
```

```ts
// src/services/new-service/index.ts
import { gql } from '@/generated/new-service';
// ... 编写 gql() 操作
```

### 4. 拉取 Schema 并生成

```bash
pnpm gqlgen:schema-ast   # 拉取新服务的 Schema
pnpm gqlgen              # 生成类型
```

### 5. 在 `app.tsx` 的 urqlConfig 中注册实例

（如果该服务需要独立的 urql 客户端）

---

## 九、常见错误排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `Cannot find module '@/generated/xxx/graphql'` | 未运行 `pnpm gqlgen` | 运行 `pnpm gqlgen` |
| `gql` 标签报类型错误 | 导入了错误的 gql 来源 | 确保从 `@/generated/<模块>` 导入 gql，不是从其他模块 |
| `Type 'XxxWhereInput' is not defined` | Schema 未更新 | 运行 `pnpm gqlgen:schema-ast` 后重新 `pnpm gqlgen` |
| 新写的 gql() 没生成类型 | 文件不在 documents 扫描路径 | 确认文件在 `src/services/<模块>/**/*.ts` 下 |
| 运行时请求到错误端点 | instanceName 未传或传错 | 检查 `query/paging/mutation` 的 opts.instanceName |
| `ignoreNoDocuments` 报错 | service 目录为空 | 正常，配置了 `ignoreNoDocuments: true` 不会报错 |
| 类型是 `Maybe<T>` 或 `Nullable` | GraphQL Schema 中字段为 nullable | 用可选链 `?.` 和空值合并 `??` 处理 |

---

## 十、枚举映射生成规范

> **核心原则**：每次根据 schema 生成 services 时，**必须同步**生成该 schema 中所有枚举类型的映射，放入 `enums.ts`，不得遗漏。

### 1. 文件格式（跨项目统一 — Object/Record 风格）

```ts
// ❌ 数组风格（仅能用于 Select.options，不能给 ProTable valueEnum）
export const EnumXxxState = [
  { label: '失效', value: -1 },
];

// ✅ Record/Object 风格（ProTable valueEnum + Tag 渲染 + Select 通用）
export const EnumXxxState: Record<number, { text: string; tagColor: string }> = {
  [-1]: { text: '失效', tagColor: 'error' },
  [0]:  { text: '初始', tagColor: 'default' },
  [100]: { text: '启用', tagColor: 'success' },
};
```

### 2. 两种枚举的处理方式

#### (A) GraphQL string 枚举 — 直接引用

```ts
import { RiskKindKindType } from '@/generated/<模块>/graphql';

export const EnumRiskKindKindType: Record<RiskKindKindType, { text: string }> = {
  [RiskKindKindType.Account]:  { text: '账户' },
  [RiskKindKindType.Event]:    { text: '事件' },
  [RiskKindKindType.Product]:  { text: '产品' },
  [RiskKindKindType.Strategy]: { text: '策略' },
};
```

#### (B) Int 型状态枚举（schema 里是 `Int` 但语义上是枚举）— 手动定义

典型场景：`state: Int` 表示 `-1=失效, 0=初始, 100=启用`。

```ts
export const EnumRiskConfigState: Record<number, { text: string; tagColor: string }> = {
  [-1]:  { text: '失效', tagColor: 'error'   },
  [0]:   { text: '初始', tagColor: 'default' },
  [100]: { text: '启用', tagColor: 'success' },
};
```

Int 型枚举的值及含义需要从 schema 注释（`"""状态,-1,失效;0:初始,100:启用"""`）中提取。

### 3. tagColor 配色约定

| 语义 | tagColor |
|------|----------|
| 成功 / 启用 / 生效 / 通过 | `success` |
| 警告 / 待审核 / 处理中 | `warning` |
| 错误 / 失败 / 已拒绝 / 失效 | `error` |
| 默认 / 初始 / 草稿 | `default` |
| 信息 / 进行中 | `processing` |
| 纯展示无状态含义 | 不填 tagColor，仅 `text` |

### 4. 命名约定

**`Enum` + 实体名 + 字段名**（PascalCase）

| Schema 字段 | 枚举映射名 |
|------------|-----------|
| `RiskConfig.state` | `EnumRiskConfigState` |
| `RiskKind.kindType` (GraphQL enum `RiskKindKindType`) | `EnumRiskKindKindType` |
| `RiskStrategy.strategyType` (GraphQL enum `RiskStrategyStrategyType`) | `EnumRiskStrategyStrategyType` |
| `RiskStrategy.approveStatus` (Int) | `EnumRiskStrategyApproveStatus` |

### 5. 文件组织与 re-export

```
src/services/<模块>/
├── index.ts          # 主服务 + re-export 所有 enums
├── enums.ts          # ← 枚举映射集中放这里
├── xxxConfig.ts
├── xxxStrategy.ts
└── ...
```

**index.ts 中必须 re-export 所有 Enum**：

```ts
// —— 枚举映射（UI 下拉 / 表格列 用） ——
export {
  EnumXxxConfigState,
  EnumXxxKindKindType,
  // ... 所有 Enum
} from './enums';
```

### 6. 消费方式

```tsx
// ProTable 列 — 直接传 valueEnum（同时获得搜索下拉 + 表格显示）
{ title: '状态', dataIndex: 'state', valueEnum: EnumXxxConfigState }

// Tag 渲染
<Tag color={EnumXxxConfigState[record.state]?.tagColor}>
  {EnumXxxConfigState[record.state]?.text}
</Tag>

// Select 下拉 — Object.entries 转换
<Select
  options={Object.entries(EnumXxxConfigState).map(([value, { text }]) => ({
    value: Number(value), label: text,
  }))}
/>
```

### 7. 检查步骤

生成 services 后，对照 `src/generated/<模块>/graphql.ts` 中的 `export enum` 列表，**逐个确认**每个 GraphQL enum 都已生成对应的 `Enum*` 映射。同时检查 schema 注释中的 Int 型状态枚举是否也一并处理。

---

## 十一、开发检查清单

新增/修改 GraphQL 接口时逐项确认：

- [ ] 确认目标模块的 schema 文件存在（`script/generated/<模块>.graphql`）
- [ ] 如果 schema 有更新，已运行 `pnpm gqlgen:schema-ast`
- [ ] gql() 操作写在 `src/services/<模块>/**/*.ts` 下
- [ ] gql 标签从正确的 `@/generated/<模块>` 导入
- [ ] 类型从 `@/generated/<模块>/graphql` 导入
- [ ] 操作名遵循命名约定（`{service}{Entity}{Action}`）
- [ ] 已运行 `pnpm gqlgen` 生成类型
- [ ] **没有重复创建已存在的接口**（查 `src/services/` 目录）
- [ ] 非 default 实例的操作传了正确的 `instanceName`
- [ ] 服务函数遵循 `getXxx` / `mutXxx` 命名
- [ ] 枚举映射已写在 `src/services/<模块>/enums.ts` 中（查 §十 规范）
- [ ] 枚举映射已从 `src/services/<模块>/index.ts` 统一 re-export
- [ ] `src/generated/` 下的文件没有被手动修改
