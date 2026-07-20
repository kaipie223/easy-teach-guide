# 陈澜 · AI工程师（意图理解 + RAG + 知识融合 + 语音）— 模块视角

> **你的定位：** 你是系统的"大脑"。教师说的话 → 你理解意图 → 你检索知识 → 你融合三路信息。
> **你是 M1（对话引导）、M3（知识库RAG）、M4（语音输入）三个模块的 Owner。**
> 本文档按 7 个功能模块列出你需要做的事情。

---

## 你在每个模块中的角色

| 模块 | 你的角色 | 关键度 | 目标日期 |
|------|---------|--------|---------|
| M1 会话与对话引导 | **Owner**：意图分析引擎 + Prompt 模板 + 对话状态机 | ⭐⭐⭐⭐⭐ | 8/2 |
| M2 文件上传与解析 | 无需直接参与（赵钰洁负责解析） | — | — |
| M3 知识库与RAG | **Owner**：RAG 全链路（加载→切分→向量化→检索） | ⭐⭐⭐⭐⭐ | 8/3 |
| M4 语音输入 | **Owner**：faster-whisper 语音转文字 | ⭐⭐⭐ | 8/2 |
| M5 课件智能生成 | 知识融合器（意图+RAG+参考 → 指令集） | ⭐⭐⭐⭐ | 8/8 |
| M6 预览与反馈 | 无需直接参与（潘卓然负责前端） | — | — |
| M7 导出下载 | 无需直接参与 | — | — |

---

## M1 — 会话与对话引导（你是 Owner！）

> **这是你最重要的模块。** 教师说的话 → 提取结构化意图 → 追问补全 → 确认锁定。

### 你需要交付的

| # | 交付物 | 文件 | 验收 |
|---|--------|------|------|
| 1 | 意图提取 Prompt | `ai/prompts/intent_extract.txt` | LLM 输出合法 JSON，字段完整 |
| 2 | 追问生成 Prompt | `ai/prompts/follow_up.txt` | 追问友好、选项合理 |
| 3 | 确认总结 Prompt | `ai/prompts/confirm.txt` | 总结格式清晰 |
| 4 | IntentAnalyzer 类 | `ai/intent/analyzer.py` | 姜文杰能 import 且 analyze() 返回 IntentResult |
| 5 | 对话状态机 | `ai/intent/state.py` | init → probing → confirming → locked |
| 6 | 意图修改支持 | `analyzer.py` 中 `update_intent()` | "修改课时为90分钟" → 更新字段 |

### 关键约定（姜文杰会这样调你）

```python
from ai.intent.analyzer import IntentAnalyzer

analyzer = IntentAnalyzer(api_key, base_url)

# 姜文杰在 Chat SSE 中调用：
result = analyzer.analyze(session_id, messages)
# result.is_complete=False → 姜文杰发 question 事件（追问）
# result.is_complete=True  → 姜文杰发 confirm 事件（确认总结）

# 姜文杰在 orchestrator 中调用：
intent = analyzer.lock_intent(session_id)  # 锁定意图
```

### Done 标准（Owner 验收，目标：8/2）

- [ ] 输入 "我想备一节Python列表的课" → is_complete=False，missing_info 非空
- [ ] follow_up_question 不为空，选项与教学场景相关
- [ ] 补全信息后 → is_complete=True，confirm_summary 格式清晰
- [ ] 姜文杰的 Chat SSE 正常 import 且不报错
- [ ] 3 轮对话 → 意图锁定 → IntentResult 字段完整

---

## M3 — 知识库与RAG（你是 Owner！）

> **这是你 Owner 的第二个模块。** 把陶克钦收集的资料向量化 → 检索时返回最相关内容。

### 你需要交付的

