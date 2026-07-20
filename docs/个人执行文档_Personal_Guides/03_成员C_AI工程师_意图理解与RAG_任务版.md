# 成员C · AI工程师（意图理解 + RAG + 知识融合 + 语音）— 任务说明

> **你的职责：** 系统"大脑"。四个模块：① 从多轮对话中提取结构化教学意图并主动追问 ② 知识库文档向量化 + 语义检索 ③ 融合"意图+检索结果+参考资料"为课件生成指令 ④ 语音转文字。
>
> **代码获取：** 具体代码片段见 `docs/08_代码速查手册_Code_Cheatsheet.md` 第三节（C · AI 代码）。
>
> **关键约束：** 你的代码被 A 直接 import（不是 HTTP 调用），因此类名、方法名、参数签名必须严格按协议文档第 4.1 节。

---

## 你需要创建的全部文件

```
ai/
├── intent/
│   ├── analyzer.py          ← IntentAnalyzer 类：analyze() / lock_intent()
│   └── state.py             ← 对话状态机：init → probing → confirming → locked
├── rag/
│   ├── loader.py            ← 遍历 knowledge-base/ 加载全部文档
│   ├── splitter.py          ← 中文友好的文档切分（chunk_size=500, overlap=50）
│   ├── embedder.py          ← 向量化 → 写入 ChromaDB
│   └── retriever.py         ← RAGRetriever 类：search(query, top_k)
├── fusion/
│   └── fusion.py            ← KnowledgeFuser 类：fuse(intent, rag_docs, refs) → GenerationInstruction
├── speech/
│   └── transcriber.py       ← SpeechTranscriber 类：transcribe(audio_path) → str
├── prompts/
│   ├── intent_extract.txt   ← 意图提取 Prompt
│   ├── follow_up.txt        ← 追问生成 Prompt
│   ├── confirm.txt          ← 确认总结 Prompt
│   └── fusion.txt           ← 知识融合 Prompt
├── build_kb.py              ← 一键知识库构建脚本（E 也会用）
├── test_llm.py              ← 验证 DeepSeek API 可调用
├── test_chroma.py           ← 验证 ChromaDB 可用
└── test_intent.py           ← 验证意图分析链路
```

---

## Phase 1：环境搭建与基础验证（Day 1-3）

---

### Day 1 — API Key + 依赖验证

#### 任务 1.1：申请 DeepSeek API Key

**要求：**
- 注册 platform.deepseek.com → API Keys → 创建 Key
- 充值 ¥20（整个项目够用）
- 把 Key 发给 A 填入 `.env` 文件
- 写 `test_llm.py`：调 `deepseek-chat` 模型，发送 "请用一句话解释什么是面向对象编程"，打印返回结果

**验证：** 控制台输出一段正常的中文回复，截图发群。

---

#### 任务 1.2：验证 ChromaDB

**要求：**
- 写 `test_chroma.py`：创建 PersistentClient → 创建 collection → 写入一条测试文档 → 用 "Python是什么" 检索 → 打印结果
- 检查 `./chroma_data` 目录是否生成

**验证：** 检索结果包含写入的测试文档内容，截图发群。

---

#### 任务 1.3：创建目录结构 + 获取测试素材

**要求：**
- 创建 `ai/intent/` / `ai/rag/` / `ai/fusion/` / `ai/speech/` / `ai/prompts/` 全部子目录
- 每个目录下创建 `__init__.py`
- 下载至少 1 本大学计算机或 Python 教材 PDF，放入 `knowledge-base/教材PDF/`，同时给 E 发一份归档

---

### Day 2 — Prompt 模板 + 意图分析器

#### 任务 2.1：写 4 个 Prompt 模板

**要求：**

**`intent_extract.txt`：**
- 系统提示：你是一个专业的教学设计师
- 输入：对话历史 `{messages}`
- 输出要求：严格 JSON，字段包括 teaching_goal / target_audience / duration_minutes / knowledge_points（含 order/title/difficulty/key_points/examples/estimated_minutes）/ logic_flow / style_preference / missing_info / is_complete
- 关键：missing_info 非空时 is_complete=false，空时 is_complete=true

