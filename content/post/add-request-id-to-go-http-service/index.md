---
title: "为 Go HTTP 服务加上 request_id：从中间件到统一响应"
date: 2025-03-14
lastmod: 2025-03-14
draft: false
tags: ["Go", "Gin", "可观测性", "中间件", "日志"]
categories: ["后端实践"]
description: "用一个 Gin 中间件为所有请求生成并透传 request_id，让它同时出现在响应头、响应体和结构化日志里，并处理客户端伪造 ID 带来的日志注入风险。"
toc: true
---

## 从一个真实的排查场景说起

用户反馈：“我刚才那次操作失败了。”

你打开日志，每秒几百行，滚动得眼睛发花。你不知道他的请求落在哪台机器、哪条日志、走了哪几个下游服务。这时候你需要的不是更详细的日志，而是一个**能把一次请求的所有痕迹串起来的锚点**——这就是 `request_id`。

一个设计良好的 `request_id` 应该出现在三个地方：

| 出口                     | 给谁看              | 作用                                   |
| ------------------------ | ------------------- | -------------------------------------- |
| 响应头 `X-Request-ID`    | 客户端、网关、Nginx | 跨服务串联链路，网关日志与业务日志对齐 |
| 响应体 `request_id` 字段 | 前端、客服          | 用户截图就能给出精确 ID，无需追问时间  |
| 日志属性 `request_id`    | 开发者              | 一次请求内的所有日志可以按 ID 聚合     |

本文以 Gin 为例，展示如何把这件事做对——**代码量不到 40 行，但有几个容易踩的坑**。

## 一、设计约定

在动手前先定三条规则，后面所有代码都围绕它们：

1. **中间件是唯一的生成点**。不要让每个 handler 各自去取或生成 ID，否则必然出现不一致。
2. **不信任客户端输入**。客户端传来的 ID 只是“建议值”，合法才透传，非法就丢弃重建。
3. **一份 ID，两个副本**：写进 Gin 上下文（内部用）和响应头（对外用）。

## 二、先解决包依赖：常量放哪里

很容易想到把 `RequestIDContextKey` 定义在 `middleware` 包，结果 `responses` 包要读上下文就不得不 import `middleware`，而 `middleware` 又要 import `responses` 来写响应——**循环依赖**。

解决办法是把这两个常量定义在更底层的 `responses` 包：

```go
package responses

const (
	// RequestIDContextKey 是 request_id 在 Gin 上下文中的键名。
	RequestIDContextKey = "request_id"
	// HeaderRequestID 是请求 ID 的响应头名称。
	HeaderRequestID = "X-Request-ID"
)
```

依赖方向变成单向的 `middleware → responses`，干净。这是一个通用技巧：**共享常量要放在依赖链最低端的包**。

## 三、核心中间件

```go
func RequestID() gin.HandlerFunc {
	return func(c *gin.Context) {
		id := sanitizeRequestID(c.GetHeader(responses.HeaderRequestID))
		if id == "" {
			id = uuid.NewString()
		}
		c.Set(responses.RequestIDContextKey, id)              // 写上下文，服务内部用
		c.Writer.Header().Set(responses.HeaderRequestID, id)  // 写响应头，对外透传
		c.Next()
	}
}
```

逻辑就三步：

1. 读客户端传来的 `X-Request-ID`；
2. 校验不通过就 `uuid.NewString()` 重新生成；
3. 把最终的 ID **同时写进两处**。

注意第 3 步的两个副本缺一不可：`c.Set` 的值不会自动进 HTTP 头，而响应头也不会自动进入 Gin 上下文。

## 四、安全：客户端传来的 ID 必须过滤

这是最容易被忽略、也最关键的一步。`request_id` 最终会进日志，如果原样接受客户端输入，就打开了一个**日志注入（log injection）**的口子：

```
X-Request-ID: admin\n2026-10-14 INFO user=root password=123456
```

攻击者可以伪造出以假乱真的日志行，污染你的日志分析系统。所以要做字符集和长度双校验：

