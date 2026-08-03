# 页面快速开发 Skill — 多项目通用模式

> 适用项目：所有基于 knockout-js 生态的 React Web 项目
> 技术栈：React 18 + ICE.js 3 + TypeScript + Ant Design 5 + ProComponents 2.8 + urql + GraphQL CodeGen + icestark 微前端
>
> **自动调用说明**：本 skill 不绑定具体项目，适用于任何使用上述技术栈的项目。
> 使用时自动按当前项目上下文适配 `instanceName`、权限 key 风格、模块路径等配置。

---

## 使用方式

开发新页面时，按以下步骤查阅本 skill：
1. 确定页面类型（列表页 / 详情页 / 表单页 / 树形管理页）
2. 复制对应的模板代码
3. 按服务层规范编写 GraphQL 服务函数
4. 运行 `pnpm gqlgen` 生成类型
5. 检查权限、国际化、枚举映射是否到位

---

## 一、统一技术栈速查

| 分类 | 技术 | 说明 |
|------|------|------|
| 框架 | ICE.js 3 (飞冰) | 文件约定式路由、插件体系 |
| UI | Ant Design 5 + ProComponents | ProTable / ModalForm / PageContainer / ProCard |
| 语言 | TypeScript 4.9 | `@/` 别名 → src/ |
| 数据层 | urql (via @knockout-js/ice-urql) | paging / query / mutation 三件套 |
| 代码生成 | graphql-codegen (client-preset) | 生成类型到 src/generated/ |
| 状态管理 | @ice/plugin-store (ICE Store) | models/ 下的 reducers + effects |
| 微前端 | icestark 子应用 | @knockout-js/layout 提供 KeepAlive |
| 国际化 | i18next + react-i18next | t('key') 调用 |
| 日期 | dayjs | |
| 包管理 | pnpm 8.6 | 禁止 npm/yarn |

### 内部共享库

| 库 | 用途 |
|----|------|
| `@knockout-js/api` | 认证/权限/GID 工具/userPermissions/gid() |
| `@knockout-js/layout` | ProLayout 布局 + KeepAlive + useLeavePrompt + CollectProviders |
| `@knockout-js/org` | 组织权限/AppSelect/UserSelect |
| `@knockout-js/ice-urql` | urql 多实例管理/paging/query/mutation/KoHeaders |

---

## 二、标准页面结构（三段式模板）

每个页面文件必须包含三个部分：

```tsx
import { definePageConfig } from 'ice';
import { PageContainer, useToken } from '@ant-design/pro-components';
import { KeepAlive } from '@knockout-js/layout';
import { routeBreadcrumb } from '@/util/hook';
import { useTranslation } from 'react-i18next';

// ============ 1. 内部业务组件（命名导出） ============
export const ListXxx = (props: { toolbarTitle?: string }) => {
  const { t } = useTranslation();
  // ... 业务逻辑
  return <ProTable ... />;
};

// ============ 2. 默认导出 — 页面包装器 ============
export default () => {
  const { token } = useToken();
  const [breadcrumbNames] = routeBreadcrumb();

  return (
    <PageContainer
      className="qeelyn-page-container"
      header={{
        breadcrumb: {
          items: breadcrumbNames.map(item => ({ title: item })),
        },
      }}
    >
      <KeepAlive clearAlive>
        <ListXxx toolbarTitle={[...breadcrumbNames].pop()} />
      </KeepAlive>
    </PageContainer>
  );
};

// ============ 3. 页面配置（路由级权限） ============
export const pageConfig = definePageConfig(() => ({
  auth: ['/模块/页面路径'],  // 对应 menu.json 中的 path
}));
```

### 关键约定

| 约定 | 说明 |
|------|------|
| `className="qeelyn-page-container"` | 项目统一页面容器类名 |
| `<KeepAlive clearAlive>` | 标签页切换时保留状态 |
| `routeBreadcrumb()` | 从 menu.json 读取面包屑 |
| `definePageConfig` | auth 数组对应菜单权限 key |
| `toolbarTitle={[...breadcrumbNames].pop()}` | 取面包屑最后一项作为工具栏标题 |

---

## 三、列表页 ProTable 模板

```tsx
import Auth, { checkAuth } from '@/components/auth';
import { ActionType, ProColumns, ProTable } from '@ant-design/pro-components';
import { KeepAlive } from '@knockout-js/layout';
import { onMousedown, saveDataSource, delDataSource } from '@/util';
import { Button, message, Modal, Space, Tag, Typography } from 'antd';
import { definePageConfig, Link } from 'ice';
import { Key, useRef, useState } from 'react';
import { useTranslation } from 'react-i18next';
import dayjs from 'dayjs';

// 数据类型扩展（加 UI 专用字段）
type XxxDataSource = GeneratedType & {
  isCancelLoading?: boolean;
  children?: XxxDataSource[];
};

const List = (props: { toolbarTitle?: string }) => {
  const { t } = useTranslation(),
    proTableRef = useRef<ActionType>(),
    [dataSource, setDataSource] = useState<XxxDataSource[]>([]),
    [selectedRowKeys, setSelectedRowKeys] = useState<Key[]>([]),

    // ====== 列定义 ======
    columns: ProColumns<XxxDataSource>[] = [
      {
        title: '编号', dataIndex: 'code', width: 120, order: 2,
      },
      {
        title: '名称', dataIndex: 'name', width: 150, order: 4,
      },
      {
        title: '状态', dataIndex: 'state', width: 100, order: 6,
        valueType: 'select',
        valueEnum: EnumXxxState,  // 从 services 导入
        render: (_, record) => (
          <Tag color={EnumXxxState[record.state]?.tagColor}>
            {EnumXxxState[record.state]?.text}
          </Tag>
        ),
      },
      {
        title: '关联实体', dataIndex: 'relatedID', width: 180, order: 8,
        // 自定义搜索表单项（按项目实际业务选择器替换）
        renderFormItem: () => <Select placeholder="请选择" />,
        // 自定义表格显示（按项目实际缓存/映射替换）
        renderText: (text) => text,
      },
      {
        title: '日期范围', dataIndex: 'dateRange', width: 200,
        valueType: 'dateRange', order: 10,
        render: (_, record) => dayjs(record.createdAt).format('YYYY-MM-DD'),
      },
      {
        title: '创建时间', dataIndex: 'createdAt', valueType: 'dateTime',
        search: false,  // 不参与搜索
      },
      {
        title: '操作', dataIndex: 'actions', fixed: 'right',
        search: false, align: 'center', width: 180,
        render: (_, record) => (
          <Space>
            <Typography.Link onClick={() => { /* 查看 */ }}>查看</Typography.Link>
            <Auth authKey="updateXxx">
              <Typography.Link onClick={() => { /* 编辑 */ }}>编辑</Typography.Link>
            </Auth>
            <Auth authKey="deleteXxx">
              <Typography.Link onClick={() => handleDelete(record)}>删除</Typography.Link>
            </Auth>
          </Space>
        ),
      },
    ];

  // ====== 删除操作 ======
  const handleDelete = (record: XxxDataSource) => {
    Modal.confirm({
      title: '删除',
      content: `是否删除：${record.name}？`,
      onOk: async () => {
        return new Promise(async (resolve, reject) => {
          const result = await mutDeleteXxx(record.id);
          if (result) {
            message.success('执行成功');
            // 本地更新，不重新请求
            setDataSource(delDataSource(dataSource, record.id));
            resolve(true);
          } else {
            reject();
          }
        });
      },
    });
  };

  return (
    <ProTable<XxxDataSource>
      actionRef={proTableRef}
      sticky={dataSource.length > 0 ? { offsetHeader: 56 } : undefined}
      rowKey="id"
      search={{
        className: 'qeelyn-pro-table-search',
        searchText: `${t('query')}`,
        resetText: `${t('reset')}`,
        labelWidth: 70,
      }}
      toolbar={{
        title: props.toolbarTitle,
        actions: [
          <Auth authKey="createXxx">
            <Button type="primary" onClick={() => { /* 新建 */ }}>
              {t('create')}
            </Button>
          </Auth>,
        ],
      }}
      scroll={{ x: 'max-content' }}
      form={{
        initialValues: {
          dateRange: [dayjs(), dayjs()],  // 默认日期
        },
      }}
      columns={columns}
      dataSource={dataSource}
      request={async (params) => {
        // 1) 构建 WhereInput
        const where: XxxWhereInput = {};
        if (params.code) where.codeContains = params.code;
        if (params.name) where.nameContains = params.name;
        if (params.state) where.state = params.state;
        if (params.dateRange?.length === 2) {
          where.createdAtGTE = dayjs(params.dateRange[0]).format('YYYY-MM-DDT00:00:00Z');
          where.createdAtLTE = dayjs(params.dateRange[1]).format('YYYY-MM-DDT23:59:59Z');
        }

        // 2) 调用服务层
        const result = await getXxxList({
          current: params.current,
          pageSize: params.pageSize,
          where,
        });

        // 3) 转换 Relay edges → 扁平数组
        const table = { data: [] as XxxDataSource[], success: true, total: 0 };
        table.total = result?.totalCount ?? 0;
        result?.edges?.forEach((item) => {
          if (item?.node) table.data.push({ ...item.node });
        });

        // 4) 批量初始化缓存（可选，用于关联实体名称显示）
        // await batchInitCacheXxx(table.data.map(i => `${i.xxxID}`));

        setDataSource(table.data);
        setSelectedRowKeys([]);
        return table;
      }}
      pagination={{
        showSizeChanger: true,
        pageSize: 20,
        pageSizeOptions: [10, 20, 50, 100],
      }}
      rowSelection={{
        type: 'radio',
        selectedRowKeys,
        onChange: setSelectedRowKeys,
      }}
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
  );
};
```

