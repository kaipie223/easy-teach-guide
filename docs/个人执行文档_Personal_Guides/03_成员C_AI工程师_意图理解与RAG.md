# 03 — 成员C · AI 工程师（意图理解 + RAG + 知识融合 + 语音）

> **你是系统的"大脑"。没有你，这只是一个文件上传下载网站。**
> 核心职责边界：**从对话中提取结构化意图 + 从知识库检索相关内容 + 融合为课件指令集，作为 D 的输入。**

---

## 你需要产出的文件（完整清单）

```
ai/
├── __init__.py
├── intent/
│   ├── __init__.py
│   ├── analyzer.py              # IntentAnalyzer 类 — A 直接 import
│   └── state.py                 # 会话状态机（init→probing→confirming→locked）
├── rag/
│   ├── __init__.py
│   ├── loader.py                # 加载知识库文档
│   ├── splitter.py              # 文档切分
│   ├── embedder.py              # 向量化 + 写入 ChromaDB
│   └── retriever.py             # RAGRetriever 类 — A 直接 import
├── fusion/
│   ├── __init__.py
│   └── fusion.py                # KnowledgeFuser 类 — A 直接 import
├── speech/
│   ├── __init__.py
│   └── transcriber.py           # SpeechTranscriber 类 — A 直接 import
├── prompts/
│   ├── intent_extract.txt       # 意图提取 Prompt
│   ├── follow_up.txt            # 追问生成 Prompt
│   ├── confirm.txt              # 确认总结 Prompt
│   └── fusion.txt               # 知识融合 Prompt
├── build_kb.py                  # 知识库构建脚本（E 也会用）
└── test_intent.py               # 你自用的测试脚本
```

---

## 你与 A 和 D 的精确对接

```
A 的 backend/services/orchestrator.py 会这样 import 你的代码:

  from ai.intent.analyzer import IntentAnalyzer     ← 你必须提供这个类
  from ai.rag.retriever import RAGRetriever          ← 你必须提供这个类
  from ai.fusion.fusion import KnowledgeFuser         ← 你必须提供这个类
  from ai.speech.transcriber import SpeechTranscriber ← 你必须提供这个类

你的 fusion/fusion.py 会这样 import D 的代码:

  from gen.parse.pdf import extract_pdf_text          ← D 必须提供这个函数
  from gen.parse.doc import extract_doc_text          ← D 必须提供这个函数
  from gen.parse.video import summarize_video         ← D 必须提供这个函数
  from gen.parse.image import describe_image          ← D 必须提供这个函数

你和 A 和 D 共享同一个数据模型:

  from backend.schemas import (
      IntentResult, RAGDocument, ReferenceMaterial,
      GenerationInstruction, KnowledgePoint
  )
```

---

## 你的技术清单

| 需求 | 安装 |
|------|------|
| DeepSeek API Key | platform.deepseek.com → 注册 → API Keys → 充值 ¥20 |
| Python 依赖 | A 的 pyproject.toml 中已包含，你只需 `poetry install` |
| Whiser 模型 | 首次运行 `faster-whisper` 时自动下载 small 模型（~1GB） |
| Embedding 模型 | `text2vec-base-chinese` 首次加载时自动下载 |

---

## Phase 1：环境搭建 + 基础验证（Day 1-3）

### Day 1 — 今天必须完成

**任务 1.1：申请 DeepSeek API Key + 测试调用（20 分钟）**

```python
# 在项目根目录创建 ai/test_llm.py，验证 LLM 可调用
import os
from dotenv import load_dotenv
load_dotenv()

from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url=os.getenv("DEEPSEEK_BASE_URL")
)

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": "你是一个专业的教学设计师。"},
        {"role": "user", "content": "请用一句话解释什么是面向对象编程。"}
    ],
    max_tokens=200
)

print(response.choices[0].message.content)
# 看到正常回复 → 截图发群
```

**任务 1.2：验证 ChromaDB（15 分钟）**