```go
// requestIDMaxLen 限制请求 ID 长度，避免超长请求头进入日志与存储。
const requestIDMaxLen = 128

// requestIDPattern 限定请求 ID 的字符集，杜绝换行等可用于日志注入的字符。
var requestIDPattern = regexp.MustCompile(`^[A-Za-z0-9._-]+$`)

// sanitizeRequestID 返回合法的请求 ID，非法输入返回空串表示应重新生成。
func sanitizeRequestID(raw string) string {
	trimmed := strings.TrimSpace(raw)
	if trimmed == "" || len(trimmed) > requestIDMaxLen {
		return ""
	}
	if !requestIDPattern.MatchString(trimmed) {
		return ""
	}
	return trimmed
}
```

几个细节：

- **用白名单而不是黑名单**：只放 `[A-Za-z0-9._-]`，而不是“过滤掉换行”。黑名单永远想不全。
- **`strings.TrimSpace` 在前**：纯空格 `"   "` 会被判为空，从而重新生成，而不是变成一个空格 ID。
- **长度上限**：防止超长头撑爆日志和存储。
- **返回空串表达“无效”**，让调用方统一走生成分支，语义清晰。

正则用 `regexp.MustCompile` 在包初始化时编译一次，不要放进热路径重复编译。

## 五、让响应体也带上 ID

统一响应结构，成功和失败都带 `request_id`：

```go
type Envelope struct {
	Data      any    `json:"data,omitempty"`
	RequestID string `json:"request_id"`
}

type ErrorEnvelope struct {
	Error     ErrorBody `json:"error"`
	RequestID string    `json:"request_id"`
}

func OK(c *gin.Context, data any) {
	c.JSON(http.StatusOK, Envelope{Data: data, RequestID: requestID(c)})
}

// Fail 以指定状态码返回统一格式的失败响应，并保证响应头携带同一 request_id。
func Fail(c *gin.Context, status int, code, message string) {
	id := requestID(c)
	if id != "" {
		c.Writer.Header().Set(HeaderRequestID, id) // 错误路径也要补写响应头
	}
	c.JSON(status, ErrorEnvelope{
		Error:     ErrorBody{Code: code, Message: message},
		RequestID: id,
	})
}
```

取值用了一个**防御式兜底**函数：

```go
// requestID 依次从 Gin 上下文与响应头中取请求 ID，取不到时返回空串。
func requestID(c *gin.Context) string {
	if id := c.GetString(RequestIDContextKey); id != "" {
		return id
	}
	return c.Writer.Header().Get(HeaderRequestID)
}
```

为什么需要兜底？因为中间件可能被漏挂——比如某个路由组忘了注册、或在单元测试里裸调 handler。先取上下文、取不到再回退响应头，保证**即使在降级路径下 ID 也不会丢**。

`Fail` 里额外 `Set` 响应头这一步值得留意：正常流程中中间件已经写过了，但这能覆盖“中间件没跑”或“响应头被中间件改写”的边界情况。

## 六、日志中间件从上下文读

```go
func Logging() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next()

		attrs := []any{
			"method", c.Request.Method,
			"path", requestPath(c),
			"status", c.Writer.Status(),
			"latency_ms", time.Since(start).Milliseconds(),
			"request_id", c.GetString(responses.RequestIDContextKey),
			"client_ip", c.ClientIP(),
		}
		// ...
		slog.Log(c.Request.Context(), level, "http request", attrs...)
	}
}
```

这里用 `c.GetString(...)` 从 Gin 上下文读，而不是去 `c.Writer.Header().Get(...)`——**从内存 map 读比解析响应头更直接，也是 Gin 传递请求级数据的惯用法**。

顺带一个日志质量技巧：路径用路由模板而不是真实 URL。

```go
func requestPath(c *gin.Context) string {
	if full := c.FullPath(); full != "" {
		return full // /users/:id，低基数
	}
	return c.Request.URL.Path
}
```

