---
title: "Go HTTP 服务的优雅关闭：从一次容器重启引发的 502 说起"
date: 2025-03-21
lastmod: 2025-03-21

draft: false
tags: ["Go", "net/http", "Gin", "Docker Compose", "优雅关闭"]
categories: ["后端"]
summary: "优雅关闭不是一个可选的美化项，而是服务能否安全上线的分水岭。本文以一个真实的 Go + Gin 服务骨架为例，拆解 signal.NotifyContext、http.Server 超时、Shutdown 宽限期与资源释放顺序，并给出 Docker Compose 侧的配套配置与一份可自查的陷阱清单。"
---

## 起因：那些只在发布时出现的 502

你有没有遇过这种情况：服务本地跑得好好的，压测也没问题，唯独每次 `docker compose up -d` 重新部署、或者 `docker compose restart`，监控上总会冒出一小撮 502 和 `connection reset by peer`。量不大，持续时间也就几秒，但它精确地出现在每次重启窗口里。

原因通常不在业务代码，而在于进程是怎么**被杀掉的**。

Docker 停止一个容器的流程是：先向容器内 PID 1 发 `SIGTERM`，等一段时间（`docker stop` 默认 10s，Compose 里由 `stop_grace_period` 控制），超时再发 `SIGKILL`。如果你的进程对 `SIGTERM` 没有任何处理，Go 运行时的默认行为就是**立即终止**——不管此刻还有多少个请求正卡在数据库查询里，也不管连接池里还有多少未提交的事务。

于是就有了那几秒的 502。

优雅关闭（Graceful Shutdown）要解决的正是这件事：**收到停止信号后，先停止接收新请求，把在途请求处理完，再释放资源退出。**

## 一、先搞清楚 Shutdown 到底做了什么

Go 1.8 引入的 `http.Server.Shutdown(ctx)` 行为非常明确，可以拆成三步：

1. **立刻关闭所有监听 socket**——新连接会被拒绝（通常是 `connection refused`，客户端/网关会重试到新实例）；
2. **把所有空闲连接置为可关闭状态**——keep-alive 但没有正在处理请求的连接直接断开；
3. **无限期等待活跃连接变为空闲**（直到 handler 返回），只有在 `ctx` 到期时才强制中断。

注意第三步的措辞：**Shutdown 会一直等，是 `ctx` 给了它上限**。所以那个 `context.WithTimeout` 不是装饰品，它是唯一的兜底。

还有两个容易被忽略的边界：

- `Shutdown` **不会**关闭被 hijack 的连接（WebSocket、SSE、`http.Hijacker`）。这些连接脱离了 `net/http` 的生命周期管理，必须自己维护注册表并主动关闭，否则 `Shutdown` 会一直挂到超时。
- `Shutdown` 也**不会**关闭你的数据库连接池、Redis 客户端、消息队列消费者。它只管 HTTP 层。

这两点决定了优雅关闭从来不是调一个 `Shutdown` 就完事的。

## 二、为什么不能直接用 Gin 的 `r.Run()`

很多 Go Web 项目是这么写的：

```go
r := gin.Default()
r.GET("/ping", handler)
r.Run(":8080")
```

这段代码能跑，但它在生产环境有两个硬伤。

`r.Run()` 内部等价于：

```go
err = http.ListenAndServe(address, engine)
// 而 ListenAndServe 等价于：
// srv := &http.Server{Addr: addr, Handler: handler}
// return srv.ListenAndServe()
```

这个隐式创建的 `http.Server`，**所有超时字段都是零值**，而且你**拿不到它的指针**。

| 能力                              | `r.Run(addr)`              | `&http.Server{...}`           |
| --------------------------------- | -------------------------- | ----------------------------- |
| 读/写/空闲超时                    | 无（零值，慢连接攻击风险） | 可精细配置                    |
| 优雅关闭                          | 做不到（拿不到句柄）       | `srv.Shutdown(ctx)`           |
| 启动失败的错误传递                | 内部打印，`main` 拿不到    | 错误可回传，`main` 决定退出码 |
| TLS / `ConnContext` / `ConnState` | 需换专用方法且仍无超时     | 同一结构体配置                |

这里要澄清一个常见的层次误解：**Gin 和 `net/http.Server` 不是替代关系**。Gin 是路由 + 中间件 + 上下文/bindings 的应用层框架，`net/http.Server` 是连接管理、超时、TLS、生命周期的传输层。而 `*gin.Engine` 实现了 `ServeHTTP` 方法，也就是 `http.Handler` 接口——所以它可以被直接塞进 `http.Server.Handler`。

