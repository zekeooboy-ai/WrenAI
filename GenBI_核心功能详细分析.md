# GenBI 三大核心功能详细实现分析

## 📊 整体架构概览

GenBI (Generative Business Intelligence) 是 Wren AI 的核心能力，包含三大功能：
1. **Text-to-SQL**: 自然语言转 SQL 查询
2. **Text-to-Charts**: 自动生成数据可视化图表
3. **AI 生成的洞察**: 提供 AI 编写的分析和摘要

---

## 🎯 一、Text-to-SQL 实现详解

### 数据流转图

```
用户问题 (User Question)
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 1. 意图分类 (Intent Classification)                          │
│    - 判断是否为 TEXT_TO_SQL / GENERAL / MISLEADING_QUERY    │
│    - 重新表述问题（如果是 follow-up）                         │
│    文件: intent_classification.py                            │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. 历史问题检索 (Historical Question Retrieval)              │
│    - 检查是否有类似的历史问题                                 │
│    - 如果有，直接返回历史 SQL                                 │
│    文件: historical_question_retrieval.py                    │
└─────────────────────────────────────────────────────────────┘
    ↓ (如果没有历史匹配)
┌─────────────────────────────────────────────────────────────┐
│ 3. 并行检索阶段                                              │
│    ┌──────────────────────┐  ┌─────────────────────────┐   │
│    │ SQL 样例检索         │  │ 用户指令检索             │   │
│    │ (SQL Pairs)          │  │ (Instructions)          │   │
│    │ sql_pairs_retrieval  │  │ instructions_retrieval  │   │
│    └──────────────────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. DB Schema 检索 (DB Schema Retrieval)                      │
│    - Embedding 用户问题                                       │
│    - 向量检索相关的表和列                                     │
│    - 构建表的 DDL                                            │
│    - 可选：列剪枝 (Column Pruning)                           │
│    文件: db_schema_retrieval.py                              │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. SQL 生成推理 (SQL Generation Reasoning) - 可选            │
│    - LLM 生成 SQL 的推理计划                                 │
│    - 思考如何构建 SQL 的步骤                                 │
│    文件: sql_generation_reasoning.py                         │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. SQL 生成 (SQL Generation)                                 │
│    输入:                                                     │
│    - 用户问题                                                │
│    - DB Schema (DDL)                                        │
│    - SQL 推理计划                                            │
│    - SQL 样例                                                │
│    - 用户指令                                                │
│    - SQL 函数列表                                            │
│    ↓                                                         │
│    LLM 生成 → SQL 后处理 → 验证                             │
│    文件: sql_generation.py                                   │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. SQL 验证和纠错 (SQL Correction) - 如果需要                │
│    - Dry Run 执行 SQL                                        │
│    - 如果失败，进行诊断 (SQL Diagnosis)                       │
│    - 自动纠错，最多重试 3 次                                  │
│    文件: sql_correction.py, sql_diagnosis.py                │
└─────────────────────────────────────────────────────────────┘
    ↓
最终 SQL 查询结果
```

### 关键代码逻辑块

#### 1.1 Ask Service 核心流程

