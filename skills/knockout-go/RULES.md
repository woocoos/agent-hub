---
name: knockout-go-rules
description: Knockout.js 后端 Go 实现的具体代码审查规则,与 knockout-go SKILL 配套使用
skill: knockout-go
---

# Knockout-Go Rules

> 与 [knockout-go SKILL](SKILL.md) 配套使用的具体代码审查规则

## R1: 模型纯净性

**规则:** Go struct 中禁止定义可由前端 `computed` 计算的字段

**优先级:** HIGH

```go
// ❌ 错误:混入展示逻辑
type User struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    FullName string `json:"full_name"` // 应由前端 computed 计算
}

// ✅ 正确:保持模型纯净
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}
```

**检查项:**
- Struct 中是否存在 `FullName`、`DisplayName` 等派生字段
- 是否有字段可由其他字段组合而成

---

## R2: JSON Tag 规范

**规则:** 所有导出字段必须带 `json` tag,使用 `snake_case` 命名

**优先级:** CRITICAL

```go
// ❌ 错误:缺少 json tag 或使用 PascalCase
type Product struct {
    ID          int
    ProductName string
}

// ✅ 正确:使用 snake_case
type Product struct {
    ID          int    `json:"id"`
    ProductName string `json:"product_name"`
}
```

**检查项:**
- 导出字段是否都有 `json` tag
- Tag 值是否为 `snake_case`
- 是否有故意排除的字段使用 `json:"-"`

---

## R3: 敏感字段过滤

**规则:** 密码、密钥等敏感字段禁止在 JSON 响应中出现

**优先级:** CRITICAL

```go
// ❌ 错误:直接暴露敏感字段
type User struct {
    ID       int    `json:"id"`
    Password string `json:"password"` // 危险!
    APIKey   string `json:"api_key"`  // 危险!
}

// ✅ 正确:使用 json:"-" 排除
type User struct {
    ID       int    `json:"id"`
    Password string `json:"-"`
    APIKey   string `json:"-"`
}
```

**检查项:**
- 字段名包含 `password`、`secret`、`key`、`token` 的是否被排除
- 是否有 `MarshalJSON()` 方法实现自定义过滤

---

## R4: API 响应统一

**规则:** 所有 API 响应必须使用统一的 `APIResponse` 结构

**优先级:** HIGH

```go
// ✅ 标准响应格式
type APIResponse struct {
    Data   interface{} `json:"data"`
    Error  string      `json:"error,omitempty"`
    Meta   *MetaInfo   `json:"meta,omitempty"`
}

type MetaInfo struct {
    Total    int `json:"total"`
    Page     int `json:"page"`
    PageSize int `json:"page_size"`
}
```

**检查项:**
- 是否直接返回 struct 而非包装在 `APIResponse` 中
- 错误响应是否包含 `error` 字段
- 分页响应是否包含 `meta` 信息

---

## R5: 实时数据推送支持

**规则:** 需要实时更新的端点必须支持 SSE 或 WebSocket

**优先级:** MEDIUM

```go
// ✅ SSE 实现示例
func HandleDataStream(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming not supported", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    
    // 推送数据...
}
```

**检查项:**
- 响应头是否设置 `Content-Type: text/event-stream`
- 是否使用 `http.Flusher` 实现流式输出
- 是否正确处理 `r.Context().Done()` 关闭连接

---

## R6: 组件数据隔离

**规则:** 不同前端组件的 API 端点必须独立,禁止共享全局状态

**优先级:** HIGH

```go
// ❌ 错误:组件间共享状态
var GlobalData = make(map[string]interface{})

// ✅ 正确:每个组件独立端点
func SetupRoutes(mux *http.ServeMux) {
    mux.Handle("/api/users/", NewUserHandler())
    mux.Handle("/api/dashboard/stats", NewStatsHandler())
}
```

**检查项:**
- 是否有全局 `var` 被多个 handler 共享
- 路由是否按组件/功能分组

---

## R7: 批量操作支持

**规则:** 高频更新场景必须提供批量操作端点,禁止单条循环请求

**优先级:** MEDIUM

```go
// ❌ 错误:循环单条请求
for _, item := range items {
    http.Post("/api/update", "application/json", toJSON(item))
}

// ✅ 正确:批量端点
func HandleBatchUpdate(w http.ResponseWriter, r *http.Request) {
    var updates []UpdateRequest
    json.NewDecoder(r.Body).Decode(&updates)
    
    // 使用事务处理
    db.Transaction(func(tx *sql.Tx) error {
        for _, update := range updates {
            applyUpdate(tx, update)
        }
        return nil
    })
}
```

**检查项:**
- 是否有批量操作端点
- 是否使用数据库事务保证一致性

---

## R8: 请求追踪

**规则:** 所有 API 请求必须携带 `X-Request-ID` 用于调试追踪

**优先级:** HIGH

```go
// ✅ 中间件实现
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()
        r = r.WithContext(context.WithValue(r.Context(), "request_id", requestID))
        w.Header().Set("X-Request-ID", requestID)
        next.ServeHTTP(w, r)
    })
}
```

**检查项:**
- 是否实现 Request ID 中间件
- 日志中是否记录 Request ID

---

## R9: 输入验证

**规则:** 所有 API 输入必须验证,禁止直接使用未校验的请求体

**优先级:** CRITICAL

```go
// ❌ 错误:未验证直接使用
var req CreateRequest
json.NewDecoder(r.Body).Decode(&req)
db.Create(&req)

// ✅ 正确:验证后使用
var req CreateRequest
if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
    respondError(w, http.StatusBadRequest, "Invalid request")
    return
}
if err := req.Validate(); err != nil {
    respondError(w, http.StatusBadRequest, err.Error())
    return
}
```

**检查项:**
- 是否对 `r.Body` 进行验证
- 错误响应是否返回明确的验证信息

---

## R10: 增量更新优先

**规则:** 数据变更应使用增量更新而非全量刷新

**优先级:** MEDIUM

| 场景 | 全量刷新 (❌) | 增量更新 (✅) |
|------|-------------|-------------|
| 列表更新 | 重新获取所有记录 | 只推送变更的记录 |
| 状态变更 | 重新加载整个页面 | 推送状态变更事件 |

**检查项:**
- API 是否支持增量查询(如 `since_id`、`updated_after`)
- 是否使用 SSE/WebSocket 推送变更