**用 `http.Server` 装载 Gin Engine，是 Go 生态的标准姿势**，Gin 官方文档的优雅关闭示例也是这么写的。

## 三、一份完整的实现，逐段拆解

下面是我们服务 `cmd/api/main.go` 的真实代码（略有精简）。

### 3.1 main 只负责退出码

```go
func main() {
	if err := run(); err != nil {
		slog.Error("service failed to start", "error", err)
		os.Exit(1)
	}
}
```

`main` 极薄，逻辑全在 `run() error`。这样做的收益是：错误沿调用栈向上传播，退出码**只在 `main` 里决定一次**。对容器化部署来说这很重要——启动失败必须非零退出，Compose 的 `restart` 策略才会生效，监控也才能报警。

### 3.2 超时常量：安全基线而非优化项

```go
// HTTP 服务器超时参数，防止慢连接耗尽服务资源。
const (
	readHeaderTimeout = 5 * time.Second
	readTimeout       = 15 * time.Second
	writeTimeout      = 15 * time.Second
	idleTimeout       = 60 * time.Second
)
```

- `ReadHeaderTimeout`：从接受连接到请求头读完的上限。**没有它，服务就暴露在 Slowloris 慢连接攻击之下**——客户端每秒发一个字节，连接长期占用，最终耗尽文件描述符和 goroutine。
- `ReadTimeout`：从连接建立到请求体读完。注意它会覆盖整个请求体上传时间，有文件上传接口时要单独放宽。
- `WriteTimeout`：从请求头读完到响应写完。它同时是长任务（SSE、大文件下载）的天花板，需要按业务调整。
- `IdleTimeout`：keep-alive 空闲连接的存活时间。**这个值必须和上游网关（Traefik / Nginx）的 idle timeout 对齐**，否则后端先关连接、网关复用了一个刚关闭的连接，就会产生随机 502。

### 3.3 用 http.Server 装载 Gin Engine

```go
srv := &http.Server{
	Addr:              cfg.HTTPAddr,
	Handler:           iamhttp.NewRouter(app),
	ReadHeaderTimeout: readHeaderTimeout,
	ReadTimeout:       readTimeout,
	WriteTimeout:      writeTimeout,
	IdleTimeout:       idleTimeout,
}
```

`NewRouter` 返回 `*gin.Engine`。这里还有一处细节值得学习——`NewRouter` 内部用的是 `gin.New()` 而不是 `gin.Default()`，自己显式挂载中间件：

```go
r := gin.New()
r.Use(gin.Recovery(), middleware.RequestID(), middleware.Logging())
```

**拒绝框架默认值、显式掌控每一层配置**，这个思路从中间件层一直贯穿到传输层，是这份代码最一致的设计取向。

### 3.4 启动 goroutine + 错误通道

```go
serverErr := make(chan error, 1)
go func() {
	slog.Info("http server listening", "addr", cfg.HTTPAddr, "env", cfg.AppEnv)
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		serverErr <- err
		return
	}
	serverErr <- nil
}()
```

三个关键点：

1. `ListenAndServe` 是阻塞的，必须放进 goroutine，主线程才能同时监听信号。
2. **`errors.Is(err, http.ErrServerClosed)` 的过滤是必须的**。`Shutdown()` 触发的关闭会让 `ListenAndServe` 返回 `http.ErrServerClosed`，这是**预期行为**不是错误。不过滤，就会在每次优雅关闭时误报一次失败。
3. **通道缓冲为 1**。保证即使 `main` 已经返回、没人再接收，这个 goroutine 也不会卡在发送上——这是一个很小但很实在的防泄漏措施。

### 3.5 信号与退出路径的汇合

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