```python
# 文件: wren-ai-service/src/web/v1/services/ask.py

class AskService:
    async def ask(self, ask_request: AskRequest):
        # 状态机驱动: understanding → searching → planning → generating → correcting → finished
        
        # 阶段 1: Understanding - 意图分类
        self._ask_results[query_id] = AskResultResponse(status="understanding")
        
        # 检查历史问题
        historical_question = await self._pipelines["historical_question"].run(
            query=user_query, project_id=ask_request.project_id
        )
        
        if historical_question_result:
            # 直接返回历史答案
            return api_results
        
        # 并行检索 SQL 样例和指令
        sql_samples_task, instructions_task = await asyncio.gather(
            self._pipelines["sql_pairs_retrieval"].run(...),
            self._pipelines["instructions_retrieval"].run(...)
        )
        
        # 意图分类
        if self._allow_intent_classification:
            intent_result = await self._pipelines["intent_classification"].run(...)
            intent = intent_result.get("intent")  # TEXT_TO_SQL, GENERAL, MISLEADING_QUERY, USER_GUIDE
            
            if intent != "TEXT_TO_SQL":
                # 调用对应的 assistance pipeline
                return results
        
        # 阶段 2: Searching - 检索相关的数据库 schema
        self._ask_results[query_id] = AskResultResponse(status="searching")
        
        retrieval_result = await self._pipelines["db_schema_retrieval"].run(
            query=user_query,
            histories=histories,
            project_id=ask_request.project_id,
            enable_column_pruning=enable_column_pruning
        )
        documents = retrieval_result["retrieval_results"]
        table_ddls = [doc.get("table_ddl") for doc in documents]
        
        # 阶段 3: Planning - SQL 生成推理
        self._ask_results[query_id] = AskResultResponse(status="planning")
        
        if allow_sql_generation_reasoning:
            sql_generation_reasoning = await self._pipelines[
                "sql_generation_reasoning"
            ].run(
                query=user_query,
                contexts=table_ddls,
                sql_samples=sql_samples,
                instructions=instructions
            )
        
        # 阶段 4: Generating - 生成 SQL
        self._ask_results[query_id] = AskResultResponse(status="generating")
        
        text_to_sql_results = await self._pipelines["sql_generation"].run(
            query=user_query,
            contexts=table_ddls,
            sql_generation_reasoning=sql_generation_reasoning,
            sql_samples=sql_samples,
            instructions=instructions,
            has_calculated_field=has_calculated_field,
            has_metric=has_metric,
            sql_functions=sql_functions,
            use_dry_plan=use_dry_plan
        )
        
        # 阶段 5: Correcting - SQL 纠错（如果需要）
        if failed_dry_run_result:
            while current_retries < max_retries:
                self._ask_results[query_id] = AskResultResponse(status="correcting")
                
                # SQL 诊断
                sql_diagnosis_results = await self._pipelines["sql_diagnosis"].run(...)
                
                # SQL 纠错
                sql_correction_results = await self._pipelines["sql_correction"].run(...)
                
                if valid_result:
                    break
        
        # 阶段 6: Finished
        return final_sql_result
```

#### 1.2 SQL 生成核心逻辑

```python
# 文件: wren-ai-service/src/pipelines/generation/sql_generation.py

sql_generation_system_prompt = """
You are a Trino SQL expert with exceptional logical thinking skills...

### TASK ###
Given a user query and relevant database schema, generate a valid Trino SQL query...

### INSTRUCTIONS ###
- Use ONLY the tables and columns provided in the schema
- Apply proper JOIN conditions using foreign keys
- Handle calculated fields and metrics correctly
- Follow SQL best practices
"""

sql_generation_user_prompt_template = """
### DATABASE SCHEMA ###
{% for document in documents %}
    {{ document }}
{% endfor %}

{% if sql_samples %}
### SQL SAMPLES ###
{% for sample in sql_samples %}
Question: {{sample.question}}
SQL: {{sample.sql}}
{% endfor %}
{% endif %}

{% if instructions %}
### USER INSTRUCTIONS ###
{% for instruction in instructions %}
{{ loop.index }}. {{ instruction }}
{% endfor %}
{% endif %}

### QUESTION ###
User's Question: {{ query }}

{% if sql_generation_reasoning %}
### REASONING PLAN ###
{{ sql_generation_reasoning }}
{% endif %}

Let's think step by step.
"""

class SQLGeneration(BasicPipeline):
    async def run(self, query, contexts, sql_generation_reasoning, ...):
        # 1. 构建 Prompt
        prompt_result = prompt_builder.run(
            query=query,
            documents=contexts,  # DDL
            sql_generation_reasoning=sql_generation_reasoning,
            sql_samples=sql_samples,
            instructions=instructions,
            sql_functions=sql_functions
        )
        
        # 2. LLM 生成 SQL
        generation_result = await generator(prompt=prompt_result["prompt"])
        
        # 3. 后处理：验证和 Dry Run
        final_result = await post_processor.run(
            generation_result["replies"],
            project_id=project_id,
            use_dry_plan=use_dry_plan,  # 使用 Wren Engine 的 dry plan 验证
            allow_dry_plan_fallback=allow_dry_plan_fallback
        )
        
        return final_result
```

#### 1.3 DB Schema 检索逻辑

