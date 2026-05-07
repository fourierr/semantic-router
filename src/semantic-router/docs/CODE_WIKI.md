# Semantic Router Code Wiki

## 1. 项目概述

### 1.1 项目简介

**vLLM Semantic Router** 是一个智能化的 LLM 请求路由系统，通过多维度信号检测（关键词、嵌入向量、领域分类、用户反馈等）将用户请求精准路由到最合适的 LLM 模型。该项目主要作为 Envoy External Processor (ExtProc) 运行，实现请求的智能分发。

### 1.2 核心能力

| 能力 | 描述 |
|------|------|
| 多信号路由 | 支持关键词、嵌入向量、领域分类、用户反馈等 15+ 种信号类型 |
| 模型选择 | 支持置信度、ELO、RL-Driven、AutoMix 等多种模型选择算法 |
| 语义缓存 | 基于向量相似度的请求缓存，减少重复计算 |
| 安全检测 | 内置 Jailbreak、PII 等安全检测能力 |
| 记忆系统 | 会话级别的上下文记忆管理 |

### 1.3 技术栈

- **语言**: Go 1.24.1
- **核心依赖**:
  - `envoyproxy/go-control-plane`: gRPC 通信
  - `openai/openai-go`: OpenAI API 兼容
  - `anthropics/anthropic-sdk-go`: Anthropic API 支持
  - `redis/go-redis/v9`: Redis 缓存
  - `milvus-sdk-go`: 向量存储
  - `prometheus/client_golang`: 指标监控

---

## 2. 项目架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Semantic Router                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────────┐    ┌──────────────────────────┐    │
│  │   Client    │───▶│  Envoy Proxy   │───▶│   ExtProc Server        │    │
│  │ (OpenAI API)│    │  (ExtProc Mode) │    │  (gRPC Service)         │    │
│  └─────────────┘    └─────────────────┘    └──────────┬─────────────┘    │
│                                                        │                   │
│  ┌─────────────────────────────────────────────────────┴───────────────┐   │
│  │                    OpenAIRouter                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │   │
│  │  │  Classifier │  │  Decision   │  │    Cache    │  │  Memory    │  │   │
│  │  │   Engine    │  │   Engine    │  │   Backend   │  │   Store    │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │   │
│  │  │    DSL      │  │   Model     │  │    AuthZ    │  │  Rate      │  │   │
│  │  │  Compiler   │  │  Selector   │  │  Resolver   │  │  Limiter   │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块依赖关系

```
cmd/main.go (入口)
    │
    ├── pkg/config (配置管理)
    │       └── 定义 CanonicalConfig, RouterConfig
    │
    ├── pkg/apiserver (HTTP API 服务器)
    │       ├── /health, /ready, /startup-status
    │       ├── /api/v1/classify/* (分类端点)
    │       └── /config/* (配置管理)
    │
    ├── pkg/extproc (Envoy ExtProc 处理器)
    │       ├── processor_core.go (请求处理核心)
    │       ├── router.go (路由决策)
    │       ├── req_filter_*.go (请求过滤器)
    │       └── res_filter_*.go (响应过滤器)
    │
    ├── pkg/classification (分类引擎)
    │       ├── classifier.go (主分类器)
    │       ├── unified_classifier.go (CGO/Rust 批处理)
    │       └── classifier_*.go (各类信号分类器)
    │
    ├── pkg/decision (决策引擎)
    │       └── engine.go (规则评估)
    │
    ├── pkg/cache (缓存系统)
    │       ├── cache.go (缓存接口)
    │       ├── redis_cache.go
    │       └── valkey_cache.go
    │
    ├── pkg/dsl (DSL 编译器)
    │       ├── parser.go (语法解析)
    │       ├── compiler.go (编译到 RouterConfig)
    │       └── ast.go (抽象语法树)
    │
    ├── pkg/memory (记忆系统)
    │       ├── store.go (存储接口)
    │       ├── valkey_store.go
    │       └── inmemory_store.go
    │
    ├── pkg/selection (模型选择)
    │       ├── selector.go
    │       └── 多种选择算法实现
    │
    └── pkg/observability (可观测性)
            ├── logging
            └── metrics
```

---

## 3. 核心模块详解

