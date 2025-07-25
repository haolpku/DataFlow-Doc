---
title: Text-to-SQL数据合成流水线
icon: material-symbols-light:checkbook-outline-rounded
createTime: 2025/06/17 02:00:31  
permalink: /zh/guide/text2sqlpipeline/  
---

# Text-to-SQL数据合成流水线

## 1. 概述

**Text-to-SQL数据合成流水线**的核心目标是通过清洗和扩充现有的Text-to-SQL数据，为每个样本生成包含训练提示词（prompt）和思维链（chain-of-thought）的高质量问答数据。本流水线能够一键完成从原始数据到最终训练数据的全流程处理，目前支持以下两种数据生成模式：

### 支持的应用场景：
* **数据优化模式**：
  - 对已有数据进行筛选、扩充和增强，生成高质量训练数据
  - 输入要求：必须包含数据库ID、自然语言问题和标准SQL答案三要素
* **数据合成模式**：
  - 直接从数据库生成训练数据
  - 特点：无需现有数据样本，零样本启动

### 处理流程：
1. **数据过滤**：
  - 执行过滤：剔除无效SQL和无法执行的SQL语句
  - 一致性过滤：确保问题、SQL与数据库Schema三者一致
2. **数据生成**：
  - SQL变体生成：基于现有SQL生成语义等效的变体
  - SQL合成：根据数据库Schema生成新的SQL语句
  - 问题生成：基于SQL和Schema生成对应的自然语言描述
3. **训练数据构建**：
  - 提示词生成：整合自然语言问题、数据库Schema和指令提示
  - 思维链生成：构建分步推理过程（Chain-of-Thought）
4. **数据分级**：
  - 语法难度分级：根据SQL语句的复杂度划分等级
  - 执行难度分级：基于SQL执行通过率评估难度

## 2. 输入数据

根据需要的不同，我们将流水线分为两条，一条数据优化流水线：从已有数据进行筛选并扩充，另一条数据合成流水线：不需要已有数据，直接从数据库中合成数据。

### 2.1 数据优化流水线

流水线的输入数据主要包括以下字段：

* **db_id**：数据库文件名称，即数据库id
* **question**：自然语言问题
* **SQL**：标准SQL答案

- **示例**（`json`）：
  ```json
  {
    "db_id": "california_schools",
    "question": "What is the highest eligible free rate for K-12 students in the schools in Alameda County?",
    "SQL": "SELECT `Free Meal Count (K-12)` / `Enrollment (K-12)` FROM frpm WHERE `County Name` = 'Alameda' ORDER BY (CAST(`Free Meal Count (K-12)` AS REAL) / `Enrollment (K-12)`) DESC LIMIT 1"
  }
  ```
- **演示数据集**：  
  `example_data/Text2SQLPipeline/pipeline_refine.jsonl`  
  包含数据库id、自然语言问题和标准SQL答案，适用于快速测试和演示。

这些输入数据可以存储在指定的文件（如`json`、`jsonl`）中，并通过`FileStorage`对象进行管理和读取。示例中会载入默认的数据路径，实际使用场景下可以根据需求修改路径以载入自定义的数据和缓存路径：

```python
self.storage = FileStorage(
    first_entry_file_name="../example_data/Text2SQLPipeline/pipeline_refine.jsonl",
    cache_path="./cache_local",
    file_name_prefix="dataflow_cache_step",
    cache_type="jsonl"
)
```

### 2.2 数据合成流水线

该流水线不需要已有数据，直接从数据库中合成数据。因此这里只需要配置数据库即可。将数据库配置好之后，送入DatabaseManager进行管理。此时不需要传入first_entry_file_name，因此将first_entry_file_name设置为`""`即可。

```python
self.storage = FileStorage(
    first_entry_file_name="",
    cache_path="./cache",
    file_name_prefix="dataflow_cache_step",
    cache_type="jsonl"
)
```

## 3. 配置说明

在开始执行流水线之前，请阅读以下配置说明，完成相关参数的配置后即可运行。

### 3.1 数据库配置

在进行数据库解析和执行时，需要配置相应的数据库信息。目前支持 SQLite 数据库和 MySQL 数据库，对其他数据库的支持正在持续更新中。

#### 3.1.1 SQLite 数据库

SQLite 是一种基于文件的数据库系统，因此在使用时需要指定数据库文件的存储路径。

