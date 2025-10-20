# GenBI 核心功能数据流转图

## 🎯 整体架构图

```mermaid
graph TB
    User[用户提问] --> AskService[Ask Service]
    
    subgraph "Text-to-SQL 流程"
        AskService --> Intent[意图分类<br/>IntentClassification]
        Intent -->|TEXT_TO_SQL| Historical[历史问题检索<br/>HistoricalQuestion]
        Historical -->|命中| DirectReturn[直接返回历史SQL]
        Historical -->|未命中| Parallel[并行检索]
        
        Parallel --> SQLSamples[SQL样例检索<br/>SQLPairsRetrieval]
        Parallel --> Instructions[用户指令检索<br/>InstructionsRetrieval]
        
        SQLSamples --> DBSchema[DB Schema检索<br/>DBSchemaRetrieval]
        Instructions --> DBSchema
        
        DBSchema --> Reasoning[SQL生成推理<br/>SQLGenerationReasoning]
        Reasoning --> SQLGen[SQL生成<br/>SQLGeneration]
        
        SQLGen --> Validate{SQL验证<br/>DryRun}
        Validate -->|通过| FinalSQL[最终SQL]
        Validate -->|失败| Diagnosis[SQL诊断<br/>SQLDiagnosis]
        Diagnosis --> Correction[SQL纠错<br/>SQLCorrection]
        Correction --> Validate
    end
    
    FinalSQL --> ChartService[Chart Service]
    
    subgraph "Text-to-Charts 流程"
        ChartService --> Execute[执行SQL<br/>SQLExecutor]
        Execute --> DataPreprocess[数据预处理<br/>ChartDataPreprocessor]
        DataPreprocess --> ChartGen[图表生成<br/>ChartGeneration]
        ChartGen --> ChartPost[后处理验证<br/>ChartPostProcessor]
        ChartPost --> VegaLite[Vega-Lite Schema]
    end
    
    FinalSQL --> AnswerService[SQL Answer Service]
    Execute --> AnswerService
    
    subgraph "AI 洞察生成流程"
        AnswerService --> AnswerPreprocess[数据预处理<br/>PreprocessSQLData]
        AnswerPreprocess --> AnswerGen[答案生成<br/>SQLAnswer]
        AnswerGen --> Stream[流式返回<br/>Streaming SSE]
    end
    
    DirectReturn --> UI[前端展示]
    FinalSQL --> UI
    VegaLite --> UI
    Stream --> UI
    
    Intent -->|GENERAL| DataAssist[数据辅助<br/>DataAssistance]
    Intent -->|MISLEADING| Mislead[误导查询处理<br/>MisleadingAssistance]
    Intent -->|USER_GUIDE| Guide[用户指南<br/>UserGuideAssistance]
    
    DataAssist --> UI
    Mislead --> UI
    Guide --> UI
    
    style AskService fill:#e1f5ff
    style ChartService fill:#fff4e1
    style AnswerService fill:#e8f5e9
    style UI fill:#f3e5f5
```

## 📋 Text-to-SQL 详细流程图

```mermaid
sequenceDiagram
    participant U as 用户
    participant AS as AskService
    participant IC as IntentClassification
    participant HQ as HistoricalQuestion
    participant DSR as DBSchemaRetrieval
    participant SGR as SQLGenerationReasoning
    participant SG as SQLGeneration
    participant SC as SQLCorrection
    participant WE as WrenEngine
    
    U->>AS: POST /v1/asks<br/>{"query": "top products by sales"}
    AS->>AS: 状态: understanding
    
    AS->>HQ: 检索历史问题
    HQ-->>AS: 未命中
    
    par 并行检索
        AS->>AS: 检索SQL样例
        AS->>AS: 检索用户指令
    end
    
    AS->>IC: 意图分类
    IC-->>AS: TEXT_TO_SQL<br/>rephrased: "What are top 10..."
    
    AS->>AS: 状态: searching
    AS->>DSR: 检索相关Schema
    
    Note over DSR: 1. Embedding 问题<br/>2. 向量检索表<br/>3. 向量检索列<br/>4. 可选列剪枝<br/>5. 构建DDL
    
    DSR-->>AS: tables: [products, sales]<br/>DDL: CREATE TABLE...
    
    AS->>AS: 状态: planning
    AS->>SGR: 生成SQL推理计划
    SGR-->>AS: reasoning: "需要JOIN products和sales..."
    
    AS->>AS: 状态: generating
    AS->>SG: 生成SQL
    
    Note over SG: Prompt构建:<br/>- DB Schema (DDL)<br/>- SQL Samples<br/>- Instructions<br/>- Reasoning Plan
    
    SG->>SG: LLM 生成
    SG->>WE: Dry Plan 验证
    
    alt SQL有效
        WE-->>SG: ✓ 验证通过
        SG-->>AS: valid SQL
        AS->>AS: 状态: finished
        AS-->>U: {"sql": "SELECT...", "status": "finished"}
    else SQL无效
        WE-->>SG: ✗ 语法错误
        SG-->>AS: invalid SQL + error
        AS->>AS: 状态: correcting
        
        loop 最多3次
            AS->>SC: SQL诊断和纠错
            SC->>WE: Dry Plan 验证
            
            alt 纠错成功
                WE-->>SC: ✓ 验证通过
                SC-->>AS: corrected SQL
                AS->>AS: 状态: finished
                AS-->>U: {"sql": "SELECT...", "status": "finished"}
            else 仍然失败
                WE-->>SC: ✗ 仍有错误
            end
        end
        
        AS->>AS: 状态: failed
        AS-->>U: {"error": "NO_RELEVANT_SQL"}
    end
```