**`follow_up.txt`：**
- 输入：当前意图 `{current_intent}` + 缺失信息 `{missing_info}`
- 输出：question_text（追问文本）+ options（2-4个选项数组）+ allow_free（true）
- 关键：每次只问 1-2 个点，选项要具体可操作

**`confirm.txt`：**
- 输入：意图 JSON `{intent_json}`
- 输出：summary（格式化确认文本，包含课程名/对象/时长/知识点列表/风格/流程）

**`fusion.txt`：**
- 输入：意图 JSON `{intent_json}` + RAG 检索结果 `{rag_context}` + 参考资料 `{reference_texts}`
- 输出：完整的 GenerationInstruction JSON（字段按协议文档 2.2 节）
- 关键：要融入参考资料中的案例和格式，补充知识库相关内容

---

#### 任务 2.2：写 IntentAnalyzer 类

**要求：** `ai/intent/analyzer.py`

核心类 `IntentAnalyzer`：
- `__init__(api_key, base_url)`：初始化 DeepSeek 客户端 + 加载 3 个 Prompt 模板
- `analyze(session_id, messages, uploaded_files=[])` → `IntentResult`：
  - 格式化对话历史 → 填入 intent_extract Prompt → 调 LLM → 解析 JSON
  - `is_complete=False` → 调 `_gen_follow_up()` 生成追问 → 填充 `follow_up_question`
  - `is_complete=True` → 调 `_gen_confirm()` 生成确认 → 填充 `confirm_summary`
  - 保存对话状态到 `self.sessions[session_id]`
- `lock_intent(session_id)` → `IntentResult`：教师确认后锁定意图
- `_gen_follow_up(intent_dict)` → dict：调 follow_up Prompt
- `_gen_confirm(intent_dict)` → dict：调 confirm Prompt

**LLM 调用参数：**
- `model="deepseek-chat"`
- `max_tokens=1000`（意图提取）/ `300`（追问）/ `500`（确认）
- `response_format={"type": "json_object"}` ← 确保 LLM 输出合法 JSON

**关键约束：**
- 返回类型必须是 `backend.schemas.IntentResult`（Pydantic Model）
- 类名和方法签名必须和协议文档 4.1 节一致（A 要 import）

---

#### 任务 2.3：写状态机 + 自测

**要求：** `ai/intent/state.py`
- 定义 4 个状态：`init`（初始）→ `probing`（追问中）→ `confirming`（等待确认）→ `locked`（已锁定）
- 状态转移逻辑：首次 analyze → probing；is_complete=True → confirming；lock_intent → locked

**自测：** 写 `test_intent.py`，模拟输入 "我想备一节Python列表的课"，验证：
1. 返回 `is_complete=False`
2. `missing_info` 非空（缺少授课对象、课时等信息）
3. `follow_up_question` 不为空且包含合理选项

---

### Day 3 — RAG 检索链路 + 知识库构建脚本

#### 任务 3.1：文档加载与切分

**要求：**

**`loader.py`：** `load_documents(kb_dir)` → 遍历 `knowledge-base/` 下的 `.pdf` / `.txt` / `.docx` → 调 D 的 `extract_pdf_text` / `extract_doc_text` → 返回 `[{"content": ..., "source": ..., "type": ...}, ...]`

**`splitter.py`：** 用 LangChain `RecursiveCharacterTextSplitter`：
- `chunk_size=500`，`chunk_overlap=50`
- 分隔符顺序：`\n\n` → `\n` → `。` → `！` → `？` → `；` → `，` → ` ` → `""`（中文友好）
- 返回 `[{"content": ..., "source": ..., "chunk_index": ...}, ...]`

---

#### 任务 3.2：向量化与检索

**要求：**

**`embedder.py`：** `build_index(chunks, persist_dir, collection_name="knowledge_base")` → ChromaDB 入库：
- 每个 chunk 的 id 为 `chunk_{i}`
- metadata 记录 source 和 chunk_index
- 使用 ChromaDB 内置 embedding（默认 sentence-transformers）

**`retriever.py`：** `RAGRetriever` 类：
- `__init__(chroma_persist_dir, collection_name)` → 连接 ChromaDB
- `search(query, top_k=5)` → `list[RAGDocument]`
- 返回的 `RAGDocument` 包含 content / source / score（如果 ChromaDB 返回 distances，score = 1 - distance）