- **数据库根目录**：用于存放所有数据库文件的目录  
  - **说明**：该目录下应包含多个 `.sqlite` 或 `.db` 格式的数据库文件。数据库管理器会自动扫描该目录并加载所有数据库文件。
  
  - **重要提示**：每个数据库文件的文件名即为 `db_id`，格式必须为 `db_id.sqlite` 或 `db_id.db`，且与输入数据中的 `db_id` 字段保持一致。

  - **支持的目录结构**：  
    数据库管理器支持任意嵌套层级的目录结构，以下为几种合法结构示例：
    ```
    databases/
      ├── california_schools.sqlite
      └── hospitals.sqlite
    ```

    ```
    databases/
      ├── forder1/
      │   └── california_schools.sqlite
      └── forder2/
          └── hospitals.sqlite
    ```

    ```
    databases/
      ├── california_schools.sqlite
      └── forder1/
          └── hospitals.sqlite
    ```

  - **演示数据库**：  
    我们提供了示例数据库用于测试，请访问：  
    [https://huggingface.co/datasets/Open-Dataflow/dataflow-Text2SQL-database-example](https://huggingface.co/datasets/Open-Dataflow/dataflow-Text2SQL-database-example)

    **使用步骤**：
    1. 下载 `databases` 压缩包并解压到本地
    2. 将解压后的路径赋值给变量 `db_root_path`
    3. 在代码中配置数据库管理器如下：

    ```python
    database_manager = DatabaseManager(
        db_type="sqlite",
        config={
            "root_path": db_root_path
        }
    )
    ```

    > 注意：`db_type` 必须设置为 `"sqlite"`，`root_path` 应为数据库文件夹的路径。

#### 3.1.2 MySQL 数据库

MySQL 数据库是以服务器形式存在的，因此需要管理连接服务器的信息。请确保您的 MySQL 服务处于开启状态，并已正确配置用户名和密码。在 DataFlow 中，我们使用 `pymysql` 库来连接 MySQL 服务器。

- **配置方式**：
  1. 准备 MySQL 服务器信息  
  2. 将 MySQL 服务器信息配置到 `database_manager`：
    ```python
    database_manager = DatabaseManager(
        db_type="mysql",
        config={
            "host": "localhost",
            "user": "root",
            "password": "password"
        }
    )
    ```
   > 其中 `db_type` 必须设定为 `mysql`，在 `config` 中，需设定 `host`、`user` 和 `password` 为 MySQL 服务器的相关信息。请确保所需使用的数据库存在于 MySQL 服务器中，并且您具有相应的访问权限。

## 3.2 模型配置

### 3.2.1 API LLM 服务配置

在 DataFlow 中，我们使用 `APILLMServing_request` 类来管理基于 API 的 LLM 服务。

```python
api_llm_serving = APILLMServing_request(
    api_url="https://api.openai.com/v1/chat/completions",
    model_name="chatgpt",
    max_workers=100
)

cot_generation_api_llm_serving = APILLMServing_request(
    api_url="https://api.openai.com/v1/chat/completions",
    model_name="gpt-4o",  # 使用性能更强的模型生成长链推理过程
    max_workers=100
)

embedding_api_llm_serving = APILLMServing_request(
    api_url="https://api.openai.com/v1/embeddings",
    model_name="text-embedding-ada-002",
    max_workers=100
)
```

其中：
- `api_llm_serving` 用于处理通用提示生成任务；
- `cot_generation_api_llm_serving` 用于生成复杂推理链（Chain-of-Thought）；
- `embedding_api_llm_serving` 用于生成文本嵌入向量。

在实际使用中，您可以根据需求更换为其他模型或 API 提供商。

### 3.2.2 本地模型服务配置

在 DataFlow 中，我们使用 `LocalModelLLMServing_vllm` 类来管理本地部署的大语言模型服务。

```python
llm_serving = LocalModelLLMServing_vllm(
    hf_model_name_or_path="Qwen/Qwen2.5-7B-Instruct", 
    vllm_tensor_parallel_size=1,
    vllm_max_tokens=8192,
)

cot_generation_llm_serving = LocalModelLLMServing_vllm(
    hf_model_name_or_path="Qwen/Qwen2.5-7B-Instruct", 
    vllm_tensor_parallel_size=1,
    vllm_max_tokens=8192,
)

embedding_serving = LocalModelLLMServing_vllm(
    hf_model_name_or_path="Alibaba-NLP/gte-Qwen2-7B-instruct", 
    vllm_max_tokens=8192
)
```

其中：
- `llm_serving` 用于处理通用提示生成任务；
- `cot_generation_llm_serving` 用于生成复杂推理链；
- `embedding_serving` 用于生成文本嵌入向量。

## 3.3 其他参数配置

### 3.3.1 难度分类配置

```python
execution_difficulty_config = {
    'thresholds': [2, 5, 9],
    'labels': ['easy', 'medium', 'hard', 'extra']
}

component_difficulty_config = {
    'thresholds': [2, 4, 6],      
    'labels': ['easy', 'medium', 'hard', 'extra']
}
```

其中：
- `execution_difficulty_config` 用于执行难度分类；
- `component_difficulty_config` 用于 SQL 组件复杂度分类。

#### 注意事项：
- `thresholds` 和 `labels` 必须同时存在；
- `thresholds` 必须按升序排列；
- `labels` 的数量必须比 `thresholds` 多 1；
- 分类依据是得分，范围为 0–10，得分越高表示难度越大，因此 `thresholds` 的值应控制在 0–10 范围内。

### 3.3.2 提示词模板配置

```python
prompt_template = '''Task Overview:
            /* Given the following database schema: */
            {schema}
            /* Answer the following: {question} */
            Let's think step by step'''
```

该模板用于构建输入给模型的提示信息，其中 `{schema}` 和 `{question}` 是占位符，分别表示数据库 Schema 和用户提出的自然语言问题。  
您可根据需要自定义模板内容，但**必须保留这两个占位符**以确保数据注入的完整性。

### 3.3.3 数据库 Schema 配置

```python
schema_config = {
    'format': 'ddl',  # 可选值：'ddl' 或 'formatted_schema'
    'use_example': False  # 是否包含示例数据
}
```

说明：
- `format`：指定输出数据库 Schema 的格式，支持 `'ddl'`（数据定义语言）和 `'formatted_schema'`（结构化展示）；
- `use_example`：是否在 Schema 中包含示例数据，取值为 `True` 或 `False`。


## 4. 数据流与流水线逻辑

### 4.1 数据过滤器

#### 4.1.1 **SQL执行过滤器（ExecutionFilter）**

**SQL执行过滤器**（`ExecutionFilter`）通过实际执行SQL语句来验证其正确性，过滤掉无法正常执行的SQL语句。

**功能：**

* 验证SQL语句的可执行性
* 过滤掉语法错误或执行失败的SQL语句

**输入**：SQL语句和数据库ID
**输出**：可正常执行的SQL语句

```python
execution_filter = ExecutionFilter(
    database_manager=database_manager
)
```

#### 4.1.2 **SQL一致性过滤器（ConsistencyFilter）**

**SQL一致性过滤器**（`ConsistencyFilter`）检查SQL语句与问题、数据库Schema之间的一致性，确保生成的SQL能够正确回答对应的问题。

**功能：**

* 验证SQL语句与问题、数据库Schema之间的一致性
* 过滤掉与问题、数据库Schema不匹配的SQL语句

**输入**：SQL语句、数据库ID和问题
**输出**：与问题一致的SQL语句

```python
consistency_filter = ConsistencyFilter(
    llm_serving=api_llm_serving,
    database_manager=database_manager
)
```

### 4.2 数据生成器

#### 4.2.1 **SQL生成器（SQLGenerator）**

**SQL生成器**（`SQLGenerator`）负责基于数据库schema生成SQL查询语句，为后续的数据处理流程提供原始SQL数据。

**功能：**

* 基于数据库schema自动生成SQL查询语句
* 支持批量生成指定数量的SQL语句

**输入**：数据库schema信息
**输出**：生成的SQL语句和对应的数据库ID

```python
sql_generator = SQLGenerator(
    llm_serving=api_llm_serving,
    database_manager=database_manager,
    generate_num=300
)
```

#### 4.2.2 **SQL变体生成器（SQLVariationGenerator）**

**SQL变体生成器**（`SQLVariationGenerator`）基于现有的SQL语句生成多个功能等价的变体，丰富数据集的多样性。

**功能：**

* 生成功能等价的SQL变体
* 增加SQL语句的多样性和复杂性

**输入**：原始SQL语句和数据库ID
**输出**：SQL变体集合

```python
sql_variation_generator = SQLVariationGenerator(
    llm_serving=api_llm_serving,
    database_manager=database_manager,
    num_variations=5
)
```

#### 4.2.3 **问题生成器（QuestionGeneration）**

**问题生成器**（`QuestionGeneration`）根据给定的SQL语句生成对应的自然语言问题，构建Text-to-SQL的问答对。

**功能：**

* 基于SQL语句生成自然语言问题
* 支持生成多个候选问题

**输入**：SQL语句和数据库ID
**输出**：自然语言问题

```python
question_generator = QuestionGeneration(
    llm_serving=api_llm_serving,
    embedding_api_llm_serving=embedding_api_llm_serving,
    database_manager=database_manager,
    question_candidates_num=5
)
```

#### 4.2.4 **提示词生成器（PromptGenerator）**

**提示词生成器**（`PromptGenerator`）根据问题和数据库schema生成用于模型训练的提示模板。

**功能：**

* 生成结构化的提示模板
* 整合问题和数据库schema信息

**输入**：问题和数据库ID
**输出**：格式化的提示模板

```python
prompt_generator = PromptGenerator(
    database_manager=database_manager,
    prompt_template=prompt_template,
    schema_config=schema_config
)
```

#### 4.2.5 **长链推理生成器（CoTGenerator）**

**长链推理生成器**（`CoTGenerator`）为SQL查询生成详细的推理过程，帮助模型理解从问题到SQL的转换逻辑。

**功能：**

* 生成SQL查询的推理过程
* 支持重试机制确保生成质量

**输入**：SQL语句、问题和数据库ID
**输出**：思维链推理过程

```python
cot_generator = CoTGenerator(
    llm_serving=cot_generation_api_llm_serving,
    database_manager=database_manager,
    schema_config=schema_config,
    max_retries=3,
    enable_retry=True
)
```

### 4.3 数据评估器

#### 4.3.1 **组件难度评估器（ComponentClassifier）**

**组件难度评估器**（`ComponentClassifier`）分析SQL语句的组件复杂度，为数据样本标注难度等级。

**功能：**

* 分析SQL语句的组件复杂度
* 为样本标注难度等级

**输入**：SQL语句
**输出**：SQL组件难度等级

```python
component_classifier = ComponentClassifier(
    difficulty_config=component_difficulty_config
)
```

#### 4.3.2 **执行难度评估器（ExecutionClassifier）**

**执行难度评估器**（`ExecutionClassifier`）评估SQL查询的执行难度，基于多次生成结果进行综合判断。

**功能：**

* 评估SQL查询的执行难度
* 基于多次生成进行难度评估

**输入**：SQL语句、数据库ID和提示
**输出**：SQL执行难度等级

```python
execution_classifier = ExecutionClassifier(
    llm_serving=api_llm_serving,
    database_manager=database_manager,
    difficulty_config=execution_difficulty_config,
    num_generations=5
)
```

## 5. **输出数据**

- **格式**：`jsonl`（每个步骤都会生成一个文件）  
- **字段说明**：
  - `db_id`: 数据库id
  - `question`: 自然语言问题
  - `SQL`: 标准SQL答案
  - `prompt`: 用于训练的提示词，包含自然语言问题、数据库Schema和提示信息
  - `cot_reasoning`: 长链推理数据，包含推理过程和最终答案，用于模型训练
  - `sql_component_difficulty`: SQL组件复杂度评估
  - `sql_execution_difficulty`: SQL执行复杂度评估
- **示例**：
  ```json
  {
      "db_id":"california_schools",
      "SQL":"SELECT AVG(s.AvgScrRead) AS average_reading_score\nFROM satscores s\nINNER JOIN frpm f ON s.cds = f.CDSCode\nINNER JOIN schools sc ON f.CDSCode = sc.CDSCode\nWHERE s.cname = 'Alameda'\n  AND f.\"Charter School (Y\/N)\" = 1\n  AND f.\"Charter Funding Type\" = 'Directly funded'\n  AND sc.County = 'Alameda';",
      "question":"What is the average reading score for directly funded charter schools in Alameda County?",
      "prompt":"Task Overview: /* Given the following database schema: ... /* Answer the following: What is the average reading score for directly funded charter schools in Alameda County? * Let's think step by step",
      "cot_reasoning":"To translate the natural language question into an executable SQLite query, we will follow these steps. ... we can construct the full SQLite query based on these steps:\n\n```sql\nSELECT AVG(s.AvgScrRead) AS average_reading_score\nFROM satscores s\nINNER JOIN frpm f ON s.cds = f.CDSCode\nINNER JOIN schools sc ON f.CDSCode = sc.CDSCode\nWHERE s.cname = 'Alameda'\n  AND f.\"Charter School (Y\/N)\" = 1\n  AND f.\"Charter Funding Type\" = 'Directly funded'\n  AND sc.County = 'Alameda';\n```\n\nThis query follows the logic outlined above and ensures alignment with the reference solution.",
      "sql_component_difficulty":"medium",
      "sql_execution_difficulty":"medium"
  }
  ```

## 6. 运行方式

这里设计了两套流水线，通过简单的Python命令执行不同的配置，满足不同的数据需求：

* **数据优化流水线**：

  ```bash
  python /pipelines/api_pipelines/text2sql_pipeline_refine.py
  ```

* **数据合成流水线**：

  ```bash
  python /pipelines/api_pipelines/text2sql_pipeline_gen.py
  ```


## 7. 流水线示例

以下给出示例流水线，演示如何使用多个算子进行推理数据处理。该示例展示了如何初始化一个推理数据处理流水线，并且顺序执行各个过滤和清理步骤。

* **数据优化流水线**：

```python
class Text2SQLPipeline():
    def __init__(self):

        self.storage = FileStorage(
            first_entry_file_name="../example_data/Text2SQLPipeline/pipeline_refine.jsonl",
            cache_path="./cache_local",
            file_name_prefix="dataflow_cache_step",
            cache_type="jsonl"
        )

        api_llm_serving = APILLMServing_request(
            api_url="http://api.openai.com/v1/chat/completions",
            model_name="gpt-4o",
            max_workers=100
        )

        cot_generation_api_llm_serving = APILLMServing_request(
            api_url="http://api.openai.com/v1/chat/completions",
            model_name="gpt-4o", 
            max_workers=100
        )

        embedding_api_llm_serving = APILLMServing_request(
            api_url="http://api.openai.com/v1/embeddings",
            model_name="text-embedding-ada-002",
            max_workers=100
        )

        execution_difficulty_config = {
            'thresholds': [2, 5, 9],
            'labels': ['easy', 'medium', 'hard', 'extra']
        }

        component_difficulty_config = {
            'thresholds': [2, 4, 6],      
            'labels': ['easy', 'medium', 'hard', 'extra']
        }

        prompt_template = '''Task Overview:
            /* Given the following database schema: */
            {schema}
            /* Answer the following: {question} */
            Let's think step by step'''

        schema_config = {
            'format': 'ddl',  
            'use_example': False  
        }

        db_root_path = "path/to/your/database"  

        database_manager = DatabaseManager(
            db_type="sqlite",
            config={
                "root_path": db_root_path
            }
        )
        
        self.sql_execution_filter_step1 = ExecutionFilter(
            database_manager=database_manager
        )

        self.sql_consistency_filter_step2 = ConsistencyFilter(
            llm_serving=api_llm_serving,
            database_manager=database_manager
        )

        self.sql_variation_generator_step3 = SQLVariationGenerator(
            llm_serving=api_llm_serving,
            database_manager=database_manager,
            num_variations=5
        )

        self.sql_execution_filter_step4 = ExecutionFilter(
            database_manager=database_manager
        )

        self.text2sql_question_generator_step5 = QuestionGeneration(
            llm_serving=api_llm_serving,
            embedding_api_llm_serving=embedding_api_llm_serving,
            database_manager=database_manager,
            question_candidates_num=5
        )

        self.text2sql_prompt_generator_step6 = PromptGenerator(
            database_manager=database_manager,
            prompt_template=prompt_template,
            schema_config=schema_config
        )

        self.sql_cot_generator_step7 = CoTGenerator(
            llm_serving=cot_generation_api_llm_serving,
            database_manager=database_manager,
            schema_config=schema_config,
            max_retries=3,
            enable_retry=True
        )

        self.sql_component_classifier_step8 = ComponentClassifier(
            difficulty_config=component_difficulty_config
        )

        self.sql_execution_classifier_step9 = ExecutionClassifier(
            llm_serving=api_llm_serving,
            database_manager=database_manager,
            difficulty_config=execution_difficulty_config,
            num_generations=5
        )
        
        
    def forward(self):

        sql_key = "SQL"
        db_id_key = "db_id"
        question_key = "question"

        self.sql_execution_filter_step1.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key
        )

        self.sql_consistency_filter_step2.run(
            storage=self.storage.step(),   
            input_sql_key=sql_key,
            input_db_id_key=db_id_key,
            input_question_key=question_key
        )

        self.sql_variation_generator_step3.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key
        )

        self.sql_execution_filter_step4.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key
        )

        self.text2sql_question_generator_step5.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key,
            output_question_key=question_key
        )

        self.text2sql_prompt_generator_step6.run(
            storage=self.storage.step(),
            input_question_key=question_key,
            input_db_id_key=db_id_key,
            output_prompt_key="prompt"
        )

        self.sql_cot_generator_step7.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_question_key=question_key,
            input_db_id_key=db_id_key,
            output_cot_key="cot_reasoning"
        )

        self.sql_component_classifier_step8.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            output_difficulty_key="sql_component_difficulty"
        )

        self.sql_execution_classifier_step9.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key,
            input_prompt_key="prompt",
            output_difficulty_key="sql_execution_difficulty"
        )

