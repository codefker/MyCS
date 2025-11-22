这是一份关于 Go 1.21+ 标准库 `log/slog` 的深度指南。

---

# 深度解析 Go 1.21+ 标准库：log/slog

自 Go 1.21 版本发布以来，Go 社区终于迎来了官方支持的**结构化日志（Structured Logging）**标准库：`log/slog`。

在此之前，Go 的标准库 `log` 仅提供简单的文本输出，导致社区中涌现了大量第三方库（如 Zap, Logrus, Zerolog）以满足结构化需求。`slog` 的出现旨在**统一日志前端接口**，让库作者可以使用标准 API 记录日志，而让最终用户决定日志的具体输出格式和后端。

---

## 1. 设计哲学与核心目标

理解 `slog` 的设计思维有助于更好地使用它：

1.  **结构化（Structured）**：日志不仅仅是给人类看的文本，更是给机器看的（便于索引、搜索、分析）。`slog` 将日志视为 "消息 + 键值对"。
2.  **高性能（High Performance）**：借鉴了 Zap 和 Zerolog 的经验，`slog` 在设计上尽量减少内存分配（Allocations），特别是在使用强类型属性时。
3.  **统一接口（Unified Interface）**：最大的野心是建立生态标准。库开发者（Library Authors）只需依赖 `slog` 接口，应用程序开发者（Application Developers）可以通过 `Handler` 决定日志是输出 JSON、文本，还是发送到 Kafka/Elasticsearch。
4.  **向后兼容与互操作**：它能够与老旧的 `log` 包无缝协作。

---

## 2. 核心架构概念

`slog` 的架构主要由三个部分组成：

### 2.1 Logger (前端)
`Logger` 是用户直接调用的结构体。它负责接收日志请求，处理上下文（Context），并将其传递给 Handler。
*   它提供了 `Info`, `Error`, `Debug`, `Warn` 等便捷方法。

### 2.2 Record (数据载体)
`Record` 代表一条日志记录。它包含了日志发生的时间、等级（Level）、消息（Message）、PC（程序计数器，用于获取源码位置）以及上下文属性（Attributes）。
*   `Record` 是设计为值类型传递的，以优化内存。

### 2.3 Handler (后端/处理逻辑)
`Handler` 是一个接口，负责将 `Record` 渲染成最终格式或发送到目的地。
*   Go 内置了两个 Handler：
    *   `TextHandler`: 输出 `key=value` 格式（类似 Logfmt）。
    *   `JSONHandler`: 输出 JSON 格式。

---

## 3. 基础 API 与使用

### 3.1 最简单的 Hello World

```go
package main

import (
	"log/slog"
)

func main() {
	// 默认使用 TextHandler，输出到 Stderr
	slog.Info("Hello World", "user", "gopher", "id", 123)
}
// 输出: 2023/10/27 10:00:00 INFO Hello World user=gopher id=123
```

### 3.2 两种风格的 API

`slog` 提供了两种传递属性（Attributes）的方式：

1.  **交替键值对（Loosely Typed）**：
    方便，但在运行时需要检查类型，稍慢，且容易写错（比如少写一个 Value）。
    ```go
    slog.Info("message", "key1", "value1", "key2", 2)
    ```

2.  **强类型属性（Strongly Typed）**：
    **推荐用于生产环境**。使用 `slog.String`, `slog.Int` 等构造函数。性能最高，类型安全。
    ```go
    slog.Info("message", slog.String("key1", "value1"), slog.Int("key2", 2))
    ```

### 3.3 日志等级 (Levels)

默认有四个等级：`Debug` (-4), `Info` (0), `Warn` (4), `Error` (8)。
*   设计为整数间隔，允许用户在中间插入自定义等级（例如 `Trace` 可以定义为 -8，`Fatal` 定义为 12）。

---

## 4. 进阶特性：属性与分组

### 4.1 Attr 与 Value
`slog.Attr` 是 Key-Value 对。
`slog.Value` 是对任何 Go 值的低分配包装器。它能高效处理基本类型，对于复杂类型则回退到 `any`。

### 4.2 Group (分组)
可以将相关的属性组合在一起，这在 JSON 输出中非常有用（会形成嵌套对象）。

```go
slog.Info("request completed",
    slog.Group("http",
        slog.String("method", "GET"),
        slog.Int("status", 200),
        slog.String("url", "/api/v1/user"),
    ),
)
```
**JSON 输出效果：**
```json
{
  "time": "...",
  "level": "INFO",
  "msg": "request completed",
  "http": {
    "method": "GET",
    "status": 200,
    "url": "/api/v1/user"
  }
}
```

### 4.3 LogValuer 接口 (自定义序列化)
如果你的结构体包含敏感信息（如密码）或复杂的内部状态，你可以实现 `LogValuer` 接口来控制它如何被记录。这类似于 `json.Marshaler`。

```go
type User struct {
    ID       int
    Password string
}

func (u User) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Int("id", u.ID),
        slog.String("password", "***MASKED***"), // 自动脱敏
    )
}

// 使用
u := User{ID: 1, Password: "secret"}
slog.Info("login", "user", u)
```

---

## 5. 深度定制：Handler 与 Options

这是 `slog` 最强大的地方。

### 5.1 配置内置 Handler

