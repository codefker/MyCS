# Go Context 包教程 - 第2部分：高级用法和最佳实践

## 1. 引言

在第一部分中，我们介绍了 Go `context` 包的基本概念和用法。本部分将深入探讨 `context` 的高级用法和最佳实践，帮助您更有效地在复杂的 Go 程序中使用 context。

## 2. Context 树

Context 在 Go 程序中形成了一个树状结构。每个派生的 Context 都是其父 Context 的子节点。这种结构有几个重要特性：

1. 当父 Context 被取消时，所有子 Context 也会被取消。
2. 子 Context 可以访问父 Context 中的值，但反之则不行。

### 2.1 构建 Context 树

```go
rootCtx := context.Background()

// 派生第一层 Context
ctx1, cancel1 := context.WithCancel(rootCtx)
defer cancel1()

// 派生第二层 Context
ctx2, cancel2 := context.WithTimeout(ctx1, 5*time.Second)
defer cancel2()

// 派生第三层 Context
ctx3 := context.WithValue(ctx2, "key", "value")
```

在这个例子中，`ctx3` 可以访问 `ctx2` 和 `ctx1` 中的值和取消信号，但 `ctx1` 不能访问 `ctx2` 或 `ctx3` 中的值。

## 3. 高级取消模式

### 3.1 组合多个取消源

有时，您可能希望一个操作在多个条件中的任何一个满足时被取消。可以使用 `select` 语句来实现这一点：

```go
func doSomething(ctx context.Context, timeout time.Duration) error {
    ctxTimeout, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    done := make(chan struct{})
    
    go func() {
        // 执行某些操作
        time.Sleep(2 * time.Second)
        close(done)
    }()

    select {
    case <-ctxTimeout.Done():
        return ctxTimeout.Err()
    case <-done:
        return nil
    }
}
```

### 3.2 优雅取消

在取消长时间运行的操作时，应该给予一些时间来清理资源：

```go
func gracefulShutdown(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    // 发送取消信号
    stopChan <- struct{}{}

    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-doneChan:
        return nil
    }
}
```

## 4. Context 值的高级用法

### 4.1 使用类型安全的键

为了避免键冲突和提高类型安全性，可以使用自定义类型作为 context 的键：

```go
type key int

const (
    userIDKey key = iota
    authTokenKey
)

func WithUserID(ctx context.Context, userID int64) context.Context {
    return context.WithValue(ctx, userIDKey, userID)
}

func UserIDFromContext(ctx context.Context) (int64, bool) {
    userID, ok := ctx.Value(userIDKey).(int64)
    return userID, ok
}
```

### 4.2 不要滥用 Context 值

Context 值应该用于请求范围的数据，而不是作为函数参数的替代品。过度使用 Context 值可能导致代码难以理解和维护。

## 5. Context 在 HTTP 服务器中的应用

Go 的 `net/http` 包在处理每个请求时都会创建一个新的 Context：

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    // 使用 ctx 进行后续操作
}
```

### 5.1 给 HTTP 请求添加超时

```go
func timeoutMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
        defer cancel()

        r = r.WithContext(ctx)
        next.ServeHTTP(w, r)
    })
}
```

## 6. Context 在数据库操作中的应用

许多数据库驱动支持使用 Context 来控制查询的生命周期：

```go
func queryWithTimeout(ctx context.Context, db *sql.DB) error {
    ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
    defer cancel()

    var result string
    err := db.QueryRowContext(ctx, "SELECT * FROM huge_table").Scan(&result)
    if err != nil {
        return err
    }
    // 处理结果
    return nil
}
```

## 7. 最佳实践

1. **总是传递 Context**：在函数调用链中，总是传递 Context，即使当前函数没有直接使用它。

2. **不要存储 Context**：Context 应该作为参数传递，而不是存储在结构体中。

3. **使用 Background() 作为根 Context**：在 main 函数或程序的入口点使用 `context.Background()`。

4. **及时取消**：当操作完成时，务必调用 cancel 函数，即使 context 已经过期。

5. **不要传递 nil Context**：如果不确定使用哪个 Context，可以使用 `context.TODO()`。

6. **Context 值只用于请求范围的数据**：不要使用 Context 值来传递可选参数。

7. **保持 Context 和取消函数成对出现**：当创建一个新的 Context 时，确保在同一个函数中调用其取消函数。

8. **注意 Context 泄露**：确保在所有情况下都能正确地取消 Context，避免 goroutine 泄露。

## 8. 小结

本部分深入探讨了 `context` 包的高级用法和最佳实践。我们学习了如何构建和管理 Context 树，实现高级取消模式，以及在 HTTP 服务器和数据库操作中应用 Context。通过遵循这些最佳实践，您可以更有效地利用 Context 来管理请求范围的数据和控制并发操作的生命周期。在下一部分中，我们将探讨 Context 在复杂系统中的应用。