select {
case err := <-serverErr:
	if err != nil {
		return fmt.Errorf("HTTP 服务异常退出：%w", err)
	}
	return nil
case <-ctx.Done():
	shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("优雅关闭失败：%w", err)
	}
	return nil
}
```

`signal.NotifyContext` 比老式的 `signal.Notify(ch)` 更省心：自动 `stop()`、直接拿到 `context`、能无缝接入 `select`。

把「服务器自己挂了」和「收到信号要退出」两条路径放进同一个 `select`，是这个模式的精髓——**退出原因被集中管理，没有遗漏的分支**。

`defer stop()` 另有一个实用价值：它恢复了默认的信号处理行为。如果用户按下第二次 `Ctrl+C`，进程会被直接杀掉，而不是继续卡在漫长的优雅关闭里。这是很好的逃生舱设计。

### 3.6 资源释放

```go
app, err := bootstrap.New(cfg)
if err != nil {
	return err
}
defer func() {
	if err := app.Close(); err != nil {
		slog.Error("failed to close app resources", "error", err)
	}
}()
```

`app.Close()` 负责关闭数据库连接池。用 `defer` 保证无论从哪条路径退出（启动失败、服务崩溃、正常关闭），资源都会被释放。

有一点值得进一步打磨：`defer` 在 `run()` 返回时才执行，顺序上恰好是对的（HTTP 已 Shutdown → 再关 DB）。但语义上更严谨的做法是用 `srv.RegisterOnShutdown(app.Close)`，让「连接全部断开」成为「释放 DB」的显式前置条件。

## 四、Docker Compose 侧的配套配置

代码写对了还不够。有一半的坑在容器编排层。

### 4.1 时间预算必须对齐

```text
stop_grace_period = 40s
└── 必须 > APP_SHUTDOWN_TIMEOUT + 资源释放耗时
```

`docker stop` 以及 `docker compose stop` / `up` 的默认宽限期只有 **10 秒**，超时就是 `SIGKILL`。这里藏着一个很容易忽略的巧合：我们 `APP_SHUTDOWN_TIMEOUT` 的默认值**也恰好是 10s**。两个 10s 打平意味着一点余量都没有——只要 `Shutdown` 卡在某个慢查询上，进程就会被强杀，优雅关闭形同虚设。

所以要在 compose 文件里把两层都显式写出来，并让外层明显大于内层：

```yaml
api:
  build: .
  entrypoint: ["/iam-api"]
  environment:
    APP_ENV: production
    APP_HTTP_ADDR: :8080
    # 应用层宽限期，必须小于 stop_grace_period
    APP_SHUTDOWN_TIMEOUT: 25s
    DATABASE_DSN: postgres://postgres:postgres@postgres:5432/iam?sslmode=disable
    LOG_LEVEL: info
  ports:
    - "8080:8080"
  # 外层兜底：给 Shutdown 与资源释放留足余量
  stop_grace_period: 40s
  restart: unless-stopped