### ProTable 列属性速查

| 属性 | 说明 |
|------|------|
| `order: number` | 搜索表单字段排序，数值越小越靠前 |
| `valueEnum` | 枚举映射对象，同时控制搜索下拉和表格显示 |
| `valueType: 'select'` | 搜索用下拉框 |
| `valueType: 'dateRange'` | 搜索用日期范围 |
| `valueType: 'dateTime'` | 表格显示日期时间 |
| `renderFormItem` | 自定义搜索表单项（如业务选择器） |
| `renderText` | 自定义表格单元格文本 |
| `render` | 完全自定义单元格渲染 |
| `search: false` | 仅展示不参与搜索 |
| `hideInTable: true` | 仅搜索不展示在表格 |
| `fieldProps: { mode: 'multiple' }` | 搜索下拉支持多选 |
| `fixed: 'right'` | 操作列固定在右侧 |

---

## 四、ModalForm 编辑器模板（CRUD 弹窗）

```tsx
import { Xxx, UpdateXxxInput } from '@/generated/模块/graphql';
import { getXxxInfo, mutCreateXxx, mutUpdateXxx } from '@/services/模块';
import { updateFormat } from '@/util';
import { ModalForm, ProFormText, ProFormSelect, ProFormTextArea } from '@ant-design/pro-components';
import { useLeavePrompt } from '@knockout-js/layout';
import { Col, Form, message, Row } from 'antd';
import { useEffect, useState } from 'react';

interface FormData {
  code?: string;
  name?: string;
  description?: string;
}

export default (props: {
  title: string;
  id?: string;
  readonly?: boolean;
  onClose: (isSuccess?: boolean, info?: Xxx) => void;
}) => {
  const [form] = Form.useForm<FormData>(),
    [checkLeave, setLeavePromptWhen] = useLeavePrompt(),
    [info, setInfo] = useState<Xxx>(),
    [saveLoading, setSaveLoading] = useState(false),
    [saveDisabled, setSaveDisabled] = useState(true),
    [loading, setLoading] = useState(false);

  useEffect(() => {
    setLeavePromptWhen(saveDisabled);
  }, [saveDisabled]);

  const onOpenChange = (open: boolean) => {
    if (!open) {
      if (checkLeave()) {
        props.onClose?.();
        setSaveDisabled(true);
      }
    } else {
      setSaveDisabled(true);
    }
  };

  return (
    <ModalForm<FormData>
      open={true}
      onOpenChange={onOpenChange}
      disabled={props.readonly}
      loading={loading}
      title={props.title}
      width={500}
      form={form}
      modalProps={{ destroyOnHidden: true }}
      submitter={props.readonly ? false : {
        searchConfig: { submitText: '保存', resetText: '取消' },
        submitButtonProps: { loading: saveLoading, disabled: saveDisabled },
      }}
      onValuesChange={() => setSaveDisabled(false)}
      request={async () => {
        setSaveLoading(false);
        setSaveDisabled(true);
        setLoading(true);
        const result: FormData = {};
        if (props.id) {
          const infoRes = await getXxxInfo(props.id);
          if (infoRes) {
            result.code = infoRes.code;
            result.name = infoRes.name;
            result.description = infoRes.description ?? undefined;
            setInfo(infoRes);
          }
        }
        setLoading(false);
        return result;
      }}
      autoFocusFirstInput
      onFinish={async (values) => {
        setSaveLoading(true);
        if (props.id) {
          // 更新：用 updateFormat 计算差异
          const result = await mutUpdateXxx(props.id, updateFormat<UpdateXxxInput>({
            code: values.code,
            name: values.name,
            description: values.description,
          }, info || {}));
          if (result?.id) {
            setSaveDisabled(true);
            props.onClose?.(true, result);
            message.success('保存成功');
          }
        } else {
          // 新建
          const result = await mutCreateXxx({
            code: values.code ?? '',
            name: values.name ?? '',
            description: values.description,
            state: XxxState.Disable,  // 新建默认禁用
          });
          if (result?.id) {
            setSaveDisabled(true);
            props.onClose?.(true, result);
            message.success('保存成功');
          }
        }
        setSaveLoading(false);
        return false;  // 阻止自动关闭
      }}
    >
      <Row gutter={16}>
        <Col span={12}>
          <ProFormText name="code" label="编码"
            rules={[{ required: true, message: '请填写编码' }]} />
        </Col>
        <Col span={12}>
          <ProFormText name="name" label="名称"
            rules={[{ required: true, message: '请填写名称' }]} />
        </Col>
        <Col span={24}>
          <ProFormTextArea name="description" label="描述" />
        </Col>
      </Row>
    </ModalForm>
  );
};
```

### ModalForm 关键模式

| 模式 | 说明 |
|------|------|
| `saveDisabled` | 初始 true，`onValuesChange` 时设 false，保存后恢复 true |
| `useLeavePrompt` | 未保存时离开提示 |
| `updateFormat(target, original)` | 对比差异，清空字段自动生成 `clearXxx: true` |
| `disabled={props.readonly}` | 查看模式 |
| `submitter={false}` | 查看模式隐藏按钮 |
| `destroyOnHidden: true` | 关闭时销毁表单状态 |
| `request={}` | 加载已有数据 |
| `return false` | onFinish 中返回 false 阻止自动关闭 |
| 新建默认 `state: XxxState.Disable` | 新建默认禁用 |

---

## 四·附、表单编辑离开警告处理规范

### 核心机制

`useLeavePrompt` 来自 `@knockout-js/layout`，覆盖两个层面的离开拦截：

| 层面 | 拦截方式 | 触发场景 |
|------|---------|---------|
| 浏览器刷新/关闭 | `beforeunload` 事件 | F5、Ctrl+R、关闭标签页 |
| SPA 内导航 | Layout 组件中 `checkLeave()` | 菜单切换、应用切换、退出登录 |

> **注意**：无法拦截浏览器前进/后退按钮（框架已知限制）。

### 实现原理

`useLeavePrompt` 内部使用**模块级共享变量** `when`（全局唯一状态），所有使用该 hook 的组件共享同一个状态：

```ts
// @knockout-js/layout 内部简化
var when = true;  // true = 无修改，允许离开

export const useLeavePrompt = () => {
  // checkLeave: when=true 直接返回 true；否则弹出 confirm 对话框
  const checkLeave = () => {
    if (when) return true;
    if (confirm('有未保存的修改，是否离开？')) return true;
    return false;
  };
  // setLeavePromptWhen: 通过 CustomEvent 更新共享的 when 变量
  const setLeavePromptWhen = (when) => {
    window.dispatchEvent(new CustomEvent('updateLeavePromptContext', { detail: when }));
  };
  return [checkLeave, setLeavePromptWhen];
};
```

> `setLeavePromptWhen(true)` = 无修改 = 允许离开；`setLeavePromptWhen(false)` = 有修改 = 阻止离开。语义与 `saveDisabled` 一致。

### Hook 返回值