---

#### 任务 3.3：知识库构建脚本

**要求：** `ai/build_kb.py` → 一键执行 `load → split → embed` 全流程：
- 打印每步进度（"1/3 加载文档…"、"2/3 切分文档…"、"3/3 向量化入库…"）
- 打印最终统计（文档数 / 块数）
- E 能独立运行：`poetry run python ai/build_kb.py`

**验证：** 用 E 整理的 `knowledge-base/` 目录跑一遍 → 检索 3 个教学相关查询 → top3 结果应与查询相关。

---

## Phase 2：核心开发与联调（Day 4-12）

---

### Day 4 — 意图分析完善

**要求：**
- `analyzer.py`：增加对话历史上下文管理（session_id → messages 映射），每次 analyze 时自动拼接历史
- `state.py`：完善状态转移逻辑 + 增加错误状态处理（LLM 调用失败时返回错误提示，不影响对话继续）
- `intent_extract.txt`：加入 3 个 few-shot 示例（Python 入门课 / 高等数学 / 大学英语），每个示例展示从模糊输入到完整意图的过程
- 测试 3 轮完整对话：模糊输入 → AI 追问 → 教师回答 → AI 再追问/确认

---

### Day 5 — 语音识别 + 配合 A 联调 SSE（关键日）

#### 任务 5.1：语音识别

**要求：** `ai/speech/transcriber.py`
- `SpeechTranscriber` 类：`__init__(model_size="small")` → 加载 faster-whisper small 模型
- `transcribe(audio_path)` → `str`：调 `model.transcribe(audio_path, language="zh")` → 拼接所有 segment.text
- 首次运行时模型会自动下载（~1GB），确认能正常下载完成

**验证：** 录一段 5 秒中文音频（"我想备一节Python入门课"）→ 转文字 → 准确率 > 90%。

---

#### 任务 5.2：配合 A 联调 SSE（重点）

**要求：**
- A 的 `routers/chat.py` 会调你的 `analyzer.analyze()`
- 如果 A 调用时报错 → 5 分钟内响应排查
- 常见问题：① import 路径不对（确认 `backend.schemas` 在 Python path 中）② 返回类型不匹配（确认返回的是 Pydantic Model 不是 dict）③ Prompt 模板路径不对（用 `os.path.dirname` 相对路径）

---

#### 任务 5.3：优化追问 Prompt

**要求：**
- 追问选项必须与教学场景相关（不要出现"你喜欢什么颜色"这类无关追问）
- 每次追问控制在 1-2 个问题点
- 语气友好但不啰嗦

---

### Day 6 — 知识融合模块

#### 任务 6.1：写 KnowledgeFuser 类

**要求：** `ai/fusion/fusion.py`
- `KnowledgeFuser` 类：`__init__(api_key, base_url)` → 加载 fusion Prompt
- `fuse(intent: IntentResult, rag_docs: list[RAGDocument], references: list[ReferenceMaterial])` → `GenerationInstruction`：
  - 组装 Prompt：意图 JSON + RAG 文本拼接 + 参考资料文本拼接
  - 调 LLM → 解析 JSON → 构造 `GenerationInstruction` Pydantic Model 返回
- LLM 参数：`max_tokens=2000`，`response_format={"type": "json_object"}`

**验证：** 用假 IntentResult + 真实 RAG 检索结果 → 生成 GenerationInstruction → 字段完整无缺失。

---

#### 任务 6.2：知识库更新

**要求：**
- 跑 `build_kb.py` 更新知识库（加入 E 新收集的资料）
- 验证新资料能被检索到

---

### Day 7 — 融合模块完善 + orchestrator 联调

#### 任务 7.1：融合中加入参考资料解析

**要求：**
- 在 `fusion.py` 中新增 `_parse_reference(file_path, file_type)` 辅助方法
- 根据 file_type 分发到 D 的不同解析函数：
  - pdf → `extract_pdf_text`
  - word → `extract_doc_text`
  - video → `summarize_video`
  - image → `describe_image`
- 解析结果封装为 `ReferenceMaterial` → 传给 `fuse()`

---

#### 任务 7.2：意图修改支持

