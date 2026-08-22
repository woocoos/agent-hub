---
name: knockout-webui
description: 基于 knockout-js 生态的 React UI 页面开发 Skill — 标准列表页、详情编辑页、弹窗编辑、行内编辑表格、权限控制、只读表单等通用模式。
---

# knockout-webui 开发 Skill — 通用页面开发基线

> 本文档是基于 knockout-js 生态的 React UI 开发的**通用 Skill 参考**
> 适用于所有基于本模板的子应用项目。
> 开发新页面时，请先通读本文档，确保遵循统一的架构约定。

## 关键规则（速记）

1. **三段式导出**：`List` 组件 + `default export`（PageContainer + KeepAlive）+ `pageConfig`
2. **列宽拖拽**：所有 ProTable 必须使用 `useResizableProTable`
3. **操作列分割线**：`<Space split={<Divider type="vertical" className="ql-divider-gray" />} size={0}>`
4. **权限包裹**：所有操作按钮必须用 `<Auth authKey="...">` 包裹
5. **编辑增量 diff**：更新操作必须使用 `updateFormat` 做增量对比
6. **离开提示**：所有表单必须接入 `useLeavePrompt`
7. **只读表单**：用 `readOnly` 不用 `disabled`，清空 `placeholder`
8. **ModalForm onFinish**：必须返回 `false` 阻止自动关闭
9. **Auth Key 来源**：优先使用 GQL mutation 名，无独立接口时自定义前端权限 Key
10. **ID 列**：`order: -999` + 默认隐藏 + 可复制
11. **操作列**：`fixed: 'right'` + `hideInSetting: true`

---

## 一、技术栈与核心依赖

```json
{
  "@ant-design/pro-components": "^2.8.7",   // ProTable / ProForm / ModalForm / PageContainer
  "@knockout-js/layout": "0.1.26",          // KeepAlive / Layout / useLeavePrompt
  "@knockout-js/api": "0.1.17",             // GraphQL 请求封装
  "@knockout-js/ice-urql": "0.1.21",        // ICE + urql 集成
  "antd": "^5.28.0",                        // 基础 UI 组件
  "react-antd-column-resize": "^1.0.3",     // 表格列宽拖拽
  "ice": "3.x",                             // 框架（文件系统路由 + icestark 微前端）
  "dayjs": "x",                             // 日期处理
  "react-i18next": "x"                      // 国际化
}
```

---

## 二、页面类型分类

| 类型 | 名称 | 典型场景 | 核心布局 |
|------|------|----------|----------|
| **A** | 标准列表页 | 基础数据管理、配置项维护 | ProTable + ModalForm 弹窗 |
| **B** | 树形管理页 | 分类树、组织架构 | Splitter（Tree + ProForm） |
| **C** | 多类型列表页 | 同一实体多种子类型 | 路由分发 + 共享列表组件 |
| **D** | 详情编辑页 | 复杂实体编辑（多 Tab） | 独立页面，多 Tab / Section |
| **E** | 属性管理页 | 属性组/属性维护 | 列表 + ProForm 内联编辑 |
| **F** | 子路由钻取页 | 主从层级数据 | 列表 → 子列表（URL 参数传递） |

---

## 三、通用页面外壳（所有类型共用）

### 3.1 三段式导出结构

每个页面 `index.tsx` **必须**包含三段式导出：

```tsx
// ① 业务组件（内部 List）
const List = () => {
  // ... 核心业务逻辑
};

// ② 默认导出（页面外壳）
export default () => {
  const [breadcrumbNames] = routeBreadcrumb();
  return (
    <PageContainer
      className="ql-page-container"
      header={{
        breadcrumb: {
          items: breadcrumbNames.map(item => ({ title: item })),
        },
      }}
    >
      <KeepAlive clearAlive>
        <List />
      </KeepAlive>
    </PageContainer>
  );
};

// ③ 页面配置（路由级权限）
export const pageConfig = definePageConfig(() => ({
  auth: ['/{module}/xxx'],  // 必须与 menu.json 路径一致
}));
```

### 3.2 关键导入清单

```tsx
// --- 框架层 ---
import { definePageConfig } from 'ice';
import { useTranslation } from 'react-i18next';

// --- UI 组件层 ---
import {
  ActionType, PageContainer, ProTable,
  ProForm, ModalForm, ProFormText, ProFormSelect,
  ProFormDigit, ProFormTextArea, ProFormRadio,
  ProFormCheckbox, ProFormDatePicker, ProFormTimePicker,
} from '@ant-design/pro-components';
import { KeepAlive, useLeavePrompt } from '@knockout-js/layout';

// --- 业务工具层 ---
import { routeBreadcrumb, useResizableProTable } from '@/util/hook';
import { onMousedown, saveDataSource, updateFormat } from '@/util';
import Auth, { checkAuth } from '@/components/auth';

// --- Ant Design 基础组件 ---
import {
  Button, Divider, message, Modal, Space, Tag, Typography,
  Col, Form, Input, Row, Alert, Checkbox, Radio,
} from 'antd';

// --- 服务层 ---
import { getXxxList, getXxxInfo, mutCreateXxx, mutUpdateXxx, mutDeleteXxx } from '@/services/{module}';
```

### 3.3 页面外壳变体

| 变体 | 使用场景 | 外壳差异 |
|------|----------|----------|
| **标准型** | A/C/E/F 类型 | `PageContainer > KeepAlive > List` |
| **卡片型** | B 树形管理页 | `PageContainer > ProCard > List`（不需要 KeepAlive） |
| **编辑型** | D 详情编辑页 | `PageContainer(header=hidden) > div.ql-detail-container > ProCard*2` |

---

## 四、Type A：标准列表页（最核心、最普遍的页面类型）

### 4.1 整体架构

```
PageContainer（面包屑 + 标题）
└── KeepAlive（页签缓存）
    └── List（业务组件）
        ├── ProTable（列表 + 搜索 + 工具栏）
        └── Editor（ModalForm 弹窗表单，条件渲染）
```

### 4.2 List 组件状态模型

```tsx
const List = () => {
  const { t } = useTranslation(),
    proTableRef = useRef<ActionType>(),
    [modal, setModal] = useState({
      open: false,       // 是否打开
      title: '',         // 弹窗标题
      id: '',            // 编辑时的记录 ID（空字符串=新建）
      readonly: false,   // 是否只读
    }),
    [dataSource, setDataSource] = useState<EntityType[]>([]),
    [selectedRowKeys, setSelectedRowKeys] = useState<Key[]>([]);

  // 可调整列宽的 ProTable
  const { columns: finalColumns, components, tableWidth } = useResizableProTable<EntityType>(() => ({
    columns: [/* ... */],
  }), [/* 依赖 */]);
};
```