```tsx
import { useLeavePrompt } from '@knockout-js/layout';

const [checkLeave, setLeavePromptWhen] = useLeavePrompt();
// checkLeave: () => boolean — 检查是否可以离开（无修改时返回 true）
// setLeavePromptWhen: (condition: boolean) => void — 设置"何时有未保存修改"
//   传入 true = 有未保存修改（阻止离开）
//   传入 false = 无未保存修改（允许离开）
```

### 标准使用模式（ModalForm 编辑器）

```tsx
export default (props: {
  title: string;
  id?: string;
  readonly?: boolean;
  onClose: (isSuccess?: boolean, info?: Xxx) => void;
}) => {
  const [form] = Form.useForm<FormData>(),
    [checkLeave, setLeavePromptWhen] = useLeavePrompt(),
    [saveDisabled, setSaveDisabled] = useState(true);

  // ① 监听 saveDisabled 变化，同步到离开提示条件
  // saveDisabled=true → 无修改 → setLeavePromptWhen(true) → 允许离开
  // saveDisabled=false → 有修改 → setLeavePromptWhen(false) → 阻止离开
  useEffect(() => {
    setLeavePromptWhen(saveDisabled);
  }, [saveDisabled]);

  // ② 弹窗关闭时的处理
  const onOpenChange = (open: boolean) => {
    if (!open) {
      // 用户点击关闭按钮 / 遮罩层 / ESC
      if (checkLeave()) {
        // 无修改 或 用户确认离开 → 关闭弹窗
        props.onClose?.();
        setSaveDisabled(true);
      }
      // checkLeave() 返回 false → 弹出确认框，用户取消则不关闭
    } else {
      // 弹窗打开时重置状态
      setSaveDisabled(true);
    }
  };

  return (
    <ModalForm<FormData>
      open={true}
      onOpenChange={onOpenChange}
      // ③ 表单值变化时标记为"有修改"
      onValuesChange={() => setSaveDisabled(false)}
      onFinish={async (values) => {
        // ④ 保存成功后恢复"无修改"状态
        const result = props.id
          ? await mutUpdateXxx(props.id, updateFormat(values, info))
          : await mutCreateXxx(values);
        if (result?.id) {
          setSaveDisabled(true);   // 恢复为"无修改"
          props.onClose?.(true, result);
          message.success('保存成功');
        }
        return false;
      }}
    >
      {/* 表单内容 */}
    </ModalForm>
  );
};
```

### 状态流转图

```
初始状态: saveDisabled=true → setLeavePromptWhen(true) → 允许离开
    ↓ 用户修改表单
onValuesChange → saveDisabled=false → setLeavePromptWhen(false) → 阻止离开
    ↓ 用户保存成功
onFinish → setSaveDisabled(true) → setLeavePromptWhen(true) → 允许离开
    ↓ 用户点击关闭
onOpenChange(false) → checkLeave() → true → props.onClose() → 弹窗关闭
```

### 在 Layout 中的全局监听

`useLeavePrompt` 的离开检查在 Layout 组件中统一处理：

```tsx
// src/components/layout/index.tsx
const [checkLeave] = useLeavePrompt();

<Layout
  onClickMenuItem={async (item, isOpen) => {
    if (checkLeave()) {          // 菜单切换前检查
      navigate(item.path);
    }
  }}
  avatarProps={{
    onLogoutClick: () => {
      if (checkLeave()) {        // 退出前检查
        logout();
      }
    },
  }}
  aggregateMenuProps={{
    onClick: async (menuItem, app, isOpen) => {
      if (checkLeave()) {        // 切换应用前检查
        navigate(url);
      }
    }
  }}
/>
```

> 这意味着任何页面/弹窗中通过 `setLeavePromptWhen(false)` 设置了未保存标记后，用户在 Layout 层的操作（切菜单、切应用、退出）都会被拦截。

### 复杂表单场景（多 Tab / 多步骤）

当表单分布在多个 Tab 或步骤中时，每个子组件都需独立处理：

```tsx
// 子组件 A — 基本信息
export default (props: { info?: Product; readonly?: boolean; productId?: string }) => {
  const [saveDisabled, setSaveDisabled] = useState(true);
  const [, setLeavePromptWhen] = useLeavePrompt();

  useEffect(() => {
    setLeavePromptWhen(saveDisabled);
  }, [saveDisabled]);

  return (
    <ProForm
      onValuesChange={() => setSaveDisabled(false)}
      onFinish={async (values) => {
        await mutUpdateProduct(props.productId!, updateFormat(values, props.info));
        setSaveDisabled(true);
        message.success('保存成功');
      }}
    >
      {/* 字段 */}
    </ProForm>
  );
};
```

### 常见错误与注意事项

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| 忘记调用 `setLeavePromptWhen` | 修改表单后仍可无提示离开 | 必须用 `useEffect` 同步 `saveDisabled` |
| `onFinish` 中不恢复 `setSaveDisabled(true)` | 保存后仍提示未保存 | 保存成功后立即 `setSaveDisabled(true)` |
| `onOpenChange` 中不调用 `checkLeave()` | 关闭弹窗时不提示 | 必须在 `!open` 分支中 `if (checkLeave())` |
| `readonly` 模式下也设置离开提示 | 查看模式也触发提示 | `readonly` 时 `saveDisabled` 保持 `true` 即可 |
| 在 `onFinish` 失败时仍 `setSaveDisabled(true)` | 保存失败后丢失离开提示 | 只在 `result?.id` 成功时才恢复 |

---

## 五、列表页 + 弹窗编辑器组合模板

```tsx
// 页面中组合使用列表 + ModalForm 编辑器
const [modal, setModal] = useState({
  open: false,
  title: '',
  id: '',
  readonly: false,
});

// 打开新建
const handleCreate = () => setModal({ open: true, title: '新建', id: '', readonly: false });

// 打开编辑
const handleEdit = (record: XxxDataSource) => setModal({
  open: true, title: `编辑 ${record.name}`, id: record.id, readonly: false,
});

// 打开查看
const handleView = (record: XxxDataSource) => setModal({
  open: true, title: `查看 ${record.name}`, id: record.id, readonly: true,
});

// 编辑器关闭回调 — 本地更新数据
const handleEditorClose = async (isSuccess?: boolean, info?: Xxx) => {
  if (isSuccess && info) {
    setDataSource(saveDataSource(dataSource, info));
  }
  setModal({ open: false, title: '', id: '', readonly: false });
};

// 渲染
return <>
  <ProTable ... />
  {modal.open ? <Editor
    title={modal.title}
    id={modal.id}
    readonly={modal.readonly}
    onClose={handleEditorClose}
  /> : <></>}
</>;
```

---

## 六、详情页模板

```tsx
import { useSearchParams } from 'ice';
import { ProCard } from '@ant-design/pro-components';
import { getXxxInfo } from '@/services/模块';
import { useEffect, useState } from 'react';

export default () => {
  const [searchParams] = useSearchParams();
  const id = searchParams.get('id') ?? '';
  const [info, setInfo] = useState<Xxx>();

  useEffect(() => {
    if (id) {
      getXxxInfo(id).then(res => res && setInfo(res));
    }
  }, [id]);

  return (
    <PageContainer className="qeelyn-page-container"
      header={{ title: info?.name, ... }}>
      <ProCard split="horizontal">
        {/* 头部信息 */}
        <ProCard>
          <Descriptions>
            <Descriptions.Item label="编号">{info?.code}</Descriptions.Item>
            <Descriptions.Item label="状态">
              <Tag color={EnumXxxState[info?.state ?? ''].tagColor}>
                {EnumXxxState[info?.state ?? ''].text}
              </Tag>
            </Descriptions.Item>
          </Descriptions>
        </ProCard>
        {/* 内容区 Tabs */}
        <ProCard tabs={{ items: [
          { key: 'info', label: '基本信息', children: <InfoTab info={info} /> },
          { key: 'related', label: '关联数据', children: <RelatedTab info={info} /> },
        ]}} />
      </ProCard>
    </PageContainer>
  );
};
```

---

## 七、服务层规范

### 文件结构

```
src/services/
├── index.ts               # instanceName 常量导出
├── 模块名/index.ts         # 该模块的 GraphQL 服务函数
└── auth/                  # 认证相关（REST）
```

### instanceName 注册表

> **项目适配**：每个项目在 `src/services/index.ts` 中定义自己的 `instanceName`，
> 值对应 `@knockout-js/ice-urql` 中注册的 GraphQL 端点名称。
> 编写服务函数时，从当前项目的 `instanceName` 中查找对应后端服务。