```python
# 文件: wren-ai-service/src/pipelines/retrieval/db_schema_retrieval.py

class DBSchemaRetrieval:
    async def run(self, query, histories, project_id, enable_column_pruning):
        # 1. Embedding 用户问题（包含历史上下文）
        if histories:
            previous_query_summaries = [h.question for h in histories]
            query = "\n".join(previous_query_summaries) + "\n" + query
        
        embedding_result = await embedder.run(query)
        
        # 2. 检索相关的表
        table_retrieval_result = await table_retriever.run(
            query_embedding=embedding_result["embedding"],
            filters={"type": "TABLE_DESCRIPTION", "project_id": project_id},
            top_k=10
        )
        
        # 3. 对每个表检索相关的列
        for table in retrieved_tables:
            column_retrieval_result = await column_retriever.run(
                query_embedding=embedding_result["embedding"],
                filters={"type": "COLUMN_DESCRIPTION", "table_name": table.name},
                top_k=20
            )
        
        # 4. 可选：列剪枝（使用 LLM 进一步筛选列）
        if enable_column_pruning:
            # 使用 LLM 分析问题，精确选择需要的列
            pruning_result = await column_pruning_llm.run(
                question=query,
                db_schemas=table_ddls
            )
            # 只保留 LLM 选中的列
            refined_columns = pruning_result["chosen_columns"]
        
        # 5. 构建表的 DDL
        table_ddls = []
        for table in tables:
            ddl = build_table_ddl(
                table_name=table.name,
                columns=table.columns,
                primary_key=table.primary_key,
                foreign_keys=table.foreign_keys
            )
            table_ddls.append(ddl)
        
        return {
            "retrieval_results": documents,
            "has_calculated_field": any(col.is_calculated for col in columns),
            "has_metric": any(table.is_metric for table in tables),
            "has_json_field": any(col.type == "json" for col in columns)
        }
```

---

## 📈 二、Text-to-Charts 实现详解

### 数据流转图

```
SQL 查询结果 + 用户问题
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 1. 数据预处理 (Chart Data Preprocessor)                      │
│    - 获取样本数据（前 N 行）                                  │
│    - 提取列值样例                                            │
│    - 数据类型分析                                            │
│    文件: chart.py - ChartDataPreprocessor                    │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. SQL 执行 (可选 - 如果没有提供数据)                        │
│    - 执行 SQL 查询                                           │
│    - 获取查询结果                                            │
│    文件: sql_executor.py                                     │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. 图表生成 (Chart Generation)                               │
│    输入:                                                     │
│    - 用户问题                                                │
│    - SQL 查询                                                │
│    - 样本数据                                                │
│    - 样本列值                                                │
│    - 语言偏好                                                │
│    - 自定义指令                                              │
│    ↓                                                         │
│    LLM 分析 → 选择图表类型 → 生成 Vega-Lite Schema          │
│    文件: chart_generation.py                                 │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 后处理 (Chart Generation Post Processor)                  │
│    - 验证 Vega-Lite Schema                                   │
│    - 填充数据到 Schema                                       │
│    - 可选：移除数据（只返回 Schema）                          │
│    文件: chart.py - ChartGenerationPostProcessor             │
└─────────────────────────────────────────────────────────────┘
    ↓
Vega-Lite 图表 Schema + 推理说明
```

### 关键代码逻辑块

#### 2.1 Chart Service 核心流程

```python
# 文件: wren-ai-service/src/web/v1/services/chart.py

class ChartService:
    async def chart(self, chart_request: ChartRequest):
        query_id = chart_request.query_id
        
        # 状态 1: Fetching - 获取 SQL 执行结果
        if not chart_request.data:
            self._chart_results[query_id] = ChartResultResponse(status="fetching")
            
            execute_sql_result = await self._pipelines["sql_executor"].run(
                sql=chart_request.sql,
                project_id=chart_request.project_id
            )
            sql_data = execute_sql_result["results"]
        else:
            sql_data = chart_request.data
        
        # 状态 2: Generating - 生成图表
        self._chart_results[query_id] = ChartResultResponse(status="generating")
        
        chart_generation_result = await self._pipelines["chart_generation"].run(
            query=chart_request.query,
            sql=chart_request.sql,
            data=sql_data,
            language=chart_request.configurations.language,
            remove_data_from_chart_schema=chart_request.remove_data_from_chart_schema,
            custom_instruction=chart_request.custom_instruction
        )
        
        chart_result = chart_generation_result["post_process"]["results"]
        
        # 状态 3: Finished
        self._chart_results[query_id] = ChartResultResponse(
            status="finished",
            response=ChartResult(
                reasoning=chart_result["reasoning"],
                chart_type=chart_result["chart_type"],
                chart_schema=chart_result["chart_schema"]
            )
        )
```

#### 2.2 图表生成核心逻辑