## 📊 Text-to-Charts 详细流程图

```mermaid
sequenceDiagram
    participant U as 用户
    participant CS as ChartService
    participant SE as SQLExecutor
    participant CDP as ChartDataPreprocessor
    participant CG as ChartGeneration
    participant CPP as ChartPostProcessor
    participant LLM as LLM Provider
    
    U->>CS: POST /v1/charts<br/>{"query": "...", "sql": "SELECT..."}
    CS->>CS: 状态: fetching
    
    alt 无数据
        CS->>SE: 执行SQL
        SE-->>CS: sql_data: [{...}, {...}]
    else 已有数据
        Note over CS: 使用请求中的data
    end
    
    CS->>CS: 状态: generating
    
    CS->>CDP: 预处理数据
    Note over CDP: 1. 转为DataFrame<br/>2. 取前50行样本<br/>3. 提取列值样本
    CDP-->>CS: sample_data, sample_column_values
    
    CS->>CG: 生成图表
    
    Note over CG: Prompt构建:<br/>- User Question<br/>- SQL Query<br/>- Sample Data<br/>- Sample Column Values<br/>- Language<br/>- Custom Instruction
    
    CG->>LLM: 图表生成请求
    
    Note over LLM: System Prompt:<br/>- 图表类型指南<br/>- 数据类型分析<br/>- Vega-Lite语法<br/><br/>分析:<br/>1. 数据类型识别<br/>2. 选择图表类型<br/>3. 生成Vega-Lite Schema
    
    LLM-->>CG: {<br/>"reasoning": "...",<br/>"chart_type": "line",<br/>"chart_schema": {...}<br/>}
    
    CG->>CPP: 后处理
    
    Note over CPP: 1. 解析LLM结果<br/>2. 添加$schema<br/>3. 填充数据到schema<br/>4. Vega-Lite验证<br/>5. 可选移除数据
    
    alt 验证通过
        CPP-->>CS: valid chart schema
        CS->>CS: 状态: finished
        CS-->>U: {<br/>"status": "finished",<br/>"response": {<br/>  "reasoning": "...",<br/>  "chart_type": "line",<br/>  "chart_schema": {...}<br/>}<br/>}
    else 验证失败
        CPP-->>CS: empty schema
        CS->>CS: 状态: failed
        CS-->>U: {"error": "NO_CHART"}
    end
```

## 💡 AI 洞察生成详细流程图

```mermaid
sequenceDiagram
    participant U as 用户
    participant SAS as SQLAnswerService
    participant PSD as PreprocessSQLData
    participant SA as SQLAnswer
    participant LLM as LLM Provider (Streaming)
    participant Queue as AsyncIO Queue
    
    U->>SAS: POST /v1/sql-answers<br/>{"query": "...", "sql": "...", "sql_data": {...}}
    
    SAS->>SAS: 状态: preprocessing
    
    SAS->>PSD: 预处理SQL数据
    
    Note over PSD: Token计数和限制:<br/>1. 转为DataFrame<br/>2. 计算每行token数<br/>3. 累计至max_tokens(10000)<br/>4. 截断数据
    
    PSD-->>SAS: preprocessed_data<br/>num_rows_used: 50
    
    alt 无数据
        SAS-->>U: {"error": "NO_DATA"}
    else 有数据
        SAS->>SAS: 状态: succeeded
        SAS-->>U: {"status": "succeeded", "num_rows_used": 50}
        
        Note over SAS: 异步启动答案生成<br/>asyncio.create_task()
        
        SAS->>SA: 生成答案（异步）
        
        Note over SA: Prompt构建:<br/>- User Question<br/>- SQL Query<br/>- Data (columns + rows)<br/>- Language<br/>- Current Time<br/>- Custom Instruction
        
        SA->>LLM: 流式生成请求
        
        Note over LLM: System Prompt:<br/>- 非技术用户友好<br/>- Markdown格式<br/>- 简洁清晰<br/>- 避免技术术语
        
        loop 流式生成
            LLM->>Queue: chunk: "Based on the data..."
            LLM->>Queue: chunk: "\n\n**Key Insights**:..."
            LLM->>Queue: chunk: "\n- Total sales: $..."
        end
        
        LLM->>Queue: <DONE>
        
        U->>SAS: GET /v1/sql-answers/{query_id}/stream
        
        loop 流式返回
            Queue-->>SAS: chunk
            SAS-->>U: SSE: data: {"message": "chunk"}
            
            Note over U: 前端实时显示<br/>Markdown渲染
        end
        
        Queue-->>SAS: <DONE>
        SAS-->>U: SSE: [DONE]
    end
```