### 3.1 cmd/main.go - 应用入口

**文件位置**: `src/semantic-router/cmd/main.go`

#### 主要职责
- 解析命令行参数和配置文件
- 初始化日志系统
- 下载必要的 ML 模型
- 启动 API Server 和 ExtProc Server
- 管理 Kubernetes Controller

#### 关键函数

| 函数 | 签名 | 说明 |
|------|------|------|
| `main` | `func main()` | 应用主入口，协调各组件初始化 |
| `parseRuntimeOptions` | `func parseRuntimeOptions() runtimeOptions` | 解析命令行参数 |
| `loadRuntimeConfigOrFatal` | `func loadRuntimeConfigOrFatal(path string) *config.RouterConfig` | 加载运行时配置 |
| `startAPIServerIfEnabled` | `func startAPIServerIfEnabled(...)` | 启动 HTTP API Server |
| `ensureModelsDownloaded` | `func ensureModelsDownloaded(...) error` | 确保模型已下载 |
| `initializeRuntimeDependencies` | `func initializeRuntimeDependencies(...)` | 初始化运行时依赖 |

#### 启动流程

```go
// 简化流程
func main() {
    logo.PrintVLLMLogo()           // 1. 打印 Logo
    opts := parseRuntimeOptions() // 2. 解析参数
    initializeRuntimeLogger()      // 3. 初始化日志

    cfg := loadRuntimeConfigOrFatal(opts.configPath)  // 4. 加载配置
    config.Replace(cfg)
    runtimeRegistry := routerruntime.NewRegistry(cfg)

    startupWriter := newStartupWriter(cfg, opts.configPath)  // 5. 启动状态写入器

    startAPIServerIfEnabled(opts, runtimeRegistry)  // 6. 启动 API Server
    ensureModelsDownloadedOrFatal(cfg, startupWriter)  // 7. 下载模型
    exitIfDownloadOnly(opts.downloadOnly)

    initializeTracing(cfg)  // 8. 初始化追踪
    initializeWindowedMetricsIfEnabled(cfg)  // 9. 初始化指标

    embeddingRuntime := initializeRuntimeDependencies(...)  // 10. 初始化依赖
    server := newExtProcServerOrFatal(...)  // 11. 创建 ExtProc Server
    warmupRouterRuntime(server, embeddingRuntime)  // 12. 预热

    markRouterReady(startupWriter)  // 13. 标记就绪
    startExtProcServerOrFatal(server, startupWriter)  // 14. 启动
}
```

---

### 3.2 pkg/config - 配置管理

**文件位置**: `src/semantic-router/pkg/config/`

#### CanonicalConfig 结构 (v0.3 公共合约)

```go
type CanonicalConfig struct {
    Version   string             `yaml:"version,omitempty"`
    Listeners []Listener         `yaml:"listeners,omitempty"`
    Providers CanonicalProviders `yaml:"providers,omitempty"`
    Routing   CanonicalRouting   `yaml:"routing,omitempty"`
    Global    *CanonicalGlobal   `yaml:"global,omitempty"`
}

type CanonicalRouting struct {
    ModelCards    []RoutingModel       // 模型卡片
    Signals       CanonicalSignals     // 路由信号
    Projections   CanonicalProjections  // 投影输出
    Decisions     []Decision          // 路由决策
    SessionStates []SessionStateConfig // 会话状态
}

type CanonicalSignals struct {
    Keywords      []KeywordRule      // 关键词信号
    Embeddings    []EmbeddingRule    // 嵌入向量信号
    Domains       []Category         // 领域分类
    FactCheck     []FactCheckRule    // 事实检查
    UserFeedbacks  []UserFeedbackRule // 用户反馈
    Reasks        []ReaskRule        // 重新提问检测
    Preferences   []PreferenceRule   // 用户偏好
    Language      []LanguageRule     // 语言检测
    Context       []ContextRule      // 上下文信号
    Structure     []StructureRule    // 请求结构
    Complexity    []ComplexityRule   // 复杂度
    Modality      []ModalityRule    // 模态
    RoleBindings  []RoleBinding      // 角色绑定
    Jailbreak     []JailbreakRule   // 越狱检测
    PII           []PIIRule         // PII 检测
    KB            []KBSignalRule    // 知识库信号
    Conversation  []ConversationRule // 对话形状
}
```