```python
# 文件: wren-ai-service/src/pipelines/generation/chart_generation.py

chart_generation_system_prompt = """
### TASK ###
You are a data analyst great at visualizing data using vega-lite! 
Given the user's question, SQL, sample data and sample column values, 
you need to generate vega-lite schema in JSON and provide suitable chart type.

### INSTRUCTIONS ###
- Chart types: Bar chart, Line chart, Multi line chart, Area chart, Pie chart, 
  Stacked bar chart, Grouped bar chart
- You can only use the chart types provided
- Generated chart should answer the user's question and based on the semantics 
  of the SQL query, and the sample data
- If the sample data is not suitable for visualization, return empty string
- The language for the chart and reasoning must be the same as user specified

### GUIDELINES TO PLOT CHART ###

1. Understanding Your Data Types
   - Nominal (Categorical): Names without order (e.g., fruits, countries)
   - Ordinal: Categorical with order (e.g., rankings)
   - Quantitative: Numerical values (e.g., sales, temperatures)
   - Temporal: Date or time data

2. Chart Types and When to Use Them
   - Bar Chart: Comparing quantities across categories
   - Line Chart: Displaying trends over continuous data (time)
   - Multi Line Chart: Comparing multiple metrics over time
   - Pie Chart: Showing parts of a whole as percentages
   - Grouped Bar Chart: Comparing sub-categories within main categories
   - Stacked Bar Chart: Showing composition and comparison
   - Area Chart: Emphasizing volume of change over time

### OUTPUT FORMAT ###
{
    "reasoning": <REASON_IN_USER_LANGUAGE>,
    "chart_type": "line" | "multi_line" | "bar" | "pie" | "grouped_bar" | "stacked_bar" | "area" | "",
    "chart_schema": <VEGA_LITE_JSON_SCHEMA>
}
"""

chart_generation_user_prompt_template = """
### INPUT ###
Question: {{ query }}
SQL: {{ sql }}
Sample Data: {{ sample_data }}
Sample Column Values: {{ sample_column_values }}
Language: {{ language }}
Custom Instruction: {{ custom_instruction }}

Please think step by step
"""

class ChartGeneration(BasicPipeline):
    async def run(self, query, sql, data, language, custom_instruction, ...):
        # 1. 数据预处理
        preprocessed = chart_data_preprocessor.run(data)
        sample_data = preprocessed["sample_data"]  # 前 50 行
        sample_column_values = preprocessed["sample_column_values"]  # 每列的前几个唯一值
        
        # 2. 构建 Prompt
        prompt_result = prompt_builder.run(
            query=query,
            sql=sql,
            sample_data=sample_data,
            sample_column_values=sample_column_values,
            language=language,
            custom_instruction=custom_instruction or ""
        )
        
        # 3. LLM 生成图表
        generation_result = await generator(prompt=prompt_result["prompt"])
        
        # 4. 后处理：验证和格式化
        final_result = post_processor.run(
            generation_result["replies"],
            vega_schema=vega_lite_v5_schema,
            sample_data=sample_data,
            remove_data_from_chart_schema=remove_data_from_chart_schema
        )
        
        return final_result
```

#### 2.3 图表数据预处理

```python
# 文件: wren-ai-service/src/pipelines/generation/utils/chart.py

class ChartDataPreprocessor:
    def run(self, data: Dict[str, Any]) -> dict:
        # 1. 转换为 DataFrame
        df = pd.DataFrame(data)
        
        # 2. 限制样本数据大小（前 50 行）
        sample_data = df.head(50).to_dict(orient="records")
        
        # 3. 提取每列的样本值（用于 LLM 理解数据特征）
        sample_column_values = {}
        for column in df.columns:
            unique_values = df[column].unique()[:5]  # 每列取前 5 个唯一值
            sample_column_values[column] = unique_values.tolist()
        
        return {
            "sample_data": sample_data,
            "sample_column_values": sample_column_values
        }

class ChartGenerationPostProcessor:
    def run(self, replies, vega_schema, sample_data, remove_data_from_chart_schema):
        # 1. 解析 LLM 生成的结果
        generation_result = orjson.loads(replies[0])
        
        chart_type = generation_result.get("chart_type", "")
        chart_schema = generation_result.get("chart_schema", {})
        reasoning = generation_result.get("reasoning", "")
        
        if not chart_schema:
            return {"chart_schema": {}, "chart_type": "", "reasoning": reasoning}
        
        # 2. 将数据填充到 Vega-Lite Schema
        chart_schema["$schema"] = "https://vega.github.io/schema/vega-lite/v5.json"
        chart_schema["data"] = {"values": sample_data}
        
        # 3. 验证 Schema 是否符合 Vega-Lite 规范
        try:
            validate(chart_schema, schema=vega_schema)
        except ValidationError as e:
            logger.error(f"Invalid Vega-Lite schema: {e}")
            return {"chart_schema": {}, "chart_type": "", "reasoning": ""}
        
        # 4. 可选：移除数据（只返回 Schema 结构）
        if remove_data_from_chart_schema:
            chart_schema["data"]["values"] = []
        
        return {
            "chart_schema": chart_schema,
            "chart_type": chart_type,
            "reasoning": reasoning
        }
```

