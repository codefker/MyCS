# gojsonschema 教程 - 第二部分：高级功能

## 1. 复杂的验证规则

### 1.1 组合 Schema

JSON Schema 允许使用逻辑组合来创建复杂的验证规则。

#### allOf

要求数据同时满足多个 schema：

```json
{
    "allOf": [
        {"type": "string"},
        {"maxLength": 5}
    ]
}
```

#### anyOf

要求数据满足至少一个 schema：

```json
{
    "anyOf": [
        {"type": "string", "maxLength": 5},
        {"type": "number", "minimum": 0}
    ]
}
```

#### oneOf

要求数据只满足一个 schema：

```json
{
    "oneOf": [
        {"type": "number", "multipleOf": 5},
        {"type": "number", "multipleOf": 3}
    ]
}
```

#### not

要求数据不满足指定的 schema：

```json
{
    "not": {"type": "string"}
}
```



### 1.2 条件验证

JSON Schema 允许使用 `if`、`then` 和 `else` 关键字进行条件验证。这是一个强大的功能，可以根据数据的某些属性动态地应用不同的验证规则。

#### 基本结构

条件验证的基本结构如下：

```json
{
    "if": {
        // 条件
    },
    "then": {
        // 如果条件满足，应用这些规则
    },
    "else": {
        // 如果条件不满足，应用这些规则
    }
}
```

#### 详细示例

让我们看一个更复杂的地址验证示例：

```json
{
    "type": "object",
    "properties": {
        "street_type": {"type": "string"}
    },
    "required": ["street_type"],
    "if": {
        "properties": {"street_type": {"const": "rural"}}
    },
    "then": {
        "properties": {
            "rural_route_number": {"type": "string"},
            "box_number": {"type": "string"}
        },
        "required": ["rural_route_number"]
    },
    "else": {
        "properties": {
            "street_name": {"type": "string"},
            "house_number": {"type": "integer"},
            "apartment": {"type": "string"}
        },
        "required": ["street_name", "house_number"]
    }
}
```

这个 schema 的工作原理：

1. 首先，它要求输入是一个对象，必须包含 `street_type` 属性。
2. 如果 `street_type` 等于 "rural"：
   - 则要求提供 `rural_route_number`（必须）和可选的 `box_number`。
3. 如果 `street_type` 不等于 "rural"：
   - 则要求提供 `street_name` 和 `house_number`（都是必须的），以及可选的 `apartment`。

#### 在 gojsonschema 中使用

下面是如何在 Go 代码中使用这个 schema 的示例：

```go
package main

import (
    "fmt"
    "github.com/xeipuuv/gojsonschema"
)

func main() {
    schemaLoader := gojsonschema.NewStringLoader(`{
        "type": "object",
        "properties": {
            "street_type": {"type": "string"}
        },
        "required": ["street_type"],
        "if": {
            "properties": {"street_type": {"const": "rural"}}
        },
        "then": {
            "properties": {
                "rural_route_number": {"type": "string"},
                "box_number": {"type": "string"}
            },
            "required": ["rural_route_number"]
        },
        "else": {
            "properties": {
                "street_name": {"type": "string"},
                "house_number": {"type": "integer"},
                "apartment": {"type": "string"}
            },
            "required": ["street_name", "house_number"]
        }
    }`)

    // 验证农村地址
    ruralAddress := gojsonschema.NewStringLoader(`{
        "street_type": "rural",
        "rural_route_number": "RR 2",
        "box_number": "Box 45"
    }`)

    result, err := gojsonschema.Validate(schemaLoader, ruralAddress)
    if err != nil {
        panic(err.Error())
    }

    if result.Valid() {
        fmt.Println("农村地址有效")
    } else {
        fmt.Println("农村地址无效:")
        for _, desc := range result.Errors() {
            fmt.Printf("- %s\n", desc)
        }
    }

    // 验证城市地址
    urbanAddress := gojsonschema.NewStringLoader(`{
        "street_type": "urban",
        "street_name": "Main St",
        "house_number": 123,
        "apartment": "Apt 4B"
    }`)

    result, err = gojsonschema.Validate(schemaLoader, urbanAddress)
    if err != nil {
        panic(err.Error())
    }

    if result.Valid() {
        fmt.Println("城市地址有效")
    } else {
        fmt.Println("城市地址无效:")
        for _, desc := range result.Errors() {
            fmt.Printf("- %s\n", desc)
        }
    }
}
```

#### 注意事项

1. `const` 关键字用于精确匹配。在这个例子中，它用于检查 `street_type` 是否恰好等于 "rural"。

