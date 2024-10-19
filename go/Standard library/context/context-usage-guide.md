# Go 编程中使用 context 的决策指南

## 1. 应该使用 context 的方法

以下类型的方法通常应该考虑使用 context：

### a) 可能长时间运行的操作
- 执行复杂计算
- 进行网络请求
- 访问数据库或文件系统

示例：
```go
func (s *Service) FetchUserData(ctx context.Context, userID string) (*UserData, error) {
    // 使用 context 进行数据库查询
    return s.db.QueryUserWithContext(ctx, userID)
}
```

### b) 可能需要被取消的操作
- 后台任务
- 周期性检查

示例：
```go
func (worker *Worker) ProcessTasks(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            task := worker.getNextTask()
            err := worker.processTask(ctx, task)
            if err != nil {
                // 处理错误
            }
        }
    }
}
```

### c) 需要传播截止时间或取消信号的操作
- 调用其他使用 context 的函数
- 在分布式系统中传播请求上下文

示例：
```go
func (s *Service) ProcessRequest(ctx context.Context, req *Request) (*Response, error) {
    // 创建一个新的 context，添加超时
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // 调用其他使用 context 的服务
    data, err := s.dataService.FetchData(ctx, req.ID)
    if err != nil {
        return nil, err
    }

    // 处理数据
    return s.processData(ctx, data)
}
```

## 2. 通常不需要使用 context 的方法

以下类型的方法通常不需要使用 context：

### a) 快速的、纯粹的内存操作
- 简单的 getter 方法
- 简单的数据结构操作

示例：
```go
func (u *User) GetID() string {
    return u.ID
}

func (list *List) Append(item interface{}) {
    list.items = append(list.items, item)
}
```

### b) 不涉及 I/O 或长时间计算的方法
- 基本的数据验证
- 简单的数学计算

示例：
```go
func ValidateEmail(email string) bool {
    // 简单的邮箱格式验证
    return regexp.MustCompile(`^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,4}$`).MatchString(email)
}

func CalculateAverage(numbers []int) float64 {
    sum := 0
    for _, num := range numbers {
        sum += num
    }
    return float64(sum) / float64(len(numbers))
}
```

### c) 不需要被取消的同步操作
- 短暂的锁操作
- 原子操作

示例：
```go
func (c *Counter) Increment() {
    atomic.AddInt64(&c.value, 1)
}

func (cache *Cache) Get(key string) (interface{}, bool) {
    cache.mu.RLock()
    defer cache.mu.RUnlock()
    val, ok := cache.data[key]
    return val, ok
}
```

## 3. 决策考虑因素

在决定是否使用 context 时，应考虑以下因素：

1. 操作的持续时间：
   - 长时间运行的操作更可能需要 context 用于取消或超时控制。

2. 是否需要取消能力：
   - 如果操作可能需要在中途被取消，应使用 context。

3. 是否需要传播截止时间：
   - 在调用链中传播超时或截止时间时，使用 context 是最佳实践。

4. 操作的复杂性和资源消耗：
   - 复杂或资源密集型操作通常受益于使用 context 进行控制。

5. 一致性：
   - 如果你的项目或包中的其他类似方法使用了 context，为了一致性可能也应该使用。

6. API 设计：
   - 公共 API 可能会从包含 context 参数中受益，以提供更大的灵活性。

## 4. 最佳实践

1. context 应该是函数的第一个参数。

2. 不要将 context 存储在结构体中，而应该显式地通过函数参数传递。

3. 使用 context 取消时，确保及时释放资源。

4. 不要传递 nil context，如果不确定要使用什么，传递 context.TODO()。

5. context.Value 应该仅用于请求范围的数据，不要用它来传递可选参数。

6. 在长时间运行的操作中定期检查 context.Done()。

## 5. 示例：在 CapabilityNucleus 中的应用

对于 CapabilityNucleus 结构，以下方法可能需要 context：

```go
func (cn *CapabilityNucleus) Execute(ctx context.Context, input interface{}) (interface{}, error)
func (cn *CapabilityNucleus) Evolve(ctx context.Context) error
func (cn *CapabilityNucleus) EvolveWithNaturalLanguage(ctx context.Context, instruction string) error
func (cn *CapabilityNucleus) validateInput(ctx context.Context, input interface{}) error
```

而以下方法可能不需要 context：

```go
func (cn *CapabilityNucleus) GetID() CapabilityID
func (cn *CapabilityNucleus) GetType() CapabilityType
func (cn *CapabilityNucleus) GetSchema() map[string]interface{}
func (cn *CapabilityNucleus) ResetPerformanceMetrics()
```

## 结论

正确使用 context 可以显著提高 Go 程序的可控性和响应性，特别是在处理长时间运行的操作、I/O 密集型任务或需要取消能力的场景中。然而，过度使用 context 可能会使代码变得复杂。在简单、快速的操作中，没有必要引入 context。关键是要在每种情况下权衡使用 context 带来的好处和增加的复杂性。

