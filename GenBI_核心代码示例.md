# GenBI 核心代码逻辑示例

## 📚 目录
- [1. Text-to-SQL 核心代码](#1-text-to-sql-核心代码)
- [2. Text-to-Charts 核心代码](#2-text-to-charts-核心代码)
- [3. AI 洞察生成核心代码](#3-ai-洞察生成核心代码)
- [4. 关键工具类](#4-关键工具类)

---

## 1. Text-to-SQL 核心代码

### 1.1 Ask Service - 主流程控制

```python
# 文件: wren-ai-service/src/web/v1/services/ask.py

class AskService:
    """Text-to-SQL 的主服务类，协调整个流程"""
    
    def __init__(self, pipelines: Dict[str, BasicPipeline], **kwargs):
        self._pipelines = pipelines
        self._ask_results: Dict[str, AskResultResponse] = TTLCache(
            maxsize=1_000_000, ttl=120
        )
        self._max_sql_correction_retries = 3
    
    @observe(name="Ask Question")
    @trace_metadata
    async def ask(self, ask_request: AskRequest) -> dict:
        """
        主流程：
        1. Understanding - 意图分类
        2. Searching - 检索相关Schema
        3. Planning - 生成SQL推理
        4. Generating - 生成SQL
        5. Correcting - 纠错（如需要）
        """
        query_id = ask_request.query_id
        user_query = ask_request.query
        histories = ask_request.histories[:5][::-1]  # 最多5个历史，倒序
        
        # ========== 阶段1: Understanding ==========
        self._ask_results[query_id] = AskResultResponse(status="understanding")
        
        # 检查历史问题
        historical_result = await self._pipelines["historical_question"].run(
            query=user_query,
            project_id=ask_request.project_id
        )
        
        if historical_result.get("documents"):
            # 找到历史答案，直接返回
            return self._build_response(historical_result["documents"][0]["sql"])
        
        # 并行检索SQL样例和用户指令
        sql_samples_task, instructions_task = await asyncio.gather(
            self._pipelines["sql_pairs_retrieval"].run(
                query=user_query,
                project_id=ask_request.project_id
            ),
            self._pipelines["instructions_retrieval"].run(
                query=user_query,
                project_id=ask_request.project_id,
                scope="sql"
            )
        )
        
        sql_samples = sql_samples_task["formatted_output"]["documents"]
        instructions = instructions_task["formatted_output"]["documents"]
        
        # 意图分类
        if self._allow_intent_classification:
            intent_result = await self._pipelines["intent_classification"].run(
                query=user_query,
                histories=histories,
                sql_samples=sql_samples,
                instructions=instructions,
                project_id=ask_request.project_id,
                configuration=ask_request.configurations
            )
            
            intent = intent_result["post_process"]["intent"]
            rephrased_question = intent_result["post_process"]["rephrased_question"]
            
            # 如果是非TEXT_TO_SQL意图，调用对应的assistance pipeline
            if intent == "GENERAL":
                asyncio.create_task(
                    self._pipelines["data_assistance"].run(...)
                )
                return self._build_response(type="GENERAL")
            elif intent == "MISLEADING_QUERY":
                asyncio.create_task(
                    self._pipelines["misleading_assistance"].run(...)
                )
                return self._build_response(type="MISLEADING_QUERY")
            
            # 使用重新表述的问题
            if rephrased_question:
                user_query = rephrased_question
        
        # ========== 阶段2: Searching ==========
        self._ask_results[query_id] = AskResultResponse(status="searching")
        
        retrieval_result = await self._pipelines["db_schema_retrieval"].run(
            query=user_query,
            histories=histories,
            project_id=ask_request.project_id,
            enable_column_pruning=ask_request.enable_column_pruning
        )
        
        documents = retrieval_result["construct_retrieval_results"]["retrieval_results"]
        if not documents:
            raise NoRelevantDataError("No relevant data found")
        
        table_names = [doc["table_name"] for doc in documents]
        table_ddls = [doc["table_ddl"] for doc in documents]
        
        # ========== 阶段3: Planning ==========
        if self._allow_sql_generation_reasoning:
            self._ask_results[query_id] = AskResultResponse(status="planning")
            
            if histories:
                # Follow-up 问题
                sql_generation_reasoning = await self._pipelines[
                    "followup_sql_generation_reasoning"
                ].run(
                    query=user_query,
                    contexts=table_ddls,
                    histories=histories,
                    sql_samples=sql_samples,
                    instructions=instructions,
                    configuration=ask_request.configurations
                )
            else:
                # 新问题
                sql_generation_reasoning = await self._pipelines[
                    "sql_generation_reasoning"
                ].run(
                    query=user_query,
                    contexts=table_ddls,
                    sql_samples=sql_samples,
                    instructions=instructions,
                    configuration=ask_request.configurations
                )
            
            reasoning_text = sql_generation_reasoning["post_process"]
        else:
            reasoning_text = None
        
        # ========== 阶段4: Generating ==========
        self._ask_results[query_id] = AskResultResponse(status="generating")
        
        # 检索SQL函数
        sql_functions = await self._pipelines["sql_functions_retrieval"].run(
            project_id=ask_request.project_id
        ) if self._allow_sql_functions_retrieval else []
        
        # 生成SQL
        if histories:
            sql_result = await self._pipelines["followup_sql_generation"].run(
                query=user_query,
                contexts=table_ddls,
                sql_generation_reasoning=reasoning_text,
                histories=histories,
                sql_samples=sql_samples,
                instructions=instructions,
                sql_functions=sql_functions,
                use_dry_plan=ask_request.use_dry_plan
            )
        else:
            sql_result = await self._pipelines["sql_generation"].run(
                query=user_query,
                contexts=table_ddls,
                sql_generation_reasoning=reasoning_text,
                sql_samples=sql_samples,
                instructions=instructions,
                sql_functions=sql_functions,
                use_dry_plan=ask_request.use_dry_plan
            )
        
        valid_result = sql_result["post_process"]["valid_generation_result"]
        invalid_result = sql_result["post_process"]["invalid_generation_result"]
        
        if valid_result:
            # SQL验证通过
            final_sql = valid_result["sql"]
            self._ask_results[query_id] = AskResultResponse(
                status="finished",
                response=[AskResult(sql=final_sql, type="llm")]
            )
            return self._build_response(final_sql)
        
        # ========== 阶段5: Correcting ==========
        if invalid_result:
            retry_count = 0
            while retry_count < self._max_sql_correction_retries:
                if invalid_result["type"] == "TIME_OUT":
                    break
                
                self._ask_results[query_id] = AskResultResponse(status="correcting")
                
                # SQL诊断
                if self._allow_sql_diagnosis:
                    diagnosis_result = await self._pipelines["sql_diagnosis"].run(
                        contexts=table_ddls,
                        original_sql=invalid_result["original_sql"],
                        invalid_sql=invalid_result["sql"],
                        error_message=invalid_result["error"]
                    )
                    diagnosis_reasoning = diagnosis_result["post_process"]["reasoning"]
                else:
                    diagnosis_reasoning = invalid_result["error"]
                
                # SQL纠错
                correction_result = await self._pipelines["sql_correction"].run(
                    contexts=table_ddls,
                    instructions=instructions,
                    invalid_generation_result={
                        "sql": invalid_result["original_sql"],
                        "error": diagnosis_reasoning
                    },
                    project_id=ask_request.project_id,
                    use_dry_plan=ask_request.use_dry_plan,
                    sql_functions=sql_functions
                )
                
                valid_result = correction_result["post_process"]["valid_generation_result"]
                if valid_result:
                    # 纠错成功
                    final_sql = valid_result["sql"]
                    self._ask_results[query_id] = AskResultResponse(
                        status="finished",
                        response=[AskResult(sql=final_sql, type="llm")]
                    )
                    return self._build_response(final_sql)
                
                invalid_result = correction_result["post_process"]["invalid_generation_result"]
                retry_count += 1
        
        # 纠错失败
        raise NoRelevantSQLError("Failed to generate valid SQL")
```

### 1.2 SQL Generation Pipeline

```python
# 文件: wren-ai-service/src/pipelines/generation/sql_generation.py

# System Prompt - 定义SQL专家的角色和规则
sql_generation_system_prompt = """
### ROLE ###
You are a Trino SQL expert with exceptional logical thinking skills and ability to understand complex data relationships.

### TASK ###
Given a user query and relevant database schema, generate a valid Trino SQL query that accurately answers the question.

### INSTRUCTIONS ###
1. **Use ONLY the tables and columns provided in the schema**
2. **Apply proper JOIN conditions** using foreign keys
3. **Handle NULL values** appropriately
4. **Use appropriate aggregation functions** (SUM, COUNT, AVG, etc.)
5. **Follow SQL best practices** for readability and performance
6. **Respect calculated fields and metrics** definitions
7. **Use correct date/time functions**
8. **Generate only the SQL query** without explanations

### OUTPUT FORMAT ###
Return ONLY the SQL query as a string, without any markdown formatting or code blocks.
"""

# User Prompt Template - 包含所有上下文信息
sql_generation_user_prompt_template = """
### DATABASE SCHEMA ###
{% for document in documents %}
{{ document }}
{% endfor %}

{% if calculated_field_instructions %}
{{ calculated_field_instructions }}
{% endif %}

{% if metric_instructions %}
{{ metric_instructions }}
{% endif %}

{% if sql_functions %}
### SQL FUNCTIONS ###
{% for function in sql_functions %}
{{ function }}
{% endfor %}
{% endif %}

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
    """SQL生成Pipeline - 使用Hamilton框架"""
    
    def __init__(self, llm_provider: LLMProvider, document_store_provider, engine):
        self._components = {
            "generator": llm_provider.get_generator(
                system_prompt=sql_generation_system_prompt,
                generation_kwargs={
                    "response_format": {"type": "json_object"},
                    "temperature": 0.0
                }
            ),
            "prompt_builder": PromptBuilder(
                template=sql_generation_user_prompt_template
            ),
            "post_processor": SQLGenPostProcessor(engine=engine)
        }
        
        # Hamilton异步驱动
        super().__init__(AsyncDriver({}, sys.modules[__name__]))
    
    # Hamilton Pipeline步骤1: 构建Prompt
    @observe(capture_input=False)
    def prompt(
        self,
        query: str,
        documents: list[str],  # DDL
        prompt_builder: PromptBuilder,
        sql_generation_reasoning: str | None = None,
        sql_samples: list[dict] | None = None,
        instructions: list[dict] | None = None,
        sql_functions: list[SqlFunction] | None = None,
        **kwargs
    ) -> dict:
        """构建最终的prompt"""
        _prompt = prompt_builder.run(
            query=query,
            documents=documents,
            sql_generation_reasoning=sql_generation_reasoning,
            sql_samples=sql_samples,
            instructions=instructions,
            sql_functions=sql_functions,
            **kwargs
        )
        return {"prompt": clean_up_new_lines(_prompt.get("prompt"))}
    
    # Hamilton Pipeline步骤2: LLM生成SQL
    @observe(as_type="generation", capture_input=False)
    @trace_cost
    async def generate_sql(
        self,
        prompt: dict,
        generator: Any,
        generator_name: str
    ) -> dict:
        """调用LLM生成SQL"""
        result = await generator(prompt=prompt.get("prompt"))
        return result, generator_name
    
    # Hamilton Pipeline步骤3: 后处理和验证
    @observe(capture_input=False)
    async def post_process(
        self,
        generate_sql: dict,
        post_processor: SQLGenPostProcessor,
        project_id: str | None = None,
        use_dry_plan: bool = False,
        **kwargs
    ) -> dict:
        """
        后处理步骤：
        1. 解析LLM响应
        2. Dry Plan验证
        3. 分类valid/invalid结果
        """
        return await post_processor.run(
            generate_sql.get("replies"),
            project_id=project_id,
            use_dry_plan=use_dry_plan,
            **kwargs
        )
    
    @observe(name="SQL Generation")
    async def run(
        self,
        query: str,
        contexts: list[str],
        **kwargs
    ) -> dict:
        """执行完整的SQL生成pipeline"""
        logger.info("SQL Generation pipeline is running...")
        
        return await self._pipe.execute(
            ["post_process"],  # 目标节点
            inputs={
                "query": query,
                "documents": contexts,
                **kwargs,
                **self._components
            }
        )
```

### 1.3 SQL Post Processor - 验证和分类

```python
# 文件: wren-ai-service/src/pipelines/generation/utils/sql.py

class SQLGenPostProcessor:
    """SQL生成后处理器 - 验证和分类SQL"""
    
    def __init__(self, engine: Engine):
        self._engine = engine
    
    async def run(
        self,
        replies: list[str],
        project_id: str,
        use_dry_plan: bool = False,
        data_source: str = "local_file",
        allow_dry_plan_fallback: bool = True,
        allow_data_preview: bool = False
    ) -> dict:
        """
        处理LLM生成的SQL：
        1. 解析响应
        2. Dry Plan验证（可选）
        3. 分类为valid或invalid
        """
        if not replies:
            return {
                "valid_generation_result": None,
                "invalid_generation_result": None
            }
        
        # 解析LLM响应
        try:
            generation_result = orjson.loads(replies[0])
            sql = generation_result.get("sql", "")
        except Exception as e:
            logger.error(f"Failed to parse SQL generation result: {e}")
            return {
                "valid_generation_result": None,
                "invalid_generation_result": {
                    "sql": "",
                    "error": str(e),
                    "type": "PARSE_ERROR"
                }
            }
        
        if not sql:
            return {
                "valid_generation_result": None,
                "invalid_generation_result": {
                    "sql": "",
                    "error": "Empty SQL generated",
                    "type": "EMPTY_SQL"
                }
            }
        
        # Dry Plan验证（不实际执行SQL，只检查语法和语义）
        if use_dry_plan:
            try:
                dry_run_result = await self._engine.dry_run_sql(
                    sql=sql,
                    project_id=project_id,
                    data_source=data_source
                )
                
                if dry_run_result.get("error"):
                    # Dry Plan失败
                    error_message = dry_run_result["error"]
                    
                    # 尝试fallback到实际执行（如果允许）
                    if allow_dry_plan_fallback:
                        execution_result = await self._engine.execute_sql(
                            sql=sql,
                            project_id=project_id,
                            dry_run=True
                        )
                        
                        if not execution_result.get("error"):
                            # 实际执行成功，Dry Plan可能误报
                            return {
                                "valid_generation_result": {
                                    "sql": sql,
                                    "type": "llm"
                                },
                                "invalid_generation_result": None
                            }
                    
                    # Dry Plan失败，SQL无效
                    return {
                        "valid_generation_result": None,
                        "invalid_generation_result": {
                            "sql": sql,
                            "original_sql": sql,
                            "error": error_message,
                            "type": "DRY_RUN_ERROR"
                        }
                    }
                
                # Dry Plan成功
                return {
                    "valid_generation_result": {
                        "sql": sql,
                        "type": "llm"
                    },
                    "invalid_generation_result": None
                }
                
            except TimeoutError:
                return {
                    "valid_generation_result": None,
                    "invalid_generation_result": {
                        "sql": sql,
                        "error": "Dry run timeout",
                        "type": "TIME_OUT"
                    }
                }
        
        # 不使用Dry Plan，直接返回valid
        return {
            "valid_generation_result": {
                "sql": sql,
                "type": "llm"
            },
            "invalid_generation_result": None
        }
```

### 1.4 DB Schema Retrieval - 向量检索

```python
# 文件: wren-ai-service/src/pipelines/retrieval/db_schema_retrieval.py

class DBSchemaRetrieval(BasicPipeline):
    """数据库Schema检索Pipeline"""
    
    # 步骤1: Embedding用户问题
    @observe(capture_input=False, capture_output=False)
    async def embedding(
        self,
        query: str,
        embedder: Any,
        histories: list[AskHistory]
    ) -> dict:
        """
        将用户问题转换为向量
        如果有历史问题，将历史问题拼接到当前问题前面
        """
        if histories:
            previous_queries = [h.question for h in histories]
            query = "\n".join(previous_queries) + "\n" + query
        
        return await embedder.run(query)
    
    # 步骤2: 检索相关的表
    @observe(capture_input=False)
    async def table_retrieval(
        self,
        embedding: dict,
        project_id: str,
        table_retriever: Any
    ) -> dict:
        """基于向量相似度检索相关的表"""
        filters = {
            "operator": "AND",
            "conditions": [
                {"field": "type", "operator": "==", "value": "TABLE_DESCRIPTION"},
                {"field": "project_id", "operator": "==", "value": project_id}
            ]
        }
        
        result = await table_retriever.run(
            query_embedding=embedding["embedding"],
            filters=filters,
            top_k=10  # 检索top 10相关的表
        )
        
        return result
    
    # 步骤3: 为每个表检索相关的列
    @observe(capture_input=False)
    async def column_retrieval(
        self,
        embedding: dict,
        table_retrieval: dict,
        column_retriever: Any
    ) -> dict:
        """为每个检索到的表，检索相关的列"""
        all_columns = []
        
        for table_doc in table_retrieval["documents"]:
            table_name = table_doc.meta.get("table_name")
            
            filters = {
                "operator": "AND",
                "conditions": [
                    {"field": "type", "operator": "==", "value": "COLUMN_DESCRIPTION"},
                    {"field": "table_name", "operator": "==", "value": table_name}
                ]
            }
            
            column_result = await column_retriever.run(
                query_embedding=embedding["embedding"],
                filters=filters,
                top_k=20  # 每个表检索top 20列
            )
            
            all_columns.extend(column_result["documents"])
        
        return {"documents": all_columns}
    
    # 步骤4: 列剪枝（可选）- 使用LLM精确选择列
    @observe(capture_input=False)
    async def column_pruning(
        self,
        query: str,
        table_retrieval: dict,
        column_retrieval: dict,
        enable_column_pruning: bool,
        llm_generator: Any
    ) -> dict:
        """
        使用LLM分析问题，精确选择需要的列
        减少不必要的列，降低token消耗
        """
        if not enable_column_pruning:
            # 不启用列剪枝，返回所有列
            return column_retrieval
        
        # 构建每个表的完整DDL
        table_ddls = []
        for table_doc in table_retrieval["documents"]:
            table_name = table_doc.meta.get("table_name")
            columns = [
                col for col in column_retrieval["documents"]
                if col.meta.get("table_name") == table_name
            ]
            ddl = self._build_table_ddl(table_name, columns)
            table_ddls.append(ddl)
        
        # 使用LLM选择列
        prompt = f"""
        ### Database Schema ###
        {chr(10).join(table_ddls)}
        
        ### Question ###
        {query}
        
        Please select ONLY the necessary columns to answer this question.
        """
        
        llm_result = await llm_generator(prompt=prompt)
        chosen_columns = self._parse_chosen_columns(llm_result)
        
        # 过滤列
        pruned_columns = [
            col for col in column_retrieval["documents"]
            if col.meta.get("column_name") in chosen_columns
        ]
        
        return {"documents": pruned_columns}
    
    # 步骤5: 构建检索结果
    @observe(capture_input=False)
    def construct_retrieval_results(
        self,
        table_retrieval: dict,
        column_retrieval: dict,
        column_pruning: dict
    ) -> dict:
        """
        构建最终的检索结果：
        - 为每个表构建DDL
        - 标记是否包含计算字段/指标/JSON字段
        """
        columns_to_use = column_pruning["documents"]
        
        retrieval_results = []
        has_calculated_field = False
        has_metric = False
        has_json_field = False
        
        for table_doc in table_retrieval["documents"]:
            table_name = table_doc.meta.get("table_name")
            table_type = table_doc.meta.get("table_type", "TABLE")
            
            # 获取该表的列
            table_columns = [
                col for col in columns_to_use
                if col.meta.get("table_name") == table_name
            ]
            
            # 检查特殊字段类型
            for col in table_columns:
                if col.meta.get("is_calculated"):
                    has_calculated_field = True
                if col.meta.get("data_type", "").lower() == "json":
                    has_json_field = True
            
            if table_type == "METRIC":
                has_metric = True
            
            # 构建DDL
            if table_type == "METRIC":
                table_ddl = self._build_metric_ddl(table_name, table_columns)
            elif table_type == "VIEW":
                table_ddl = self._build_view_ddl(table_name, table_doc.meta)
            else:
                table_ddl = build_table_ddl(
                    table_name=table_name,
                    columns=table_columns,
                    primary_key=table_doc.meta.get("primary_key"),
                    foreign_keys=table_doc.meta.get("foreign_keys", [])
                )
            
            retrieval_results.append({
                "table_name": table_name,
                "table_type": table_type,
                "table_ddl": table_ddl
            })
        
        return {
            "retrieval_results": retrieval_results,
            "has_calculated_field": has_calculated_field,
            "has_metric": has_metric,
            "has_json_field": has_json_field
        }
    
    @observe(name="DB Schema Retrieval")
    async def run(
        self,
        query: str,
        histories: list[AskHistory],
        project_id: str,
        enable_column_pruning: bool = False
    ) -> dict:
        """执行完整的Schema检索pipeline"""
        return await self._pipe.execute(
            ["construct_retrieval_results"],
            inputs={
                "query": query,
                "histories": histories,
                "project_id": project_id,
                "enable_column_pruning": enable_column_pruning,
                **self._components
            }
        )
```

---

## 2. Text-to-Charts 核心代码

### 2.1 Chart Service

```python
# 文件: wren-ai-service/src/web/v1/services/chart.py

class ChartService:
    """图表生成服务"""
    
    def __init__(self, pipelines: Dict[str, BasicPipeline]):
        self._pipelines = pipelines
        self._chart_results: Dict[str, ChartResultResponse] = TTLCache(
            maxsize=1_000_000, ttl=120
        )
    
    @observe(name="Generate Chart")
    @trace_metadata
    async def chart(self, chart_request: ChartRequest) -> dict:
        """
        图表生成主流程：
        1. Fetching - 执行SQL获取数据（如果没有提供）
        2. Generating - 生成图表
        3. Finished - 返回结果
        """
        query_id = chart_request.query_id
        trace_id = kwargs.get("trace_id")
        
        # ========== 阶段1: Fetching ==========
        if not chart_request.data:
            self._chart_results[query_id] = ChartResultResponse(
                status="fetching",
                trace_id=trace_id
            )
            
            # 执行SQL
            execute_result = await self._pipelines["sql_executor"].run(
                sql=chart_request.sql,
                project_id=chart_request.project_id
            )
            
            sql_data = execute_result["execute_sql"]["results"]
            error_message = execute_result["execute_sql"].get("error_message")
            
            if error_message:
                self._chart_results[query_id] = ChartResultResponse(
                    status="failed",
                    error=ChartError(code="OTHERS", message=error_message),
                    trace_id=trace_id
                )
                return {"error": error_message}
        else:
            sql_data = chart_request.data
        
        # ========== 阶段2: Generating ==========
        self._chart_results[query_id] = ChartResultResponse(
            status="generating",
            trace_id=trace_id
        )
        
        chart_result = await self._pipelines["chart_generation"].run(
            query=chart_request.query,
            sql=chart_request.sql,
            data=sql_data,
            language=chart_request.configurations.language,
            remove_data_from_chart_schema=chart_request.remove_data_from_chart_schema,
            custom_instruction=chart_request.custom_instruction
        )
        
        chart_output = chart_result["post_process"]["results"]
        
        # 检查是否成功生成图表
        if not chart_output.get("chart_schema") and not chart_output.get("reasoning"):
            self._chart_results[query_id] = ChartResultResponse(
                status="failed",
                error=ChartError(code="NO_CHART", message="Chart generation failed"),
                trace_id=trace_id
            )
            return {"error": "NO_CHART"}
        
        # ========== 阶段3: Finished ==========
        self._chart_results[query_id] = ChartResultResponse(
            status="finished",
            response=ChartResult(
                reasoning=chart_output["reasoning"],
                chart_type=chart_output["chart_type"],
                chart_schema=chart_output["chart_schema"]
            ),
            trace_id=trace_id
        )
        
        return {"chart_result": chart_output}
    
    def get_chart_result(self, query_id: str) -> ChartResultResponse:
        """获取图表生成结果"""
        result = self._chart_results.get(query_id)
        if not result:
            return ChartResultResponse(
                status="failed",
                error=ChartError(code="OTHERS", message=f"{query_id} not found")
            )
        return result
```

### 2.2 Chart Generation Pipeline

```python
# 文件: wren-ai-service/src/pipelines/generation/chart_generation.py

# System Prompt - 图表生成专家
chart_generation_system_prompt = """
### TASK ###
You are a data analyst great at visualizing data using vega-lite! 
Given the user's question, SQL, sample data and sample column values, 
you need to generate vega-lite schema in JSON and provide suitable chart type.

### CHART TYPES ###
- Bar chart: Comparing quantities across categories
- Line chart: Showing trends over time
- Multi line chart: Comparing multiple metrics over time
- Area chart: Emphasizing volume of change over time
- Pie chart: Showing parts of a whole as percentages
- Stacked bar chart: Showing composition and comparison across categories
- Grouped bar chart: Comparing sub-categories within main categories

### INSTRUCTIONS ###
- You can ONLY use the chart types listed above
- Generated chart should answer the user's question
- If the sample data is not suitable for visualization, return empty string
- The language for reasoning must be the same language provided by the user
- Make sure all fields in the encoding section are present in the column names

### DATA TYPE GUIDELINES ###
1. Nominal (Categorical): Names without order → Use bar chart or pie chart
2. Ordinal: Categorical with order → Use bar chart
3. Quantitative: Numerical values → Use line chart or area chart
4. Temporal: Date/time data → Use line chart or area chart

### OUTPUT FORMAT ###
{
    "reasoning": <REASON_IN_USER_LANGUAGE>,
    "chart_type": "line" | "multi_line" | "bar" | "pie" | "grouped_bar" | "stacked_bar" | "area" | "",
    "chart_schema": <VEGA_LITE_JSON_SCHEMA>
}
"""

# User Prompt Template
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
    """图表生成Pipeline"""
    
    def __init__(self, llm_provider: LLMProvider):
        self._components = {
            "prompt_builder": PromptBuilder(
                template=chart_generation_user_prompt_template
            ),
            "generator": llm_provider.get_generator(
                system_prompt=chart_generation_system_prompt,
                generation_kwargs={
                    "response_format": {
                        "type": "json_schema",
                        "json_schema": {
                            "name": "chart_generation_schema",
                            "schema": ChartGenerationResults.model_json_schema()
                        }
                    }
                }
            ),
            "chart_data_preprocessor": ChartDataPreprocessor(),
            "post_processor": ChartGenerationPostProcessor()
        }
        
        # 加载Vega-Lite Schema用于验证
        with open("src/pipelines/generation/utils/vega-lite-schema-v5.json") as f:
            self._vega_schema = orjson.loads(f.read())
    
    # 步骤1: 数据预处理
    @observe(capture_input=False)
    def preprocess_data(
        self,
        data: Dict[str, Any],
        chart_data_preprocessor: ChartDataPreprocessor
    ) -> dict:
        """
        预处理数据：
        - 限制样本大小（前50行）
        - 提取列值样本
        """
        return chart_data_preprocessor.run(data)
    
    # 步骤2: 构建Prompt
    @observe(capture_input=False)
    def prompt(
        self,
        query: str,
        sql: str,
        preprocess_data: dict,
        language: str,
        custom_instruction: str,
        prompt_builder: PromptBuilder
    ) -> dict:
        """构建图表生成prompt"""
        result = prompt_builder.run(
            query=query,
            sql=sql,
            sample_data=preprocess_data["sample_data"],
            sample_column_values=preprocess_data["sample_column_values"],
            language=language,
            custom_instruction=custom_instruction
        )
        return {"prompt": clean_up_new_lines(result["prompt"])}
    
    # 步骤3: LLM生成图表
    @observe(as_type="generation", capture_input=False)
    @trace_cost
    async def generate_chart(
        self,
        prompt: dict,
        generator: Any,
        generator_name: str
    ) -> dict:
        """调用LLM生成图表"""
        result = await generator(prompt=prompt["prompt"])
        return result, generator_name
    
    # 步骤4: 后处理和验证
    @observe(capture_input=False)
    def post_process(
        self,
        generate_chart: dict,
        vega_schema: Dict[str, Any],
        remove_data_from_chart_schema: bool,
        preprocess_data: dict,
        post_processor: ChartGenerationPostProcessor
    ) -> dict:
        """
        后处理：
        1. 解析LLM结果
        2. 验证Vega-Lite Schema
        3. 填充数据
        """
        return post_processor.run(
            generate_chart["replies"],
            vega_schema,
            preprocess_data["sample_data"],
            remove_data_from_chart_schema
        )
    
    @observe(name="Chart Generation")
    async def run(
        self,
        query: str,
        sql: str,
        data: dict,
        language: str,
        remove_data_from_chart_schema: bool = True,
        custom_instruction: Optional[str] = None
    ) -> dict:
        """执行完整的图表生成pipeline"""
        return await self._pipe.execute(
            ["post_process"],
            inputs={
                "query": query,
                "sql": sql,
                "data": data,
                "language": language,
                "remove_data_from_chart_schema": remove_data_from_chart_schema,
                "custom_instruction": custom_instruction or "",
                "vega_schema": self._vega_schema,
                **self._components
            }
        )
```

### 2.3 Chart Data Preprocessor

```python
# 文件: wren-ai-service/src/pipelines/generation/utils/chart.py

class ChartDataPreprocessor:
    """图表数据预处理器"""
    
    def run(self, data: Dict[str, Any]) -> dict:
        """
        预处理数据：
        1. 转换为DataFrame
        2. 限制样本大小（前50行）
        3. 提取每列的样本值
        """
        if not data:
            return {
                "sample_data": [],
                "sample_column_values": {}
            }
        
        try:
            # 转换为DataFrame
            df = pd.DataFrame(data)
            
            if df.empty:
                return {
                    "sample_data": [],
                    "sample_column_values": {}
                }
            
            # 限制样本大小 - 只取前50行
            sample_df = df.head(50)
            sample_data = sample_df.to_dict(orient="records")
            
            # 提取每列的样本值 - 用于LLM理解数据特征
            sample_column_values = {}
            for column in df.columns:
                # 获取该列的唯一值（前5个）
                unique_values = df[column].dropna().unique()[:5]
                sample_column_values[column] = [
                    str(val) for val in unique_values
                ]
            
            return {
                "sample_data": sample_data,
                "sample_column_values": sample_column_values
            }
            
        except Exception as e:
            logger.error(f"Chart data preprocessing error: {e}")
            return {
                "sample_data": [],
                "sample_column_values": {}
            }


class ChartGenerationPostProcessor:
    """图表生成后处理器"""
    
    def run(
        self,
        replies: list[str],
        vega_schema: Dict[str, Any],
        sample_data: list[dict],
        remove_data_from_chart_schema: bool = True
    ) -> dict:
        """
        后处理LLM生成的图表：
        1. 解析JSON结果
        2. 添加Vega-Lite必需的字段
        3. 验证Schema
        4. 填充数据
        """
        if not replies:
            return self._empty_result()
        
        try:
            # 解析LLM响应
            generation_result = orjson.loads(replies[0])
            
            reasoning = generation_result.get("reasoning", "")
            chart_type = generation_result.get("chart_type", "")
            chart_schema = generation_result.get("chart_schema", {})
            
            # 如果没有生成schema，返回空结果
            if not chart_schema:
                return {
                    "chart_schema": {},
                    "chart_type": chart_type,
                    "reasoning": reasoning
                }
            
            # 有时LLM返回的是字符串格式的JSON
            if isinstance(chart_schema, str):
                chart_schema = orjson.loads(chart_schema)
            
            # 添加Vega-Lite Schema版本
            chart_schema["$schema"] = "https://vega.github.io/schema/vega-lite/v5.json"
            
            # 填充数据到schema
            chart_schema["data"] = {"values": sample_data}
            
            # 验证Schema是否符合Vega-Lite规范
            try:
                validate(chart_schema, schema=vega_schema)
            except ValidationError as e:
                logger.error(f"Invalid Vega-Lite schema: {e}")
                return {
                    "chart_schema": {},
                    "chart_type": "",
                    "reasoning": reasoning
                }
            
            # 可选：移除数据（节省传输大小）
            if remove_data_from_chart_schema:
                chart_schema["data"]["values"] = []
            
            return {
                "chart_schema": chart_schema,
                "chart_type": chart_type,
                "reasoning": reasoning
            }
            
        except Exception as e:
            logger.error(f"Chart generation post processing error: {e}")
            return self._empty_result()
    
    def _empty_result(self) -> dict:
        """返回空结果"""
        return {
            "chart_schema": {},
            "chart_type": "",
            "reasoning": ""
        }


# Pydantic模型定义 - 用于JSON Schema验证
class ChartGenerationResults(BaseModel):
    """图表生成结果的Pydantic模型"""
    reasoning: str
    chart_type: Literal[
        "line", "multi_line", "bar", "pie", 
        "grouped_bar", "stacked_bar", "area", ""
    ]
    chart_schema: dict  # Vega-Lite JSON Schema

# 示例：生成的Vega-Lite Schema
"""
{
    "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
    "mark": {"type": "bar"},
    "encoding": {
        "x": {
            "field": "product_category",
            "type": "nominal",
            "title": "Product Category"
        },
        "y": {
            "field": "total_sales",
            "type": "quantitative",
            "title": "Total Sales ($)"
        },
        "color": {
            "field": "region",
            "type": "nominal",
            "title": "Region"
        }
    },
    "data": {
        "values": [
            {"product_category": "Electronics", "total_sales": 150000, "region": "North"},
            {"product_category": "Clothing", "total_sales": 120000, "region": "South"},
            ...
        ]
    }
}
"""
```

---

## 3. AI 洞察生成核心代码

### 3.1 SQL Answer Service

```python
# 文件: wren-ai-service/src/web/v1/services/sql_answer.py

class SqlAnswerService:
    """SQL答案生成服务 - 流式返回AI洞察"""
    
    def __init__(self, pipelines: Dict[str, BasicPipeline]):
        self._pipelines = pipelines
        self._sql_answer_results: Dict[str, SqlAnswerResultResponse] = TTLCache(
            maxsize=1_000_000, ttl=120
        )
    
    @observe(name="SQL Answer")
    @trace_metadata
    async def sql_answer(self, sql_answer_request: SqlAnswerRequest) -> dict:
        """
        SQL答案生成主流程：
        1. Preprocessing - 预处理数据
        2. Streaming Generation - 异步流式生成答案
        """
        query_id = sql_answer_request.query_id
        trace_id = kwargs.get("trace_id")
        
        # ========== 阶段1: Preprocessing ==========
        self._sql_answer_results[query_id] = SqlAnswerResultResponse(
            status="preprocessing",
            trace_id=trace_id
        )
        
        # 预处理SQL数据 - 限制token数量
        preprocessed = self._pipelines["preprocess_sql_data"].run(
            sql_data=sql_answer_request.sql_data
        )["preprocess"]
        
        num_rows_used = preprocessed.get("num_rows_used_in_llm", 0)
        
        if num_rows_used == 0:
            return {"error": "NO_DATA", "message": "No data to answer"}
        
        # ========== 阶段2: Succeeded - 启动异步生成 ==========
        self._sql_answer_results[query_id] = SqlAnswerResultResponse(
            status="succeeded",
            num_rows_used_in_llm=num_rows_used,
            trace_id=trace_id
        )
        
        # 异步启动答案生成（不等待完成）
        asyncio.create_task(
            self._pipelines["sql_answer"].run(
                query=sql_answer_request.query,
                sql=sql_answer_request.sql,
                sql_data=preprocessed["sql_data"],
                language=sql_answer_request.configurations.language,
                current_time=sql_answer_request.configurations.show_current_time(),
                query_id=query_id,
                custom_instruction=sql_answer_request.custom_instruction
            )
        )
        
        return {"status": "succeeded", "num_rows_used": num_rows_used}
    
    async def get_sql_answer_streaming_result(self, query_id: str):
        """
        流式返回答案（SSE）
        """
        if not self._sql_answer_results.get(query_id):
            yield SSEEvent(data={"error": "Query ID not found"}).serialize()
            return
        
        if self._sql_answer_results[query_id].status != "succeeded":
            yield SSEEvent(data={"error": "Not ready for streaming"}).serialize()
            return
        
        # 从pipeline获取流式结果
        async for chunk in self._pipelines["sql_answer"].get_streaming_results(query_id):
            event = SSEEvent(
                data=SSEEvent.SSEEventMessage(message=chunk)
            )
            yield event.serialize()
```

### 3.2 SQL Answer Pipeline

```python
# 文件: wren-ai-service/src/pipelines/generation/sql_answer.py

# System Prompt - 面向非技术用户的数据分析师
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
4. Generate a concise and clear answer in string format to answer the user's 
   question based on the data and sql.
5. If answer is in list format, only list top few examples, and tell users 
   there are more results omitted.
6. Answer must be in the same language user specified.
7. Do not include ```markdown or ``` in the answer.
8. If the user provides a custom instruction, follow it strictly.

### OUTPUT FORMAT
Please provide your response in proper Markdown string format.
"""

# User Prompt Template
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
    """SQL答案生成Pipeline - 支持流式输出"""
    
    def __init__(self, llm_provider: LLMProvider):
        # 为每个查询维护一个队列，用于流式传输
        self._user_queues: Dict[str, asyncio.Queue] = {}
        
        self._components = {
            "prompt_builder": PromptBuilder(
                template=sql_to_answer_user_prompt_template
            ),
            "generator": llm_provider.get_generator(
                system_prompt=sql_to_answer_system_prompt,
                streaming_callback=self._streaming_callback  # 关键：流式回调
            )
        }
    
    def _streaming_callback(self, chunk, query_id: str):
        """
        LLM流式生成的回调函数
        每次LLM生成一个chunk，就调用这个函数
        """
        # 确保query_id有对应的队列
        if query_id not in self._user_queues:
            self._user_queues[query_id] = asyncio.Queue()
        
        # 将chunk放入队列
        asyncio.create_task(
            self._user_queues[query_id].put(chunk.content)
        )
        
        # 如果是最后一个chunk，放入结束标记
        if chunk.meta.get("finish_reason"):
            asyncio.create_task(
                self._user_queues[query_id].put("<DONE>")
            )
    
    async def get_streaming_results(self, query_id: str):
        """
        流式返回生成的答案
        这是一个异步生成器，会持续yield直到生成完成
        """
        # 确保队列存在
        if query_id not in self._user_queues:
            self._user_queues[query_id] = asyncio.Queue()
        
        while True:
            try:
                # 从队列中获取chunk（最多等待120秒）
                chunk = await asyncio.wait_for(
                    self._user_queues[query_id].get(),
                    timeout=120
                )
                
                # 检查是否结束
                if chunk == "<DONE>":
                    # 清理队列
                    del self._user_queues[query_id]
                    break
                
                # yield chunk给调用者
                if chunk:
                    yield chunk
                    
            except asyncio.TimeoutError:
                # 超时，停止生成
                logger.warning(f"Streaming timeout for query_id: {query_id}")
                break
    
    # Pipeline步骤1: 构建Prompt
    @observe(capture_input=False)
    def prompt(
        self,
        query: str,
        sql: str,
        sql_data: dict,
        language: str,
        current_time: str,
        custom_instruction: str,
        prompt_builder: PromptBuilder
    ) -> dict:
        """构建SQL答案生成的prompt"""
        result = prompt_builder.run(
            query=query,
            sql=sql,
            sql_data=sql_data,
            language=language,
            current_time=current_time,
            custom_instruction=custom_instruction
        )
        return {"prompt": clean_up_new_lines(result["prompt"])}
    
    # Pipeline步骤2: LLM流式生成答案
    @observe(as_type="generation", capture_input=False)
    @trace_cost
    async def generate_answer(
        self,
        prompt: dict,
        generator: Any,
        query_id: str,
        generator_name: str
    ) -> dict:
        """
        调用LLM流式生成答案
        注意：通过streaming_callback将结果放入队列
        """
        result = await generator(
            prompt=prompt["prompt"],
            query_id=query_id  # 传递query_id给回调函数
        )
        return result, generator_name
    
    @observe(name="SQL Answer Generation")
    async def run(
        self,
        query: str,
        sql: str,
        sql_data: dict,
        language: str,
        current_time: str,
        query_id: str,
        custom_instruction: Optional[str] = None
    ) -> dict:
        """执行完整的SQL答案生成pipeline"""
        logger.info("SQL Answer Generation pipeline is running...")
        
        return await self._pipe.execute(
            ["generate_answer"],
            inputs={
                "query": query,
                "sql": sql,
                "sql_data": sql_data,
                "language": language,
                "current_time": current_time,
                "query_id": query_id,
                "custom_instruction": custom_instruction or "",
                **self._components
            }
        )
```

### 3.3 SQL Data Preprocessor

```python
# 文件: wren-ai-service/src/pipelines/retrieval/preprocess_sql_data.py

class PreprocessSqlData:
    """SQL数据预处理 - 限制token数量"""
    
    def run(self, sql_data: Dict) -> dict:
        """
        预处理SQL查询结果：
        1. 转换为DataFrame
        2. 计算每行的token数
        3. 逐行添加，直到达到token限制
        4. 返回截断后的数据
        """
        if not sql_data:
            return {
                "sql_data": {"columns": [], "data": []},
                "num_rows_used_in_llm": 0
            }
        
        try:
            df = pd.DataFrame(sql_data)
            
            if df.empty:
                return {
                    "sql_data": {"columns": [], "data": []},
                    "num_rows_used_in_llm": 0
                }
            
            # 使用tiktoken计算token数量
            encoding = tiktoken.encoding_for_model("gpt-4")
            
            # Token限制 - 防止超过LLM的上下文窗口
            MAX_TOKENS = 10000
            
            # 计算列名的token数
            columns_str = str(df.columns.tolist())
            columns_tokens = len(encoding.encode(columns_str))
            current_tokens = columns_tokens
            
            # 逐行添加数据
            preprocessed_rows = []
            for idx, row in df.iterrows():
                # 将行转换为JSON字符串
                row_str = row.to_json()
                row_tokens = len(encoding.encode(row_str))
                
                # 检查是否超过限制
                if current_tokens + row_tokens > MAX_TOKENS:
                    logger.info(f"Reached token limit at row {idx}")
                    break
                
                preprocessed_rows.append(row.to_dict())
                current_tokens += row_tokens
            
            return {
                "sql_data": {
                    "columns": df.columns.tolist(),
                    "data": preprocessed_rows
                },
                "num_rows_used_in_llm": len(preprocessed_rows),
                "total_tokens": current_tokens
            }
            
        except Exception as e:
            logger.error(f"Preprocess SQL data error: {e}")
            return {
                "sql_data": {"columns": [], "data": []},
                "num_rows_used_in_llm": 0
            }
```

---

## 4. 关键工具类

### 4.1 Hamilton Pipeline 基类

```python
# 文件: src/core/pipeline.py

class BasicPipeline(ABC):
    """
    所有Pipeline的基类
    使用Hamilton框架实现函数式DAG
    """
    
    def __init__(self, driver: AsyncDriver):
        self._pipe = driver
    
    @abstractmethod
    async def run(self, **kwargs) -> dict:
        """执行pipeline"""
        pass


# 使用示例
class MyPipeline(BasicPipeline):
    def __init__(self, llm_provider):
        # 定义pipeline的组件
        self._components = {
            "generator": llm_provider.get_generator(),
            "prompt_builder": PromptBuilder(template="...")
        }
        
        # 初始化Hamilton驱动
        super().__init__(
            AsyncDriver({}, sys.modules[__name__], result_builder=base.DictResult())
        )
    
    # 定义pipeline步骤（Hamilton会自动构建DAG）
    def step1(self, input_data, component1):
        return component1.process(input_data)
    
    def step2(self, step1, component2):
        return component2.process(step1)
    
    async def run(self, input_data):
        # 执行pipeline到目标节点
        return await self._pipe.execute(
            ["step2"],  # 目标节点
            inputs={
                "input_data": input_data,
                **self._components
            }
        )
```

### 4.2 LLM Provider 抽象

```python
# 文件: src/core/provider.py

class LLMProvider(ABC):
    """LLM提供商抽象基类"""
    
    @abstractmethod
    def get_generator(
        self,
        system_prompt: str,
        generation_kwargs: dict = None,
        streaming_callback: Callable = None
    ):
        """获取LLM生成器"""
        pass
    
    @abstractmethod
    def get_model(self) -> str:
        """获取模型名称"""
        pass


class OpenAIProvider(LLMProvider):
    """OpenAI实现"""
    
    def __init__(self, api_key: str, model: str = "gpt-4"):
        self._api_key = api_key
        self._model = model
    
    def get_generator(self, system_prompt, generation_kwargs=None, streaming_callback=None):
        return OpenAIChatGenerator(
            api_key=self._api_key,
            model=self._model,
            system_prompt=system_prompt,
            generation_kwargs=generation_kwargs or {},
            streaming_callback=streaming_callback
        )
    
    def get_model(self) -> str:
        return self._model
```

### 4.3 向量检索抽象

```python
# 文件: src/core/provider.py

class DocumentStoreProvider(ABC):
    """文档存储提供商抽象基类"""
    
    @abstractmethod
    def get_store(self, store_name: str):
        """获取文档存储"""
        pass
    
    @abstractmethod
    def get_retriever(self, document_store):
        """获取检索器"""
        pass


class QdrantProvider(DocumentStoreProvider):
    """Qdrant向量数据库实现"""
    
    def __init__(self, url: str, api_key: str):
        self._url = url
        self._api_key = api_key
    
    def get_store(self, store_name: str):
        return QdrantDocumentStore(
            url=self._url,
            api_key=self._api_key,
            index=store_name,
            embedding_dim=768
        )
    
    def get_retriever(self, document_store):
        return QdrantEmbeddingRetriever(
            document_store=document_store,
            top_k=10
        )
```

### 4.4 Observability - Langfuse集成

```python
# 文件: src/utils.py

from langfuse.decorators import observe

# 用法1: 追踪整个pipeline
@observe(name="SQL Generation")
async def sql_generation_pipeline(...):
    pass

# 用法2: 追踪LLM调用
@observe(as_type="generation", capture_input=False)
async def generate_sql(prompt, generator):
    return await generator(prompt=prompt)

# 用法3: 追踪成本
@trace_cost
async def llm_call(...):
    pass

# 用法4: 添加元数据
@trace_metadata
async def service_method(..., **kwargs):
    trace_id = kwargs.get("trace_id")
    # 自动记录trace_id到Langfuse
    pass
```

---

## 📊 完整示例：端到端流程

```python
# 完整的问答流程示例

# 1. 用户提问
user_question = "What are the top 5 products by revenue in 2023?"

# 2. Text-to-SQL
ask_result = await ask_service.ask(
    AskRequest(
        query=user_question,
        project_id="project_123",
        configurations=Configuration(language="en")
    )
)
# 返回: {"sql": "SELECT product_name, SUM(revenue) as total_revenue..."}

# 3. Text-to-Charts
chart_result = await chart_service.chart(
    ChartRequest(
        query=user_question,
        sql=ask_result["sql"],
        configurations=Configuration(language="en")
    )
)
# 返回: {
#   "chart_type": "bar",
#   "reasoning": "Bar chart is best for comparing quantities",
#   "chart_schema": {...}  # Vega-Lite Schema
# }

# 4. AI洞察生成
answer_result = await sql_answer_service.sql_answer(
    SqlAnswerRequest(
        query=user_question,
        sql=ask_result["sql"],
        sql_data=ask_result["data"],
        configurations=Configuration(language="en")
    )
)

# 5. 流式获取答案
async for chunk in sql_answer_service.get_sql_answer_streaming_result(query_id):
    print(chunk, end="", flush=True)

# 输出示例:
"""
Based on the sales data for 2023, here are the **top 5 products by revenue**:

1. **Product A** - $1,250,000 in total revenue
2. **Product B** - $980,000 in total revenue
3. **Product C** - $875,000 in total revenue
4. **Product D** - $720,000 in total revenue
5. **Product E** - $650,000 in total revenue

**Key Insights**:
- Product A dominates the market with 27% of total revenue
- Top 3 products account for over 60% of total revenue
- There's a significant drop-off after the top 3 products

**Recommendation**: Focus marketing and inventory efforts on the top 3 products 
to maximize profitability.
"""
```

---

这些核心代码展示了 GenBI 三大功能的完整实现逻辑，包括：
1. **模块化设计**: 每个功能都是独立的Pipeline
2. **流式处理**: 支持实时响应和流式输出
3. **可观测性**: 完整的Langfuse追踪
4. **错误处理**: 多重验证和自动纠错
5. **性能优化**: 并行执行、缓存、token限制

希望这些代码示例能帮助您理解 GenBI 的核心实现！