if __name__ == "__main__":
    model = Text2SQLPipeline()
    model.forward()
```

* **数据合成流水线**：
```python
class Text2SQLPipeline():
    def __init__(self):

        self.storage = FileStorage(
            first_entry_file_name="",
            cache_path="./cache",
            file_name_prefix="dataflow_cache_step",
            cache_type="jsonl",
        )

        api_llm_serving = APILLMServing_request(
            api_url="http://api.openai.com/v1/chat/completions",
            model_name="gpt-4o",
            max_workers=100
        )

        cot_generation_api_llm_serving = APILLMServing_request(
            api_url="http://api.openai.com/v1/chat/completions",
            model_name="gpt-4o", 
            max_workers=100
        )

        embedding_api_llm_serving = APILLMServing_request(
            api_url="http://api.openai.com/v1/embeddings",
            model_name="text-embedding-ada-002",
            max_workers=100
        )

        execution_difficulty_config = {
            'thresholds': [2, 5, 9],
            'labels': ['easy', 'medium', 'hard', 'extra']
        }

        component_difficulty_config = {
            'thresholds': [2, 4, 6],      
            'labels': ['easy', 'medium', 'hard', 'extra']
        }

        prompt_template = '''Task Overview:
            /* Given the following database schema: */
            {schema}
            /* Answer the following: {question} */
            Let's think step by step'''

        schema_config = {
            'format': 'ddl',  
            'use_example': True  
        }

        db_root_path = "path/to/your/database"  

        database_manager = DatabaseManager(
            db_type="sqlite",
            config={
                "root_path": db_root_path
            }
        )
        
        self.sql_generator_step1 = SQLGenerator(
            llm_serving=api_llm_serving,
            database_manager=database_manager,
            generate_num=300
        )

        self.sql_execution_filter_step2 = ExecutionFilter(
            database_manager=database_manager
        )

        self.text2sql_question_generator_step3 = QuestionGeneration(
            llm_serving=api_llm_serving,
            embedding_api_llm_serving=embedding_api_llm_serving,
            database_manager=database_manager,
            question_candidates_num=5
        )

        self.text2sql_prompt_generator_step4 = PromptGenerator(
            database_manager=database_manager,
            prompt_template=prompt_template,
            schema_config=schema_config
        )

        self.sql_cot_generator_step5 = CoTGenerator(
            llm_serving=cot_generation_api_llm_serving,
            database_manager=database_manager,
            schema_config=schema_config,
            max_retries=3,
            enable_retry=True
        )

        self.sql_component_classifier_step6 = ComponentClassifier(
            difficulty_config=component_difficulty_config
        )

        self.sql_execution_classifier_step7 = ExecutionClassifier(
            llm_serving=api_llm_serving,
            database_manager=database_manager,
            difficulty_config=execution_difficulty_config,
            num_generations=5
        )
        
        
    def forward(self):

        sql_key = "SQL"
        db_id_key = "db_id"
        question_key = "question"

        self.sql_generator_step1.run(
            storage=self.storage.step(),
            output_sql_key=sql_key,
            output_db_id_key=db_id_key
        )

        self.sql_execution_filter_step2.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key
        )

        self.text2sql_question_generator_step3.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key,
            output_question_key=question_key
        )

        self.text2sql_prompt_generator_step4.run(
            storage=self.storage.step(),
            input_question_key=question_key,
            input_db_id_key=db_id_key,
            output_prompt_key="prompt"
        )

        self.sql_cot_generator_step5.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_question_key=question_key,
            input_db_id_key=db_id_key,
            output_cot_key="cot_reasoning"
        )

        self.sql_component_classifier_step6.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            output_difficulty_key="sql_component_difficulty"
        )

        self.sql_execution_classifier_step7.run(
            storage=self.storage.step(),
            input_sql_key=sql_key,
            input_db_id_key=db_id_key,
            input_prompt_key="prompt",
            output_difficulty_key="sql_execution_difficulty"
        )

if __name__ == "__main__":
    model = Text2SQLPipeline()
    model.forward()
```