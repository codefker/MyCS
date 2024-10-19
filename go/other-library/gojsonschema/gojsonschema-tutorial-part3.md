# gojsonschema 教程 - 第三部分：高级主题和最佳实践

## 1. 自定义关键字

gojsonschema 允许你定义自定义关键字，以扩展 JSON Schema 的功能。

### 1.1 创建自定义关键字

```go
import (
    "github.com/xeipuuv/gojsonschema"
)

// 定义自定义关键字
type CustomKeywordValidator struct{}

func (v CustomKeywordValidator) Validate(data interface{}) bool {
    // 实现自定义验证逻辑
    str, ok := data.(string)
    if !ok {
        return false
    }
    return len(str) % 2 == 0 // 例如：验证字符串长度是否为偶数
}

// 注册自定义关键字
gojsonschema.RegisterKeyword("customEvenLength", CustomKeywordValidator{})

// 使用自定义关键字
schemaLoader := gojsonschema.NewStringLoader(`{
    "type": "string",
    "customEvenLength": true
}`)

documentLoader := gojsonschema.NewStringLoader(`"abcd"`)
result, err := gojsonschema.Validate(schemaLoader, documentLoader)
// ... 处理结果
```

## 2. Schema 生成

有时，你可能需要从 Go 结构体生成 JSON Schema。虽然 gojsonschema 本身不提供这个功能，但你可以使用其他库（如 `jsonschema`）来实现这一点，然后与 gojsonschema 结合使用。

### 2.1 使用 jsonschema 生成 Schema

首先，安装 jsonschema 库：

```bash
go get github.com/invopop/jsonschema
```

然后，你可以这样使用它：

```go
import (
    "encoding/json"
    "github.com/invopop/jsonschema"
    "github.com/xeipuuv/gojsonschema"
)

type User struct {
    Name  string `json:"name" jsonschema:"required"`
    Email string `json:"email" jsonschema:"format=email"`
    Age   int    `json:"age" jsonschema:"minimum=18"`
}

func main() {
    // 生成 Schema
    reflector := jsonschema.Reflector{}
    schema := reflector.Reflect(&User{})

    // 将 Schema 转换为 JSON
    schemaJSON, _ := json.Marshal(schema)

    // 使用 gojsonschema 验证
    schemaLoader := gojsonschema.NewBytesLoader(schemaJSON)
    documentLoader := gojsonschema.NewStringLoader(`{
        "name": "John Doe",
        "email": "john@example.com",
        "age": 30
    }`)

    result, err := gojsonschema.Validate(schemaLoader, documentLoader)
    // ... 处理结果
}
```

## 3. 与其他库的集成

### 3.1 与 gin 框架集成

以下是如何在 gin web 框架中使用 gojsonschema 进行请求验证的示例：

```go
import (
    "github.com/gin-gonic/gin"
    "github.com/xeipuuv/gojsonschema"
    "net/http"
)

func validateJSON() gin.HandlerFunc {
    schemaLoader := gojsonschema.NewStringLoader(`{
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "email": {"type": "string", "format": "email"},
            "age": {"type": "integer", "minimum": 18}
        },
        "required": ["name", "email"]
    }`)

    return func(c *gin.Context) {
        var json interface{}
        if err := c.ShouldBindJSON(&json); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid JSON"})
            c.Abort()
            return
        }

        result, err := gojsonschema.Validate(schemaLoader, gojsonschema.NewGoLoader(json))
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            c.Abort()
            return
        }

        if !result.Valid() {
            errors := []string{}
            for _, desc := range result.Errors() {
                errors = append(errors, desc.String())
            }
            c.JSON(http.StatusBadRequest, gin.H{"errors": errors})
            c.Abort()
            return
        }

        c.Next()
    }
}

func main() {
    r := gin.Default()
    r.POST("/user", validateJSON(), func(c *gin.Context) {
        // 处理验证通过的请求
        c.JSON(http.StatusOK, gin.H{"message": "User created successfully"})
    })
    r.Run()
}
```

## 4. 最佳实践

### 4.1 Schema 复用

对于大型应用，创建一个集中的 schema 存储可以提高可维护性：

```go
var schemas = map[string]*gojsonschema.Schema{
    "user": gojsonschema.MustNewSchema(gojsonschema.NewStringLoader(`{
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "email": {"type": "string", "format": "email"}
        },
        "required": ["name", "email"]
    }`)),
    "product": gojsonschema.MustNewSchema(gojsonschema.NewStringLoader(`{
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "price": {"type": "number", "minimum": 0}
        },
        "required": ["name", "price"]
    }`)),
}

func validateAgainstSchema(schemaName string, data interface{}) (bool, []string) {
    schema, exists := schemas[schemaName]
    if !exists {
        return false, []string{"Schema not found"}
    }

    result, err := schema.Validate(gojsonschema.NewGoLoader(data))
    if err != nil {
        return false, []string{err.Error()}
    }

    if !result.Valid() {
        errors := make([]string, len(result.Errors()))
        for i, err := range result.Errors() {
            errors[i] = err.String()
        }
        return false, errors
    }

    return true, nil
}
```

### 4.2 性能优化

- 预编译 Schema：对于频繁使用的 schema，使用 `gojsonschema.NewSchema()` 预编译可以提高性能。
- 使用适当的 Loader：对于静态 JSON 数据，使用 `StringLoader` 或 `BytesLoader` 而不是 `JSONLoader`。
- 合理使用缓存：对于动态生成的 schema，考虑实现缓存机制。

### 4.3 错误处理

创建一个通用的错误处理函数可以使代码更加清晰：

```go
func handleValidationErrors(result *gojsonschema.Result) []string {
    if result.Valid() {
        return nil
    }

    errors := make([]string, len(result.Errors()))
    for i, err := range result.Errors() {
        errors[i] = fmt.Sprintf("%s: %s", err.Field(), err.Description())
    }
    return errors
}
```

## 5. 常见陷阱和解决方案

### 5.1 处理未知字段

默认情况下，gojsonschema 不会验证额外的未知字段。如果你想禁止额外字段，可以使用 `additionalProperties: false`：

```json
{
    "type": "object",
    "properties": {
        "name": {"type": "string"}
    },
    "additionalProperties": false
}
```

### 5.2 日期时间验证

使用 `format` 关键字来验证日期和时间：

```json
{
    "type": "string",
    "format": "date-time"
}
```

### 5.3 处理空值

在某些情况下，你可能需要区分 `null` 和未提供的字段：

```json
{
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": ["integer", "null"]}
    },
    "required": ["name"]
}
```

这个 schema 允许 `age` 字段是整数或 `null`，但 `name` 是必需的且不能为 `null`。

## 结论

gojsonschema 是一个强大的库，可以帮助你在 Go 应用程序中实现复杂的 JSON 验证逻辑。通过本教程的三个部分，我们已经覆盖了从基础使用到高级技巧的广泛内容。合理使用 gojsonschema 可以显著提高你的应用程序的数据完整性和可靠性。

记住，JSON Schema 不仅仅是一个验证工具，它还可以作为一种文档形式，清晰地描述你的 API 预期的数据结构。在实际项目中，结合 gojsonschema 与其他工具和最佳实践，可以创建出更加健壮和可维护的应用程序。