**要求：**
- `analyzer.py` 新增 `update_intent(session_id, modification_text)` 方法
- 教师在 confirm 后说 "修改课时为 90 分钟"、"增加一个知识点" → analyzer 更新对应 IntentResult 字段
- 更新后重新判断 is_complete → 如果信息完整则再次生成确认总结

---

#### 任务 7.3：配合 A 调试 orchestrator

**要求：**
- A 的 orchestrator Step 1-4 会调你的 `lock_intent()` / `RAGRetriever.search()` / `KnowledgeFuser.fuse()`
- 确保三个函数在 A 的环境下不报错
- 如果 import 报错 → 确认 `backend.schemas` 和 D 的 `gen.parse` 都可正常 import

---

### Day 8 — 全链路联调 + Prompt 调优

#### 任务 8.1：全链路联调

**要求：** 配合 A 跑通 orchestrator 完整链路（Step 1-6）：
- lock_intent → 返回正确的 IntentResult
- search → 返回 ≥ 3 条相关结果
- fuse → 返回字段完整的 GenerationInstruction

---

#### 任务 8.2：优化 fusion Prompt

**要求：**
- 根据 A 联调中 LLM 实际输出的 JSON 质量调优 Prompt
- 重点：确保 knowledge_points 数组完整、logic_flow 合理、style_preference 不丢失
- 如果 LLM 输出字段缺失 → Prompt 中增加 "必须包含以下所有字段" 的约束

---

#### 任务 8.3：模型预热 + 边缘案例收集

**要求：**
- `transcriber.py`：首次启动时预加载模型（放在 `__init__` 中），避免首次调用等待太久
- 收集边缘案例：教师输入极短（"函数"）、极长（粘贴整段教材）、切换主题（"算了，还是讲数据库吧"）、重复确认（"再确认一次"）→ 这些场景下 Prompt 如何处理

---

### Day 9 — LLM 调用健壮性 + Demo 准备

#### 任务 9.1：LLM 调用加强

**要求：**
- 所有 LLM 调用加 retry 机制：失败重试 3 次，每次间隔 2 秒
- 加 timeout：`timeout=30`（秒）
- 加异常捕获：网络错误 / JSON 解析错误 / Pydantic 验证错误 → 记录日志 → 返回友好的错误描述
- 重试 3 次仍失败 → 返回带有 `error` 字段的兜底结果，不要抛未捕获异常

---

#### 任务 9.2：Prompt 版本管理

**要求：**
- 在 `ai/prompts/` 下创建 `v1/` 子文件夹
- 把当前 Prompt 复制进去，后续迭代在根目录的 `.txt` 中修改
- `v1/` 文件夹作为备份，方便回滚

---

#### 任务 9.3：准备 Demo 对话

**要求：**
- 准备 2-3 个不同学科的演示场景（如 Python 入门 / 大学计算机基础 / 数据结构）
- 每个场景设计完整的对话路径：教师输入 → AI 追问 → 教师回答 → 确认 → 生成
- 确保这些预置对话在 LLM 下能稳定产出预期结果

---

### Day 10 — 语音长音频 + RAG 混合检索

#### 任务 10.1：语音长音频支持

**要求：**
- `transcriber.py` 新增分段处理：音频 > 30 秒 → 按 30 秒分段 → 逐段识别 → 合并结果
- 分段边界在静音处（VAD 检测，如无法实现则在固定间隔处）

---

#### 任务 10.2：RAG 混合检索

**要求：**
- 在 `retriever.py` 中实现混合检索：
  - 向量检索：ChromaDB query（语义相似度）
  - 关键词检索：用简单的 TF-IDF 或 BM25 对 chunks 做关键词匹配
  - 融合分数：`final_score = 0.7 × vector_score + 0.3 × keyword_score`
- 对比测试：纯向量检索 vs 混合检索，混合检索的 top3 相关性更高

---

#### 任务 10.3：RAG 检索延迟测试

**要求：**
- 准备 20 个教学相关查询
- 每个查询跑 5 次 → 记录平均响应时间
- 目标：单次检索 < 2 秒

---

### Day 11 — 知识库扩充 + Prompt 最终调优

#### 任务 11.1：知识库扩充