2. 你可以在 `if` 条件中使用更复杂的逻辑，例如使用 `anyOf`、`allOf` 等。

3. `then` 和 `else` 子句可以包含完整的 schema 定义，允许你根据条件应用完全不同的验证规则。

4. 条件验证可以嵌套，允许创建复杂的验证逻辑树。

5. 当使用条件验证时，确保考虑到所有可能的情况，以避免意外的验证结果。

通过使用条件验证，你可以创建非常灵活和强大的 schema，能够处理复杂的数据结构和业务规则。这在处理有多种可能格式或依赖于某些字段值的数据时特别有用。



## 2. 自定义格式

gojsonschema 允许你定义和使用自定义格式：

```go
import "github.com/xeipuuv/gojsonschema"

// 添加自定义格式
gojsonschema.FormatCheckers.Add("my-format", gojsonschema.FormatCheckerFunc(
    func(input interface{}) bool {
        // 实现自定义格式检查逻辑
        str, ok := input.(string)
        if !ok {
            return false
        }
        return len(str) == 8 // 例如：检查字符串长度是否为 8
    },
))

// 在 schema 中使用自定义格式
schemaLoader := gojsonschema.NewStringLoader(`{
    "type": "string",
    "format": "my-format"
}`)

// 验证
documentLoader := gojsonschema.NewStringLoader(`"12345678"`)
result, err := gojsonschema.Validate(schemaLoader, documentLoader)
// ... 处理结果
```

## 3. 引用和复用 Schema

### 3.1 使用 $ref

你可以使用 `$ref` 关键字来引用其他的 schema 定义：

```json
{
    "type": "object",
    "properties": {
        "billing_address": {"$ref": "#/definitions/address"},
        "shipping_address": {"$ref": "#/definitions/address"}
    },
    "definitions": {
        "address": {
            "type": "object",
            "properties": {
                "street": {"type": "string"},
                "city": {"type": "string"}
            },
            "required": ["street", "city"]
        }
    }
}
```

### 3.2 外部引用

你也可以引用外部文件或 URL 中的 schema：

```json
{
    "properties": {
        "user": {"$ref": "http://example.com/schemas/user.json"}
    }
}
```

## 4. 性能优化

对于需要重复使用的 schema，可以预先编译以提高性能：

```go
schema, err := gojsonschema.NewSchema(gojsonschema.NewStringLoader(`{
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer", "minimum": 0}
    },
    "required": ["name", "age"]
}`))
if err != nil {
    panic(err.Error())
}

// 现在可以多次使用这个预编译的 schema
result, err := schema.Validate(gojsonschema.NewStringLoader(`{"name": "John", "age": 30}`))
// ... 处理结果

result, err = schema.Validate(gojsonschema.NewStringLoader(`{"name": "Jane", "age": 25}`))
// ... 处理结果
```

## 5. 高级错误处理

gojsonschema 提供了详细的错误信息，你可以根据需要处理这些信息：

```go
result, err := gojsonschema.Validate(schemaLoader, documentLoader)
if err != nil {
    panic(err.Error())
}

if !result.Valid() {
    for _, err := range result.Errors() {
        // 获取具体的错误信息
        fmt.Printf("- Field: %s\n", err.Field())
        fmt.Printf("  Error: %s\n", err.Description())
        fmt.Printf("  Details: %v\n", err.Details())
    }
}
```

## 6. 实际应用示例

### 6.1 API 请求验证

```go
type APIHandler struct {
    inputSchema *gojsonschema.Schema
}

func NewAPIHandler() *APIHandler {
    schema, err := gojsonschema.NewSchema(gojsonschema.NewStringLoader(`{
        "type": "object",
        "properties": {
            "username": {"type": "string", "minLength": 3},
            "email": {"type": "string", "format": "email"},
            "age": {"type": "integer", "minimum": 18}
        },
        "required": ["username", "email"]
    }`))
    if err != nil {
        panic(err)
    }
    return &APIHandler{inputSchema: schema}
}

func (h *APIHandler) HandleRequest(w http.ResponseWriter, r *http.Request) {
    var body map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    result, err := h.inputSchema.Validate(gojsonschema.NewGoLoader(body))
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    if !result.Valid() {
        errors := make([]string, len(result.Errors()))
        for i, err := range result.Errors() {
            errors[i] = err.String()
        }
        json.NewEncoder(w).Encode(map[string]interface{}{
            "errors": errors,
        })
        return
    }

    // 处理有效的请求
    // ...
}
```

这个例子展示了如何在 API 处理程序中使用 gojsonschema 来验证输入数据。

在下一部分中，我们将探讨更多高级主题，包括自定义关键字、schema 生成、与其他库的集成等。