通过 `HandlerOptions` 可以控制日志细节：
*   `Level`: 设置最小日志等级（支持动态修改）。
*   `AddSource`: 是否添加源码位置（文件名:行号）。
*   `ReplaceAttr`: 一个极其强大的钩子函数，用于在写入前修改或重写属性。

**示例：创建一个 JSON Logger，开启 Debug 模式，并标准化时间格式**

```go
import (
	"context"
	"log/slog"
	"os"
	"time"
)

func main() {
	opts := &slog.HandlerOptions{
		Level: slog.LevelDebug, // 开启 Debug
		AddSource: true,        // 显示源码位置

		// ReplaceAttr 钩子
		ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
			// 修改 Time 字段的输出格式
			if a.Key == slog.TimeKey {
				t := a.Value.Time()
				return slog.String(a.Key, t.Format(time.RFC3339))
			}
			// 可以在这里统一修改字段名，例如把 "msg" 改为 "message"
			if a.Key == slog.MessageKey {
				return slog.Attr{Key: "message", Value: a.Value}
			}
			return a
		},
	}

	logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))

	// 设置为全局默认 logger
	slog.SetDefault(logger)

	slog.Debug("System starting...")
}
```

### 5.2 Logger.With (预设属性)
如果你有一批日志都需要带上某些字段（例如 `service_name`, `environment`），使用 `With` 可以创建一个带有预设属性的新 Logger，且性能极高。

```go
// 创建一个带有公共字段的 logger
reqLogger := logger.With(
    slog.String("service", "payment"),
    slog.String("env", "prod"),
)

reqLogger.Info("transaction started") // 自动包含 service 和 env
```

---

## 6. 上下文集成 (Context)

在 Go 中，`context` 是传递请求范围数据的标准方式（如 TraceID）。
`slog` 提供了 `InfoContext`, `ErrorContext` 等方法。

**注意**：默认的 `TextHandler` 和 `JSONHandler` **不会**自动从 Context 中提取数据打印。你需要编写中间件或自定义 Handler 来实现 TraceID 的自动注入。

**最佳实践：从 Context 提取 TraceID**

你需要自定义 Handler 的 `Handle` 方法：

```go
type TraceHandler struct {
    slog.Handler
}

func (h *TraceHandler) Handle(ctx context.Context, r slog.Record) error {
    // 假设 traceID 存在于 ctx 中
    if id, ok := ctx.Value("trace_id").(string); ok {
        r.AddAttrs(slog.String("trace_id", id))
    }
    return h.Handler.Handle(ctx, r)
}
```

---

## 7. 性能与最佳实践

### 7.1 使用 IsEnabled 避免昂贵计算
虽然 `slog` 很快，但构建参数也需要开销。如果某个日志级别未开启，通过检查可以避免参数构造。

```go
if logger.Enabled(ctx, slog.LevelDebug) {
    // heavyCalculation() 只有在 Debug 开启时才会执行
    logger.Debug("result", "val", heavyCalculation())
}
```

### 7.2 避免使用 slog.Any
尽量使用 `slog.String`, `slog.Int` 等明确类型的构造函数。`slog.Any` 会导致反射（Reflection）和逃逸分析带来的堆分配，影响性能。

### 7.3 静态属性前置
对于整个应用生命周期不变的属性（如 `pid`, `hostname`, `version`），在程序启动时使用 `logger.With()` 注入，而不是在每条日志里重复写。

---

## 8. 与旧生态的兼容

### 8.1 接管标准库 log
你可以让 `slog` 接管老旧的 `log.Printf` 输出，实现全项目日志格式统一。

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger) // 将 slog 设置为默认

// 标准库 log 会自动桥接到 slog 的 Handler
log.Printf("This will be formatted as JSON now")
```

### 8.2 第三方库桥接
目前主流的日志库（Zap, Zerolog）都在开发或已经提供了适配 `slog.Handler` 的桥接器。这意味着你可以继续使用 Zap 的后端逻辑，但使用 `slog` 的前端 API。

---

## 9. 常见陷阱与注意事项

1.  **格式化字符串**：`slog` **不支持** `printf` 风格的格式化（如 `%s`, `%d`）。
    *   错误：`slog.Info("User %s logged in", user)`
    *   正确：`slog.Info("User logged in", "user", user)`
    *   原因：这是为了强制实施结构化设计，方便机器解析。

2.  **Key 冲突**：如果你在一个记录中重复使用了相同的 Key，`slog` 默认会全部保留（取决于 Handler 实现），这在 JSON 中可能导致解析歧义。

3.  **Lazy Evaluation**：`slog` 没有像 Zap 那样内置显式的 `ObjectMarshaler` 这种极度懒加载机制（除 `LogValuer` 外）。参数在传递给 `Info` 函数时就已经求值了。

---

## 10. 总结

Go 1.21 的 `log/slog` 是一个里程碑。

*   **对于库作者**：请立即停止在你的库中使用 `zap` 或 `logrus`，改用 `log/slog`。这让你的库对调用者更加友好。
*   **对于应用开发者**：
    *   新项目：直接使用 `slog`。
    *   旧项目：可以考虑逐步迁移，或者通过 Adapter 将现有的日志库后端对接到 `slog` 前端。

它的设计在易用性（类似 `fmt.Print`）和高性能结构化（类似 `zap`）之间找到了极佳的平衡。