```

**超时参数必须由同一份时间预算推导出来，而不是各自拍脑袋。**

### 4.2 信号必须真的到达你的进程

Compose 场景下一个高频事故：容器里的 PID 1 不是你的 Go 程序。

如果写了 `ENTRYPOINT ["/bin/sh", "-c", "..."]` 或 `command: sh -c "..."` 这种 shell 形式，PID 1 就是 shell。而 shell 默认**不会把 `SIGTERM` 转发给子进程**，也不会等它退出——表现就是宽限期一到直接 `SIGKILL`，优雅关闭一次都没执行过。

我们的 Dockerfile 用的是 exec 形式的 `ENTRYPOINT`，二进制自己就是 PID 1，信号直达：

```dockerfile
# exec 形式：二进制即 PID 1，直接收到 SIGTERM
ENTRYPOINT ["/iam-api"]
```

如果确实需要 shell 做启动前的准备工作，用 `exec` 结尾把进程替换掉：`sh -c "... ; exec /iam-api"`。另外确认 `stop_signal` 没有被改掉（默认即为 `SIGTERM`）。

### 4.3 上游摘流与 keep-alive 是异步的

`docker compose up -d` 重建容器时，旧容器停止与新容器就绪之间天然存在一个窗口；如果前面还挂了 Traefik / Nginx 做反向代理，问题会放大：

- 反代的健康检查是**周期性**的，它不会在旧容器收到 `SIGTERM` 的瞬间立刻摘掉这台上游；
- 反代到后端的 **keep-alive 长连接**可能正指向旧容器，`Shutdown` 关闭监听后，反代若复用了这条刚被关闭的连接，就会产生 `connection reset by peer`。

Kubernetes 里这类问题靠 `preStop` sleep 缓解，而 **Compose 没有 preStop hook**。更干净的做法是让应用自己"装死"：**收到 `SIGTERM` 后立即让 `/health/ready` 返回 503**——在信号处理器里置一个原子标志位即可，反代下一轮探测就会把流量摘走，比固定 sleep 更精确也更省时间。

同时把两边的空闲超时对齐（见 3.2）：反代的 `keepalive` / proxy 空闲时间不要大于后端的 `IdleTimeout`，否则会变成随机 502。

### 4.4 别把 Compose 的 healthcheck 当成 k8s 探针

Compose 的 `healthcheck` 与 Kubernetes 探针的语义差别很大，很容易想当然：

```text
/health/live   不触碰任何依赖，恒 200
/health/ready  校验 DB、迁移状态、Casbin 策略，失败返回 503
```

- `depends_on: {condition: service_healthy}` 会读 healthcheck 结果，这是它最主要的用途——比如我们的 `migrate` 就是等 Postgres 健康之后才执行；
- 但 **Compose 不会因为 healthcheck 失败而重启容器**，`restart` 策略只看进程是否退出。想要"不健康就重启"，需要自愈容器（如 autoheal）或交给上层编排；
- healthcheck **不会自动摘流量**，摘流是反代的职责（对应 4.3）。

即便如此，两个探针仍必须分开：`live` 用于判断"要不要重启"，`ready` 用于判断"能不能接流量"。如果合并成一个会真实校验数据库的探针，一次数据库抖动就会让所有实例同时被判定为不可用，雪崩。

还有一个工程细节：我们的运行镜像是 `distroless/static`，容器里没有 shell，也没有 `curl` / `wget`，所以 `test: ["CMD-SHELL", "curl -f http://localhost:8080/health/ready"]` 这种常见写法**根本跑不起来**。可行做法是给二进制加一个一次性自检子命令作为 healthcheck 入口，或者用一个外部探测容器从网络侧打 `/health/ready`。

## 五、陷阱清单

按被踩频率排序，可以当自查表用：

| #   | 陷阱                                    | 后果                     | 解法                                   |
| --- | --------------------------------------- | ------------------------ | -------------------------------------- |
| 1   | 不过滤 `http.ErrServerClosed`           | 每次优雅关闭都误报失败   | `errors.Is(err, http.ErrServerClosed)` |
| 2   | signal channel 无缓冲                   | goroutine 泄漏           | 缓冲设为 1                             |
| 3   | 只 `Shutdown` 不关 DB/Redis/MQ          | 连接泄漏、事务中断       | `defer` + 资源统一 `Close()`           |
| 4   | `Shutdown` 时间超过 `stop_grace_period` | 一定会被 `SIGKILL`       | 时间预算统一推导                       |
| 5   | 零值超时                                | Slowloris 攻击面         | 显式设置四个超时                       |
| 6   | WebSocket / SSE 连接                    | `Shutdown` 挂到超时      | 自行维护连接注册表并主动关闭           |
| 7   | 网关 idle timeout 与后端不一致          | 随机 502                 | 两边对齐                               |
| 8   | 反代摘流与 keep-alive 异步              | 关闭期仍收到新请求       | 收到 `SIGTERM` 即让 ready 返回 503     |
| 9   | 后台 goroutine 无退出通道               | 关闭期仍在写已关闭的资源 | 用 `context` 串联所有后台任务          |
| 10  | 二次信号无法强制退出                    | 卡死的关闭无法被打断     | `defer stop()` 恢复默认行为            |
| 11  | shell 形式 `ENTRYPOINT` 占用 PID 1      | `SIGTERM` 到不了进程     | exec 形式，或 `sh -c "...; exec app"`  |

## 六、最小可用模板

如果你只想抄一段，这是去掉业务细节后的骨架：

```go
func run() error {
	srv := &http.Server{
		Addr:              ":8080",
		Handler:           router,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	serverErr := make(chan error, 1)
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			serverErr <- err
			return
		}
		serverErr <- nil
	}()

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	select {
	case err := <-serverErr:
		return err
	case <-ctx.Done():
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		return srv.Shutdown(shutdownCtx)
	}
}
```

## 小结

优雅关闭的本质是**把「进程何时退出」的控制权，从操作系统手里拿回到自己手里**。

它需要三层配合，缺一不可：

1. **传输层**：显式配置 `http.Server` 的超时，拿到句柄才能 `Shutdown`；
2. **应用层**：所有后台任务与资源用 `context` 串联，可中断、可释放；
3. **编排层**：`stop_grace_period`、healthcheck、反代摘流策略与代码里的超时参数必须来自同一份时间预算，并且要保证 `SIGTERM` 真的能送到你的进程手里。

回头看，这件事的成本不过是十来行代码。但它决定的是：你的服务在每次发布时，是悄无声息地完成交接，还是在监控图上留下一道刺眼的红色尖峰。
