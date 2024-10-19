# gojsonschema 教程 - 第一部分：基础介绍和简单使用

## 1. 介绍

gojsonschema 是一个用于验证 JSON 数据是否符合预定义 JSON Schema 的 Go 库。它实现了 JSON Schema 规范，允许你定义复杂的验证规则来确保 JSON 数据的结构和内容是正确的。

### 1.1 主要特性

- 支持 JSON Schema draft 4、draft 6 和 draft 7
- 高性能和易用性
- 详细的验证错误报告
- 支持引用外部 schema

### 1.2 安装

使用以下命令安装 gojsonschema：

```bash
go get github.com/xeipuuv/gojsonschema
```

## 2. 基本使用

### 2.1 简单示例

以下是一个基本的使用示例：

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
            "name": {"type": "string"},
            "age": {"type": "integer", "minimum": 0}
        },
        "required": ["name", "age"]
    }`)

    documentLoader := gojsonschema.NewStringLoader(`{
        "name": "John Doe",
        "age": 30
    }`)

    result, err := gojsonschema.Validate(schemaLoader, documentLoader)
    if err != nil {
        panic(err.Error())
    }

    if result.Valid() {
        fmt.Println("The document is valid")
    } else {
        fmt.Println("The document is not valid. see errors :")
        for _, desc := range result.Errors() {
            fmt.Printf("- %s\n", desc)
        }
    }
}
```

### 2.2 解释

1. 我们首先创建了一个 `schemaLoader`，它包含了 JSON Schema 的定义。
2. 然后创建了一个 `documentLoader`，包含要验证的 JSON 数据。
3. 使用 `gojsonschema.Validate()` 函数进行验证。
4. 检查 `result.Valid()` 来确定文档是否有效。
5. 如果无效，我们可以遍历 `result.Errors()` 来获取详细的错误信息。

## 3. Loader 类型

gojsonschema 提供了多种加载 JSON 数据的方式：

### 3.1 StringLoader

用于从字符串加载 JSON：

```go
loader := gojsonschema.NewStringLoader(`{"type": "string"}`)
```

### 3.2 BytesLoader

用于从字节切片加载 JSON：

```go
data := []byte(`{"type": "string"}`)
loader := gojsonschema.NewBytesLoader(data)
```

### 3.3 FileLoader

用于从文件加载 JSON：

```go
loader := gojsonschema.NewReferenceLoader("file:///path/to/schema.json")
```

### 3.4 ReferenceLoader

用于从 URL 加载 JSON：

```go
loader := gojsonschema.NewReferenceLoader("http://example.com/schema.json")
```

## 4. 基本验证规则

JSON Schema 允许你定义各种验证规则。以下是一些常用的规则：

### 4.1 类型验证

```json
{
    "type": "string"
}
```

### 4.2 数值范围

```json
{
    "type": "integer",
    "minimum": 0,
    "maximum": 100
}
```

### 4.3 字符串长度

```json
{
    "type": "string",
    "minLength": 2,
    "maxLength": 50
}
```

### 4.4 枚举值

```json
{
    "type": "string",
    "enum": ["red", "green", "blue"]
}
```

### 4.5 正则表达式

```json
{
    "type": "string",
    "pattern": "^[A-Z][a-z]*$"
}
```

## 5. 结构化数据验证

对于复杂的 JSON 对象，我们可以定义嵌套的 schema：

```go
schemaLoader := gojsonschema.NewStringLoader(`{
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer", "minimum": 0},
        "address": {
            "type": "object",
            "properties": {
                "street": {"type": "string"},
                "city": {"type": "string"}
            },
            "required": ["street", "city"]
        }
    },
    "required": ["name", "age"]
}`)

documentLoader := gojsonschema.NewStringLoader(`{
    "name": "John Doe",
    "age": 30,
    "address": {
        "street": "123 Main St",
        "city": "Anytown"
    }
}`)

result, err := gojsonschema.Validate(schemaLoader, documentLoader)
// ... 处理结果
```

这个例子展示了如何验证包含嵌套对象的 JSON 数据。

在下一部分中，我们将深入探讨更高级的 gojsonschema 功能，包括复杂的验证规则、自定义格式、引用和复用 schema 等内容。