```ts
// src/services/index.ts
// 每个项目按需定义，以下为示例结构
export const instanceName = {
  // 按项目实际后端服务配置
  DEFAULT: 'default',
  // ... 其他服务实例
};
```

### 标准 CRUD 服务函数

```ts
import { gql } from '@/generated/模块名';
import { KoHeaders, mutation, paging, query } from '@knockout-js/ice-urql/request';
import { gid } from '@knockout-js/api';
import { instanceName } from '..';

// ====== 枚举映射导出 ======
export const EnumXxxState: Record<string, { text: string; tagColor: string }> = {
  Enable: { text: '启用', tagColor: 'success' },
  Disable: { text: '禁用', tagColor: 'error' },
};

// ====== 1. 列表查询（分页） ======
const queryXxxList = gql(`
  query xxxList($first: Int, $orderBy: XxxOrder, $where: XxxWhereInput) {
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
    instanceName: instanceName.XXX,  // 从当前项目 services/index.ts 查找
    fetchOptions: { headers: KoHeaders.noCache },
  });
  return result.data?.xxxs;
};

// ====== 2. 单条查询 ======
const queryXxxInfo = gql(`
  query xxxInfo($gid: GID!) {
    node(id: $gid) {
      ... on Xxx { id code name state ... }
    }
  }
`);

export const getXxxInfo = async (id: string) => {
  const result = await query(queryXxxInfo, { gid: gid('Xxx', id) }, {
    instanceName: instanceName.XXX,
    fetchOptions: { headers: KoHeaders.noCache },
  });
  if (result.data?.node?.__typename === 'Xxx') return result.data.node;
  return null;
};

// ====== 3. 创建 ======
const mutationCreateXxx = gql(`
  mutation createXxx($input: CreateXxxInput!) {
    createXxx(input: $input) { id code name state ... }
  }
`);

export const mutCreateXxx = async (input: CreateXxxInput) => {
  const result = await mutation(mutationCreateXxx, { input }, {
    instanceName: instanceName.XXX,
  });
  return result.data?.createXxx;
};

// ====== 4. 更新 ======
const mutationUpdateXxx = gql(`
  mutation updateXxx($id: ID!, $input: UpdateXxxInput!) {
    updateXxx(id: $id, input: $input) { id code name state ... }
  }
`);

export const mutUpdateXxx = async (id: string, input: UpdateXxxInput) => {
  const result = await mutation(mutationUpdateXxx, { id, input }, {
    instanceName: instanceName.XXX,
  });
  return result.data?.updateXxx;
};
// ⚠️ 调用 mutUpdateXxx 时，input 参数必须经过 updateFormat 处理
// 正确：mutUpdateXxx(id, updateFormat<UpdateXxxInput>(formValues, originalInfo))
// 错误：mutUpdateXxx(id, formValues)  ← 直接传表单对象

// ====== 5. 删除 ======
const mutationDeleteXxx = gql(`
  mutation deleteXxx($id: ID!) { deleteXxx(id: $id) }
`);

export const mutDeleteXxx = async (id: string) => {
  const result = await mutation(mutationDeleteXxx, { id }, {
    instanceName: instanceName.XXX,
  });
  return result.data?.deleteXxx;
};

// ====== 6. 校验函数（可选） ======
export const isCanDelete = (record: Xxx): {
  type: 'error' | 'warning';
  string: string;
} | undefined => {
  if (record.state === 'Enable') {
    return { type: 'warning', string: '请先禁用再删除' };
  }
  return undefined;
};
```

### 三个 urql 辅助函数

| 函数 | 用途 | 返回 |
|------|------|------|
| `paging(doc, args, page, opts)` | Relay 游标分页列表 | `{ totalCount, edges, pageInfo }` |
| `query(doc, vars, opts)` | 单实体查询 | `result.data` |
| `mutation(doc, vars, opts)` | 写操作 | `result.data` |

### GraphQL 命名约定

```graphql
# 查询：{service}{Entity}{Action}
query userOrderList($first: Int, ...) { ... }
query userConditionOrderList(...) { ... }
query bizTypes($first: Int, ...) { ... }

# Mutation：{service}{Action}
mutation userCancelOrder($req: CancelOrderRequest) { ... }
mutation createBizType($input: CreateBizTypeInput!) { ... }
```

### 代码生成工作流

```bash
# 1. 从远程拉取 Schema（需 .env.local 中配置 Token）
pnpm gqlgen:schema-ast

# 2. 生成 TypeScript 类型
pnpm gqlgen

# 3. 监听模式（开发时）
pnpm gqlgen:watch
```

**流程：** 在 `src/services/<模块>/*.ts` 写 `gql()` → 运行 `pnpm gqlgen` → 从 `@/generated/<模块>/graphql` 导入类型

---

## 八、权限控制

### 8.1 权限体系概述

权限系统分为三层：

```
menu.json（页面路由权限） → userPermissions API（后端权限列表） → Auth 组件（按钮级控制）
```

| 层级 | 控制目标 | 配置位置 | 示例 |
|------|---------|---------|------|
| 路由权限 | 页面能否访问 | `pageConfig.auth` + `menu.json` | `/ui/biz-type` |
| 操作权限 | 按钮能否显示 | `<Auth authKey="...">` | `createBizType` |
| 开发模式 | 全部权限放行 | `NODE_ENV=development` | 自动通过 |

### 8.2 路由级权限

```tsx
import { definePageConfig } from 'ice';

// auth 数组中的 path 必须与 menu.json 中的 path 完全一致
export const pageConfig = definePageConfig(() => ({
  auth: ['/ui/biz-type'],
}));
```

**menu.json 结构：**

```json
[
  {
    "name": "基础数据",
    "icon": "icon-icon_zhanghu1",
    "children": [
      { "name": "业务类型管理", "path": "/ui/biz-type" },
      { "name": "交易类型管理", "path": "/ui/trade-type" }
    ]
  }
]
```

### 8.3 权限 Key 获取规范

#### 权限 Key 的来源与加载流程

```
1. 应用启动 → authConfig (app.tsx)
2. getMenuAppActions() → 从 menu.json 提取所有 path 作为初始权限（开发环境全部为 true）
3. userPermissions(ICE_APP_CODE, headers) → 从后端获取用户操作权限列表
4. 后端返回 ups[] → 每个 item.name 就是权限 key（如 "createBizType"）
5. 合并到 initialAuth → { "/ui/biz-type": true, "createBizType": true, ... }
```

**app.tsx 中的关键代码：**

```tsx
import { userPermissions } from '@knockout-js/api';
import { getMenuAppActions } from '@/util';

export const authConfig = defineAuthConfig(async (appData) => {
  // 1. 从 menu.json 生成路由权限（开发环境自动全部放行）
  const initialAuth = getMenuAppActions();

  // 2. 从后端获取操作权限
  const ups = await userPermissions(ICE_APP_CODE, {
    Authorization: getRequestHeaderAuthorization(token, ...),
    'X-Tenant-ID': tenantId,
  });

  // 3. 合并到 initialAuth
  ups?.forEach(item => {
    if (item) initialAuth[item.name] = true;
  });

  return { initialAuth };
});
```

#### 权限 Key 命名规则

> **项目适配**：不同项目的权限 key 风格可能不同，开发前参考同项目已有页面确认风格。

| 风格 | 适用项目类型 | 规则 | 示例 |
|------|---------|------|------|
| camelCase 动词+实体 | 多数项目 | `create/update/delete` + 实体名 | `createBizType`、`updateUnit`、`deleteTradeRule` |
| snake_case 应用\_实体\_操作 | 部分项目 | `app_entity_action_btn` | `account_unit_add_btn`、`account_unit_config_btn` |

> **重要**：开发新页面前需确认后端已配置对应权限 key。参考当前项目已有页面判断风格。

#### 开发环境权限自动放行

`getMenuAppActions()` 在 `NODE_ENV=development` 时将所有 `menu.json` 中的 path 设为 `true`：

```tsx
// src/util/index.ts
export const getMenuAppActions = (list?: MenuJsonData[]) => {
  const initialAuth: Record<string, true> = {};
  if (process.env.NODE_ENV === 'development') {
    menuJsonList?.forEach(item => {
      if (item.path) initialAuth[item.path] = true;
      if (item.children) { /* 递归子菜单 */ }
    });
  }
  return initialAuth;
};
```

> **注意**：`getMenuAppActions()` 只处理路由权限（path），操作权限（如 `createBizType`）在开发环境下由 `checkAuth` 中的 `NODE_ENV === 'development'` 判断自动放行。

### 8.4 操作按钮权限规范

#### 组件包裹模式（推荐）

```tsx
import Auth, { checkAuth } from '@/components/auth';