否则 `/users/1`、`/users/2`… 会变成高基数标签，拖垮日志聚合系统。

## 七、注册顺序

```go
r := gin.New()
r.Use(gin.Recovery(), middleware.RequestID(), middleware.Logging())
```

需要遵守这个顺序：

> 中间件顺序固定为 RequestID → Logging，保证日志与错误响应都能携带 request_id。

这里有个**值得说透的细节**：由于 `Logging` 是在 `c.Next()` **之后**才输出日志，即使把 `Logging` 注册在 `RequestID` 之前，日志里其实也能读到 ID——因为 `RequestID` 的 `c.Set` 发生在 `Logging` 调用的 `c.Next()` 嵌套链里，早已执行完毕。

那为什么还要固定顺序？因为**“先生成、再使用”是最不容易被后续重构破坏的约定**。一旦有人把 `Logging` 改成请求前也打一条日志，或者新增一个在 `c.Next()` 之前就要用 ID 的中间件，顺序错了就会静默地拿到空字符串——这类 bug 极难排查。把它放在最前面，成本为零，收益是确定性。

## 八、一个高频疑问：两处取值是同一个东西吗

经常有人问：

```go
c.GetString(RequestIDContextKey)              // ?
c.Writer.Header().Get(HeaderRequestID)        // ?
```

**不是同一个东西，只是在正常流程下值相等。**

|                  | `c.GetString(RequestIDContextKey)` | `c.Writer.Header().Get(HeaderRequestID)` |
| ---------------- | ---------------------------------- | ---------------------------------------- |
| 存储             | Gin 上下文 `Keys` map（进程内存）  | HTTP 响应头缓冲区                        |
| 键               | `"request_id"`                     | `"X-Request-ID"`                         |
| 写入             | `c.Set(...)`                       | `c.Writer.Header().Set(...)`             |
| 是否会发给客户端 | 否                                 | 是                                       |
| 典型读取方       | 日志、业务代码                     | 客户端、网关、测试断言                   |

它们的关系是：**中间件这个唯一写入点把同一个值写进了两个地方**，所以看起来一致。既然是两个独立副本，就可能分叉——某处只改了响应头没改上下文，或中间件压根没执行，二者就不同了。这正是 `requestID()` 需要兜底的原因。

理解这一点，比死记“它们相等”重要得多。

## 九、测试要覆盖哪些分支

中间件虽小，边界不少，建议这几条用例：

| 用例       | 输入       | 期望                             |
| ---------- | ---------- | -------------------------------- |
| 缺失时生成 | 无头       | 返回 UUID 格式（按 `-` 分 5 段） |
| 合法值透传 | `abc-123`  | 原样返回                         |
| 拒绝注入   | 含 `\r\n`  | 重新生成，且结果不含换行         |
| 超长截断   | > 128 字符 | 重新生成，长度 ≤ 128             |
| 纯空白     | `"   "`    | 视为空，重新生成                 |

测试里断言响应头即可，无需暴露内部实现：

```go
rec := doGet(t, newTestRouter(), "abc-123")
assert.Equal(t, "abc-123", rec.Header().Get(responses.HeaderRequestID))
```

## 十、小结

把 `request_id` 做对，总结成几条可复用的经验：

1. **单一生成点**：用一个中间件搞定，别散落在 handler 里。
2. **三个出口**：响应头给网关、响应体给用户、日志给开发者。
3. **两个副本**：Gin 上下文供内部、响应头供外部，中间件负责同步。
4. **绝不信任客户端输入**：白名单字符集 + 长度上限，防日志注入。
5. **防御式取值**：上下文优先、响应头兜底，降级路径也不丢 ID。
6. **顺序显式化**：生成中间件放最前，避免重构踩坑。
7. **共享常量下沉**：放在依赖链最低端，规避循环 import。

最终收益是：用户报障时给你一个 ID，你 `grep` 一次就能看到这次请求的完整生命周期。这 40 行代码，是整个可观测性体系里性价比最高的一笔投资。