#### 关键函数

| 函数 | 签名 | 说明 |
|------|------|------|
| `LoadConfig` | `func LoadConfig(path string) (*RouterConfig, error)` | 从文件加载配置 |
| `normalizeCanonicalConfig` | `func normalizeCanonicalConfig(c *CanonicalConfig) (*RouterConfig, error)` | 规范化配置 |
| `validateCanonicalContract` | `func validateCanonicalContract(c *CanonicalConfig) error` | 验证配置合约 |

---

### 3.3 pkg/apiserver - HTTP API 服务器

**文件位置**: `src/semantic-router/pkg/apiserver/server.go`

#### ClassificationAPIServer 结构

```go
type ClassificationAPIServer struct {
    classificationSvc     classificationService  // 分类服务
    config                *config.RouterConfig  // 运行时配置
    runtimeConfig         *liveRuntimeConfig    // 动态配置更新
    runtimeRegistry       *routerruntime.Registry
    configPath            string
    memoryStore           memory.Store          // 记忆存储
    knowledgeBaseMapCache *knowledgeBaseMapCache
}
```

#### 路由注册

```go
func (s *ClassificationAPIServer) setupRoutes() *http.ServeMux {
    mux := http.NewServeMux()
    s.registerCoreRoutes(mux)           // 健康检查
    s.registerClassificationRoutes(mux)  // 分类端点
    s.registerEmbeddingRoutes(mux)       // 嵌入端点
    s.registerInfoRoutes(mux)            // 信息端点
    s.registerConfigRoutes(mux)         // 配置端点
    s.registerMemoryRoutes(mux)         // 记忆端点
    registerVectorStoreRoutes(mux, s)
    registerFileRoutes(mux, s)
    return mux
}
```

#### 主要 API 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/health` | GET | 健康检查 |
| `/ready` | GET | 就绪检查 |
| `/startup-status` | GET | 启动状态 |
| `/api/v1/classify/intent` | POST | 意图分类 |
| `/api/v1/classify/pii` | POST | PII 检测 |
| `/api/v1/classify/fact-check` | POST | 事实检查 |
| `/api/v1/classify/combined` | POST | 组合分类 |
| `/api/v1/classify/batch` | POST | 批量分类 |
| `/api/v1/embeddings` | POST | 嵌入生成 |
| `/config/kbs` | GET/POST | 知识库管理 |
| `/config/router` | GET/PATCH/PUT | 路由配置 |
| `/v1/memory/{id}` | GET/DELETE | 记忆管理 |

---

### 3.4 pkg/classification - 分类引擎

**文件位置**: `src/semantic-router/pkg/classification/`

#### Classifier 结构

```go
type Classifier struct {
    // 树内分类器
    categoryInitializer         CategoryInitializer
    categoryInference           CategoryInference
    jailbreakInitializer        JailbreakInitializer
    jailbreakInference          JailbreakInference
    piiInitializer              PIIInitializer
    piiInference                PIIInference
    keywordClassifier           *KeywordClassifier
    keywordEmbeddingClassifier  *EmbeddingClassifier
    factCheckClassifier         *FactCheckClassifier
    hallucinationDetector       *HallucinationDetector
    feedbackDetector            *FeedbackDetector
    reaskClassifier            *ReaskClassifier
    preferenceClassifier        *PreferenceClassifier
    languageClassifier          *LanguageClassifier
    contextClassifier           *ContextClassifier
    structureClassifier         *StructureClassifier
    complexityClassifier        *ComplexityClassifier
    contrastiveJailbreakClassifiers map[string]*ContrastiveJailbreakClassifier
    authzClassifier             *AuthzClassifier
    kbClassifiers              map[string]*KnowledgeBaseClassifier

    // 配置
    Config           *config.RouterConfig
    CategoryMapping  *CategoryMapping
    PIIMapping       *PIIMapping
    JailbreakMapping *JailbreakMapping
    MMLUToGeneric    map[string]string
    GenericToMMLU    map[string][]string
}
```

#### UnifiedClassifier (CGO/Rust 批处理)

**文件位置**: `src/semantic-router/pkg/classification/unified_classifier.go`