// 新建按钮
<Auth authKey="createBizType">
  <Button type="primary" onClick={handleCreate}>新建</Button>
</Auth>

// 编辑按钮
<Auth authKey="updateBizType">
  <Typography.Link onClick={() => handleEdit(record)}>编辑</Typography.Link>
</Auth>

// 删除按钮
<Auth authKey="deleteBizType">
  <Typography.Link onClick={() => handleDelete(record)}>删除</Typography.Link>
</Auth>
```

#### 操作列完整示例（CRUD 四按钮）

```tsx
{
  title: '操作', dataIndex: 'actions', fixed: 'right',
  search: false, align: 'center', width: 180,
  render(_, record) {
    return <Space>
      {/* 查看：无需权限控制 */}
      <Typography.Link onClick={() => handleView(record)}>查看</Typography.Link>

      {/* 编辑：受 update 权限控制 */}
      <Auth authKey="updateBizType">
        <Typography.Link
          disabled={record.state === BizTypeState.Enable}
          onClick={() => handleEdit(record)}
        >编辑</Typography.Link>
      </Auth>

      {/* 删除：受 delete 权限控制 */}
      <Auth authKey="deleteBizType">
        <Typography.Link
          disabled={record.state === BizTypeState.Enable}
          onClick={() => handleDelete(record)}
        >删除</Typography.Link>
      </Auth>

      {/* 启用/禁用：受 update 权限控制 */}
      <Auth authKey="updateBizType">
        <Typography.Link onClick={() => handleToggleState(record)}>
          {record.state === BizTypeState.Enable ? '禁用' : '启用'}
        </Typography.Link>
      </Auth>
    </Space>;
  }
}
```

#### 多权限组合

```tsx
// AND 模式（默认）：需要同时满足所有权限
<Auth authKey={['perm1', 'perm2']}>
  <Button>操作</Button>
</Auth>

// OR 模式：满足任一权限即可
<Auth authKey={['perm1', 'perm2']} keyAndOr="or">
  <Button>操作</Button>
</Auth>

// 无权限时的占位内容
<Auth authKey="specialPerm" fallback={<span>无权限</span>}>
  <Button>操作</Button>
</Auth>
```

#### 编程式权限检查

```tsx
import { checkAuth } from '@/components/auth';

// 在非 JSX 场景使用
const handleBatchDelete = () => {
  if (!checkAuth('deleteBizType')) {
    message.warning('无删除权限');
    return;
  }
  // 执行批量删除...
};
```

#### Auth 组件实现原理

```tsx
// src/components/auth/index.tsx
// checkAuth 内部使用 useAuth() 获取权限表
// 开发环境下（NODE_ENV === 'development'）直接返回 true
export const checkAuth = (authKey: string, auth?: AuthType) => {
  if (!auth) [auth] = useAuth();
  return NODE_ENV === 'development' || auth[authKey];
};
```

### 8.5 权限检查清单

- [ ] `pageConfig.auth` 中的 path 与 `menu.json` 一致
- [ ] 每个操作按钮用 `<Auth authKey="...">` 包裹
- [ ] 查看操作不设置权限控制
- [ ] 编辑/启用/禁用共用 `update` 权限
- [ ] 删除使用 `delete` 权限
- [ ] 新建使用 `create` 权限
- [ ] 权限 key 已在后端管理系统配置

---

## 九、确认弹窗模式

```tsx
import { ConfirmKnown } from '@/components/modalConfirm';

// 标准确认弹窗
Modal.confirm({
  title: '删除',
  content: `是否删除：${record.name}？`,
  onOk: async () => {
    return new Promise(async (resolve, reject) => {
      const result = await mutDeleteXxx(record.id);
      if (result) {
        message.success('执行成功');
        resolve(true);
      } else {
        reject();
      }
    });
  },
});

// 带"我已知悉"确认的危险操作弹窗
const mc = Modal.confirm({
  title: '撤单',
  content: (
    <ConfirmKnown
      showCheck={warningStr.length > 0}
      onChange={(check) => mc.update({ okButtonProps: { disabled: !check } })}
    >
      <div>是否对{record.orderNo}进行撤单操作？</div>
    </ConfirmKnown>
  ),
  onOk: async () => { /* 同上 Promise 模式 */ },
});
```

---

## 十、通用工具函数

### 数据操作

| 函数 | 用途 | 示例 |
|------|------|------|
| `saveDataSource(dataSource, data)` | 新增或更新扁平数组数据 | `setDataSource(saveDataSource(list, newItem))` |
| `delDataSource(dataSource, id)` | 按 ID 移除 | `setDataSource(delDataSource(list, id))` |
| `updateFormat(target, original)` | 对比差异生成更新 patch | `mutUpdate(id, updateFormat(form, info))` |
| `formatTreeData(allList, parentList?, defineKey?)` | 扁平数组 → 树结构 | 构建 Ant Design Tree 数据 |
| `loopTreeData(data, key, callback)` | 遍历树找节点 | 查找并修改树中某个节点 |
| `saveTreeData(treeList, updateData)` | 树中插入/更新节点 | 树形 CRUD 后本地更新 |
| `delTreeData(treeList, id)` | 树中删除节点 | |
| `getTreeDropData(treeData, dragInfo)` | 处理 Tree 拖拽结果 | 返回 `{ sourceId, targetId, action, newTreeData }` |

### 其他工具

| 函数 | 用途 |
|------|------|
| `firstUpper(str)` | 首字母大写 |
| `randomId(len)` | 随机字符串 |
| `browserLanguage()` | 检测浏览器语言 |
| `getMenuAppActions(list?)` | 开发模式从 menu.json 生成权限 map |
| `getANDResult(list, value)` | 位与运算（多选标记字段展示） |
| `getORResult(list)` | 位或运算（多选标记字段存储） |
| `openDetailUrl(data, type)` | 生成详情页 URL |
| `errTextFormat(str)` | 格式化 KO 错误消息 |
| `roundUp(num, decimals)` | 向上保留小数 |
| `roundDown(num, decimals)` | 向下保留小数 |
| `onMousedown({ target, click, doubleClick, textSelection })` | 区分单击/双击/文本选中 |

### updateFormat 详解

> **核心规则：所有更新接口（`mutUpdateXxx`）必须使用 `updateFormat` 处理数据。**
>
> 不要直接传整个表单对象给 update mutation，必须用 `updateFormat(values, originalInfo)` 做差异对比。

#### 为什么必须用

| 问题 | 直接传整个表单 | 用 updateFormat |
|------|--------------|----------------|
| 未修改的字段 | 也会发送，浪费带宽 | 只发变化的字段 |
| 清空字段 | 传 `null`/`undefined`，后端不知道是"清空"还是"没传" | 自动生成 `clearXxx: true`，后端明确知道要清空 |
| 后端校验 | 可能触发不必要的必填校验 | 只校验真正变化的字段 |

#### 用法

```ts
import { updateFormat } from '@/util';

// 对比 target 和 original，只返回变化的字段
// 清空字段时自动生成 clearXxx: true
updateFormat(
  { name: 'new', description: null, code: 'same' },
  { name: 'old', description: 'desc', code: 'same' }
)
// 结果: { name: 'new', clearDescription: true }
// code 没变，不在结果中
```

#### 正确 vs 错误示例

```tsx
// ✅ 正确：用 updateFormat 做差异对比
onFinish={async (values) => {
  const result = await mutUpdateXxx(props.id, updateFormat<UpdateXxxInput>({
    name: values.name,
    description: values.description,
    appID: Number(values.appID),
  }, info || {}));
  // ...
}}

// ❌ 错误：直接传整个表单
onFinish={async (values) => {
  const result = await mutUpdateXxx(props.id, values);  // 不要这样做！
  // ...
}}