#### 2.4 图表类型定义和示例

```python
# 支持的图表类型及其 Pydantic 模型

class LineChartSchema(ChartSchema):
    class LineChartEncoding(BaseModel):
        x: TemporalChartEncoding | ChartSchema.ChartEncoding  # 时间或分类轴
        y: ChartSchema.ChartEncoding  # 数值轴
        color: ChartSchema.ChartEncoding  # 可选的颜色编码
    
    mark: {"type": "line"}
    encoding: LineChartEncoding

class BarChartSchema(ChartSchema):
    class BarChartEncoding(BaseModel):
        x: ChartSchema.ChartEncoding
        y: ChartSchema.ChartEncoding
        color: ChartSchema.ChartEncoding
    
    mark: {"type": "bar"}
    encoding: BarChartEncoding

class PieChartSchema(ChartSchema):
    class PieChartEncoding(BaseModel):
        theta: ChartSchema.ChartEncoding  # 角度（值）
        color: ChartSchema.ChartEncoding  # 分类（颜色）
    
    mark: {"type": "arc"}
    encoding: PieChartEncoding

# 示例：生成的 Vega-Lite Schema
{
    "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
    "mark": {"type": "bar"},
    "encoding": {
        "x": {
            "field": "category",
            "type": "nominal",
            "title": "Product Category"
        },
        "y": {
            "field": "sales",
            "type": "quantitative",
            "title": "Total Sales"
        },
        "color": {
            "field": "region",
            "type": "nominal"
        }
    },
    "data": {"values": [...]}
}
```

#### 2.5 图表调整功能

```python
# 文件: wren-ai-service/src/pipelines/generation/chart_adjustment.py

# 用户可以调整已生成的图表
class ChartAdjustment(BasicPipeline):
    async def run(self, query, sql, data, adjustment_option, chart_schema):
        # adjustment_option 包含：
        # - chart_type: 切换图表类型
        # - x: 更改 X 轴字段
        # - y: 更改 Y 轴字段
        # - color: 更改颜色编码字段
        # - x_offset: 用于分组柱状图
        # - theta: 用于饼图
        
        # 重新生成图表，基于调整选项
        prompt = f"""
        Original Chart: {chart_schema}
        User Adjustments:
        - Chart Type: {adjustment_option.chart_type}
        - X Axis: {adjustment_option.x}
        - Y Axis: {adjustment_option.y}
        ...
        
        Please regenerate the Vega-Lite schema based on these adjustments.
        """
        
        return await self.generate_adjusted_chart(prompt)
```

---

## 💡 三、AI 生成的洞察实现详解

### 数据流转图

```
SQL 查询结果 + 用户问题
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 1. SQL 数据预处理                                            │
│    - 限制行数（避免超过 LLM token 限制）                      │
│    - 格式化数据结构                                          │
│    文件: preprocess_sql_data.py                              │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. SQL Answer Generation (流式生成)                          │
│    输入:                                                     │
│    - 用户问题                                                │
│    - SQL 查询                                                │
│    - 查询结果数据                                            │
│    - 语言偏好                                                │
│    - 当前时间（用于时间相关分析）                             │
│    - 自定义指令                                              │
│    ↓                                                         │
│    LLM 流式生成 → Markdown 格式答案                          │
│    文件: sql_answer.py                                       │
└─────────────────────────────────────────────────────────────┘
    ↓
流式返回 AI 生成的分析洞察
```

### 关键代码逻辑块

#### 3.1 SQL Answer Service 核心流程

