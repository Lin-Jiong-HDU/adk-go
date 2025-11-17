# ADK-Go 模型接口文档

## 项目概述

Agent Development Kit (ADK) for Go 是一个开源的、以代码为核心的 Go 工具包，用于构建、评估和部署复杂的 AI 代理，具有灵活性和控制性。该项目虽然为 Gemini 进行了优化，但是模型无关的，意味着可以支持多种不同的语言模型。

## 模型接口架构

### 核心接口定义

ADK-Go 的模型系统设计简洁而强大，主要通过 `model.LLM` 接口定义了语言模型的标准行为。

#### 主要接口：`model.LLM`

位置：`model/llm.go:26`

```go
type LLM interface {
    Name() string
    GenerateContent(ctx context.Context, req *LLMRequest, stream bool) iter.Seq2[*LLMResponse, error]
}
```

**接口方法说明：**

1. **`Name() string`**
   - 返回模型的名称标识符
   - 例如："gemini-2.5-flash", "gpt-4" 等

2. **`GenerateContent(ctx context.Context, req *LLMRequest, stream bool) iter.Seq2[*LLMResponse, error]`**
   - 核心方法，用于生成内容
   - 支持同步和流式两种模式
   - 使用 Go 1.23+ 的迭代器模式返回结果序列

### 数据结构定义

#### 请求结构：`LLMRequest`

位置：`model/llm.go:32`

```go
type LLMRequest struct {
    Model    string
    Contents []*genai.Content
    Config   *genai.GenerateContentConfig
    Tools    map[string]any `json:"-"`
}
```

**字段说明：**

- `Model`: 目标模型名称
- `Contents`: 对话内容，使用 genai.Content 格式
- `Config`: 生成配置参数
- `Tools`: 可用工具集合

#### 响应结构：`LLMResponse`

位置：`model/llm.go:42`

```go
type LLMResponse struct {
    Content           *genai.Content
    CitationMetadata  *genai.CitationMetadata
    GroundingMetadata *genai.GroundingMetadata
    UsageMetadata     *genai.GenerateContentResponseUsageMetadata
    CustomMetadata    map[string]any
    LogprobsResult    *genai.LogprobsResult
    Partial           bool        // 流式模式下的部分响应标志
    TurnComplete      bool        // 流式模式下的回合完成标志
    Interrupted       bool        // 中断标志
    ErrorCode         string
    ErrorMessage      string
    FinishReason      genai.FinishReason
    AvgLogprobs       float64
}
```

## 现有实现：Gemini 模型

### Gemini 实现

位置：`model/gemini/gemini.go`

Gemini 模型实现展示了如何实现 `model.LLM` 接口：

```go
type geminiModel struct {
    client             *genai.Client
    name               string
    versionHeaderValue string
}
```

**关键实现特性：**

1. **构造函数模式**

   ```go
   func NewModel(ctx context.Context, modelName string, cfg *genai.ClientConfig) (model.LLM, error)
   ```

2. **HTTP 头管理**
   - 自动添加版本信息和用户代理头
   - 支持 `x-goog-api-client` 和 `user-agent` 头设置

3. **内容处理**
   - 自动添加用户内容以确保模型能够继续输出
   - 智能处理对话上下文

4. **流式支持**
   - 使用 `StreamingResponseAggregator` 处理流式响应
   - 支持部分响应聚合

## 支持新模型需要实现的功能

### 1. 核心接口实现

必须实现 `model.LLM` 接口的两个方法：

```go
type YourModel struct {
    // 模型特定的字段
    client *YourModelClient
    name   string
    // 其他配置...
}

func (m *YourModel) Name() string {
    return m.name
}

func (m *YourModel) GenerateContent(ctx context.Context, req *model.LLMRequest, stream bool) iter.Seq2[*model.LLMResponse, error] {
    // 实现内容生成逻辑
}
```

### 2. 数据转换器

需要创建转换器来桥接你的模型 API 和 ADK 的标准格式：

```go
// 参考位置：internal/llminternal/converters/converters.go
func YourAPIResponseToLLMResponse(yourResp *YourAPIResponse) *model.LLMResponse {
    // 转换逻辑
}
```

### 3. 流式响应处理（可选）

如果模型支持流式输出，需要实现流式响应聚合：

```go
// 参考位置：internal/llminternal/stream_aggregator.go
type YourStreamingAggregator struct {
    // 聚合器字段
}

func (s *YourStreamingAggregator) ProcessResponse(ctx context.Context, resp *YourStreamResponse) iter.Seq2[*model.LLMResponse, error] {
    // 流式处理逻辑
}
```

### 4. 错误处理

需要正确处理各种错误情况：

- API 调用错误
- 内容过滤错误
- 网络超时错误
- 认证错误

错误信息应该映射到 `LLMResponse` 的 `ErrorCode` 和 `ErrorMessage` 字段。

### 5. 配置管理

需要提供配置选项来初始化模型：

```go
type YourModelConfig struct {
    APIKey      string
    Endpoint    string
    ModelName   string
    Temperature float32
    MaxTokens   int
    // 其他配置...
}
```

### 6. 元数据支持

需要支持各种元数据：

- **使用统计** (`UsageMetadata`): token 使用量、调用次数等
- **引用信息** (`CitationMetadata`): 内容来源引用
- **基础信息** (`GroundingMetadata`): 事实基础信息
- **概率信息** (`LogprobsResult`): token 概率分布
- **自定义元数据** (`CustomMetadata`): 模型特定的元数据

### 7. 工具调用支持

如果模型支持工具/函数调用，需要：

1. 解析 `LLMRequest.Tools` 中的工具定义
2. 将工具定义转换为目标模型的格式
3. 处理模型的工具调用响应
4. 将工具调用结果转换回 ADK 格式

### 8. 测试实现

参考 `model/llm_test.go` 和 `model/gemini/gemini_test.go`：

- 单元测试：测试各个方法的正确性
- 集成测试：测试与 ADK 框架的集成
- HTTP 重放测试：使用 `.httprr` 文件进行可重复的 API 测试

## 实现建议

### 1. 遵循 Go 惯例

- 使用 Go 的错误处理模式
- 实现 context.Context 支持
- 使用 Go 1.23+ 的迭代器模式
- 遵循 Go 的命名约定

### 2. 性能优化

- 支持并发请求
- 实现连接池
- 优化内存使用
- 考虑流式响应的缓冲策略

### 3. 可观测性

- 添加适当的日志记录
- 支持指标收集
- 实现追踪功能
- 添加健康检查端点

### 4. 配置灵活性

- 支持环境变量配置
- 支持配置文件
- 提供合理的默认值
- 支持运行时配置更新

## 总结

ADK-Go 的模型接口设计简洁而强大，主要围绕 `model.LLM` 接口展开。要为新模型提供支持，需要：

1. 实现 `LLM` 接口的核心方法
2. 创建数据转换器来桥接 API 格式
3. 正确处理流式响应和错误情况
4. 支持各种元数据和工具调用
5. 提供完整的测试覆盖

通过遵循这些指导原则，可以为 ADK-Go 框架添加对新模型的支持，同时保持与现有代码的兼容性。