// ❌ 错误：手动拼装但漏了 clearXxx
onFinish={async (values) => {
  const result = await mutUpdateXxx(props.id, {
    name: values.name,
    description: values.description,
    // description 清空了但没传 clearDescription: true，后端无法清空该字段
  });
  // ...
}}
```

#### 带排除字段的用法

```ts
// 第三个参数 excludeTargetKey 可排除不需要对比的字段
updateFormat<UpdateXxxInput>(
  { name: 'new', state: 'Enable', code: 'same' },
  { name: 'old', state: 'Enable', code: 'same' },
  ['state']  // state 不参与差异对比
)
// 结果: { name: 'new' }
// state 被排除，即使没变也不影响
```

#### 实现原理

```ts
// src/util/index.ts
export const updateFormat = <T>(
  target: T,
  original: Record<string, any>,
  excludeTargetKey?: string[]
) => {
  const ud: Record<string, any> = {};
  for (const key in target) {
    if (excludeTargetKey && excludeTargetKey.includes(key)) continue;
    const tValue = target[key];
    if (tValue !== original[key]) {
      ud[key] = tValue;
      // 非 boolean 类型的空值 → 生成 clearXxx: true
      if (typeof tValue != 'boolean' && !tValue) {
        const clearKey = `clear${firstUpper(key)}`.replace('ID', '');
        ud[clearKey] = true;
      }
    }
  }
  return ud as T;
};
```

> **注意**：`clearXxx` 的命名规则是 `clear` + 首字母大写字段名，但 `ID` 后缀会被去掉。例如 `appID` → `clearApp`，不是 `clearAppID`。

### onMousedown 详解

```ts
// 解决表格行点击和文本选中冲突
onMousedown({
  target: e.target as HTMLElement,
  exclusionClassNames: ['checkbox', 'expand'],  // 排除的元素类名
  click: () => { /* 单击：选中行 */ },
  doubleClick: () => { /* 双击：展开/编辑 */ },
  textSelection: () => { /* 文本选中：不触发点击 */ },
});
```

---

## 十之一、用户信息缓存（cacheUser / getCacheUser）

用于将 **用户 ID**（`createdBy` / `updatedBy` / `traderID` / `owner` 等数字字段）解析为用户显示名（`displayName`）。

### 适用场景

| 缓存 | 解析对象 | 适用字段 | 来源 |
|------|---------|---------|------|
| **cacheUser** | 系统用户（会员/员工） | `createdBy` / `updatedBy` / `traderID` / `owner` / 任意 user ID | `@knockout-js/api` |

"创建人"、"更新人" 等通常是 user ID → 用 `cacheUser`。

### 导入

```ts
// 单条查询
import { getCacheUser } from '@knockout-js/api';
import { User } from '@knockout-js/api/ucenter';   // User 类型

// 列表批量
import { batchInitCacheUser, cacheUser } from '@knockout-js/api/esm/ucenter';
```

### 详情页 — 单条解析

```tsx
const [info, setInfo] = useState<Xxx>();
const [createdByUserInfo, setCreatedByUserInfo] = useState<User>();

const reqInfo = async () => {
  const result = await getXxxInfo(id);
  if (result) {
    setInfo(result);
    if (result.createdBy) {
      const userInfo = await getCacheUser(`${result.createdBy}`);
      setCreatedByUserInfo(userInfo);
    }
  }
};

// JSX 渲染：优先 displayName，兜底原始 ID
<Typography.Text>
  <span className="label">{t('created_by')}：</span>
  {createdByUserInfo?.displayName ?? info.createdBy ?? '-'}
</Typography.Text>
```

### 列表页 — 批量解析

在 `request` 中拉取数据后批量初始化缓存：

```tsx
request={async (params) => {
  // ... 请求数据
  const table = { data: [...], success: true, total: 0 };
  // ...

  // 批量缓存用户信息（过滤掉无效 ID）
  await batchInitCacheUser(
    table.data.filter(item => Number(item.createdBy) > 0).map(item => `${item.createdBy}`)
  );

  return table;
}}
```

列定义中通过 `cacheUser` 字典直接读取：

```tsx
{
  title: '创建人', dataIndex: 'createdBy', width: 100,
  renderText: (text, record) => {
    const user = record.createdBy ? cacheUser[record.createdBy] : undefined;
    return user?.displayName ?? record.createdBy;
  },
}
```

### 关键规则

1. **ID 必须转字符串**：`getCacheUser(`${id}`)` — 参数类型是 string
2. **批量前先过滤无效 ID**：`filter(item => Number(item.xxxID) > 0)`
3. **兜底显示原始 ID**：`userInfo?.displayName ?? record.xxxID` — 缓存未命中时不显示空白
4. **User 类型字段**：`displayName`（显示名）、`id`、`userType` 等

---

## 十一、localStorage 封装

每个项目使用命名空间隔离：

```ts
// src/pkg/localStore.ts
const module = '项目名';  // 如 'meta'、'user'、'msg' — 按当前项目配置

getItem<T>(key: string): T | null    // 读取
setItem<T>(key: string, value: T)    // 写入
removeItem(key: string)              // 删除
monitorKeyChange(keys)               // 跨标签页同步（storage 事件 + 200ms 节流）
```

用法：
```ts
import { getItem, setItem, removeItem } from '@/pkg/localStore';
setItem<string>('token', token);
const token = getItem<string>('token');
```

---

## 十二、状态管理（ICE Store）

### Model 定义

```ts
// src/models/user.ts
import { createModel } from 'ice';

export default createModel({
  state: { token: '', user: null } as ModelState,
  reducers: {
    updateToken(prevState, payload: string) {
      if (payload) setItem('token', payload);
      else removeItem('token');
      prevState.token = payload;
    },
  },
  effects: () => ({
    async logout() { this.updateToken(''); },
  }),
});
```

### 组件中使用

```tsx
import store from '@/store';

const [userState] = store.useModel('user');
const [appState, appDispatcher] = store.useModel('app');
appDispatcher.updateLocale('en-US');
```

---

## 十三、日期处理

```ts
import dayjs from 'dayjs';

// 格式化
dayjs().format('YYYY-MM-DD');
dayjs().format('YYYY-MM-DDTHH:mm:ss.SSSZ');

// 日期范围 → WhereInput
where.createdAtGTE = dayjs(params.dateRange[0]).format('YYYY-MM-DDT00:00:00Z');
where.createdAtLTE = dayjs(params.dateRange[1]).format('YYYY-MM-DDT23:59:59Z');

// 表格显示
render: (_, record) => dayjs(record.createdAt).format('YYYY-MM-DD')
```

---

## 十四、国际化

```tsx
import { useTranslation } from 'react-i18next';

const { t } = useTranslation();

// 使用
<Button>{t('create')}</Button>
searchText: `${t('query')}`
resetText: `${t('reset')}`
message.success(`${t('action_success')}`)
```

常用 key：`query`、`reset`、`create`、`editor`、`view`、`operation`、`action_success`

---

## 十五、CSS 类名约定

| 类名 | 用途 |
|------|------|
| `qeelyn-page-container` | PageContainer 标准类名 |
| `qeelyn-pro-table-search` | ProTable 搜索区域标准类名 |

---

## 十六、开发命令速查

| 命令 | 说明 |
|------|------|
| `pnpm dev` | 启动开发服务器（带 mock） |
| `pnpm start` | 启动开发服务器（禁用 mock） |
| `pnpm build` | 生产构建 |
| `pnpm eslint` | ESLint 检查 |
| `pnpm eslint:fix` | ESLint 自动修复 |
| `pnpm stylelint` | Stylelint 检查 |
| `pnpm gqlgen` | GraphQL 代码生成 |
| `pnpm gqlgen:schema-ast` | 从远程拉取 Schema |
| `pnpm gqlgen:watch` | 监听模式生成 |

---

## 十七、枚举映射命名约定

| 模式 | 示例 |
|------|------|
| `Enum{Entity}{Field}` | `EnumOrderOrdStatus`、`EnumBizTypeBizTypeState` |
| `{ text, tagColor, textCn? }` | 每个枚举值包含显示文本和 Tag 颜色 |

```ts
// 完整映射
export const EnumOrderOrdStatus: Record<OrderOrdStatus, { text: string; tagColor: string; textCn?: string }> = {
  [OrderOrdStatus.New]: { text: 'New', tagColor: '#ffcc5f', textCn: '已报' },
  [OrderOrdStatus.Filled]: { text: 'Filled', tagColor: '#30c880', textCn: '已成' },
  [OrderOrdStatus.Canceled]: { text: 'Canceled', tagColor: '#ff4d4f', textCn: '已撤' },
};