使用 Rust 实现的高性能批处理分类器，通过 CGO 调用。

```go
type UnifiedClassifier struct {
    initialized     bool
    mu              sync.Mutex
    stats           UnifiedClassifierStats
    useLoRA         bool              // 使用高置信度 LoRA 模型
    loraModelPaths  *LoRAModelPaths
    loraInitialized bool
}

type UnifiedBatchResults struct {
    IntentResults   []IntentResult   // 意图分类结果
    PIIResults      []PIIResult      // PII 检测结果
    SecurityResults []SecurityResult // 安全检测结果
    BatchSize       int
}
```

**关键方法**:

| 方法 | 说明 |
|------|------|
| `ClassifyBatch(texts []string)` | 批量分类，自动选择 LoRA 或传统模型 |
| `ClassifyIntent(texts []string)` | 提取意图分类结果 |
| `ClassifyPII(texts []string)` | 提取 PII 检测结果 |
| `ClassifySecurity(texts []string)` | 提取安全检测结果 |

---

### 3.5 pkg/decision - 决策引擎

**文件位置**: `src/semantic-router/pkg/decision/engine.go`

#### DecisionEngine 结构

```go
type DecisionEngine struct {
    keywordRules   []config.KeywordRule
    embeddingRules []config.EmbeddingRule
    categories     []config.Category
    decisions      []config.Decision
    strategy       string  // "priority" | "confidence"
}
```

#### SignalMatches - 信号匹配结果

```go
type SignalMatches struct {
    KeywordRules      []string  // 匹配的关键词规则
    EmbeddingRules    []string  // 匹配的嵌入规则
    DomainRules       []string  // 匹配的领域规则
    FactCheckRules    []string  // 事实检查结果
    UserFeedbackRules []string  // 用户反馈结果
    ReaskRules        []string  // 重新提问检测
    PreferenceRules   []string  // 偏好匹配
    LanguageRules     []string  // 语言代码
    ContextRules      []string  // 上下文规则
    StructureRules    []string  // 结构规则
    ComplexityRules   []string  // 复杂度规则
    ModalityRules     []string  // 模态分类
    AuthzRules        []string  // 授权规则
    JailbreakRules    []string  // 越狱检测
    PIIRules          []string  // PII 规则
    KBRules           []string  // 知识库规则
    ConversationRules []string  // 对话形状规则
    ProjectionRules   []string  // 投影规则

    SignalConfidences map[string]float64  // 信号置信度
}
```

#### 决策评估流程

```go
// 1. 评估所有决策
func (e *DecisionEngine) EvaluateDecisionsWithSignals(signals *SignalMatches) (*DecisionResult, error) {
    var results []DecisionResult
    for _, decision := range e.decisions {
        matched, confidence, matchedRules := e.evaluateDecisionWithSignals(decision, signals)
        if matched {
            results = append(results, DecisionResult{
                Decision:     decision,
                Confidence:   confidence,
                MatchedRules: matchedRules,
            })
        }
    }
    return e.selectBestDecision(results), nil
}

// 2. 递归评估规则树
func (e *DecisionEngine) evalNode(node config.RuleNode, signals *SignalMatches) (bool, float64, []string) {
    if node.IsLeaf() {
        return e.evalLeaf(node.Type, node.Name, signals)
    }
    switch strings.ToUpper(node.Operator) {
    case "AND": return e.evalAND(node.Conditions, signals)
    case "NOT": return e.evalNOT(node.Conditions, signals)
    default:    return e.evalOR(node.Conditions, signals)  // OR
    }
}
```

---

### 3.6 pkg/extproc - Envoy ExtProc 处理器

**文件位置**: `src/semantic-router/pkg/extproc/`

#### OpenAIRouter 结构

```go
type OpenAIRouter struct {
    Config                *config.RouterConfig
    Classifier            *classification.Classifier
    ClassificationService *services.ClassificationService
    Cache                 cache.CacheBackend
    ToolsDatabase         *tools.ToolsDatabase
    ResponseAPIFilter     *ResponseAPIFilter
    ReplayRecorder        *routerreplay.Recorder
    ModelSelector         *selection.Registry
    MemoryStore           memory.Store
    MemoryExtractor       *memory.MemoryExtractor
    CredentialResolver    *authz.CredentialResolver
    RateLimiter          *ratelimit.RateLimitResolver
    RuntimeRegistry      *routerruntime.Registry
}
```