## 🔄 完整集成流程（端到端）

```mermaid
flowchart TB
    Start([用户输入问题]) --> Ask[Text-to-SQL]
    
    subgraph Ask [" "]
        direction TB
        A1[意图分类] --> A2{意图类型}
        A2 -->|TEXT_TO_SQL| A3[Schema检索]
        A2 -->|GENERAL| A4[数据辅助]
        A2 -->|MISLEADING| A5[误导处理]
        A2 -->|USER_GUIDE| A6[用户指南]
        
        A3 --> A7[生成推理]
        A7 --> A8[SQL生成]
        A8 --> A9{验证}
        A9 -->|失败| A10[纠错]
        A10 --> A9
        A9 -->|成功| A11[最终SQL]
    end
    
    A4 --> End1([返回文本答案])
    A5 --> End1
    A6 --> End1
    
    A11 --> B[并行处理]
    
    subgraph B [" "]
        direction LR
        B1[Text-to-Charts]
        B2[AI洞察生成]
    end
    
    A11 --> B1
    A11 --> B2
    
    subgraph Charts [" "]
        direction TB
        C1[执行SQL] --> C2[数据预处理]
        C2 --> C3[图表生成]
        C3 --> C4[Vega-Lite验证]
        C4 --> C5[图表Schema]
    end
    
    B1 --> C1
    
    subgraph Insights [" "]
        direction TB
        D1[数据预处理] --> D2[LLM流式生成]
        D2 --> D3[Markdown答案]
    end
    
    B2 --> D1
    
    C5 --> UI[前端展示]
    D3 --> UI
    A11 --> UI
    
    UI --> End2([完整结果展示])
    
    style Ask fill:#e1f5ff,stroke:#0288d1
    style Charts fill:#fff4e1,stroke:#ff9800
    style Insights fill:#e8f5e9,stroke:#4caf50
    style UI fill:#f3e5f5,stroke:#9c27b0
```

## 🏗️ 系统架构层次图

```mermaid
graph TB
    subgraph Frontend ["前端层 (wren-ui)"]
        UI1[React组件]
        UI2[GraphQL客户端]
        UI3[SSE处理]
        UI4[Vega-Lite渲染]
    end
    
    subgraph APIGateway ["API网关层"]
        API1[REST API]
        API2[GraphQL API]
        API3[WebSocket/SSE]
    end
    
    subgraph Services ["服务层 (wren-ai-service)"]
        S1[AskService]
        S2[ChartService]
        S3[SQLAnswerService]
        S4[其他服务...]
    end
    
    subgraph Pipelines ["Pipeline层"]
        direction LR
        P1[Generation<br/>Pipelines]
        P2[Retrieval<br/>Pipelines]
        P3[Indexing<br/>Pipelines]
    end
    
    subgraph Core ["核心层"]
        C1[LLM Provider<br/>OpenAI/Anthropic/...]
        C2[Embedder Provider<br/>OpenAI/HuggingFace]
        C3[Document Store<br/>Qdrant]
        C4[Wren Engine<br/>SQL执行/验证]
    end
    
    subgraph Data ["数据层"]
        D1[向量数据库<br/>Qdrant]
        D2[关系数据库<br/>PostgreSQL]
        D3[语义层<br/>MDL]
    end
    
    Frontend --> APIGateway
    APIGateway --> Services
    Services --> Pipelines
    Pipelines --> Core
    Core --> Data
    
    style Frontend fill:#e3f2fd
    style APIGateway fill:#fff3e0
    style Services fill:#e8f5e9
    style Pipelines fill:#fce4ec
    style Core fill:#f3e5f5
    style Data fill:#e0f2f1
```