```python
# 文件: wren-ai-service/src/web/v1/services/sql_answer.py

class SqlAnswerService:
    async def sql_answer(self, sql_answer_request: SqlAnswerRequest):
        query_id = sql_answer_request.query_id
        
        # 状态 1: Preprocessing - 预处理数据
        self._sql_answer_results[query_id] = SqlAnswerResultResponse(
            status="preprocessing"
        )
        
        preprocessed_sql_data = self._pipelines["preprocess_sql_data"].run(
            sql_data=sql_answer_request.sql_data
        )["preprocess"]
        
        num_rows_used = preprocessed_sql_data.get("num_rows_used_in_llm")
        
        if num_rows_used == 0:
            # 没有数据可分析
            return {"error": "No data to answer"}
        
        # 状态 2: Succeeded - 异步开始生成答案（流式）
        self._sql_answer_results[query_id] = SqlAnswerResultResponse(
            status="succeeded",
            num_rows_used_in_llm=num_rows_used
        )
        
        # 异步流式生成答案
        asyncio.create_task(
            self._pipelines["sql_answer"].run(
                query=sql_answer_request.query,
                sql=sql_answer_request.sql,
                sql_data=preprocessed_sql_data["sql_data"],
                language=sql_answer_request.configurations.language,
                current_time=sql_answer_request.configurations.show_current_time(),
                query_id=query_id,
                custom_instruction=sql_answer_request.custom_instruction
            )
        )
    
    async def get_sql_answer_streaming_result(self, query_id: str):
        # 流式返回答案
        async for chunk in self._pipelines["sql_answer"].get_streaming_results(query_id):
            event = SSEEvent(data=SSEEvent.SSEEventMessage(message=chunk))
            yield event.serialize()
```

#### 3.2 SQL Answer 生成核心逻辑

```python
# 文件: wren-ai-service/src/pipelines/generation/sql_answer.py

sql_to_answer_system_prompt = """
### TASK
You are a data analyst that great at answering non-technical user's questions 
based on the data and sql, so that even non technical users can easily understand.
Please answer the user's question in concise and clear manner in Markdown format.

### INSTRUCTIONS
1. Read the user's question and understand the user's intention.
2. Read the sql and understand the data.
3. Make sure the answer is aimed for non-technical users, so don't mention any 
   technical terms such as SQL syntax.
4. Generate a concise and clear answer in string format to answer the user's question 
   based on the data and sql.
5. If answer is in list format, only list top few examples, and tell users there 
   are more results omitted.
6. Answer must be in the same language user specified.
7. Do not include ```markdown or ``` in the answer.
8. If the user provides a custom instruction, follow it strictly.

### OUTPUT FORMAT
Please provide your response in proper Markdown string format.
"""

sql_to_answer_user_prompt_template = """
### Inputs ###
User's question: {{ query }}
SQL: {{ sql }}
Data: 
columns: {{ sql_data.columns }}
rows: {{ sql_data.data }}
Language: {{ language }}
Current Time: {{ current_time }}
Custom Instruction: {{ custom_instruction }}

Please think step by step and answer the user's question.
"""

class SQLAnswer(BasicPipeline):
    def __init__(self, llm_provider: LLMProvider):
        self._user_queues = {}  # 为每个用户维护一个队列，用于流式传输
        
        self._components = {
            "generator": llm_provider.get_generator(
                system_prompt=sql_to_answer_system_prompt,
                streaming_callback=self._streaming_callback  # 流式回调
            )
        }
    
    def _streaming_callback(self, chunk, query_id):
        """LLM 流式生成的回调函数"""
        if query_id not in self._user_queues:
            self._user_queues[query_id] = asyncio.Queue()
        
        # 将生成的 chunk 放入队列
        asyncio.create_task(self._user_queues[query_id].put(chunk.content))
        
        # 生成完成标记
        if chunk.meta.get("finish_reason"):
            asyncio.create_task(self._user_queues[query_id].put("<DONE>"))
    
    async def get_streaming_results(self, query_id):
        """流式返回生成的答案"""
        if query_id not in self._user_queues:
            self._user_queues[query_id] = asyncio.Queue()
        
        while True:
            try:
                result = await asyncio.wait_for(
                    self._user_queues[query_id].get(), 
                    timeout=120
                )
                
                if result == "<DONE>":
                    del self._user_queues[query_id]
                    break
                
                if result:
                    yield result
            except TimeoutError:
                break
    
    async def run(self, query, sql, sql_data, language, current_time, query_id, custom_instruction):
        # 1. 构建 Prompt
        prompt_result = prompt_builder.run(
            query=query,
            sql=sql,
            sql_data=sql_data,  # {"columns": [...], "data": [...]}
            language=language,
            current_time=current_time,
            custom_instruction=custom_instruction or ""
        )
        
        # 2. 流式生成答案（通过回调函数传递到队列）
        result = await generator(
            prompt=prompt_result["prompt"],
            query_id=query_id
        )
        
        return result
```

#### 3.3 数据预处理逻辑