### 4.3 ProTable 标准配置

```tsx
<ProTable<EntityType>
  // --- Ref & 数据源 ---
  actionRef={proTableRef}
  dataSource={dataSource}
  rowKey="id"

  // --- 吸顶（仅有数据时启用）---
  sticky={dataSource.length > 0 ? { offsetHeader: 56 } : undefined}

  // --- 搜索区域 ---
  search={{
    className: 'ql-pro-table-search',
    searchText: `${t('query')}`,
    resetText: `${t('reset')}`,
    labelWidth: 70,
  }}

  // --- 工具栏 ---
  toolbar={{
    actions: [
      <Auth authKey="createXxx">
        <Button type="primary" onClick={() => {
          setModal({ open: true, title: '新建', id: '', readonly: false });
        }}>新建</Button>
      </Auth>,
    ],
  }}

  // --- 横向滚动 & 列宽拖拽 ---
  scroll={{ x: tableWidth }}
  components={components}

  // --- 列定义 ---
  columns={finalColumns}
  columnsState={{ defaultValue: { id: { show: false } } }}

  // --- 数据请求（Relay Connection 模式）---
  request={async (params) => {
    const table = { data: [] as EntityType[], success: true, total: 0 };
    const where: EntityWhereInput = {};
    if (params.name) where.nameContains = params.name;
    if (params.code) where.codeContains = params.code;

    const result = await getXxxList({
      current: params.current,
      pageSize: params.pageSize,
      where,
    });
    table.total = result?.totalCount ?? 0;
    result?.edges?.forEach(edge => {
      if (edge?.node) table.data.push(edge.node);
    });
    setDataSource(table.data);
    setSelectedRowKeys([]);
    return table;
  }}

  // --- 分页 ---
  pagination={{ showSizeChanger: true }}

  // --- 行选择（单选） ---
  rowSelection={{
    type: 'radio',
    hideSelectAll: false,
    selectedRowKeys,
    onChange: (rowKeys) => setSelectedRowKeys(rowKeys),
  }}

  // --- 行点击选中 ---
  onRow={(record) => ({
    onMouseDown: (e) => {
      onMousedown({
        target: e.target as HTMLElement,
        click: () => {
          setSelectedRowKeys(prev =>
            prev.includes(record.id) ? [] : [record.id]
          );
        },
      });
    },
  })}
/>
```

### 4.4 列定义标准模式

#### 列顺序约定

| 位置 | 列类型 | 说明 |
|------|--------|------|
| 1 | **ID 列** | 主键标识，默认隐藏 |
| 2~N-2 | **业务列** | 编码、名称、状态等 |
| N-1 | **占位列** | `{ search: false, hideInSetting: true }` |
| N | **操作列** | `fixed: 'right'` |

#### ID 列（必备，首位）

```tsx
{
  title: 'ID',
  dataIndex: 'id',
  width: 100,
  order: -999,                              // 搜索表单中排最后
  render(_, record) {
    return (
      <Typography.Text
        copyable={{ text: record.id, tooltips: ['复制ID', '已复制'] }}
        onClick={(e) => e.stopPropagation()}
      >
        {record.id}
      </Typography.Text>
    );
  },
},
```

#### 业务列

```tsx
{ title: '编码', dataIndex: 'code', width: 120 },
{ title: '名称', dataIndex: 'name', minWidth: 160 },
{ title: '描述', dataIndex: 'description', width: 200, search: false, ellipsis: true },
{ title: '排序', dataIndex: 'listOrder', width: 100, search: false, align: 'center' },
```

#### 状态列（枚举 Tag）

```tsx
{
  title: '状态',
  dataIndex: 'state',
  valueType: 'select',
  width: 120,
  align: 'center',
  valueEnum: EnumXxxState,
  render(_, record) {
    return (
      <Tag color={EnumXxxState[record.state]?.tagColor}>
        {EnumXxxState[record.state]?.text}
      </Tag>
    );
  },
},
```

#### 日期列

```tsx
{
  title: '新建时间',
  dataIndex: 'createdAt',
  valueType: 'date',
  search: false,
  width: 120,
  renderText: (text) => text ? dayjs(text).format('YYYY-MM-DD') : '-',
},
```

#### 数值列（右对齐）

```tsx
{
  title: '行情价',
  dataIndex: 'last',
  width: 100,
  align: 'right',
  search: false,
  render: (_, record) => record.last ? record.last.toFixed(3) : '-',
},
```

#### 占位列（操作列前，固定写法）

```tsx
{ search: false, hideInSetting: true },
```

### 4.5 操作列配置

```tsx
{
  title: '操作',
  dataIndex: 'actions',
  fixed: 'right',
  search: false,
  align: 'center',
  width: 120,          // 根据按钮数量调整
  hideInSetting: true,
  render(_, record) {
    return (
      <Space split={<Divider type="vertical" className="ql-divider-gray" />} size={0}>
        <Auth authKey="updateXxx">
          <Typography.Link onClick={() => {
            setModal({ open: true, title: `编辑 ${record.name}`, id: record.id, readonly: false });
          }}>编辑</Typography.Link>
        </Auth>
        <Auth authKey="deleteXxx">
          <Typography.Link onClick={() => {
            Modal.confirm({
              title: '删除',
              content: `是否删除：${record.name}？`,
              onOk: async () => {
                return new Promise(async (resolve, reject) => {
                  const result = await mutDeleteXxx(record.id);
                  if (result) {
                    message.success('执行成功');
                    setDataSource(prev => prev.filter(item => item.id !== record.id));
                    resolve(true);
                  } else { reject(); }
                });
              },
            });
          }}>删除</Typography.Link>
        </Auth>
      </Space>
    );
  },
},
```

**操作列宽度参考：**

| 按钮组合 | 建议宽度 |
|---------|---------|
| 编辑 + 删除 | `120` |
| 查看 + 编辑 + 删除 | `180` |
| 查看 + 编辑 + 启用/禁用 + 删除 | `220` ~ `240` |
| 5+ 按钮（含更多操作） | `280` ~ `310` |

