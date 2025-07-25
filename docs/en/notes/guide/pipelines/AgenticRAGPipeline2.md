---
title: Agentic RAG Data Synthesis Pipeline-Beta
icon: solar:palette-round-linear
createTime: 2025/07/14 16:37:14  
permalink: /en/guide/agenticrag_pipeline2/  
---

# Agentic RAG Data Synthesis Pipeline

## 1. Overview

The **Agentic RAG Data Synthesis Pipeline** is an end-to-end framework to:  
- Support RL-based agentic RAG training.
- Generate high-quality pairs of questions and answers from provided text contents.

This pipeline only need text contexts for generating high-quality questions and answers for further training  

---

## 2. Data Flow and Pipeline Logic

### 1. **Input Data**

The input data for the pipeline includes the following fields:

* **text**: various text contents 

These input data can be stored in designated files (such as `json` or `jsonl`) and managed and read via the `FileStorage` object. In the provided example, the default data path is loaded. In practical use, you can modify the path to load custom data and cache paths:

```python
self.storage = FileStorage(
    first_entry_file_name="../dataflow/example/AgenticRAGPipeline/pipeline_small_chunk.json",
    cache_path="./cache_local",
    file_name_prefix="dataflow_cache_step",
    cache_type="jsonl",
)
```

### 2. **Atomic Task Generation**

#### 2.1 **Generating Task**

The first step of the process is to use the **Atomic Task Generator** operator (`AtomicTaskGenerator`) to generate the question, reference answer, refined reference answer, optional verifiable answers, and the LLM’s answer to the question when provided with the original document—all from a large dataset.

**Functionality:**

* Generate the question, reference answer, refined reference answer, optional verifiable answers, and the LLM’s answer to the question when provided with the original document—all from a large dataset.

**Input:** Original text content

**Output:** Question, reference answer, refined reference answer, optional verifiable answers, and the LLM’s answer to the question when provided with the original document—all from a large dataset.

```python
llm_serving = APILLMServing_request(
                    api_url="https://api.openai.com/v1/chat/completions",
                    model_name="gpt-4o-mini",
                    max_workers=500
        )

atomic_task_generator = AtomicTaskGenerator(
                            llm_serving=llm_serving
                        )
result = atomic_task_generator.run(
            storage = self.storage.step(),
            input_key = "text",
        )
```

### 3. **Task Quality Evaluation**

#### 3.1 **F1 Scorer**

The second step of the process is to use the **F1 Scorer** operator (`F1Scorer`) evaluate the F1 score between the refined reference answer and the LLM’s answer to the question when provided with the original document. This step ensures that each constructed question, when paired with correct document retrieval, receives an appropriate reward, thereby maintaining the training quality of reinforcement learning.

**Functionality:**

* Evaluate the F1 score between the refined reference answer and the LLM’s answer to the question given the original document.

**Input:** refined reference answer, the LLM’s answer to the question given the original document.
**Output:** F1 scores

```python
f1_scorer = F1Scorer(
            prediction_key="refined_answer",
            ground_truth_key="golden_doc_answer"
        )
result = f1_scorer.run(
            storage=self.storage.step(),
            output_key="F1Score"
        )
```

---

## 3. Running the Process

Run the complete process:

```python
import pandas as pd
from dataflow.operators.eval import *

from dataflow.operators.generate import (
    AtomicTaskGenerator,
    DepthQAGenerator,
    WidthQAGenerator
)

from dataflow.operators.filter import *
from dataflow.utils.storage import FileStorage
from dataflow.serving import APILLMServing_request, LocalModelLLMServing
from dataflow.core import LLMServingABC

class AgenticRAGEvalPipeline():

    def __init__(self, llm_serving=None):

        self.storage = FileStorage(
            first_entry_file_name="../dataflow/example/AgenticRAGPipeline/pipeline_small_chunk.json",
            cache_path="./agenticRAG_eval_cache",
            file_name_prefix="agentic_rag_eval",
            cache_type="jsonl",
        )

        llm_serving = APILLMServing_request(
            api_url="https://api.openai.com/v1/chat/completions",
            model_name="gpt-4o-mini",
            max_workers=500
        )

        self.task_step1 = AtomicTaskGenerator(
            llm_serving=llm_serving
        )

        self.task_step2 = F1Scorer(
            prediction_key="refined_answer",
            ground_truth_key="golden_doc_answer"
        )
        
    def forward(self):

        self.task_step1.run(
            storage = self.storage.step(),
            input_key = "contents",
        )

        self.task_step2.run(
            storage=self.storage.step(),
            output_key="F1Score"
        )

if __name__ == "__main__":
    model = AgenticRAGEvalPipeline()
    model.forward()
```