```python
# 文件: wren-ai-service/src/pipelines/retrieval/preprocess_sql_data.py

class PreprocessSqlData:
    def run(self, sql_data: Dict) -> dict:
        """
        预处理 SQL 查询结果，限制数据量以适应 LLM token 限制
        """
        df = pd.DataFrame(sql_data)
        
        # 计算 token 数量
        encoding = tiktoken.encoding_for_model("gpt-4")
        
        # 逐步添加行，直到达到 token 限制
        max_tokens = 10000  # 限制数据占用的 token 数
        preprocessed_data = []
        current_tokens = 0
        
        for idx, row in df.iterrows():
            row_str = row.to_json()
            row_tokens = len(encoding.encode(row_str))
            
            if current_tokens + row_tokens > max_tokens:
                break
            
            preprocessed_data.append(row.to_dict())
            current_tokens += row_tokens
        
        return {
            "sql_data": {
                "columns": df.columns.tolist(),
                "data": preprocessed_data
            },
            "num_rows_used_in_llm": len(preprocessed_data)
        }
```

#### 3.4 实际生成示例

```python
# 示例输入
query = "What are the top 5 products by revenue?"
sql = "SELECT product_name, SUM(revenue) as total_revenue FROM sales GROUP BY product_name ORDER BY total_revenue DESC LIMIT 5"
sql_data = {
    "columns": ["product_name", "total_revenue"],
    "data": [
        {"product_name": "Product A", "total_revenue": 150000},
        {"product_name": "Product B", "total_revenue": 120000},
        {"product_name": "Product C", "total_revenue": 95000},
        {"product_name": "Product D", "total_revenue": 78000},
        {"product_name": "Product E", "total_revenue": 65000}
    ]
}

# LLM 生成的答案（Markdown 格式）
"""
Based on the sales data, here are the **top 5 products by revenue**:

1. **Product A** - $150,000 in total revenue
2. **Product B** - $120,000 in total revenue
3. **Product C** - $95,000 in total revenue
4. **Product D** - $78,000 in total revenue
5. **Product E** - $65,000 in total revenue

**Key Insight**: Product A significantly outperforms other products, generating 
25% more revenue than the second-best product. This suggests focusing marketing 
and inventory efforts on Product A could maximize returns.
"""
```

---

## 🔄 整体集成流程

### 完整的用户问答流程（Ask → SQL → Chart → Answer）

```
用户提问: "Show me monthly sales trends for last year"
    ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 1: ASK SERVICE (Text-to-SQL)                            │
│ ├─ Intent Classification: TEXT_TO_SQL                        │
│ ├─ DB Schema Retrieval: sales, orders, time_dim              │
│ ├─ SQL Generation Reasoning: "Need to join sales with        │
│ │                            time_dim, group by month"        │
│ ├─ SQL Generation:                                           │
│ │   SELECT                                                   │
│ │     DATE_TRUNC('month', order_date) as month,              │
│ │     SUM(revenue) as monthly_sales                          │
│ │   FROM sales                                               │
│ │   WHERE YEAR(order_date) = YEAR(CURRENT_DATE) - 1          │
│ │   GROUP BY 1                                               │
│ │   ORDER BY 1                                               │
│ └─ SQL Validation: ✓ (dry run success)                       │
└─────────────────────────────────────────────────────────────┘
    ↓
    Response: {"sql": "SELECT ...", "status": "finished"}
    ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: CHART SERVICE (Text-to-Charts)                       │
│ ├─ SQL Execution: 执行上面的 SQL                             │
│ │   Results: [                                               │
│ │     {"month": "2023-01", "monthly_sales": 45000},          │
│ │     {"month": "2023-02", "monthly_sales": 52000},          │
│ │     ...                                                    │
│ │   ]                                                        │
│ ├─ Chart Generation:                                         │
│ │   - Analyzing data: temporal x-axis, quantitative y-axis   │
│ │   - Best chart type: LINE CHART (time series trend)        │
│ │   - Generate Vega-Lite schema                              │
│ └─ Output:                                                   │
│     {                                                        │
│       "chart_type": "line",                                  │
│       "reasoning": "Line chart best shows trends over time", │
│       "chart_schema": {                                      │
│         "mark": {"type": "line"},                            │
│         "encoding": {                                        │
│           "x": {"field": "month", "type": "temporal"},       │
│           "y": {"field": "monthly_sales", "type": "quantitative"} │
│         }                                                    │
│       }                                                      │
│     }                                                        │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: SQL ANSWER SERVICE (AI 洞察生成)                     │
│ ├─ Data Preprocessing: 12 rows of data                       │
│ ├─ Streaming Answer Generation:                              │
│ │   "The monthly sales data for last year shows an          │
│ │   interesting pattern:                                     │
│ │                                                            │
│ │   - **Overall Growth**: Sales grew from $45,000 in        │
│ │     January to $72,000 in December, a 60% increase.       │
│ │                                                            │
│ │   - **Seasonal Trends**: Notable peaks in Q4 (Oct-Dec)    │
│ │     suggest holiday season boost.                          │
│ │                                                            │
│ │   - **Lowest Performance**: February showed the weakest   │
│ │     performance at $52,000.                                │
│ │                                                            │
│ │   **Recommendation**: Consider ramping up inventory and   │
│ │   marketing budget during Q3 to prepare for Q4 demand."   │
│ └─ Output: Markdown formatted streaming response             │
└─────────────────────────────────────────────────────────────┘
    ↓
最终用户界面展示:
├─ SQL 查询
├─ 交互式图表 (Vega-Lite)
└─ AI 生成的洞察分析 (Markdown)
```