## 📊 数据流向详解

### 1. 向量检索流程

```
用户问题: "What are the top selling products?"
    ↓
[Embedder] 文本向量化
    ↓
embedding: [0.234, -0.567, 0.891, ...]  (768维向量)
    ↓
[Qdrant 向量数据库] 相似度检索
    ↓
检索结果:
├─ 表: products (相似度: 0.95)
├─ 表: sales (相似度: 0.92)
├─ 列: product_name (相似度: 0.88)
├─ 列: quantity_sold (相似度: 0.85)
└─ 指标: total_revenue (相似度: 0.83)
    ↓
[构建DDL]
    ↓
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  product_name VARCHAR(255),
  category VARCHAR(100)
);

CREATE TABLE sales (
  sale_id INT PRIMARY KEY,
  product_id INT,
  quantity_sold INT,
  revenue DECIMAL(10,2)
);
```

### 2. LLM Prompt 构建流程

```
[System Prompt]
You are a Trino SQL expert...

[User Prompt]
### DATABASE SCHEMA ###
CREATE TABLE products (...)
CREATE TABLE sales (...)

### SQL SAMPLES ###
Question: Show me customer orders
SQL: SELECT * FROM orders WHERE...

### USER INSTRUCTIONS ###
1. Always use proper JOIN conditions
2. Format dates as YYYY-MM-DD

### QUESTION ###
User's Question: What are the top selling products?

### REASONING PLAN ###
1. JOIN products and sales tables
2. GROUP BY product
3. ORDER BY total quantity
4. LIMIT to top 10

Let's think step by step.
```

### 3. SQL 生成和验证流程

```
[LLM 生成]
SELECT 
  p.product_name,
  SUM(s.quantity_sold) as total_sold
FROM products p
JOIN sales s ON p.product_id = s.product_id
GROUP BY p.product_name
ORDER BY total_sold DESC
LIMIT 10
    ↓
[Wren Engine Dry Plan]
- 语法检查: ✓
- 表存在性: ✓
- 列存在性: ✓
- JOIN条件: ✓
- 类型兼容: ✓
    ↓
[返回验证结果]
status: valid
execution_plan: [...]
```

### 4. 图表生成决策树

```
[数据分析]
列1: product_name (类型: string, 类别: nominal)
列2: total_sold (类型: number, 类别: quantitative)
数据行数: 10
    ↓
[图表类型决策]
- 有1个分类变量 + 1个数值变量
- 目的: 比较不同类别的数值
- 数据行数适中 (10行)
    ↓
[选择: BAR CHART]
理由: 柱状图最适合比较不同类别的数值大小
    ↓
[生成 Vega-Lite Schema]
{
  "mark": {"type": "bar"},
  "encoding": {
    "x": {"field": "product_name", "type": "nominal"},
    "y": {"field": "total_sold", "type": "quantitative"}
  }
}
```

### 5. AI 洞察生成流程

```
[输入数据]
columns: ["product_name", "total_sold"]
data: [
  {"product_name": "Product A", "total_sold": 1500},
  {"product_name": "Product B", "total_sold": 1200},
  ...
]
    ↓
[Token预算计算]
max_tokens: 10000
当前数据tokens: 2500 ✓
    ↓
[LLM 流式生成]
chunk_1: "Based on the sales data, "
chunk_2: "here are the **top selling products**:\n\n"
chunk_3: "1. **Product A** - 1,500 units sold\n"
chunk_4: "2. **Product B** - 1,200 units sold\n"
...
chunk_n: "\n**Key Insight**: Product A dominates..."
    ↓
[流式返回]
通过 AsyncIO Queue + SSE 实时传输到前端
```

## 🎯 关键技术点

### 状态机管理
```
Ask Pipeline States:
understanding → searching → planning → generating → correcting → finished/failed/stopped

Chart Pipeline States:
fetching → generating → finished/failed/stopped

SQL Answer Pipeline States:
preprocessing → succeeded/failed
```

### 缓存策略
```python
# TTL Cache配置
TTLCache(
    maxsize=1_000_000,  # 支持100万并发请求
    ttl=120             # 2分钟过期
)
```

### 并发控制
```python
# 并行执行多个检索任务
await asyncio.gather(
    sql_pairs_retrieval(),
    instructions_retrieval(),
    sql_functions_retrieval()
)
```

### 错误重试
```python
# SQL纠错重试机制
max_retries = 3
for attempt in range(max_retries):
    if sql_valid:
        break
    sql = correct_sql(sql, error)
```

---

这些流程图清晰地展示了 GenBI 三大核心功能的完整数据流转过程，从用户输入到最终结果输出的每一个环节都有详细说明。