**操作按钮间分割线规范：**
- 使用 `<Space split={<Divider type="vertical" className="ql-divider-gray" />} size={0}>`
- `ql-divider-gray` 颜色为 `#bcbec3`（浅灰色竖线）
- `Space` 的 `size={0}`，由 `split` 属性自动插入分割线
- 此模式在项目中出现 50+ 次，是**全局统一的操作列分割线方案**

---

## 五、列宽拖拽处理（useResizableProTable）

### 5.1 Hook 用法

```tsx
import { useResizableProTable } from '@/util/hook';

const { columns: finalColumns, components, tableWidth } = useResizableProTable<EntityType>(() => ({
  columns: [/* 原始列定义 */],
  minWidth?: 100,   // 可选，默认 100
  maxWidth?: number, // 可选
}), [/* 依赖数组 */]);
```

### 5.2 关键行为

- 包装 `react-antd-column-resize` 的 `useAntdColumnResize`
- **自动剥离** `fixed: 'right'` 列的 `onHeaderCell`，防止操作列出现拖拽手柄
- 返回 `tableWidth - 200` 作为 `scroll.x` 的宽度（留出 padding）
- 接受工厂函数 + 依赖数组（类似 `useMemo`/`useEffect`）

### 5.3 ProTable 接线（固定写法）

```tsx
<ProTable
  scroll={{ x: tableWidth }}
  components={components}
  columns={finalColumns}
/>
```

### 5.4 列宽参考值

| 列类型 | 宽度范围 | 说明 |
|--------|---------|------|
| ID | `100` | UUID 标准宽度 |
| 编码 | `120` ~ `200` | |
| 名称 | `140` ~ `220` | 用 `minWidth` 可拉伸 |
| 短文本（符号、区号） | `100` ~ `120` | |
| 状态 | `100` ~ `120` | `align: 'center'` |
| 日期 | `120` ~ `160` | |
| 描述/备注 | `200` | `ellipsis: true` |
| 操作列 | `120` ~ `310` | 按按钮数量调整 |

---

## 六、ModalForm 弹窗编辑处理

### 6.1 弹窗状态管理

```tsx
const [modal, setModal] = useState({
  open: false,
  title: '',
  id: '',
  readonly: false,
});
```

### 6.2 条件渲染

```tsx
{modal.open ? <Editor
  title={modal.title}
  id={modal.id}
  readonly={modal.readonly}
  onClose={async (isSuccess, info) => {
    if (isSuccess && info) {
      setDataSource(saveDataSource(dataSource, info));
    }
    setModal({ open: false, title: '', id: '', readonly: false });
  }}
/> : <></>}
```

### 6.3 Editor 组件标准模板

```tsx
export default (props: {
  title: string;
  onClose: (isSuccess?: boolean, info?: EntityType) => void;
  readonly?: boolean;
  id?: string;
}) => {
  const [form] = Form.useForm(),
    [checkLeave, setLeavePromptWhen] = useLeavePrompt(),
    [info, setInfo] = useState<EntityType>(),
    [saveLoading, setSaveLoading] = useState(false),
    [saveDisabled, setSaveDisabled] = useState(true),
    [loading, setLoading] = useState(false);

  // 离开提示绑定
  useEffect(() => { setLeavePromptWhen(saveDisabled); }, [saveDisabled]);

  // 弹窗关闭处理
  const onOpenChange = (open: boolean) => {
    if (!open) {
      if (checkLeave()) {
        setSaveDisabled(true);
        requestAnimationFrame(() => { props.onClose?.(); });
      }
    } else { setSaveDisabled(true); }
  };

  return (
    <ModalForm<FormData>
      open={true}
      onOpenChange={onOpenChange}
      requiredMark={!props.readonly}
      loading={loading}
      title={props.title}
      width={500}
      form={form}
      modalProps={{ destroyOnHidden: true }}
      submitter={props.readonly ? false : {
        searchConfig: { submitText: '保存', resetText: '取消' },
        submitButtonProps: { loading: saveLoading, disabled: saveDisabled },
      }}
      onValuesChange={() => { setSaveDisabled(false); }}
      request={async () => {
        const result: FormData = {};
        setSaveLoading(false);
        setSaveDisabled(true);
        setLoading(true);
        if (props.id) {
          const infoRes = await getXxxInfo(props.id);
          if (infoRes) {
            result.name = infoRes.name ?? undefined;
            setInfo(infoRes);
          }
        }
        setLoading(false);
        return result;
      }}
      autoFocusFirstInput
      onFinish={async (values: FormData) => {
        setSaveLoading(true);
        if (props.id) {
          // 编辑：必须使用 updateFormat 做增量 diff
          const result = await mutUpdateXxx(props.id, updateFormat<UpdateXxxInput>({
            name: values.name,
          }, info || {}));
          if (result?.id) {
            setSaveDisabled(true);
            props.onClose?.(true, result as EntityType);
            message.success('保存成功');
          }
        } else {
          // 新建
          const result = await mutCreateXxx({ name: values.name ?? '' });
          if (result?.id) {
            setSaveDisabled(true);
            props.onClose?.(true, result as EntityType);
            message.success('保存成功');
          }
        }
        setSaveLoading(false);
        return false;  // 阻止自动关闭
      }}
    >
      <Row gutter={16}>
        <Col span={12}>
          <ProFormText
            name="name"
            label="名称"
            rules={[{ required: true, message: '请填写名称' }]}
            placeholder={props.readonly ? '' : '请输入名称'}
            fieldProps={{ readOnly: props.readonly }}
          />
        </Col>
      </Row>
    </ModalForm>
  );
};
```

### 6.4 弹窗关闭后的数据更新模式

| 模式 | 代码 | 适用场景 |
|------|------|---------|
| `saveDataSource()` | `setDataSource(saveDataSource(dataSource, info))` | 最通用，自动处理新增/更新 |
| 带排序 | `saveDataSource(dataSource, info, { sort: 'DESC', sortField: 'listOrder' })` | 有序列表 |
| 手动处理 | `idx === -1 ? [info, ...prev] : next[idx] = info` | 自定义插入逻辑 |
| `proTableRef.reload()` | `proTableRef.current?.reload()` | 简单但性能稍差 |

---

## 七、详情编辑页布局（Type D）

### 7.1 整体结构