| # | 交付物 | 文件 | 验收 |
|---|--------|------|------|
| 1 | 文档加载器 | `ai/rag/loader.py` | 遍历 knowledge-base/ → 加载全部文档 |
| 2 | 文本分割器 | `ai/rag/splitter.py` | chunk_size=500, overlap=50, 中文友好 |
| 3 | 向量化入库 | `ai/rag/embedder.py` | ChromaDB 入库，chroma_data/ 目录生成 |
| 4 | 检索器 RAGRetriever 类 | `ai/rag/retriever.py` | search(query, top_k) → list[RAGDocument] |
| 5 | 一键构建脚本 | `ai/build_kb.py` | 陶克钦能独立运行 |
| 6 | 混合检索（可选，8/10前） | `retriever.py` 升级 | 向量 0.7 + 关键词 0.3 |

### 关键约定（姜文杰会这样调你）

```python
from ai.rag.retriever import RAGRetriever

retriever = RAGRetriever(chroma_persist_dir)

# 姜文杰在 orchestrator 中调用：
docs = retriever.search("Python列表操作 基本语法", top_k=5)
# 返回 list[RAGDocument]，每个含 content / source / score
```

### 关键约定（你会这样调赵钰洁）

```python
# loader.py 中加载 PDF：
from gen.parse.pdf import extract_pdf_text
text = extract_pdf_text("knowledge-base/教材PDF/xxx.pdf")

# loader.py 中加载 Word：
from gen.parse.doc import extract_doc_text
text = extract_doc_text("knowledge-base/教案样例/xxx.docx")
```

### Done 标准（Owner 验收，目标：8/3）

- [ ] `poetry run python ai/build_kb.py` 跑通，陶克钦能独立执行
- [ ] search("Python列表常用方法", top_k=3) → 3 条结果与查询相关
- [ ] RAGRetriever 被姜文杰正常 import
- [ ] 5 个教学查询 × top3，Precision@3 ≥ 60%

---

## M4 — 语音输入（你是 Owner！）

> **这是你 Owner 的第三个模块。** 语音转文字，规模最小但独立完整。

### 你需要交付的

| # | 交付物 | 文件 | 验收 |
|---|--------|------|------|
| 1 | SpeechTranscriber 类 | `ai/speech/transcriber.py` | transcribe(audio_path) → 文字 |
| 2 | 长音频分段支持（8/10前） | 同上 | > 30 秒音频 → 分段 → 合并 |

### 关键约定（姜文杰会这样调你）

```python
from ai.speech.transcriber import SpeechTranscriber

transcriber = SpeechTranscriber(model_size="small")
text = transcriber.transcribe("temp_audio.wav")
# 返回: "我想备一节Python入门课"
```

### Done 标准（Owner 验收，目标：8/2）

- [ ] 5 秒中文音频 → 转写准确率 > 90%
- [ ] 姜文杰的 speech.py 正常 import 且不报错
- [ ] 首次运行时模型自动下载完成

---

## M5 — 课件智能生成（你不是 Owner，但是核心贡献者）

> Owner 是赵钰洁，但你提供**知识融合器**——这是三路信息交汇的核心节点。

### 你需要交付的

| # | 交付物 | 文件 | 验收 |
|---|--------|------|------|
| 1 | 融合 Prompt | `ai/prompts/fusion.txt` | LLM 输出完整 GenerationInstruction JSON |
| 2 | KnowledgeFuser 类 | `ai/fusion/fusion.py` | fuse(intent, rag_docs, refs) → GenerationInstruction |

### 关键约定（姜文杰会这样调你）

```python
from ai.fusion.fusion import KnowledgeFuser

fuser = KnowledgeFuser(api_key, base_url)

# 姜文杰在 orchestrator Step 4 中调用：
instruction = fuser.fuse(intent, rag_docs, references)
# 返回 GenerationInstruction → 直接传给赵钰洁的 gen_ppt / gen_doc / gen_animation
```

### 关键约定（fusion.py 会这样调赵钰洁）

```python
from gen.parse.pdf import extract_pdf_text
from gen.parse.doc import extract_doc_text
from gen.parse.video import summarize_video
from gen.parse.image import describe_image
```

### Done 标准（目标：8/8）