```python
# ai/test_chroma.py
import chromadb
client = chromadb.PersistentClient(path="./chroma_data")

# 创建测试集合
collection = client.get_or_create_collection("test")

# 写入测试数据
collection.add(
    documents=["Python是一种解释型、面向对象的高级编程语言。"],
    metadatas=[{"source": "test"}],
    ids=["doc_001"]
)

# 检索
results = collection.query(query_texts=["Python是什么"], n_results=1)
print(results["documents"])
# 看到检索结果 → 截图发群
```

**任务 1.3：创建 ai/ 目录结构，所有文件至少有一个占位 `__init__.py`**

```bash
cd "C:\Users\kaipie\Desktop\省服务外包大赛\多模态AI教学系统"
mkdir -p ai/intent ai/rag ai/fusion ai/speech ai/prompts
touch ai/__init__.py ai/intent/__init__.py ai/rag/__init__.py ai/fusion/__init__.py ai/speech/__init__.py
```

---

### Day 2 — Prompt 模板 + 意图分析原型

**任务 2.1：写 4 个核心 Prompt 模板**

```text
# ai/prompts/intent_extract.txt
你是一个专业的教学设计师。请从以下教师对话中提取结构化的教学意图。

输出严格的 JSON 格式，不要输出任何其他内容：
{
  "teaching_goal": "一句话总结教学目标",
  "target_audience": "授课对象（如大一新生、在职培训等，未知则填null）",
  "duration_minutes": 课时长度（未知则填null）,
  "knowledge_points": [
    {"order": 1, "title": "知识点名", "difficulty": "basic/intermediate/advanced", "key_points": ["要点1", "要点2"], "examples": ["案例1"], "estimated_minutes": 10}
  ],
  "logic_flow": ["导入方式", "新授", "练习", "总结"],
  "style_preference": "严谨学术/生动活泼/案例驱动/实操为主（未知填null）",
  "missing_info": ["缺失的关键信息1", "缺失的关键信息2"],
  "is_complete": false
}

对话历史：
{messages}

请分析：
```

```text
# ai/prompts/follow_up.txt
以下是教师当前的课程设计需求，但缺失了一些关键信息。

当前已知：
{current_intent}

缺失的信息：
{missing_info}

请生成 1 个友好的追问，帮助教师补充缺失信息。追问应该：
1. 每次只问 1-2 个点
2. 给出 2-4 个选项供教师选择
3. 也允许教师自由输入

输出 JSON：
{
  "question_text": "追问文本",
  "options": ["选项A", "选项B", "选项C"],
  "allow_free": true
}
```

```text
# ai/prompts/confirm.txt
根据以下教学意图，生成一份结构化的"需求确认书"。

意图信息：
{intent_json}

请生成：
{
  "summary": "格式化的确认文本，包含课程名称、对象、时长、知识点列表、教学风格、教学流程"
}
```

```text
# ai/prompts/fusion.txt
你是一个教学课件设计专家。请综合以下三方面信息，生成一份完整的课件生成指令集。

1. 教师意图：
{intent_json}

2. 知识库补充材料：
{rag_context}

3. 参考资料内容：
{reference_texts}

请确保最终指令集：
- 知识点完整，逻辑清晰
- 融入参考资料中的案例和格式
- 补充知识库中的相关内容

输出严格的 JSON 格式（按 backend.schemas.GenerationInstruction 的结构）：
{
  "session_id": "...",
  "teaching_goal": "...",
  ...（完整字段见协议文档 2.2 节 GenerationInstruction 定义）
}
```

**任务 2.2：写 `analyzer.py` — 这是你的核心产出**