```tsx
<PageContainer header={{ style: { display: 'none' } }}>
  <div className="ql-detail-container">
    {/* ① 头部 ProCard：标题 + 操作按钮 + 信息栏 */}
    <ProCard
      title={
        <div className="ql-detail-title">
          {info?.name ? `${isReadonly ? '' : '编辑-'}实体名称：${info.name}` : '新建-实体名称'}
        </div>
      }
      loading={loading && !info}
      split="horizontal"
      extra={<Space>
        {isReadonly && hasId && (
          <Auth authKey="updateXxx">
            <Button type="primary" onClick={handleEdit}>编辑</Button>
          </Auth>
        )}
      </Space>}
    >
      {hasId && info && (
        <div className="ql-detail-subTitle ql-detail-subTitle-end-tabs">
          ...信息栏内容...
        </div>
      )}
    </ProCard>

    {/* ② 内容区域（根据业务复杂度选择不同模式，见 7.1.1） */}
    {loading ? <></> : <ContentArea ... />}
  </div>
</PageContainer>
```

**层级关系：**
```
PageContainer（header 隐藏）
└── div.ql-detail-container
    ├── ProCard（头部）
    │   ├── title: div.ql-detail-title（动态标题）
    │   ├── extra: Space（操作按钮组）
    │   └── body: div.ql-detail-subTitle（信息栏，仅 hasId 时显示）
    └── 内容区域（loading 时不渲染）
```

#### 7.1.1 内容区域的三种模式

**模式一：ProCard Tab 页签模式（最常用，适合多板块复杂实体）**

```tsx
<ProCard size="small" tabs={{
  className: 'ql-detail-tabs',
  size: 'small',
  items: [
    { key: 'basic', label: '基本信息', children: <BasicInfo ... /> },
    { key: 'detail', label: '详细资料', disabled: !hasId, children: hasId ? <Detail ... /> : null },
  ],
}} />
```

**模式二：ProCard 直出模式（适合内容简单、无需分 Tab 的实体）**

```tsx
<ProCard size="small">
  <BasicInfo info={info} readonly={isReadonly} ... />
</ProCard>
```

**模式三：自定义布局模式（适合特殊交互需求）**

根据业务需要自由组合，如 Splitter 左右分栏、多 ProCard 纵向排列、内嵌表格等：

```tsx
{/* 示例：左右分栏 */}
<Splitter>
  <Splitter.Panel defaultSize={400}><LeftContent /></Splitter.Panel>
  <Splitter.Panel><RightContent /></Splitter.Panel>
</Splitter>

{/* 示例：多 Card 纵向排列 */}
<Card size="small" style={{ marginBottom: 16 }}><SectionA /></Card>
<Card size="small" style={{ marginBottom: 16 }}><SectionB /></Card>
```

### 7.2 头部操作按钮（extra 区域）

ProCard 的 `extra` 区域放置操作按钮，按**只读/编辑两种模式**分组显示：

```tsx
extra={<Space>
  {/* 只读模式：显示"编辑"按钮 → 移除 URL 中的 readonly 参数 */}
  {isReadonly && hasId && (
    <Auth authKey="updateXxx">
      <Button type="primary" onClick={() => {
        const params = new URLSearchParams(searchParams);
        params.delete('readonly');
        setSearchParams(params);
      }}>
        编辑
      </Button>
    </Auth>
  )}

  {/* 编辑模式：显示业务操作按钮（按实体需求添加） */}
  {!isReadonly && hasId && (
    <>
      <Button onClick={handleToggleState}>
        {info?.state === 1 ? '禁用' : '启用'}
      </Button>
      {/* 其他操作按钮... */}
    </>
  )}
</Space>}
```

**按钮显示规则：**

| 条件 | 含义 | 展示按钮 |
|------|------|---------|
| `isReadonly && hasId` | 只读查看已有记录 | **编辑**（进入编辑模式） |
| `!isReadonly && hasId` | 编辑已有记录 | 启用/禁用、业务操作等 |
| `!isReadonly && !hasId` | 新建记录 | 通常无额外按钮（保存由 Tab 内表单处理） |

> **核心逻辑：** "编辑"按钮通过 `params.delete('readonly')` 切换 URL 参数实现只读→编辑的模式切换，而非跳转到新页面。

### 7.3 页面标题规范

详情页有**两处标题**需要保持一致：ProCard 内的 `ql-detail-title` 和浏览器标签页的 `document.title`。

#### 标题文本规则

| 状态 | 格式 | 示例 |
|------|------|------|
| 新建 | `新建-{实体类型}` | `新建-产品信息` |
| 编辑 | `编辑-{实体类型}：{名称/编码}` | `编辑-产品信息：AAPL` |
| 只读查看 | `{实体类型}：{名称/编码}`（无"编辑-"前缀） | `产品信息：AAPL` |

#### ProCard title（`ql-detail-title`）

```tsx
<ProCard
  title={
    <div className="ql-detail-title">
      {info?.name
        ? `${isReadonly ? '' : '编辑-'}产品信息：${info.name}`
        : '新建-产品信息'}
    </div>
  }
>
```

#### document.title（浏览器标签页标题）

通过 `useEffect` 同步设置，文本内容与 `ql-detail-title` 保持一致：

```tsx
useEffect(() => {
  document.title = info?.name
    ? `${isReadonly ? '' : '编辑-'}产品信息：${info.name}`
    : '新建-产品信息';
}, [info, isReadonly]);
```

> **注意：** 部分实体使用 `code`（编码）而非 `name`（名称）作为标题展示字段，如标的资料页使用 `info.code`。根据业务场景选择合适的标识字段。

### 7.4 URL 参数驱动

```tsx
const [searchParams, setSearchParams] = useSearchParams();
const id = searchParams.get('id') ?? '';
const isReadonly = searchParams.get('readonly') === 'true';
```

### 7.5 信息栏模板

```tsx
{hasId && info && (
  <div className="ql-detail-subTitle ql-detail-subTitle-end-tabs">
    <Typography.Text>
      <span className="label">ID：</span>{info.id}
    </Typography.Text>
    <Divider type="vertical" className="ql-divider-gray" />
    <Typography.Text>
      <span className="label">状态：</span>{stateEnum?.text ?? '-'}
    </Typography.Text>
    <Divider type="vertical" className="ql-divider-gray" />
    <Typography.Text>
      <span className="label">创建人：</span>{createUser?.displayName ?? info.createdBy ?? '-'}
    </Typography.Text>
    <Divider type="vertical" className="ql-divider-gray" />
    <Typography.Text>
      <span className="label">创建时间：</span>
      {info.createdAt ? dayjs(info.createdAt).format('YYYY-MM-DD HH:mm:ss') : '-'}
    </Typography.Text>
  </div>
)}
```