- [ ] Input: IntentResult + RAGDocument[] + ReferenceMaterial[] → 完整 GenerationInstruction
- [ ] GenerationInstruction 字段无缺失
- [ ] 姜文杰正常 import KnowledgeFuser 且 fuse() 不报错

---

## 你全部模块的交付物一览

```
ai/
├── intent/
│   ├── analyzer.py          ← M1：IntentAnalyzer 类（姜文杰 import）
│   └── state.py             ← M1：对话状态机
├── rag/
│   ├── loader.py            ← M3：文档加载（调赵钰洁的解析函数）
│   ├── splitter.py          ← M3：中文分块
│   ├── embedder.py          ← M3：ChromaDB 入库
│   └── retriever.py         ← M3：RAGRetriever 类（姜文杰 import）
├── fusion/
│   └── fusion.py            ← M5：KnowledgeFuser 类（姜文杰 import，调赵钰洁的解析函数）
├── speech/
│   └── transcriber.py       ← M4：SpeechTranscriber 类（姜文杰 import）
├── prompts/
│   ├── intent_extract.txt   ← M1：意图提取 Prompt
│   ├── follow_up.txt        ← M1：追问生成 Prompt
│   ├── confirm.txt          ← M1：确认总结 Prompt
│   └── fusion.txt           ← M5：知识融合 Prompt
├── build_kb.py              ← M3：一键知识库构建（陶克钦使用）
├── test_llm.py              ← M1：LLM 调用验证
├── test_chroma.py           ← M3：ChromaDB 验证
└── test_intent.py           ← M1：意图分析自测
```

---

## 你的开发时间线

```
7/22-7/24:  环境：API Key + ChromaDB + 目录 + 测试脚本
            M1：4 个 Prompt 模板 + IntentAnalyzer 初版
            M3：loader + splitter（等赵钰洁的解析函数）

7/25-7/29:  M3：embedder + retriever + build_kb.py（交陶克钦使用）
            M4：SpeechTranscriber + 语音测试

7/30-8/2:   M1：完善对话状态机 + few-shot 优化 Prompt
            联调姜文杰：确保 IntentAnalyzer 被正确调用
            M5：KnowledgeFuser + fusion Prompt

8/3-8/8:    M3：混合检索 + 知识库扩充
            M5：与姜文杰联调 orchestrator 全链路
            Prompt 调优 + 边缘案例处理

8/9-8/22:   联调测试 + 配合陶克钦验收 + 彩排
```

---

## 你需要从每个人那里拿到什么

| 从谁 | 需要什么 | 用到哪个模块 | 何时需要 |
|------|---------|-------------|---------|
| 姜文杰 | `backend/schemas.py`（Pydantic 模型） | M1/M3/M5 | 7/24 |
| 姜文杰 | `.env` 中的 DeepSeek API Key | M1/M4/M5 | 7/22 |
| 赵钰洁 | `gen/parse/pdf.py` → `extract_pdf_text` | M3（loader） | 7/28 |
| 赵钰洁 | `gen/parse/doc.py` → `extract_doc_text` | M3（loader） | 7/28 |
| 赵钰洁 | `gen/parse/video.py` → `summarize_video` | M5（fusion） | 8/5 |
| 赵钰洁 | `gen/parse/image.py` → `describe_image` | M5（fusion） | 8/5 |
| 陶克钦 | `knowledge-base/` 整理好的资料 | M3 | 7/28 |

## 你需要交给谁什么

| 交给谁 | 什么 | 何时 |
|--------|------|------|
| 姜文杰 | `IntentAnalyzer` 类（可 import） | 7/28（原型）→ 8/2（完善） |
| 姜文杰 | `RAGRetriever` 类（可 import） | 7/29 |
| 姜文杰 | `KnowledgeFuser` 类（可 import） | 8/8 |
| 姜文杰 | `SpeechTranscriber` 类（可 import） | 7/30 |
| 陶克钦 | `build_kb.py` + 使用说明 | 7/29 |