```python
# ai/intent/analyzer.py
import json
import os
from openai import OpenAI
from backend.schemas import IntentResult, KnowledgePoint

class IntentAnalyzer:
    """教师意图分析器 — A 在后端启动时创建单例"""

    def __init__(self, api_key: str, base_url: str):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.sessions = {}         # session_id → 对话状态
        self._load_prompts()

    def _load_prompts(self):
        base = os.path.dirname(os.path.dirname(__file__))  # ai/
        with open(f"{base}/prompts/intent_extract.txt", encoding="utf-8") as f:
            self.prompt_extract = f.read()
        with open(f"{base}/prompts/follow_up.txt", encoding="utf-8") as f:
            self.prompt_followup = f.read()
        with open(f"{base}/prompts/confirm.txt", encoding="utf-8") as f:
            self.prompt_confirm = f.read()

    def analyze(
        self,
        session_id: str,
        messages: list[dict],
        uploaded_files: list[str] = []
    ) -> IntentResult:
        """
        分析对话，返回 IntentResult。
        - is_complete=False → follow_up_question 有值 → A 发 question 事件
        - is_complete=True  → confirm_summary 有值   → A 发 confirm 事件
        """
        # Step 1: 调 LLM 提取意图
        formatted = "\n".join([f"{m['role']}: {m['content']}" for m in messages])
        prompt = self.prompt_extract.format(messages=formatted)

        resp = self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=1000,
            response_format={"type": "json_object"}
        )
        raw = json.loads(resp.choices[0].message.content)

        # Step 2: 检查是否完整
        if not raw.get("is_complete") and raw.get("missing_info"):
            # 生成追问
            follow_up = self._gen_follow_up(raw)
            raw["follow_up_question"] = follow_up["question_text"]
            raw["confirm_summary"] = None
        else:
            # 生成确认总结
            summary = self._gen_confirm(raw)
            raw["confirm_summary"] = summary["summary"]
            raw["follow_up_question"] = None
            raw["is_complete"] = True

        # 保存状态
        self.sessions[session_id] = raw

        # 构造 Pydantic Model 返回
        return IntentResult(**raw)

    def lock_intent(self, session_id: str) -> IntentResult:
        """教师确认后锁定意图"""
        state = self.sessions.get(session_id)
        if not state:
            raise ValueError(f"Session {session_id} not found")
        state["is_complete"] = True
        return IntentResult(**state)

    def _gen_follow_up(self, intent: dict) -> dict:
        prompt = self.prompt_followup.format(
            current_intent=json.dumps(intent, ensure_ascii=False),
            missing_info=", ".join(intent.get("missing_info", []))
        )
        resp = self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=300,
            response_format={"type": "json_object"}
        )
        return json.loads(resp.choices[0].message.content)

    def _gen_confirm(self, intent: dict) -> dict:
        prompt = self.prompt_confirm.format(intent_json=json.dumps(intent, ensure_ascii=False))
        resp = self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=500,
            response_format={"type": "json_object"}
        )
        return json.loads(resp.choices[0].message.content)
```

**任务 2.3：写 `test_intent.py` 自测**

```python
# ai/test_intent.py
import os
from dotenv import load_dotenv
load_dotenv()

from ai.intent.analyzer import IntentAnalyzer

analyzer = IntentAnalyzer(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url=os.getenv("DEEPSEEK_BASE_URL")
)

# 模拟模糊输入
result = analyzer.analyze("test_session", [
    {"role": "user", "content": "我想备一节Python列表的课"}
])

print(f"is_complete: {result.is_complete}")
print(f"teaching_goal: {result.teaching_goal}")
print(f"missing_info: {result.missing_info}")
print(f"follow_up: {result.follow_up_question}")

# 预期：is_complete=False，AI 追问授课对象、课时等信息
```

```bash
poetry run python ai/test_intent.py
```

---

### Day 3 — RAG 检索链路

**任务 3.1：`loader.py` + `splitter.py`**

```python
# ai/rag/loader.py
import os
from pathlib import Path

def load_documents(kb_dir: str) -> list[dict]:
    """加载 knowledge-base/ 下所有文档"""
    docs = []
    kb_path = Path(kb_dir)

    for file_path in kb_path.rglob("*.pdf"):
        from gen.parse.pdf import extract_pdf_text  # 调 D 的 PDF 解析
        text = extract_pdf_text(str(file_path))
        docs.append({"content": text, "source": file_path.name, "type": "pdf"})

    for file_path in kb_path.rglob("*.txt"):
        text = file_path.read_text(encoding="utf-8")
        docs.append({"content": text, "source": file_path.name, "type": "txt"})

    for file_path in kb_path.rglob("*.docx"):
        from gen.parse.doc import extract_doc_text    # 调 D 的 Word 解析
        text = extract_doc_text(str(file_path))
        docs.append({"content": text, "source": file_path.name, "type": "docx"})

    return docs
```