// 简单映射
export const EnumUnitUnitState = {
  Disable: { text: '禁用', tagColor: 'error' },
  Enable: { text: '启用', tagColor: 'success' },
};
```

---

## 十八、命名规范速查

| 类型 | 规则 | 示例 |
|------|------|------|
| 组件 | PascalCase | `ListOrder`、`ConfirmKnown` |
| 查询函数 | `get` + 实体 + 动作 | `getOrderList`、`getOrderInfo` |
| Mutation 函数 | `mut` + 动作 | `mutCancelOrder`、`mutNewOrder` |
| 枚举映射 | `Enum` + 实体名 | `EnumOrderOrdStatus` |
| 校验函数 | `is` + 动作 | `isCancelOrder` |
| 事件处理 | `on` + 动作 | `onClick`、`onChange`、`onFinish` |
| 类型/接口 | PascalCase | `OrderWhereInput`、`MyComponentRef` |
| 页面文件 | `index.tsx` | `pages/ui/list/index.tsx` |
| 服务文件 | `index.ts` | `services/ui/index.ts` |

---

## 十九、组件编写模式

### 函数组件（默认）

```tsx
// 匿名箭头 + 内联 Props
export default (props: { title: string; onSuccess?: () => void }) => {
  return <div>...</div>;
};
```

### forwardRef（暴露方法给父组件）

```tsx
export interface MyComponentRef {
  reload: () => void;
  clearAutoReload: () => void;
}

export default forwardRef<MyComponentRef, Props>((props, ref) => {
  useImperativeHandle(ref, () => ({
    reload: () => { /* ... */ },
    clearAutoReload: () => { /* ... */ },
  }));
  return <div>...</div>;
});
```

---

## 二十、微前端通信

```tsx
import { starkStore, starkEvent } from '@ice/stark-data';
import { isInIcestark } from '@ice/stark-app';

// 读取宿主状态
const user = starkStore.getItem('user');

// 发送事件
starkEvent.emit('set-user', user);
starkEvent.emit('set-token', newToken);
```

---

## 二十一、字典下拉与业务选择器

### 字典下拉（meta 项目常用）

使用 `getDictionaryValueList` 从 meta 服务获取字典数据，通过 `typeCode` 过滤：

```tsx
import { getDictionaryValueList } from '@/services/meta';

// 方式 1：单个 typeCode
const loadDict = async () => {
  const result = await getDictionaryValueList({
    where: { typeCode: 'BizModule', isEnabled: 'Y' },
  });
  return result?.edges?.map(e => ({
    label: e.node.name,
    value: e.node.code,
  })) ?? [];
};

// 在 ProFormSelect 中使用
<ProFormSelect
  name="moduleCode"
  label="业务模块"
  request={loadDict}
/>

// 方式 2：多个 typeCode 批量加载
const typeCodes = ['BizModule', 'OrderType', 'MarketType'];
const [dictData, setDictData] = useState<Record<string, { label: string; value: string }[]>>({});

useEffect(() => {
  const loadAll = async () => {
    const data: Record<string, { label: string; value: string }[]> = {};
    for (const code of typeCodes) {
      const res = await getDictionaryValueList({ where: { typeCode: code, isEnabled: 'Y' } });
      data[code] = res?.edges?.map(e => ({
        label: e.node.name,
        value: e.node.code,
      })) ?? [];
    }
    setDictData(data);
  };
  loadAll();
}, []);

// 在表格列中使用字典值显示
{
  title: '业务模块', dataIndex: 'moduleCode', width: 120,
  renderText: (text) => dictData['BizModule']?.find(d => d.value === text)?.label ?? text,
}
```

### AppSelect 应用选择器

```tsx
import { AppSelect } from '@knockout-js/org';

// 方式 1：返回完整对象（默认）
<ProFormSelect
  name="appID"
  label="所属应用"
  addonAfter={<AppSelect />}  // 弹窗形式
/>

// 方式 2：只返回 id
<ProFormSelect
  name="appID"
  label="所属应用"
  addonAfter={<AppSelect changeValue="id" />}
/>

// 方式 3：作为独立表单组件
<AppSelect
  value={appID}
  onChange={(val) => setAppID(val)}
/>
```

---

## 二十二、非标准 Query/Mutation 模式

### 非标准 Query（自定义输入参数）

当 GraphQL schema 定义的查询不是标准 Relay 分页时：

```ts
// Schema: markets(input: MarketsInput!): MarketsResponse!
const queryMarkets = gql(`
  query markets($input: MarketsInput!) {
    markets(input: $input) {
      markets { id code name description }
    }
  }
`);

export const getMarkets = async (input: MarketsInput) => {
  const result = await query(queryMarkets, { input }, {
    instanceName: instanceName.XXX,
    fetchOptions: { headers: KoHeaders.noCache },
  });
  return result.data?.markets;
};
```

### 非标准 Mutation（多参数）

当 mutation 需要多个独立参数而非单一 input：

```ts
// Schema: deleteWatchlist(materialID: ID!, scene: WatchScene!): Boolean!
const mutationDeleteWatchlist = gql(`
  mutation deleteWatchlist($materialID: ID!, $scene: WatchScene!) {
    deleteWatchlist(materialID: $materialID, scene: $scene)
  }
`);

export const mutDeleteWatchlist = async (materialID: string, scene: WatchScene) => {
  const result = await mutation(mutationDeleteWatchlist, { materialID, scene }, {
    instanceName: instanceName.XXX,
  });
  return result.data?.deleteWatchlist;
};

// 调用示例
await mutDeleteWatchlist(record.materialID, WatchScene.Trading);
```

### 聚合查询（返回统计信息）

```ts
const queryDashboardStats = gql(`
  query dashboardStats($where: StatsWhereInput) {
    stats(where: $where) {
      totalOrders
      totalVolume
      totalCommission
    }
  }
`);

export const getDashboardStats = async (where?: StatsWhereInput) => {
  const result = await query(queryDashboardStats, { where });
  return result.data?.stats;
};
```

---

## 二十三、服务层 JSDoc 注释规范

每个导出的函数和枚举必须添加 JSDoc 注释：

```ts
/**
 * 订单状态枚举映射
 */
export const EnumOrderStatus: Record<string, { text: string; tagColor: string }> = {
  New: { text: '新建', tagColor: 'processing' },
  Filled: { text: '已成', tagColor: 'success' },
};

/**
 * 订单列表 GraphQL 查询
 */
const queryOrderList = gql(`...`);

/**
 * 获取订单列表（分页）
 * @param gather - 分页和过滤参数
 * @returns Relay 连接对象，包含 totalCount 和 edges
 */
export const getOrderList = async (gather: {
  current?: number;
  pageSize?: number;
  where?: OrderWhereInput;
  orderBy?: OrderOrder;
}) => {
  // ...
};

/**
 * 获取订单详情
 * @param id - 订单 ID
 * @returns 订单对象或 null
 */
export const getOrderInfo = async (id: string) => {
  // ...
};

/**
 * 创建订单
 * @param input - 创建订单输入
 * @returns 创建的订单对象
 */
export const mutCreateOrder = async (input: CreateOrderInput) => {
  // ...
};

/**
 * 更新订单
 * @param id - 订单 ID
 * @param input - 更新输入
 * @returns 更新后的订单对象
 */
export const mutUpdateOrder = async (id: string, input: UpdateOrderInput) => {
  // ...
};

/**
 * 删除订单
 * @param id - 订单 ID
 * @returns 是否成功
 */
export const mutDeleteOrder = async (id: string) => {
  // ...
};
```

### 文件头注释

每个服务文件开头应包含模块说明：

```ts
/**
 * 订单管理 GraphQL API 服务接口
 * 配合 gqlgen 使用，运行 `pnpm gqlgen` 生成类型
 */
import { gql } from '@/generated/order';
// ...
```

---

## 二十四、gqlgen 配置与工作流

### gqlgen 配置文件

`script/gqlgen.ts` 控制代码生成行为：

```ts
export default {
  // 指定 documents 路径，确保包含服务目录
  documents: [
    "src/services/**/*.ts",  // 按项目实际模块目录配置
  ],
  // 输出生成文件到 src/generated/
  generates: {
    "src/generated/": {
      // ...
    },
  },
};
```

**新增服务模块时，必须更新 `documents` 数组**，否则 `gql()` 中的类型无法解析。

### 完整工作流

```bash
# 1. 更新 Schema（可选，网络不通时跳过）
pnpm gqlgen:schema-ast