#### 请求处理流程

```
Envoy ExtProc ──▶ RequestHeaders ──▶ RequestBody ──▶ ResponseHeaders ──▶ ResponseBody
      │                │                  │                │                │
      │         headers         ├─▶ 分类决策          ├─▶ 缓存写入        ├─▶ 记忆存储
      │         验证             │  缓存查找           │  响应处理         │  幻觉检测
      │         授权             │  RAG 处理          │                   │
      │         模型选择          │  工具过滤           │                   │
```

#### 请求过滤器 (req_filter_*.go)

| 过滤器 | 说明 |
|--------|------|
| `req_filter_classification.go` | 分类决策 |
| `req_filter_cache.go` | 语义缓存 |
| `req_filter_rag.go` | RAG 处理 |
| `req_filter_memory.go` | 记忆检索 |
| `req_filter_modality.go` | 模态检测 |
| `req_filter_tools.go` | 工具过滤 |
| `req_filter_jailbreak.go` | 越狱检测 |
| `req_filter_pii.go` | PII 检测 |

#### 响应过滤器 (res_filter_*.go)

| 过滤器 | 说明 |
|--------|------|
| `res_filter_hallucination.go` | 幻觉检测 |
| `res_filter_jailbreak.go` | 越狱检测响应 |

---

### 3.7 pkg/dsl - DSL 编译器

**文件位置**: `src/semantic-router/pkg/dsl/`

#### 编译器结构

```go
type Compiler struct {
    prog            *Program           // AST
    config          *config.RouterConfig
    pluginTemplates map[string]*PluginDecl
    errors          []error
}
```

#### DSL 示例

```yaml
# 信号定义
SIGNAL keyword ai_questions {
    keywords: ["how", "what", "why", "when", "where"]
    operator: OR
}

SIGNAL domain coding {
    mmlu_categories: ["computer science", "programming"]
}

SIGNAL complexity hard {
    threshold: 0.8
}

# 路由决策
ROUTE fast_response
WHEN keyword ai_questions AND complexity hard
MODEL gpt-4-turbo
PLUGIN fast_response { message: "I'll think about this..." }

ROUTE code_generation
WHEN domain coding AND complexity hard
MODEL code-llama
MODEL codellama-34b
ALGORITHM confidence { threshold: 0.8 }
```

#### 编译流程

```go
func (c *Compiler) compile() {
    // 1. 注册插件模板
    for _, p := range c.prog.Plugins {
        c.pluginTemplates[p.Name] = p
    }
    // 2. 编译信号
    c.compileSignals()
    // 3. 编译投影分区
    c.compileProjectionPartitions()
    // 4. 编译投影
    c.compileProjectionScores()
    c.compileProjectionMappings()
    // 5. 编译模型目录
    c.compileModels()
    // 6. 编译会话状态
    c.compileSessionStates()
    // 7. 编译路由
    c.compileRoutes()
}
```

---

### 3.8 pkg/cache - 缓存系统

**文件位置**: `src/semantic-router/pkg/cache/`

#### 缓存接口

```go
type CacheBackend interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, value []byte, ttl time.Duration) error
    Delete(ctx context.Context, key string) error
    Exists(ctx context.Context, key string) (bool, error)
    Close() error
}
```

#### 支持的后端

| 后端 | 文件 | 说明 |
|------|------|------|
| Redis | `redis_cache.go` | Redis 单机/集群 |
| Valkey | `valkey_cache.go` | Redis 兼容存储 |
| Milvus | `milvus_cache.go` | 向量相似度缓存 |
| 内存 | `inmemory_cache.go` | 开发/测试用 |
| 混合 | `hybrid_cache.go` | 多级缓存 |

#### 用户作用域缓存

```go
// ScopeQueryToUser 为缓存查询添加用户命名空间
func ScopeQueryToUser(query string, userID string) string {
    namespace := userScopeNamespace(userID)
    return fmt.Sprintf("cache-scope %s %s", namespace, query)
}
```

---

### 3.9 pkg/memory - 记忆系统