```python
# ai/rag/splitter.py
from langchain.text_splitter import RecursiveCharacterTextSplitter

def split_documents(docs: list[dict], chunk_size: int = 500, overlap: int = 50) -> list[dict]:
    """中文友好的文档切分"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=overlap,
        separators=["\n\n", "\n", "。", "！", "？", "；", "，", " ", ""]
    )

    chunks = []
    for doc in docs:
        texts = splitter.split_text(doc["content"])
        for i, t in enumerate(texts):
            chunks.append({"content": t, "source": doc["source"], "chunk_index": i})
    return chunks
```

**任务 3.2：`embedder.py` + `retriever.py`**

```python
# ai/rag/embedder.py
import chromadb
from chromadb.config import Settings

def build_index(chunks: list[dict], persist_dir: str, collection_name: str = "knowledge_base"):
    """将文档块向量化并写入 ChromaDB"""
    client = chromadb.PersistentClient(path=persist_dir, settings=Settings(anonymized_telemetry=False))
    collection = client.get_or_create_collection(collection_name)

    # 注意：ChromaDB 内置 embedding 支持，或用 text2vec 手动向量化
    # 这里用 ChromaDB 默认的 sentence-transformers
    for i, chunk in enumerate(chunks):
        collection.add(
            documents=[chunk["content"]],
            metadatas=[{"source": chunk["source"], "chunk_index": chunk["chunk_index"]}],
            ids=[f"chunk_{i}"]
        )

    print(f"已入库 {len(chunks)} 个文档块")
    return collection
```

```python
# ai/rag/retriever.py
import chromadb
from backend.schemas import RAGDocument

class RAGRetriever:
    """RAG 检索器"""

    def __init__(self, chroma_persist_dir: str, collection_name: str = "knowledge_base"):
        self.client = chromadb.PersistentClient(path=chroma_persist_dir)
        self.collection = self.client.get_or_create_collection(collection_name)

    def search(self, query: str, top_k: int = 5) -> list[RAGDocument]:
        results = self.collection.query(query_texts=[query], n_results=top_k)
        docs = []
        if results["documents"] and results["documents"][0]:
            for i, content in enumerate(results["documents"][0]):
                docs.append(RAGDocument(
                    content=content,
                    source=results["metadatas"][0][i].get("source", "unknown"),
                    score=1.0 - results["distances"][0][i] if results.get("distances") else 0.0
                ))
        return docs
```

---

## Phase 2：核心开发（Day 4-12）

### 开发顺序

```
Day 4-5: 完善 RAG → 写 build_kb.py → 让 E 能跑通知识库入库
Day 6-7: 完善意图分析 → 对话状态机 → A 联调 SSE
Day 8:   完善语音识别 → transcriber.py
Day 9-10: 知识融合 → fusion.py → 联调 D 的解析函数
Day 11-12: 全链路联调 + Prompt 调优
```

### `fusion.py` 关键代码

```python
# ai/fusion/fusion.py
import json, os
from openai import OpenAI
from backend.schemas import IntentResult, RAGDocument, ReferenceMaterial, GenerationInstruction
from gen.parse.pdf import extract_pdf_text      # ← D 的函数
from gen.parse.doc import extract_doc_text      # ← D 的函数
from gen.parse.video import summarize_video     # ← D 的函数
from gen.parse.image import describe_image      # ← D 的函数

class KnowledgeFuser:
    def __init__(self, api_key: str, base_url: str):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        base = os.path.dirname(os.path.dirname(__file__))
        with open(f"{base}/prompts/fusion.txt", encoding="utf-8") as f:
            self.prompt_fusion = f.read()

    def fuse(
        self,
        intent: IntentResult,
        rag_docs: list[RAGDocument],
        references: list[ReferenceMaterial]
    ) -> GenerationInstruction:
        # 组装 prompt
        rag_text = "\n---\n".join([d.content for d in rag_docs])
        ref_text = "\n---\n".join([r.extracted_text for r in references])

        prompt = self.prompt_fusion.format(
            intent_json=intent.model_dump_json(indent=2),
            rag_context=rag_text,
            reference_texts=ref_text
        )

        resp = self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=2000,
            response_format={"type": "json_object"}
        )

        data = json.loads(resp.choices[0].message.content)
        # 把 LLM 输出的 dict 转成 Pydantic Model
        return GenerationInstruction(**data)
```