**信息栏规范：**
- 字段顺序：`ID` → `状态` → `创建人` → `创建时间`
- 标签颜色：`#686a8f`（`.label` 类）
- 分隔符：`<Divider type="vertical" className="ql-divider-gray" />`
- 空值兜底：`?? '-'`
- 日期格式：`YYYY-MM-DD HH:mm:ss`

### 7.6 Tab 配置

```tsx
const tabItems = [
  { key: 'basic', label: '基本信息', children: <BasicInfo ... /> },
  { key: 'detail', label: '详细资料', disabled: !hasId,
    children: hasId ? <DetailComponent ... /> : null },
];
```

**规则：** 新建时第一个 Tab 可编辑，其余 Tab `disabled: !hasId`，`children` 设为 `null`。

### 7.7 表单区块间隔

```tsx
<div style={{ fontSize: 14, fontWeight: 500, marginBottom: 16 }}>标识信息</div>
<Row gutter={24}>
  ...表单字段...
</Row>

<div style={{ fontSize: 14, fontWeight: 500, marginBottom: 16, marginTop: 24 }}>其他信息</div>
<Row gutter={24}>
  ...表单字段...
</Row>
```

**间距规范：**
- 区块标题：`fontSize: 14, fontWeight: 500`
- 标题下方：`marginBottom: 16`
- 非首区块上方：`marginTop: 24`
- Row gutter：编辑页 `24`，弹窗 `16`，复杂表单 `20`

**Col span 规范：**
- 4 列布局（编辑页）：`span={6}`
- 2 列布局（弹窗）：`span={12}`
- 全宽（TextArea）：`span={24}`

---

## 八、EditableProTable 行内编辑表格

在详情页或弹窗中，经常需要对子表数据进行行内编辑（如产品的关联标的、规则参数列表等）。使用 `EditableProTable` 实现行内编辑，配合 `value/onChange` 接口作为表单字段直接使用。

### 8.1 组件接口（受控表单字段模式）

```tsx
export default (props: {
  value?: RowType[];
  readonly?: boolean;
  onChange?: (value?: RowType[]) => void;
}) => {
  // ...
};
```

> 遵循 antd `Form.Item` 的 `value/onChange` 约定，可直接嵌入 ProForm 中作为表单字段。

### 8.2 核心状态

```tsx
const [editableKeys, setEditableRowKeys] = useState<Key[]>([]);
```

### 8.3 列定义要点

```tsx
const columns = [
  // 选择列（不可编辑，仅展示）
  {
    title: '产品',
    dataIndex: 'productId',
    valueType: 'select',
    editable: false,                        // 该行不可编辑
    valueEnum: prodEnum,                    // 用于展示映射
  },

  // 数字列（带格式化）
  {
    title: '最大数量',
    dataIndex: 'maxQty',
    valueType: 'digit',
    fieldProps: {
      formatter: (v: number) => v?.toLocaleString(),
      parser: (v: string) => v?.replace(/,/g, ''),
    },
  },

  // Checkbox 列
  {
    title: '允许拆分',
    dataIndex: 'allowSplit',
    valueType: 'checkbox',
    formItemProps: { valuePropName: 'checked' },
    renderFormItem: () => <Checkbox>允许</Checkbox>,
    render: (_, record) => record.allowSplit ? '允许' : '不允许',
  },

  // 占位列
  { search: false, hideInSetting: true, editable: false },

  // 操作列（仅编辑模式显示）
  ...(!props.readonly ? [{
    title: '操作',
    dataIndex: 'action',
    valueType: 'option',
    width: 120,
    align: 'center',
    render: (_, record, __, action) => (
      <Space split={<Divider type="vertical" className="ql-divider-gray" />} size={0}>
        <Typography.Link onClick={() => action?.startEditable?.(record.id)}>
          编辑
        </Typography.Link>
        <Popconfirm title="是否删除？" onConfirm={() => {
          props.onChange?.(props.value?.filter(item => item.id !== record.id));
        }}>
          <Typography.Link>删除</Typography.Link>
        </Popconfirm>
      </Space>
    ),
  }] : []),
];
```

### 8.4 EditableProTable 配置

```tsx
<EditableProTable<RowType>
  rowKey="id"
  columns={columns}
  value={props.value}
  onChange={props.onChange}
  controlled                                // 受控模式
  recordCreatorProps={false}                // 禁用内置新增按钮（使用自定义方式）
  scroll={{ x: tableWidth }}
  components={components}
  editable={{
    type: 'single',                         // 单行编辑（同时只能编辑一行）
    editableKeys,
    onChange: setEditableRowKeys,

    // 保存回调：将编辑后的数据通过 onChange 传回父级
    onSave: async (rowKey, data) => {
      props.onChange?.(props.value?.map(item =>
        item.id === data.id ? { ...item, ...data } : item
      ));
    },

    // 编辑态操作按钮样式（保存 | 取消，与全局分割线风格一致）
    actionRender: (row, config, defaultDom) => [
      <Space split={<Divider type="vertical" className="ql-divider-gray" />} size={0}>
        {defaultDom.save}
        {defaultDom.cancel}
      </Space>,
    ],
  }}
/>
```

### 8.5 自定义新增行

禁用内置 `recordCreatorProps`，通过工具栏按钮（Dropdown、Button 等）自定义新增：

```tsx
// 方式一：Dropdown 选择后新增
<Dropdown menu={{
  items: availableItems.map(item => ({ label: item.name, key: `${item.id}` })),
  onClick: ({ key }) => {
    const v = [...(props.value ?? [])];
    v.push({ id: key, /* 初始值 */ });
    props.onChange?.(v);
    setEditableRowKeys([key]);             // 新增后立即进入编辑态
  },
}}>
  <Button>添加<DownOutlined /></Button>
</Dropdown>

// 方式二：普通按钮新增
<Button onClick={() => {
  const newId = `temp_${Date.now()}`;
  props.onChange?.([...(props.value ?? []), { id: newId }]);
  setEditableRowKeys([newId]);             // 新增后立即进入编辑态
}}>
  新增
</Button>
```

> **关键：** 新增后立刻调用 `setEditableRowKeys([newKey])` 让用户马上填写字段。

### 8.6 只读模式处理