# 2. 分析 Schema（读取 script/generated/xxx.graphql）
# - 识别 Query（列表/详情）
# - 识别 Mutation（create/update/delete）
# - 识别 Enum（需导出给 UI 的枚举）

# 3. 更新 gqlgen 配置（如果是新模块）
# 编辑 script/gqlgen.ts，添加 documents 路径

# 4. 编写服务函数
# 在 src/services/模块名/index.ts 中写 gql() 模板

# 5. 生成 TypeScript 类型
pnpm gqlgen

# 6. 验证类型正确性
# 检查无类型错误，确保导入路径正确
```

### Schema 分析要点

读取 `script/generated/模块名.graphql` 时关注：

| 类型 | 识别方式 | 处理方式 |
|------|---------|---------|
| 标准列表查询 | `xxx(first: Int, where: XxxWhereInput)` | 用 `paging()` |
| 单条查询 | `node(id: GID!) { ... on Xxx }` | 用 `query()` + `gid()` |
| 自定义查询 | `markets(input: MarketsInput!)` | 直接传参，用 `query()` |
| 标准 Mutation | `createXxx(input: CreateXxxInput!)` | 用 `mutation()` |
| 多参数 Mutation | `deleteXxx(id: ID!, scene: Scene!)` | 多个参数分别传递 |
| 枚举 | `enum XxxState { Enable Disable }` | 导出为 `EnumXxxState` 映射 |

---

## 二十五、分页参数详解

### paging() 函数签名

```ts
paging<T>(
  document: DocumentNode,      // gql 查询文档
  variables: {                  // 查询变量
    first?: number;             // 每页数量
    after?: string;             // 游标
    where?: WhereInput;         // 过滤条件
    orderBy?: OrderInput;       // 排序
  },
  page: number,                 // 当前页码（从 1 开始）
  options?: {
    instanceName?: string;      // GraphQL 实例名
    fetchOptions?: {
      headers?: Headers;        // 请求头（如 KoHeaders.noCache）
    };
  }
): Promise<{ data: { xxxs: Connection<T> } }>
```

### 分页参数转换

ProTable 的 `params.current` 从 1 开始，`paging()` 也使用 1-based：

```ts
const result = await paging(queryList, {
  first: gather.pageSize || 20,
  where: gather.where,
  orderBy: gather.orderBy ?? {
    direction: OrderDirection.Desc,
    field: XxxOrderField.CreatedAt,
  },
}, gather.current || 1, {        // 默认第 1 页
  instanceName: instanceName.XXX,
  fetchOptions: { headers: KoHeaders.noCache },
});
```

### Relay Connection 结构

```ts
interface Connection<T> {
  totalCount: number;
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor: string;
    endCursor: string;
  };
  edges: Array<{
    cursor: string;
    node: T;
  }>;
}
```

---

## 二十六、本地数据更新模式

### saveDataSource - 新增或更新

```ts
import { saveDataSource } from '@/util';

// 新增：如果数组中没有该 id，则追加
// 更新：如果数组中有该 id，则替换
setDataSource(saveDataSource(dataSource, newItem));

// 典型场景：编辑弹窗保存成功后
const handleEditorClose = async (isSuccess?: boolean, info?: Xxx) => {
  if (isSuccess && info) {
    setDataSource(saveDataSource(dataSource, info));
  }
  setModal({ open: false });
};
```

### delDataSource - 删除

```ts
import { delDataSource } from '@/util';

// 按 id 移除数组中的项
setDataSource(delDataSource(dataSource, record.id));

// 典型场景：删除成功后本地更新，不重新请求
Modal.confirm({
  onOk: async () => {
    const result = await mutDeleteXxx(record.id);
    if (result) {
      setDataSource(delDataSource(dataSource, record.id));
      message.success('删除成功');
    }
  },
});
```

### updateFormat - 差异对比

```ts
import { updateFormat } from '@/util';

// 对比 target 和 original，只返回变化的字段
// 清空字段时自动生成 clearXxx: true
const patch = updateFormat<UpdateXxxInput>(
  { name: 'newName', description: null, code: 'sameCode' },
  { name: 'oldName', description: 'oldDesc', code: 'sameCode' }
);
// patch = { name: 'newName', clearDescription: true }
// code 没变，不在结果中

// 在 ModalForm 中使用
onFinish={async (values) => {
  const result = await mutUpdateXxx(props.id, updateFormat<UpdateXxxInput>({
    name: values.name,
    description: values.description,
  }, info || {}));
  if (result?.id) {
    props.onClose?.(true, result);
    message.success('保存成功');
  }
}}
```

---

## 二十七、ProTable 行选择与点击处理

### onMousedown 区分点击类型

```tsx
import { onMousedown } from '@/util';

<ProTable
  onRow={(record) => ({
    onMouseDown: (e) => {
      onMousedown({
        target: e.target as HTMLElement,
        exclusionClassNames: ['ant-checkbox', 'ant-table-row-expand-icon'],
        click: () => {
          // 单击：选中当前行
          setSelectedRowKeys(prev =>
            prev.includes(record.id) ? [] : [record.id]
          );
        },
        doubleClick: () => {
          // 双击：打开编辑弹窗
          handleEdit(record);
        },
        textSelection: () => {
          // 文本选中：不触发点击
        },
      });
    },
  })}
/>
```

### rowSelection 配置

```tsx
<ProTable
  rowSelection={{
    type: 'radio',  // 单选用 'radio'，多选用 'checkbox'
    selectedRowKeys,
    onChange: setSelectedRowKeys,
  }}
/>
```

---

## 二十八、树形数据结构处理

### formatTreeData - 扁平数组转树

```ts
import { formatTreeData } from '@/util';

// 将扁平列表转换为 Ant Design Tree 需要的树结构
const treeData = formatTreeData(allList, parentList, 'parentID');
// allList: 所有节点
// parentList: 顶级节点（可选，不传则自动识别）
// 'parentID': 父节点字段名（可选，默认 'parentID'）
```

### loopTreeData - 遍历树

```ts
import { loopTreeData } from '@/util';

// 遍历树找到目标节点并修改
loopTreeData(treeData, targetId, (node) => {
  node.name = newName;
});
```

### saveTreeData / delTreeData - 树形 CRUD

```ts
import { saveTreeData, delTreeData } from '@/util';

// 插入或更新节点
setTreeData(saveTreeData(treeData, newNode));

// 删除节点
setTreeData(delTreeData(treeData, nodeId));
```

### getTreeDropData - 拖拽排序

```ts
import { getTreeDropData } from '@/util';

const onDrop = (info: any) => {
  const result = getTreeDropData(treeData, {
    dragNode: info.dragNode,
    dropNode: info.node,
    dropPosition: info.dropPosition,
  });
  // result: { sourceId, targetId, action, newTreeData }
  setTreeData(result.newTreeData);
  // 调用 API 保存排序
  mutUpdateOrder(result.sourceId, { afterId: result.targetId });
};
```

---

## 二十九、开发检查清单

新建页面时逐项确认：

- [ ] 文件放在 `src/pages/模块/页面名/index.tsx`
- [ ] 三段式结构：业务组件 + 默认导出 + pageConfig
- [ ] `className="qeelyn-page-container"` 和 `<KeepAlive clearAlive>`
- [ ] `routeBreadcrumb()` 面包屑
- [ ] ProTable 使用 `search={{ className: 'qeelyn-pro-table-search' }}`
- [ ] 操作按钮用 `<Auth authKey="...">` 包裹
- [ ] 枚举映射在 `services/` 中导出（带 `text` 和 `tagColor`）
- [ ] GraphQL 函数遵循 `getXxx` / `mutXxx` 命名
- [ ] 服务函数添加 JSDoc 注释
- [ ] 运行 `pnpm gqlgen` 生成类型
- [ ] 新增模块时更新 `script/gqlgen.ts` 的 `documents` 配置
- [ ] UI 文案使用 `t('key')` 国际化
- [ ] 删除/危险操作用 `Modal.confirm` + Promise 模式
- [ ] **更新接口必须用 `updateFormat(values, info)` 处理**，不直接传整个表单对象
- [ ] `saveDataSource` / `delDataSource` 本地更新（不重新请求列表）
- [ ] `onMousedown` 处理行点击与文本选中冲突
- [ ] 字典下拉用 `getDictionaryValueList`（按项目实际字典服务替换）
- [ ] 非标准 Query/Mutation 按 Schema 定义参数