**文件位置**: `src/semantic-router/pkg/memory/`

#### Store 接口

```go
type Store interface {
    Store(ctx context.Context, memory *Memory) error
    Retrieve(ctx context.Context, opts RetrieveOptions) ([]*RetrieveResult, error)
    Get(ctx context.Context, id string) (*Memory, error)
    Update(ctx context.Context, id string, memory *Memory) error
    List(ctx context.Context, opts ListOptions) (*ListResult, error)
    Forget(ctx context.Context, id string) error
    ForgetByScope(ctx context.Context, scope MemoryScope) error
    IsEnabled() bool
    CheckConnection(ctx context.Context) error
    Close() error
}
```

#### 记忆类型

```go
type Memory struct {
    ID        string                 `json:"id"`
    UserID    string                 `json:"user_id"`
    Content   string                 `json:"content"`
    Metadata  map[string]interface{} `json:"metadata"`
    Type      MemoryType             `json:"type"`
    CreatedAt time.Time              `json:"created_at"`
    UpdatedAt time.Time              `json:"updated_at"`
}

type MemoryType string
const (
    MemoryTypeUserMessage     MemoryType = "user_message"
    MemoryTypeAssistantReply  MemoryType = "assistant_reply"
    MemoryTypeSystemReflection MemoryType = "system_reflection"
    MemoryTypeToolResult      MemoryType = "tool_result"
)
```

---

### 3.10 pkg/selection - 模型选择

**文件位置**: `src/semantic-router/pkg/selection/`

#### 支持的算法

| 算法 | 说明 |
|------|------|
| `confidence` | 置信度选择 |
| `ratings` | 评分驱动 |
| `elo` | ELO 评分系统 |
| `remom` | Reasoning on Mixture of Models |
| `router_dc` | 路由决策边界 |
| `automix` | 自动混合选择 |
| `hybrid` | 混合算法 |
| `rl_driven` | 强化学习驱动 |
| `gmtrouter` | GMT 路由器 |
| `latency_aware` | 延迟感知 |

---

## 4. 配置示例

### 4.1 完整配置示例

```yaml
version: "0.3"

listeners:
  - port: 8080
    protocol: http

providers:
  defaults:
    default_model: gpt-4
    reasoning_families:
      default:
        default_effort: medium
  models:
    - name: gpt-4
      backend_refs:
        - endpoint: api.openai.com:443
          type: openai
          weight: 1
      pricing:
        input: 0.03
        output: 0.06
    - name: gpt-3.5-turbo
      backend_refs:
        - endpoint: api.openai.com:443
          type: openai
          weight: 3

routing:
  modelCards:
    - name: gpt-4
      param_size: 175B
      context_window_size: 128000
    - name: gpt-3.5-turbo
      param_size: 6B
      context_window_size: 16385

  signals:
    keywords:
      - name: coding
        keywords: ["code", "programming", "function", "debug"]
        operator: OR
    domains:
      - name: coding
        mmlu_categories: ["computer science"]

  decisions:
    - name: simple_query
      priority: 10
      rules:
        operator: OR
        conditions:
          - type: keyword
            name: coding
      modelRefs:
        - model: gpt-3.5-turbo

    - name: complex_query
      priority: 5
      rules:
        operator: AND
        conditions:
          - type: domain
            name: coding
      modelRefs:
        - model: gpt-4
      plugins:
        - type: semantic-cache
          configuration:
            similarity_threshold: 0.95

global:
  cache:
    enabled: true
    backend: redis
    ttl: 3600
  memory:
    enabled: true
```

---

## 5. 运行方式

### 5.1 本地运行

```bash
# 构建
cd src/semantic-router
go build -o vllm-sr ./cmd/main.go

# 运行
./vllm-sr serve --config config/router.yaml --port 8080
```

### 5.2 Docker 运行

```bash
# 构建镜像
make vllm-sr-dev ENV=cpu

# 运行
docker run -p 8080:8080 -p 8081:8081 \
  -v $(pwd)/config:/config \
  vllm-sr:latest serve --config /config/router.yaml
```

### 5.3 Kubernetes 部署