```tsx
// 只读时：操作列不渲染、新增按钮隐藏、editable 不配置
<EditableProTable<RowType>
  columns={columns}                         // columns 中已按 readonly 过滤操作列
  value={props.value}
  {...(!props.readonly && {
    editable: { editableKeys, onChange: setEditableRowKeys, onSave, actionRender },
  })}
/>
```

### 8.7 关键规范总结

| 规范 | 说明 |
|------|------|
| `controlled` | 必须开启受控模式，数据由 `value/onChange` 驱动 |
| `type: 'single'` | 推荐单行编辑，避免多行同时编辑的混乱 |
| `actionRender` | 用 `Space` + `Divider` 包裹 `defaultDom.save/cancel`，与全局操作列风格统一 |
| `recordCreatorProps={false}` | 通常禁用内置新增，改用自定义按钮/Dropdown |
| 新增后 `setEditableRowKeys` | 新增行立即进入编辑态 |
| 去重过滤 | 自定义新增时过滤已有项，防止重复 |
| 操作列条件渲染 | `readonly` 时不渲染操作列 |

---

## 九、权限系统（Auth）

### 9.1 三层权限架构

| 层级 | 机制 | 用途 |
|------|------|------|
| **路由级** | `definePageConfig({ auth: [...] })` | 整个页面权限门控 |
| **按钮级** | `<Auth authKey="...">` 组件 | 条件渲染 UI 元素 |
| **编程式** | `checkAuth("...")` 函数 | 基于权限分支逻辑 |

### 9.2 Auth 组件用法

```tsx
import Auth, { checkAuth } from '@/components/auth';

// 按钮级权限
<Auth authKey="createXxx">
  <Button>新建</Button>
</Auth>

// 多 Key（AND 逻辑，默认）
<Auth authKey={['permA', 'permB']} keyAndOr="and">
  <Button>操作</Button>
</Auth>

// 多 Key（OR 逻辑）
<Auth authKey={['permA', 'permB']} keyAndOr="or">
  <Button>操作</Button>
</Auth>

// 带 fallback
<Auth authKey="updateXxx" fallback={<span>无权限</span>}>
  <Button>编辑</Button>
</Auth>

// 编程式检查
const canMove = checkAuth('moveXxx');
```

### 9.3 Auth Key 获取规则

Auth Key **不是固定的命名模式**，而是分两种来源：

#### 来源一：GQL 真实接口名（优先）

当后端 GraphQL 接口有独立的操作 mutation 时，Auth Key 直接对应 mutation 名称：

| GQL Mutation | Auth Key | 说明 |
|---|---|---|
| `createCurrency` | `createCurrency` | 新建接口 |
| `updateCurrency` | `updateCurrency` | 更新接口 |
| `deleteCurrency` | `deleteCurrency` | 删除接口 |
| `moveCategory` | `moveCategory` | 移动接口 |

#### 来源二：自定义前端权限 Key

当某个操作**没有独立的 GQL mutation**（如"启用/禁用"复用 update 接口、"导出"无后端接口、"审核"在同一个 mutation 中处理）时，由前端自定义权限 Key：

| 场景 | 自定义 Key | 说明 |
|---|---|---|
| 启用/禁用（复用 update） | `toggleXxxState` 或 `updateXxxState` | 无独立 mutation |
| 导出功能 | `exportXxx` | 纯前端操作 |
| 审核操作 | `approveXxx` / `rejectXxx` | 合并在 update 中 |
| 批量操作 | `batchDeleteXxx` / `batchUpdateXxx` | 可能无独立 mutation |

**格式：** camelCase，动词 + 实体名（首字母大写）。

**判断规则：**
1. 先看 GQL schema 中是否有对应的独立 mutation → 有则直接用 mutation 名
2. 无独立 mutation 时 → 自定义 Key，并在后端权限系统中同步注册

### 9.4 权限初始化流程

```
App 启动 → defineAuthConfig()
  ├── 开发环境：getMenuAppActions() → 读取 menu.json 授予所有路由权限
  └── 生产环境：userPermissions(APP_CODE, headers) → 服务端返回用户权限列表
      → 合并到 initialAuth（Record<string, boolean>）
```

### 9.5 开发环境自动全权限

```tsx
// src/util/index.ts — getMenuAppActions()
if (process.env.NODE_ENV === 'development') {
  menuJsonList?.forEach((item) => {
    if (item.path) initialAuth[item.path] = true;
    // 递归处理子菜单
  });
}

// src/components/auth/index.tsx — checkAuth()
export const checkAuth = (authKey: string) => {
  return NODE_ENV === 'development' || auth[authKey];
};
```

### 9.6 权限在组件中的使用位置

| 位置 | 示例 |
|------|------|
| 工具栏新建按钮 | `<Auth authKey="createXxx"><Button>新建</Button></Auth>` |
| 操作列编辑按钮 | `<Auth authKey="updateXxx"><Typography.Link>编辑</Typography.Link></Auth>` |
| 操作列删除按钮 | `<Auth authKey="deleteXxx"><Typography.Link>删除</Typography.Link></Auth>` |
| 操作列启用/禁用 | `<Auth authKey="updateXxx"><Typography.Link>启用/禁用</Typography.Link></Auth>` |
| 详情页编辑按钮 | `<Auth authKey="updateXxx"><Button>编辑</Button></Auth>` |
| 树形拖拽开关 | `const canMove = checkAuth('moveXxx');` |

---

## 十、只读表单处理规范

### 10.1 核心原则

1. **不使用 `disabled`** — disabled 导致输入框变灰、无法选中复制文本
2. **使用 `readOnly`** — 保留外观、允许文本选中
3. **`requiredMark={!readonly}`** — 只读模式隐藏必填星号
4. **`submitter={readonly ? false : {...}}`** — 只读模式隐藏提交按钮
5. **`placeholder={readonly ? '' : '请输入...'}`** — 只读模式清空提示文本

### 10.2 三类处理方式

| 组件类型 | 是否支持 readOnly | 处理方式 |
|---------|-------------------|---------|
| `ProFormText` | ✅ | `fieldProps={{ readOnly: readonly }}` |
| `ProFormDigit` | ✅ | `fieldProps={{ readOnly: readonly }}` |
| `ProFormTextArea` | ✅ | `fieldProps={{ readOnly: readonly }}` |
| `InputNumber` | ✅ | `readOnly={readonly}` |
| `ProFormSelect` | ❌ | 条件渲染：`<Form.Item>` + `<Input readOnly>` 展示 label |
| `ProFormDatePicker` | ❌ | 条件渲染：`<Form.Item>` + `<Input readOnly>` 展示格式化日期 |
| `TreeSelect` | ❌ | 条件渲染：递归查找 title |
| `ProFormRadio.Group` | ❌ | 条件渲染或 disabled Radio.Group |
| `Switch` / `Checkbox` | ❌ | 条件渲染：`<Form.Item>` + `<Input readOnly>` 或 disabled 组件 |