### `build_kb.py` — 给 E 用的知识库构建脚本

```python
# ai/build_kb.py
"""知识库构建脚本。E 只需运行: poetry run python ai/build_kb.py"""
from ai.rag.loader import load_documents
from ai.rag.splitter import split_documents
from ai.rag.embedder import build_index
import os

KB_DIR = os.getenv("KB_DIR", "./knowledge-base")
CHROMA_DIR = os.getenv("CHROMA_PERSIST_DIR", "./chroma_data")

print("1/3 加载文档...")
docs = load_documents(KB_DIR)
print(f"  已加载 {len(docs)} 个文档")

print("2/3 切分文档...")
chunks = split_documents(docs)
print(f"  已切分 {len(chunks)} 个块")

print("3/3 向量化入库...")
build_index(chunks, CHROMA_DIR)
print("✅ 知识库构建完成！")
```

---

## Phase 3-4：联调 + 交付（Day 10-19）

### 与 A 联调的 Checklist

```
□ IntentAnalyzer.analyze() 能在 A 的 chat.py 中被正确调用
□ RAGRetriever.search() 能返回非空结果
□ KnowledgeFuser.fuse() 能生成符合 GenerationInstruction schema 的输出
□ SpeechTranscriber.transcribe() 能处理 A 传来的音频文件
□ A 的 orchestrator.py 完整链路不报错
```

### 你需要交给其他人的东西

| 交给谁 | 是什么 | 何时 |
|--------|--------|------|
| A | `IntentAnalyzer` 类（`ai/intent/analyzer.py`） | Day 3 晚（原型） → Day 7 晚（完善版） |
| A | `RAGRetriever` 类（`ai/rag/retriever.py`） | Day 5 晚 |
| A | `KnowledgeFuser` 类（`ai/fusion/fusion.py`） | Day 10 晚 |
| A | `SpeechTranscriber` 类（`ai/speech/transcriber.py`） | Day 8 晚 |
| E | `build_kb.py` + 知识库入库流程文档 | Day 5 晚 |
| E（归档） | Prompt 设计文档 | Day 19 |

### 你需要从谁那里拿到什么

| 从谁 | 是什么 | 何时 |
|------|--------|------|
| A | `backend/schemas.py`（Pydantic 模型） | Day 2 晚 |
| A | `.env` 中的 API Key 配置 | Day 1 |
| D | `gen/parse/pdf.py` 的 `extract_pdf_text` | Day 3 晚 |
| D | `gen/parse/doc.py` 的 `extract_doc_text` | Day 3 晚 |
| D | `gen/parse/video.py` 的 `summarize_video` | Day 3 晚 |
| D | `gen/parse/image.py` 的 `describe_image` | Day 3 晚 |
| E | 整理好的知识库资料 | Day 3 晚 |

---

## 今天（Day 1）立刻做这三件事

**① 注册 DeepSeek 开放平台 + 申请 Key + 充值 ¥20**

打开 platform.deepseek.com → 注册 → API Keys → 创建 Key → 充值 ¥20（支付宝/微信）。
复制 Key，告诉 A 填入 `.env` 文件。然后跑 `ai/test_llm.py` 验证调用成功，截图发群。

**② 安装 Python 依赖 + 创建目录**

```bash
cd "C:\Users\kaipie\Desktop\省服务外包大赛\多模态AI教学系统"
poetry install
mkdir -p ai/intent ai/rag ai/fusion ai/speech ai/prompts
```

跑 `ai/test_chroma.py` 验证 ChromaDB 可用，截图发群。

**③ 找一本教材 PDF**

从网上找一本大学计算机或 Python 教材 PDF（你后续测试意图分析和 RAG 的素材），存到 `knowledge-base/教材PDF/` 下。同时在群文件里发一份给 E 归档。