```bash
# 使用 operator 部署
kubectl apply -f deploy/operator/config/samples/

# 部署自定义配置
kubectl apply -f - <<EOF
apiVersion: vllm.ai/v1alpha1
kind: SemanticRouter
metadata:
  name: my-router
spec:
  config: |
    version: "0.3"
    routing:
      decisions:
        - name: default
          modelRefs:
            - model: gpt-4
EOF
```

---

## 6. 关键依赖关系图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              依赖层级                                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Level 5: 应用层                                                             │
│  ┌─────────────────────┐                                                     │
│  │  cmd/main.go        │  入口点，协调所有组件                                │
│  └─────────┬───────────┘                                                     │
│            │                                                                  │
│  Level 4: 服务层                                                             │
│  ┌─────────┴───────────┐  ┌─────────────────┐                               │
│  │  pkg/apiserver     │  │  pkg/extproc     │                               │
│  │  (HTTP Server)      │  │  (gRPC Server)   │                               │
│  └─────────┬───────────┘  └─────────┬─────────┘                               │
│            │                        │                                          │
│  Level 3: 业务逻辑层                                                          │
│  ┌─────────┴───────────┐  ┌─────────┴───────────┐  ┌─────────────────┐      │
│  │  pkg/classification │  │  pkg/decision       │  │  pkg/selection   │      │
│  │  (分类引擎)         │  │  (决策引擎)         │  │  (模型选择)       │      │
│  └─────────┬───────────┘  └─────────────────────┘  └────────┬────────┘      │
│            │                                                    │             │
│  Level 2: 数据层                                                              │
│  ┌─────────┴───────────┐  ┌─────────┴───────────┐  ┌─────────┴────────┐     │
│  │  pkg/cache         │  │  pkg/memory         │  │  pkg/dsl         │     │
│  │  (缓存系统)        │  │  (记忆存储)         │  │  (DSL 编译)       │     │
│  └─────────┬───────────┘  └─────────┬───────────┘  └──────────────────┘     │
│            │                        │                                          │
│  Level 1: 基础设施                                                            │
│  ┌─────────┴───────────┐  ┌─────────┴───────────┐  ┌─────────┴────────┐     │
│  │  pkg/config         │  │  pkg/observability │  │  External libs   │     │
│  │  (配置管理)        │  │  (日志/指标)        │  │  (gRPC, Redis)   │     │
│  └────────────────────┘  └────────────────────┘  └──────────────────┘     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 扩展指南

### 7.1 添加新的信号类型

1. 在 `pkg/config/` 中定义信号配置结构
2. 在 `pkg/classification/` 中实现分类器
3. 在 `pkg/decision/` 中添加规则匹配逻辑
4. 在 `pkg/dsl/` 中添加 DSL 语法支持

### 7.2 添加新的模型选择算法

1. 在 `pkg/selection/` 中实现 `Selector` 接口
2. 在配置中注册算法
3. 在决策引擎中调用

### 7.3 添加新的缓存后端

1. 在 `pkg/cache/` 中实现 `CacheBackend` 接口
2. 在缓存工厂中注册

---

## 8. 测试指南

### 8.1 运行测试

```bash
# 运行所有测试
cd src/semantic-router
go test ./...

# 运行特定模块测试
go test ./pkg/classification/...

# 运行集成测试
go test -tags=integration ./...
```

### 8.2 测试覆盖

| 模块 | 测试文件模式 | 说明 |
|------|-------------|------|
| classification | `*_test.go` | 单元测试 |
| decision | `*_test.go` | 规则评估测试 |
| dsl | `*_test.go`, `*_roundtrip_test.go` | 编译/反编译测试 |
| cache | `*_integration_test.go` | 集成测试 |

---

## 9. 贡献指南

### 9.1 代码规范

- 使用 `gofmt` 格式化代码
- 遵循 Go 命名约定
- 添加适当的注释和文档
- 确保测试覆盖率

### 9.2 提交流程

1. Fork 仓库
2. 创建功能分支
3. 编写代码和测试
4. 运行 `make agent-validate`
5. 提交 Pull Request

### 9.3 验证命令

```bash
make agent-validate    # 基础验证
make agent-lint       # 代码检查
make agent-ci-gate    # CI 门禁
make agent-feature-gate ENV=cpu CHANGED_FILES="..."  # 特性验证
```