### 10.3 模式一：支持 readOnly 的组件

```tsx
<ProFormText
  name="name"
  label="名称"
  placeholder={props.readonly ? '' : '请输入名称'}
  fieldProps={{ readOnly: props.readonly }}
/>

<ProFormDigit
  name="multiplier"
  label="乘数"
  placeholder={props.readonly ? '' : '请输入乘数'}
  fieldProps={{ precision: 0, readOnly: props.readonly }}
/>
```

### 10.4 模式二：Select 单选只读

```tsx
{props.readonly ? (
  <Form.Item label="状态">
    <Input value={EnumXxxState[info?.state ?? 0]?.text ?? ''} readOnly />
  </Form.Item>
) : (
  <ProFormSelect label="状态" name="state" options={...} />
)}
```

### 10.5 模式二：Select 多选只读（Tag 展示）

```tsx
{props.readonly ? (
  <Form.Item label="业务类型">
    <div className="ql-readonly-tags">
      {info?.bizTypes?.split(',').filter(Boolean)
        .map(v => bizTypes.find(b => `+${b.id}` === v)?.bizName ?? v)
        .map((name, index) => (
          <Tag key={index} style={{ margin: 0 }}>{name}</Tag>
        ))}
    </div>
  </Form.Item>
) : (
  <ProFormSelect name="bizTypes" label="业务类型" mode="multiple" options={...} />
)}
```

> **`ql-readonly-tags`** 全局样式：`min-height: 32px`、`border: 1px solid #d9d9d9`、`border-radius: 6px`、`display: flex; flex-wrap: wrap; gap: 4px`，hover 时边框变 `#1677ff`。

### 10.6 模式二：DatePicker 只读

```tsx
{props.readonly ? (
  <Form.Item label="发布日期">
    <Input value={info?.pubTime ? dayjs(info.pubTime).utc().format('YYYY-MM-DD') : ''} readOnly />
  </Form.Item>
) : (
  <DatePicker format="YYYY-MM-DD" style={{ width: '100%' }} />
)}
```

**规则：** 没值用 `''` 不用 `'-'`；格式必须与 DatePicker 的 `format` prop 一致。

### 10.7 模式三：Checkbox / Radio 只读

```tsx
// Checkbox 只读
{props.readonly ? (
  <Form.Item name="triggerAlarm">
    <Checkbox checked={info?.triggerAlarm ?? undefined}>预警性规则</Checkbox>
  </Form.Item>
) : (
  <Form.Item name="triggerAlarm" valuePropName="checked">
    <Checkbox>预警性规则</Checkbox>
  </Form.Item>
)}
```

> 此模式使用 `disabled` 而非 `readOnly`，因为 Checkbox/Radio 原生不支持 `readOnly`。

---

## 十一、枚举与状态 Tag 规范

### 11.1 枚举定义位置

`src/services/{module}/enums.ts`，按业务模块分文件。

### 11.2 枚举命名规则

| 类型 | 命名模式 | 示例 |
|------|---------|------|
| GraphQL 字符串枚举 | `Enum` + 实体名 + GraphQL枚举名 | `EnumBizTypeState` |
| Int 型状态枚举 | `Enum` + 实体名 + `State` | `EnumExchangeState` |
| 审核状态枚举 | `Enum` + 实体名 + `ApproveStatus` | `EnumProductApproveStatus` |
| Y/N 字符串枚举 | `Enum` + 实体名 + `IsEnabled` | `EnumRecordIsEnabled` |

### 11.3 枚举结构

```tsx
// 统一结构：Record<KeyType, { ...自定义字段 }>
// KeyType 根据实际数据类型决定（number / string / enum）
// 字段按需扩展，text 和 tagColor 是最常用的两个，可追加 description、icon、color 等任意字段
export const EnumXxxState: Record<number, { text: string; tagColor: string }> = {
  0: { text: '禁用', tagColor: '#ff3030' },
  1: { text: '启用', tagColor: '#30c880' },
};
```

### 11.4 颜色体系

| 语义 | 色值 | 适用场景 |
|------|------|---------|
| **启用 / 成功 / 通过** | `#30c880` | 启用状态、审核通过 |
| **禁用 / 失败 / 拒绝** | `#ff3030` | 禁用状态、禁止交易 |
| **初始 / 草稿 / 中性** | `#dcdee1` | 初始状态、未知状态 |
| **待审核 / 警告** | `#ffcc5f` | 待审核、停牌 |
| **信息 / 进行中** | `#00C8FF` | 进行中状态 |
| **强调 / 主要** | `#3080FF` | 高优先级、特殊标记 |

### 11.5 安全渲染

```tsx
render(_, record) {
  const stateVal = record.state ?? 0;        // 空值兜底
  const enumItem = EnumXxxState[stateVal];
  return <Tag color={enumItem?.tagColor}>{enumItem?.text ?? '-'}</Tag>;
},
```

**规则：** `?.` 安全访问 + `?? '-'` 文本兜底，防止枚举值缺失导致白屏。

---

## 十二、共享工具函数与 Hooks

### 12.1 Hooks

| Hook | 来源 | 用途 |
|------|------|------|
| `routeBreadcrumb()` | `@/util/hook` | 根据路由自动生成面包屑名称数组 |
| `useResizableProTable(config, deps)` | `@/util/hook` | 为 ProTable 增加列宽拖拽能力 |
| `useLeavePrompt()` | `@knockout-js/layout` | 返回 `[checkLeave, setLeavePromptWhen]`，表单未保存离开拦截 |

### 12.2 工具函数