**要求：**
- 与 E 协作，入库 10+ 份新资料（教材章节 / 教案 / 试题 / 课件参考）
- 重新运行 `build_kb.py`
- 对比扩充前后检索效果（同一个查询的前后 top3 对比）

---

#### 任务 11.2：Prompt 最终版

**要求：**
- 根据实际测试中各 Prompt 的常见缺陷，针对性地加约束和 few-shot 示例
- 冻结 `v1.0` 版本 → 复制到 `ai/prompts/v1.0/`
- 写 Prompt 设计文档：每个 Prompt 的设计思路、输入输出、调优过程 → 给 E 归档

---

### Day 12 — Phase 2 收尾

**要求：**
1. 自查四个模块（intent / rag / fusion / speech）全部稳定
2. 写 `ai/README.md`：各模块用途、依赖、如何单独测试
3. 确保全部类能被 A 正常 import（让 A 跑一遍 import 确认）
4. 整理 Prompt 设计文档 → 给 E

---

## Phase 3：联调测试（Day 13-16）

### Day 13 — RAG 压力测试 + 边缘测试

**要求：**
- RAG 压力测试：20 个查询并发 → 记录平均/最大响应时间
- 意图分析边缘测试：10 种边缘输入（空输入 / 纯表情 / 英文 / 中英混合 / 超长输入 / 特殊字符 / SQL注入尝试 / 只有标点 / 重复语句 / 话题跳转）→ 记录每种情况的处理结果
- 修复测试中发现的任何异常

---

### Day 14-15 — 彩排配合

**要求：**
- 配合 E 的 Demo 彩排 → 确保对话流畅自然
- 调整 Prompt 中追问的语气和选项（根据彩排反馈）
- 准备备用方案：如果 DeepSeek API 在演示时不可用 → 准备离线 fallback 文本或预设回复
- 代码 freeze

---

### Day 16 — 最终检查

**要求：**
- 确认四个模块全部可被 A import
- 确认 `build_kb.py` 可重建知识库（删掉 `chroma_data/` → 重新运行 → 检索结果一致）
- 写 AI 模块交接文档（给 E 归档）

---

## Phase 4：文档与交付（Day 17-19）

### Day 17-18 — 提供文档素材

**要求：**
- 画 RAG 流程图（文档加载 → 切分 → 向量化 → 检索）
- 画意图理解流程图（对话 → 状态机 → 追问/确认 → 锁定）
- 写 Prompt 设计理念说明（为什么要分 extract/follow-up/confirm 三步）
- 审查 E 写的 AI 技术章节

---

### Day 19 — 最终检查

**要求：**
- 代码全部 push
- 知识库数据可复现确认
- LLM API Key 余额确认（演示前充值充足）
- 参与最终评审

---

## 依赖关系

### 你需要从谁那里拿到什么

| 从谁 | 什么 | 何时 | 用于什么 |
|------|------|------|---------|
| A | DeepSeek API Key（.env） | Day 1 | 所有 LLM 调用 |
| A | `backend/schemas.py`（Pydantic 模型） | Day 2 晚 | `IntentResult` / `RAGDocument` / `GenerationInstruction` 等类型 |
| D | `gen/parse/pdf.py` → `extract_pdf_text` | Day 3 晚 | `loader.py` 中加载 PDF |
| D | `gen/parse/doc.py` → `extract_doc_text` | Day 3 晚 | `loader.py` 中加载 Word |
| D | `gen/parse/video.py` → `summarize_video` | Day 3 晚 | `fusion.py` 中解析视频 |
| D | `gen/parse/image.py` → `describe_image` | Day 3 晚 | `fusion.py` 中解析图片 |
| E | `knowledge-base/` 目录（整理好的资料） | Day 3 晚 | 知识库构建 |

### 你需要交给谁什么

| 交给谁 | 什么 | 何时 |
|--------|------|------|
| A | `IntentAnalyzer` 类 | Day 2 晚（原型）→ Day 7 晚（完善） |
| A | `RAGRetriever` 类 | Day 5 晚 |
| A | `KnowledgeFuser` 类 | Day 10 晚 |
| A | `SpeechTranscriber` 类 | Day 8 晚 |
| E | `build_kb.py` | Day 5 晚 |
| E | Prompt 设计文档 | Day 19 |
