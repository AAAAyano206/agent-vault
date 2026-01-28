# agent-vault
Agent即服务，支持多租户、高并发、工具调用、长记忆的Agent服务

# 核心架构
┌─────────────────────────────────────────────────────────────┐
│                        客户端层                              │
│  (OpenAI兼容API / Web控制台 / CLI工具)                       │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│                   Gateway (Rust + Axum)                     │
│  • JWT多租户认证 │ Token Bucket限流 │ 请求路由 │ 健康检查      │
│  • 并发控制(Semaphore) │ 超时管理 │ 日志追踪(Tracing)         │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│              Scheduler (Rust)                               │
│  • GPU显存池管理 │ PagedAttention实现 │ 动态批处理(Batching)  │
│  • 模型热切换 │ 优先级队列 │ OOM防护 │ 显存碎片整理            │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│          Inference Engine (Python vLLM + Rust FFI)          │
│  • Llama 3.1 8B/70B-int4 │ Embedding模型 │ 量化推理          │
│  • Tensor并行(模拟) │ 流水线并行 │ Continuous Batching        │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│              Agent Core (Rust + Python沙箱)                 │
│  • ReAct引擎 │ Actor模型 │ 工具注册表 │ 异步执行器             │
│  • 分层记忆系统(短期/长期) │ 状态持久化 │ 工作流编排            │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│            RAG Kernel (Rust手写向量检索)                     │
│  • HNSW索引 │ IVF索引 │ 增量索引 │ 向量量化                   │
│  • GPU加速检索 │ 重排序(Rerank) │ 文档分块                    │
└─────────────────────────────────────────────────────────────┘

# 目录树规划
agent-vault/
├── Cargo.toml                    # Workspace配置，定义所有成员
├── .env                          # 本地环境变量(API密钥/端口等)
├── .gitignore
├── README.md
├── docs/
│   ├── ARCHITECTURE.md           # 架构设计文档
│   └── API_SPEC.md              # OpenAI API兼容规范
├── gateway/                      # API网关服务
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs              # 入口：初始化Tokio运行时、配置加载
│       ├── config.rs            # 配置管理：环境变量→结构体
│       ├── error.rs             # 全局错误处理：统一错误响应格式
│       ├── state.rs             # 共享状态：Semaphore、HTTP客户端池
│       ├── lib.rs               # 模块导出(用于集成测试)
│       ├── middleware/
│       │   ├── mod.rs           # 中间件模块聚合
│       │   ├── auth.rs          # JWT验证中间件
│       │   ├── rate_limit.rs    # Token Bucket限流
│       │   └── tracing.rs       # 请求日志追踪
│       ├── handlers/
│       │   ├── mod.rs           # Handler路由聚合
│       │   ├── chat.rs          # /v1/chat/completions 核心接口
│       │   └── health.rs        # /health 健康检查
│       └── models/
│           ├── mod.rs           # 模型模块导出
│           ├── protocol.rs      # OpenAI API协议结构体
│           └── stream.rs        # SSE流式响应结构
├── scheduler/                    # GPU显存调度器
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── memory_pool.rs       # 显存池：分配/释放/碎片整理
│       ├── paged_attention.rs   # PagedAttention实现
│       └── batcher.rs           # 动态批处理队列
├── agent-core/                   # Agent运行时
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── react/
│       │   ├── mod.rs
│       │   ├── engine.rs        # ReAct主循环
│       │   └── state.rs         # 对话状态机
│       ├── tools/
│       │   ├── mod.rs           # 工具注册表
│       │   └── executor.rs      # 沙箱执行器
│       └── memory/
│           ├── mod.rs
│           ├── short_term.rs    # 滑动窗口缓存
│           └── long_term.rs     # 向量记忆接口
├── rag-kernel/                   # 向量检索引擎
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── hnsw.rs              # HNSW索引实现
│       ├── ivf.rs               # IVF索引(备选)
│       └── quantization.rs      # 向量量化
└── shared/                       # 共享库(类型定义/工具函数)
    ├── Cargo.toml
    └── src/
        ├── lib.rs
        └── types.rs             # 通用类型定义(RequestId等)