| 函数 | 来源 | 用途 |
|------|------|------|
| `onMousedown({ target, click })` | `@/util` | 区分单击/双击/文本选择的鼠标事件 |
| `saveDataSource(dataSource, item, options?)` | `@/util` | 新增/更新数据源中的记录 |
| `delDataSource(dataSource, id)` | `@/util` | 从数据源中删除记录 |
| `updateFormat<T>(newValues, oldValues)` | `@/util` | 增量 diff，生成 `clearXxx: true` |
| `formatTreeData(items, rootId?, options?)` | `@/util` | 扁平数组 → 树结构 |
| `delTreeData(tree, key, options?)` | `@/util` | 从树中删除节点 |
| `getTreeDropData(treeData, info)` | `@/util` | 计算拖拽后的新树结构 |
| `exportExcel(filename, sheetData)` | `@/util/excel` | 导出 Excel |

### 12.3 updateFormat 使用规范

```tsx
// 编辑时，必须使用 updateFormat 做增量 diff
// 只发送实际变更的字段，清除的字段自动生成 clearXxx: true
const result = await mutUpdateXxx(id, updateFormat<UpdateXxxInput>({
  name: values.name,
  description: values.description,
}, info || {}));

// 示例输出：{ name: '新名称', clearDescription: true }
```

---

## 十三、数据层约定（GraphQL + Relay）

### 13.1 服务函数命名规范

| 操作 | 函数名 | 返回值 |
|------|--------|--------|
| 分页列表 | `getXxxList({ current, pageSize, where, orderBy? })` | Relay Connection |
| 单条查询 | `getXxxInfo(id)` | 实体对象 |
| 新建 | `mutCreateXxx(input)` | 新建后的实体 |
| 更新 | `mutUpdateXxx(id, updateInput)` | 更新后的实体 |
| 删除 | `mutDeleteXxx(id)` | boolean |
| 字典查询 | `getDictionaryValuesByTypeCodes(codes)` | DictionaryValue[] |

### 13.2 Relay Connection 转换

```tsx
const result = await getXxxList({ current, pageSize, where });
const table = { data: [] as EntityType[], success: true, total: 0 };
table.total = result?.totalCount ?? 0;
result?.edges?.forEach(edge => {
  if (edge?.node) table.data.push(edge.node);
});
return table;
```

### 13.3 字典数据加载

```tsx
const typeCodes = ['TypeA', 'TypeB'];
const dictRes = await getDictionaryValuesByTypeCodes(typeCodes);
const dictionary: Record<string, DictionaryValue[]> = {};
typeCodes.forEach(code => { dictionary[code] = []; });
dictRes.forEach(item => {
  if (item?.typeCode) dictionary[item.typeCode].push(item);
});
for (const key in dictionary) {
  dictionary[key] = dictionary[key].sort((a, b) =>
    (a.listOrder ?? 0) - (b.listOrder ?? 0)
  );
}
```

---

## 十四、启用/禁用切换操作模板

```tsx
<Auth authKey="updateXxx">
  <Typography.Link onClick={() => {
    const isEnabled = record.state === 1;
    Modal.confirm({
      title: isEnabled ? '禁用' : '启用',
      content: `是否${isEnabled ? '禁用' : '启用'}：${record.name}？`,
      onOk: async () => {
        return new Promise(async (resolve, reject) => {
          const result = await mutUpdateXxx(record.id, {
            state: isEnabled ? 0 : 1,
          });
          if (result) {
            message.success('执行成功');
            setDataSource(prev => prev.map(item =>
              item.id === record.id ? { ...item, state: result.state } : item
            ));
            resolve(true);
          } else { reject(); }
        });
      },
    });
  }}>
    {record.state === 1 ? '禁用' : '启用'}
  </Typography.Link>
</Auth>
```

---

## 十五、CSS 类名约定

| 类名 | 用途 |
|------|------|
| `ql-page-container` | PageContainer 标准样式 |
| `ql-pro-table-search` | ProTable 搜索区域样式 |
| `ql-divider-gray` | 操作列灰色分割线（`#bcbec3`） |
| `ql-detail-container` | 编辑页容器 |
| `ql-detail-subTitle` | 详情页信息栏 |
| `ql-detail-subTitle-end-tabs` | 信息栏 + Tab 模式 |
| `ql-detail-tabs` | 详情页 Tab 导航 |
| `ql-readonly-tags` | 只读多选标签展示 |
| `ql-detail-title` | 编辑页标题（18px） |

---

## 十六、尺寸基线

| 项目 | 值 |
|------|-----|
| ProTable 吸顶偏移 | `offsetHeader: 56` |
| Splitter 高度 | `calc(100vh - 120px)` |
| Splitter 左面板宽度 | `380px`（min: 280, max: 500） |
| ModalForm 宽度 | `500 / 600 / 800` |
| Row gutter（编辑页） | `24` |
| Row gutter（弹窗） | `16` |
| Row gutter（复杂表单） | `20` |
| 表单区 padding | `24px 32px` |
| 页面边框色 | `#eeeef1` |
| 标签文字色 | `#686a8f` |

---

## 十七、开发检查清单

### 新建 Type A 列表页

- [ ] 创建 `src/pages/{module}/xxx/index.tsx`：三段式导出
- [ ] 创建 `src/pages/{module}/xxx/components/editor.tsx`：ModalForm 编辑器
- [ ] 在 `services/{module}/` 中定义 CRUD 函数
- [ ] 在 `services/{module}/enums.ts` 中定义枚举
- [ ] 配置 `menu.json` 菜单路径
- [ ] 定义权限 Key（createXxx / updateXxx / deleteXxx）

### 关键规则（必须遵守）

1. **所有编辑操作必须使用 `updateFormat`** 做增量 diff
2. **所有列表必须使用 `useResizableProTable`** 支持列宽拖拽
3. **所有表单必须接入 `useLeavePrompt`** 防止误关
4. **所有权限操作必须包裹 `<Auth>` 组件**
5. **ID 列：** `order: -999` + `columnsState: { show: false }` + 可复制
6. **操作列：** `fixed: 'right'` + `hideInSetting: true` + `align: 'center'`
7. **操作按钮分割线：** `<Space split={<Divider type="vertical" className="ql-divider-gray" />} size={0}>`
8. **删除操作：** `Modal.confirm` + Promise 模式
9. **ModalForm `onFinish`：** 必须返回 `false` 阻止自动关闭
10. **只读表单：** 用 `readOnly` 不用 `disabled`，清空 `placeholder`
11. **状态列：** `valueEnum` + `Tag` 渲染，安全访问 `?.` + 兜底 `?? '-'`
12. **数值列：** `align: 'right'`
13. **搜索表单：** `className: 'ql-pro-table-search'`
14. **行选择：** `type: 'radio'` + `onMousedown` 区分点击与文本选择