---

## 🎯 关键技术亮点

### 1. 语义层 (Semantic Layer)
- **MDL (Modeling Definition Language)**: 定义数据模型、指标、关系
- **向量检索**: 使用 embedding 检索相关的表、列、指标
- **上下文增强**: SQL 样例、用户指令、历史问题

### 2. LLM Pipeline 设计
- **Hamilton Framework**: 函数式 DAG pipeline，易于追踪和调试
- **Langfuse**: 完整的 LLM observability，追踪每个步骤
- **流式生成**: 使用 AsyncIO Queue 实现流式响应

### 3. SQL 生成优化
- **Dry Plan**: 使用 Wren Engine 的 dry plan 验证 SQL 而不执行
- **自动纠错**: SQL 诊断 + 纠错，最多重试 3 次
- **列剪枝**: 使用 LLM 精确选择需要的列，减少 token 消耗

### 4. 图表智能选择
- **数据类型感知**: 分析数据类型（nominal, ordinal, quantitative, temporal）
- **自动推荐**: 根据数据特征和用户问题自动选择最合适的图表类型
- **Vega-Lite**: 标准化的声明式可视化语法

### 5. 状态管理
- **TTL Cache**: 使用 cachetools 管理请求状态，自动过期
- **状态机**: 清晰的状态转换（understanding → searching → planning → generating → finished）
- **流式响应**: 支持 SSE (Server-Sent Events) 流式返回结果

---

## 📊 性能和可扩展性

### 并发处理
```python
# 多个检索任务并行执行
sql_samples_task, instructions_task = await asyncio.gather(
    self._pipelines["sql_pairs_retrieval"].run(...),
    self._pipelines["instructions_retrieval"].run(...)
)
```

### 缓存策略
```python
# TTL Cache 避免重复计算
self._ask_results: Dict[str, AskResultResponse] = TTLCache(
    maxsize=1_000_000,  # 最多缓存 100 万个结果
    ttl=120  # 120 秒后过期
)
```

### Token 优化
- SQL 数据预处理：限制传给 LLM 的行数
- 列剪枝：只检索和传递必要的列
- 样本数据：图表生成只使用前 50 行

---

## 🔧 配置和扩展

### LLM Provider 抽象
```python
# 支持多种 LLM 提供商
class LLMProvider:
    def get_generator(self, system_prompt, generation_kwargs):
        # OpenAI, Azure OpenAI, Anthropic, Google, etc.
        pass
```

### 可配置参数
- `allow_intent_classification`: 是否启用意图分类
- `allow_sql_generation_reasoning`: 是否生成 SQL 推理
- `enable_column_pruning`: 是否启用列剪枝
- `max_sql_correction_retries`: SQL 纠错最大重试次数
- `use_dry_plan`: 是否使用 dry plan 验证

---

## 📝 总结

Wren AI 的 GenBI 实现展示了一个完整的 AI 驱动的商业智能系统架构：

1. **Text-to-SQL**: 通过意图分类、语义检索、推理规划、SQL 生成和自动纠错，实现高质量的 SQL 生成
2. **Text-to-Charts**: 智能分析数据特征，自动选择和生成合适的可视化图表
3. **AI 洞察**: 将数据和 SQL 转换为易于理解的商业洞察

核心优势：
- **语义层驱动**: MDL + 向量检索确保 LLM 理解数据语义
- **Pipeline 设计**: 模块化、可观测、可扩展
- **用户体验**: 流式响应、自动纠错、多语言支持
- **可靠性**: 状态管理、错误处理、验证机制

这是一个生产级的 GenBI 实现，值得深入学习和借鉴。
