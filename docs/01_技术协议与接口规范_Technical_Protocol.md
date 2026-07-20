# 01 — 技术协议与接口规范 · Technical Protocol

> **本文档是项目唯一的接口"合同"。所有成员必须遵守本文档定义的协议。**
> 任何接口变更必须更新本文档并通知所有下游依赖方。

---

## 〇、统一项目目录结构

> 每个成员必须在自己本地复刻这个目录结构。目录名不可自行修改。

```
多模态AI教学系统/                    # 项目根目录
│
├── CLAUDE.md                        # 项目记忆入口
├── 要求.md                          # 竞赛原始需求
│
├── docs/                            # 所有文档
│   ├── 00_总执行手册_Master_Plan.md
│   ├── 01_技术协议与接口规范_Technical_Protocol.md  ← 本文档
│   ├── test-plan.md
│   ├── test-cases.xlsx
│   └── 个人执行文档_Personal_Guides/
│       ├── 01_成员A_后端工程师.md
│       ├── 02_成员B_前端工程师.md
│       ├── 03_成员C_AI工程师_意图理解与RAG.md
│       ├── 04_成员D_AI工程师_课件生成与多模态解析.md
│       └── 05_成员E_PM_测试_数据.md
│
├── backend/                         # 成员A 工作区
│   ├── main.py                      # FastAPI 入口
│   ├── config.py                    # 全局配置
│   ├── schemas.py                   # 所有 Pydantic 模型（统一出口）
│   ├── models/                      # SQLAlchemy ORM 模型
│   │   ├── __init__.py
│   │   ├── session.py
│   │   ├── file.py
│   │   └── task.py
│   ├── routers/                     # API 路由
│   │   ├── __init__.py
│   │   ├── session.py
│   │   ├── file.py
│   │   ├── chat.py
│   │   ├── generate.py
│   │   └── download.py
│   ├── services/                    # 业务编排层
│   │   ├── __init__.py
│   │   ├── orchestrator.py          # 课件生成编排
│   │   └── file_mgr.py              # 文件管理
│   └── uploads/                     # 用户上传文件存储（.gitignore）
│
├── frontend/                        # 成员B 工作区
│   ├── public/
│   ├── src/
│   │   ├── main.js
│   │   ├── App.vue
│   │   ├── router/
│   │   │   └── index.js
│   │   ├── views/
│   │   │   ├── Home.vue
│   │   │   ├── Chat.vue
│   │   │   ├── Preview.vue
│   │   │   └── Download.vue
│   │   ├── components/
│   │   │   ├── chat/
│   │   │   │   ├── MessageBubble.vue
│   │   │   │   ├── ChatInput.vue
│   │   │   │   └── VoiceInput.vue
│   │   │   ├── upload/
│   │   │   │   └── FileUploader.vue
│   │   │   └── preview/
│   │   │       └── EditPanel.vue
│   │   ├── stores/                  # Pinia 状态
│   │   │   ├── session.js
│   │   │   └── task.js
│   │   ├── utils/
│   │   │   ├── request.js           # Axios 封装
│   │   │   └── sse.js               # SSE 流式接收
│   │   └── types/                   # TypeScript 接口定义
│   │       └── index.ts
│   ├── package.json
│   └── vite.config.js
│
├── ai/                              # 成员C 工作区（意图 + RAG + 语音）
│   ├── __init__.py
│   ├── intent/
│   │   ├── __init__.py
│   │   ├── analyzer.py              # 意图分析入口
│   │   └── state.py                 # 对话状态机
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── loader.py
│   │   ├── splitter.py
│   │   ├── embedder.py
│   │   └── retriever.py
│   ├── fusion/
│   │   ├── __init__.py
│   │   └── fusion.py
│   ├── speech/
│   │   ├── __init__.py
│   │   └── transcriber.py
│   ├── prompts/                     # Prompt 模板文件
│   │   ├── intent_extract.txt
│   │   ├── follow_up.txt
│   │   ├── confirm.txt
│   │   └── fusion.txt
│   └── build_kb.py                  # 知识库构建脚本（E 也会用）
│
├── gen/                             # 成员D 工作区（课件生成 + 多模态解析）
│   ├── __init__.py
│   ├── pptx/
│   │   ├── __init__.py
│   │   ├── generator.py
│   │   ├── slides.py
│   │   └── layout.py
│   ├── docx/
│   │   ├── __init__.py
│   │   ├── generator.py
│   │   └── sections.py
│   ├── anim/
│   │   ├── __init__.py
│   │   └── generator.py
│   ├── parse/
│   │   ├── __init__.py
│   │   ├── pdf.py
│   │   ├── doc.py
│   │   ├── image.py
│   │   └── video.py
│   └── prompts/
│       ├── ppt_outline.txt
│       ├── ppt_content.txt
│       ├── doc_plan.txt
│       └── anim_idea.txt
│
├── knowledge-base/                  # 成员E 维护的知识库原始资料
│   ├── 教材PDF/
│   ├── 教案样例/
│   ├── 课件参考/
│   ├── 教学视频/
│   └── 试题库/
│
├── .gitignore
└── README.md
```

---

## 一、环境与版本锁定

### 1.1 全局约束

| 工具 | 精确版本 | 检查命令 |
|------|---------|----------|
| Python | **3.11.9** | `python --version` |
| Node.js | **20.15.0 LTS** | `node --version` |
| Poetry | ≥ 1.8.0 | `poetry --version` |
| pnpm | ≥ 9.0.0 | `pnpm --version` |
| Git | ≥ 2.40 | `git --version` |
| FFmpeg | ≥ 6.0 | `ffmpeg -version` |

### 1.2 后端依赖（backend/ + ai/ + gen/ 共用同一个 Poetry 项目）

> **说明**：`backend/`、`ai/`、`gen/` 三个目录不是独立项目，而是同一个 Python 包的不同子包。Poetry 项目创建在项目根目录。

```toml
# 项目根目录/pyproject.toml (Poetry)
[tool.poetry]
name = "edu-agent"
version = "0.1.0"
description = "多模态AI互动式教学智能体"
python = "^3.11"

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "0.111.0"
uvicorn = { version = "0.30.0", extras = ["standard"] }
sqlalchemy = "2.0.30"
pydantic = "2.7.0"
python-multipart = "0.0.9"
python-dotenv = "1.0.1"

# 成员C 依赖
openai = "1.35.0"           # DeepSeek 兼容 OpenAI SDK
langchain = "0.3.0"
langchain-community = "0.3.0"
chromadb = "0.5.0"
faster-whisper = "1.0.3"
text2vec = "1.2.0"

# 成员D 依赖
python-pptx = "0.6.23"
python-docx = "1.1.2"
pymupdf = "1.24.5"
opencv-python = "4.9.0.80"
pillow = "10.3.0"
paddleocr = "2.8.0"

[tool.poetry.group.dev.dependencies]
pytest = "8.2.0"
httpx = "0.27.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### 1.3 前端依赖

```json
// frontend/package.json 关键依赖
{
  "dependencies": {
    "vue": "3.4.31",
    "vue-router": "4.4.0",
    "pinia": "2.1.7",
    "axios": "1.7.2",
    "element-plus": "2.7.6",
    "@element-plus/icons-vue": "2.3.1"
  },
  "devDependencies": {
    "vite": "5.3.3",
    "@vitejs/plugin-vue": "5.0.5"
  }
}
```

### 1.4 环境变量（根目录 .env）

```bash
# .env — 所有成员共用，A 创建后分发
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxx
DEEPSEEK_BASE_URL=https://api.deepseek.com
WHISPER_MODEL=small           # tiny / small / medium
CHROMA_PERSIST_DIR=./chroma_data
DATABASE_URL=sqlite:///./data.db
UPLOAD_DIR=./backend/uploads
OUTPUT_DIR=./output
```

---

## 二、核心数据模型（唯一的 Schema 定义）

> **这里定义的 Pydantic Model 是后端、AI 模块、前端之间的唯一数据合同。**
> 成员A 负责在 `backend/schemas.py` 中维护。成员C/D 的 Python 模块使用同一个 Model。
> 成员B 在 `frontend/src/types/index.ts` 中维护对应的 TypeScript 类型。

### 2.1 会话与消息

```python
# ===== backend/schemas.py =====
from pydantic import BaseModel
from datetime import datetime
from enum import Enum

# --- 会话 ---
class SessionCreate(BaseModel):
    """创建会话请求"""
    teacher_name: str = ""
    subject: str = ""

class SessionInfo(BaseModel):
    """会话信息"""
    session_id: str
    teacher_name: str
    subject: str
    status: str            # "active" | "completed"
    created_at: datetime

# --- 消息 ---
class MessageRole(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"

class MessageType(str, Enum):
    TEXT = "text"
    QUESTION = "question"        # AI 追问（前端渲染为选择卡片）
    CONFIRM = "confirm"          # AI 确认总结（前端渲染为确认面板）

class ChatRequest(BaseModel):
    """发送消息请求"""
    message: str                 # 文字内容

class ChatEvent(BaseModel):
    """SSE 推送的单个事件（JSON 序列化后通过 SSE data 字段传输）"""
    event_type: MessageType      # "text" | "question" | "confirm"
    content: str                 # 展示给用户的文字
    data: dict | None = None     # 结构化附加数据

# --- 文件 ---
class FileType(str, Enum):
    PDF = "pdf"
    WORD = "word"
    PPT = "ppt"
    IMAGE = "image"
    VIDEO = "video"

class FileInfo(BaseModel):
    """上传成功后返回"""
    file_id: str
    original_name: str
    file_type: FileType
    size_kb: float
    upload_time: datetime
    ref_description: str | None = None    # 教师备注：参考了哪个知识点/格式

# --- 任务 ---
class TaskStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

class GenerateRequest(BaseModel):
    """触发课件生成请求"""
    session_id: str

class TaskInfo(BaseModel):
    """任务状态查询返回"""
    task_id: str
    session_id: str
    status: TaskStatus
    progress: int              # 0-100
    outputs: list["OutputFile"] | None = None   # 完成后填充
    error: str | None = None

class OutputFile(BaseModel):
    """生成的课件文件描述"""
    file_id: str
    file_type: str             # "pptx" | "docx" | "html"
    file_name: str
    size_kb: float
    download_url: str          # /api/v1/download/{file_id}
```

对应的 TypeScript 定义：

```typescript
// ===== frontend/src/types/index.ts =====
export interface SessionCreate {
  teacher_name: string;
  subject: string;
}

export interface SessionInfo {
  session_id: string;
  teacher_name: string;
  subject: string;
  status: 'active' | 'completed';
  created_at: string;
}

export interface ChatRequest {
  message: string;
}

export interface ChatEvent {
  event_type: 'text' | 'question' | 'confirm';
  content: string;
  data?: Record<string, any> | null;
}

export type FileType = 'pdf' | 'word' | 'ppt' | 'image' | 'video';

export interface FileInfo {
  file_id: string;
  original_name: string;
  file_type: FileType;
  size_kb: number;
  upload_time: string;
  ref_description?: string | null;
}

export interface GenerateRequest {
  session_id: string;
}

export type TaskStatus = 'pending' | 'processing' | 'completed' | 'failed';

export interface TaskInfo {
  task_id: string;
  session_id: string;
  status: TaskStatus;
  progress: number;
  outputs?: OutputFile[] | null;
  error?: string | null;
}

export interface OutputFile {
  file_id: string;
  file_type: 'pptx' | 'docx' | 'html';
  file_name: string;
  size_kb: number;
  download_url: string;
}
```

### 2.2 意图理解与课件指令（C ↔ A ↔ D 的核心数据流）

```python
# ===== backend/schemas.py（续）=====

# --- 知识点 ---
class KnowledgePoint(BaseModel):
    order: int                       # 序号 (1, 2, 3...)
    title: str                       # 知识点标题
    difficulty: str                  # "basic" | "intermediate" | "advanced"
    key_points: list[str]            # 要点列表
    examples: list[str] = []         # 相关案例
    estimated_minutes: int = 5       # 预计讲解时长

# --- 意图分析结果（C 输出）---
class IntentResult(BaseModel):
    """成员C 的 intent/analyzer.py 输出此结构"""
    teaching_goal: str               # 一句话教学目标
    target_audience: str             # "大一新生" / "计算机专业大三" 等
    duration_minutes: int            # 课时长度
    knowledge_points: list[KnowledgePoint]
    logic_flow: list[str]            # 教学流程 ["导入", "新授", "练习", "总结"]
    style_preference: str            # "严谨学术" / "生动活泼" / "案例驱动"
    missing_info: list[str] = []     # 缺失的关键信息（非空就需要追问）
    follow_up_question: str | None = None  # 追问文本（missing_info 非空时）
    confirm_summary: str | None = None     # 确认总结（missing_info 为空时）
    is_complete: bool = False        # True 表示信息足够，可以确认

# --- RAG 检索结果 ---
class RAGDocument(BaseModel):
    content: str                     # 检索到的文本片段
    source: str                      # 来源文件
    score: float                     # 相似度 0-1

# --- 参考资料解析结果 ---
class ReferenceMaterial(BaseModel):
    file_id: str
    file_type: FileType
    extracted_text: str              # 解析后的纯文本
    key_topics: list[str]            # 涉及的知识点/主题
    format_notes: str | None = None  # 格式/风格参考说明

# --- 课件生成指令集（C → A → D 的核心产物）---
class GenerationInstruction(BaseModel):
    """这是整个 AI 管道的最终输出，直接驱动课件生成"""
    session_id: str
    teaching_goal: str
    target_audience: str
    duration_minutes: int
    knowledge_points: list[KnowledgePoint]
    logic_flow: list[str]
    style_preference: str
    rag_context: list[RAGDocument]          # RAG 检索到的补充材料
    reference_materials: list[ReferenceMaterial]  # 教师上传的资料解析结果
    extra_requirements: str = ""            # 教师的特殊要求
```

---

## 三、REST API 契约（A ↔ B 的唯一界面）

> 基准路径：`http://localhost:8000/api/v1`

### 3.1 会话管理

```
POST /api/v1/sessions
  请求体: { "teacher_name": "张老师", "subject": "Python程序设计" }
  响应 201: { "session_id": "s_abc123", "teacher_name": "张老师", "subject": "...", "status": "active", "created_at": "..." }
  错误 422: 请求体格式不符

GET /api/v1/sessions/{session_id}
  响应 200: SessionInfo
  错误 404: { "error_code": "SESSION_NOT_FOUND", "detail": "会话不存在" }

GET /api/v1/sessions
  响应 200: [SessionInfo, ...]   # 会话列表
```

### 3.2 对话（SSE 流式）

```
POST /api/v1/sessions/{session_id}/chat
  请求体: { "message": "我想备一节关于Python列表的课" }
  响应: text/event-stream (SSE)

  SSE 事件流格式（A 必须严格按照此格式发送，B 按此格式解析）:

  ── 当 AI 正常回复文字时 ──
  data: {"event_type":"text","content":"好的","data":null}

  data: {"event_type":"text","content":"我了解到您想准备一节关于Python列表的课程","data":null}

  data: {"event_type":"text","content":"为了更好地设计课件，我想确认几个信息：","data":null}

  ── 当 AI 需要追问时 ──
  data: {"event_type":"question","content":"请问授课对象是哪个年级？","data":{"options":["大一新生","大二/大三","研究生","在职培训"],"allow_free":true}}

  ── 当 AI 完成信息收集，输出确认总结时 ──
  data: {"event_type":"confirm","content":"课程名称：Python列表操作\n授课对象：大一新生\n课时：45分钟\n教学风格：案例驱动\n知识点：列表定义、索引、切片、常用方法\n\n请确认以上信息，或告诉我需要修改的地方。","data":null}

  ── 流结束 ──
  data: [DONE]

  重要约束:
  - A 后端从 C 的 IntentResult 构造 ChatEvent
  - is_complete=False 时 → 发送 question 事件
  - is_complete=True 时 → 发送 confirm 事件
  - 流过程中 B 收到的每个 data 行都是完整 JSON，B 逐条解析并渲染
  - [DONE] 表示该次对话处理完毕
```

### 3.3 文件上传

```
POST /api/v1/files/upload
  Content-Type: multipart/form-data
  字段:
    file: (binary)              # 文件本体
    ref_description: "参考第3章的例题格式"   # 可选

  支持格式: .pdf / .docx / .pptx / .jpg / .png / .mp4 / .avi
  大小限制: 视频 ≤ 500MB，其余 ≤ 50MB

  响应 201:
  {
    "file_id": "f_xyz789",
    "original_name": "Python教案.pdf",
    "file_type": "pdf",
    "size_kb": 2048.5,
    "upload_time": "2026-07-17T10:30:00",
    "ref_description": "参考第3章的例题格式"
  }

GET /api/v1/files/{file_id}
  响应 200: FileInfo
```

### 3.4 课件生成（异步）

```
POST /api/v1/generate
  请求体: { "session_id": "s_abc123" }
  响应 202: { "task_id": "task_001", "session_id": "s_abc123", "status": "pending", "progress": 0, "outputs": null, "error": null }

GET /api/v1/tasks/{task_id}/status
  响应 200:
  ── 处理中 ──
  { "task_id": "task_001", "session_id": "s_abc123", "status": "processing", "progress": 60, "outputs": null, "error": null }

  ── 完成 ──
  {
    "task_id": "task_001",
    "session_id": "s_abc123",
    "status": "completed",
    "progress": 100,
    "outputs": [
      { "file_id": "ppt_001", "file_type": "pptx", "file_name": "Python列表操作.pptx", "size_kb": 2048, "download_url": "/api/v1/download/ppt_001" },
      { "file_id": "doc_001", "file_type": "docx", "file_name": "Python列表操作_教案.docx", "size_kb": 512, "download_url": "/api/v1/download/doc_001" },
      { "file_id": "html_001", "file_type": "html", "file_name": "列表知识闯关.html", "size_kb": 128, "download_url": "/api/v1/download/html_001" }
    ],
    "error": null
  }

  ── 失败 ──
  { "task_id": "task_001", ..., "status": "failed", "progress": 30, "outputs": null, "error": "PPT生成失败：LLM返回格式异常" }

  B 的轮询策略:
  - 间隔 2 秒轮询一次
  - status="completed" 时停止轮询，展示下载链接
  - status="failed" 时显示错误信息 + 重试按钮
```

### 3.5 文件下载

```
GET /api/v1/download/{file_id}
  响应 200: 文件流
    Content-Type: application/vnd.openxmlformats-officedocument.presentationml.presentation  (pptx)
    Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document    (docx)
    Content-Type: text/html                                                                  (html)
    Content-Disposition: attachment; filename="Python列表操作.pptx"
```

### 3.6 语音识别

```
POST /api/v1/speech/transcribe
  Content-Type: multipart/form-data
  字段: audio (binary, .wav 或 .webm)
  响应 200: { "text": "我想备一节Python入门课" }
```

### 3.7 统一错误响应格式

```json
{
  "error_code": "FILE_TOO_LARGE",
  "detail": "文件大小 600MB 超过限制 500MB",
  "timestamp": "2026-07-17T10:30:00Z"
}
```

错误码枚举：`SESSION_NOT_FOUND` / `FILE_TOO_LARGE` / `UNSUPPORTED_FORMAT` / `TASK_NOT_FOUND` / `GENERATION_FAILED` / `INTERNAL_ERROR`

---

## 四、Python 模块间调用协议（C/D → A 的函数契约）

> **A 直接 import 使用 C 和 D 的函数，不走 HTTP。以下签名是强制约定。**

### 4.1 成员C 暴露给 A 的函数

```python
# ========================
# ai/intent/analyzer.py
# ========================
from backend.schemas import IntentResult

class IntentAnalyzer:
    """意图分析器 — A 在后端启动时初始化一次，保持单例"""

    def __init__(self, api_key: str, base_url: str):
        """初始化 DeepSeek 客户端"""
        ...

    def analyze(
        self,
        session_id: str,
        messages: list[dict],       # [{"role":"user","content":"..."}, {"role":"assistant","content":"..."}]
        uploaded_files: list[str] = []
    ) -> IntentResult:
        """
        分析当前对话，返回结构化意图。
        - is_complete=False → 使用 follow_up_question 追问
        - is_complete=True  → 使用 confirm_summary 让教师确认
        """
        ...

    def lock_intent(self, session_id: str) -> IntentResult:
        """教师确认后锁定意图，返回最终 IntentResult 供融合使用"""
        ...


# ========================
# ai/rag/retriever.py
# ========================
from backend.schemas import RAGDocument

class RAGRetriever:
    """RAG 检索器 — A 初始化一次"""

    def __init__(self, chroma_persist_dir: str, embedding_model: str = "text2vec-base-chinese"):
        ...

    def search(self, query: str, top_k: int = 5) -> list[RAGDocument]:
        """检索知识库，返回 top_k 个相关文档片段"""
        ...


# ========================
# ai/fusion/fusion.py
# ========================
from backend.schemas import IntentResult, RAGDocument, ReferenceMaterial, GenerationInstruction

class KnowledgeFuser:
    """知识融合器 — A 初始化一次"""

    def __init__(self, api_key: str, base_url: str):
        ...

    def fuse(
        self,
        intent: IntentResult,
        rag_docs: list[RAGDocument],
        references: list[ReferenceMaterial]
    ) -> GenerationInstruction:
        """
        融合三路信息（意图 + RAG + 参考资料）→ 课件生成指令集。
        这是生成课件的唯一输入。
        """
        ...


# ========================
# ai/speech/transcriber.py
# ========================
class SpeechTranscriber:
    """语音转文字 — A 初始化一次"""

    def __init__(self, model_size: str = "small"):
        ...

    def transcribe(self, audio_path: str) -> str:
        """输入音频文件路径，返回识别文字"""
        ...
```

### 4.2 成员D 暴露给 A 的函数

```python
# ========================
# gen/pptx/generator.py
# ========================
from backend.schemas import GenerationInstruction
from pathlib import Path

def gen_ppt(
    instruction: GenerationInstruction,
    template: str = "default",
    output_dir: str = "./output"
) -> Path:
    """
    输入指令集 → 生成 .pptx 文件 → 返回文件路径

    返回值示例: Path("./output/Python列表操作.pptx")
    生成的 PPT 结构:
      - 页面1: 封面（标题 + 副标题 + 教师信息）
      - 页面2: 目录（按 logic_flow 生成章节列表）
      - 页面3-N: 内容页（每个 KnowledgePoint 1-2 页）
      - 最后页: 总结（回顾所有 key_points）
    """
    ...


# ========================
# gen/docx/generator.py
# ========================
from backend.schemas import GenerationInstruction
from pathlib import Path

def gen_doc(
    instruction: GenerationInstruction,
    output_dir: str = "./output"
) -> Path:
    """
    输入指令集 → 生成 .docx 教案 → 返回文件路径

    教案结构:
      1. 教学目标（三维目标）
      2. 教学重难点
      3. 教学过程（按 logic_flow 详细展开）
      4. 教学方法
      5. 课堂活动设计
      6. 课后作业与评价
    """
    ...


# ========================
# gen/anim/generator.py
# ========================
from backend.schemas import GenerationInstruction
from pathlib import Path

def gen_animation(
    instruction: GenerationInstruction,
    anim_type: str = "quiz_game",     # "quiz_game" | "concept_anim"
    output_dir: str = "./output"
) -> Path:
    """
    生成 HTML5 互动内容 → 返回 .html 文件路径
    """
    ...


# ========================
# gen/parse/pdf.py
# ========================
def extract_pdf_text(pdf_path: str) -> str:
    """提取 PDF 纯文本，保留段落结构"""
    ...

def extract_pdf_images(pdf_path: str, output_dir: str) -> list[str]:
    """提取 PDF 中所有图片，返回图片路径列表"""
    ...


# ========================
# gen/parse/doc.py
# ========================
def extract_doc_text(docx_path: str) -> str:
    """提取 Word 文档文本和标题结构"""
    ...


# ========================
# gen/parse/image.py
# ========================
def describe_image(image_path: str, api_key: str) -> str:
    """调用 DeepSeek Vision 描述图片内容"""
    ...


# ========================
# gen/parse/video.py
# ========================
def extract_keyframes(video_path: str, interval_sec: int = 10, output_dir: str = "./frames") -> list[str]:
    """每 interval_sec 秒提取一帧，返回帧图片路径列表"""
    ...

def summarize_video(video_path: str, api_key: str) -> str:
    """提取关键帧 → 描述帧 → 汇总生成视频摘要"""
    ...
```

### 4.3 成员C 调用成员D 的协议

```python
# C 的 fusion/fusion.py 中需要解析教师上传的参考资料
# C 直接 import D 的解析函数：

from gen.parse.pdf import extract_pdf_text
from gen.parse.doc import extract_doc_text
from gen.parse.image import describe_image
from gen.parse.video import summarize_video

# 使用方式（在 fuse() 内部）:
def _parse_reference(self, file_path: str, file_type: str) -> ReferenceMaterial:
    if file_type == "pdf":
        text = extract_pdf_text(file_path)
    elif file_type == "word":
        text = extract_doc_text(file_path)
    elif file_type == "video":
        text = summarize_video(file_path, self.api_key)
    # ... 封装为 ReferenceMaterial 返回
```

---

## 五、A 的服务编排协议（关键！）

> `backend/services/orchestrator.py` 中的 `run_generation()` 是整个系统的核心调度函数。
> 它串联了 C 和 D 的全部能力。A 必须严格按照以下顺序调用。

```python
# backend/services/orchestrator.py

from backend.schemas import *
from ai.intent.analyzer import IntentAnalyzer
from ai.rag.retriever import RAGRetriever
from ai.fusion.fusion import KnowledgeFuser
from gen.pptx.generator import gen_ppt
from gen.docx.generator import gen_doc
from gen.anim.generator import gen_animation
from gen.parse.pdf import extract_pdf_text
from gen.parse.video import summarize_video
from gen.parse.doc import extract_doc_text

class Orchestrator:
    def __init__(self, config):
        self.intent_analyzer = IntentAnalyzer(config.deepseek_key, config.deepseek_url)
        self.rag_retriever = RAGRetriever(config.chroma_dir)
        self.fuser = KnowledgeFuser(config.deepseek_key, config.deepseek_url)

    def run_generation(self, session_id: str) -> TaskInfo:
        """课件生成全流程编排"""

        # Step 1: 锁定意图（C 已通过对话完成意图收集）
        intent: IntentResult = self.intent_analyzer.lock_intent(session_id)

        # Step 2: RAG 检索（C）
        query = f"{intent.teaching_goal} {' '.join(kp.title for kp in intent.knowledge_points)}"
        rag_docs: list[RAGDocument] = self.rag_retriever.search(query, top_k=5)

        # Step 3: 解析参考资料（C 调 D）
        references: list[ReferenceMaterial] = []
        for f in session_files(session_id):  # 该会话已上传的文件
            if f.file_type == FileType.PDF:
                text = extract_pdf_text(f.path)
            elif f.file_type == FileType.WORD:
                text = extract_doc_text(f.path)
            elif f.file_type == FileType.VIDEO:
                text = summarize_video(f.path, self.config.deepseek_key)
            else:
                continue
            references.append(ReferenceMaterial(
                file_id=f.file_id,
                file_type=f.file_type,
                extracted_text=text,
                key_topics=[],
                format_notes=f.ref_description
            ))

        # Step 4: 知识融合（C）
        instruction: GenerationInstruction = self.fuser.fuse(intent, rag_docs, references)

        # Step 5: 课件生成（D）
        pptx_path = gen_ppt(instruction, output_dir=self.config.output_dir)
        docx_path = gen_doc(instruction, output_dir=self.config.output_dir)
        html_path = gen_animation(instruction, output_dir=self.config.output_dir)

        # Step 6: 注册输出文件，生成下载链接
        return self._build_task_result(pptx_path, docx_path, html_path)
```

---

## 六、角色间数据流全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│  教师 (浏览器)                                                        │
│  │                                                                    │
│  ▼                                                                    │
│ ┌──────────────────────────────────────────────────────────────┐     │
│ │ 成员B · 前端 (Vue 3)                                          │     │
│ │                                                                │     │
│ │  Chat.vue ──SSE──▶ 解析 ChatEvent ──▶ 渲染气泡/追问卡片/确认面板 │     │
│ │  FileUploader.vue ──multipart──▶ 上传文件                       │     │
│ │  Preview.vue ──轮询──▶ GET /tasks/{id}/status                   │     │
│ │  Download.vue ──▶ GET /download/{file_id}                       │     │
│ └──────────────────────────┬───────────────────────────────────┘     │
│                            │ RESTful API (JSON + SSE + multipart)     │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────────┐
│  成员A · 后端 (FastAPI)                                                │
│                                                                        │
│  routers/chat.py ──▶ services/orchestrator.py                          │
│  routers/generate.py ──▶ orchestrator.run_generation()                 │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │ orchestrator.run_generation() 调度顺序:                        │     │
│  │                                                                │     │
│  │  1. intent_analyzer.lock_intent(session_id)                    │     │
│  │                          │                                     │     │
│  │                          ▼                                     │     │
│  │  2. rag_retriever.search(query)    ◀── 成员C 提供              │     │
│  │                          │                                     │     │
│  │                          ▼                                     │     │
│  │  3. parse_pdf / summarize_video    ◀── 成员D 提供              │     │
│  │                          │                                     │     │
│  │                          ▼                                     │     │
│  │  4. fuser.fuse(intent, rag, refs)  ◀── 成员C 提供              │     │
│  │                          │                                     │     │
│  │                          ▼                                     │     │
│  │  5. gen_ppt / gen_doc / gen_anim   ◀── 成员D 提供              │     │
│  │                                                                │     │
│  └──────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  routers/chat.py ──▶ intent_analyzer.analyze() ◀── SSE 推给 B         │
│  routers/speech.py ──▶ transcriber.transcribe() ◀── 成员C 提供        │
└──────┬──────────────────┬────────────────────┬─────────────────────┘
       │                  │                    │
       │ import           │ import             │ import
       ▼                  ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
│ 成员C · ai/   │  │ 成员D · gen/  │  │ 成员C+D 共用      │
│              │  │              │  │ backend/schemas.py │
│ intent/      │  │ pptx/        │  │ (Pydantic Models)  │
│ rag/         │  │ docx/        │  └──────────────────┘
│ fusion/      │  │ anim/        │
│ speech/      │  │ parse/       │
│ prompts/     │  │ prompts/     │
└──────────────┘  └──────────────┘
```

---

## 七、文件命名与代码风格强制规范

### 7.1 文件/目录命名

| 类型 | 约定 | 正例 | 反例 |
|------|------|------|------|
| Python 文件 | `snake_case.py` | `pdf_parser.py` | `PDFParser.py`, `pdf-parser.py` |
| Python 包目录 | `snake_case` | `rag/` | `RAG/`, `retrieval_augmented_generation/` |
| Vue 组件文件 | `PascalCase.vue` | `MessageBubble.vue` | `messageBubble.vue`, `message_bubble.vue` |
| Vue 视图文件 | `PascalCase.vue` | `Chat.vue` | `chat.vue` |
| JS/TS 工具文件 | `camelCase.js` | `request.js` | `request_util.js` |
| Markdown 文档 | `kebab-case.md` | `test-plan.md` | `Test Plan.md` |
| Prompt 模板 | `snake_case.txt` | `intent_extract.txt` | `IntentExtractPrompt.txt` |

### 7.2 Python 代码规范

```python
# ✅ 正确示范
def parse_pdf(path: str) -> str:          # 函数名: 动词_名词，简洁
    """Extract text from PDF file."""     # docstring 英文
    ...

class PPTGenerator:                       # 类名: PascalCase
    """Generate PPT from instruction."""

    MAX_SLIDES = 30                       # 常量: UPPER_SNAKE

    def __init__(self, template: str):
        self.template = template          # 实例变量: snake_case

    def gen_slide(self, title: str, body: str) -> Slide:
        ...

# ❌ 禁止
def parse_the_pdf_file_and_extract_all_text_from_it(path): ...  # 太冗长
class powerpoint_presentation_generator: ...                     # 应为 PascalCase
maxSlides = 30                                                   # 应为 UPPER_SNAKE
```

### 7.3 Vue/JS 代码规范

```javascript
// ✅ 正确示范
const sessionId = ref('')                   // 变量: camelCase
function fetchSessions() { ... }            // 函数: camelCase

const props = defineProps({                 // Props: camelCase
  sessionId: String,
  messages: Array
})

const emit = defineEmits(['select'])        // Events: kebab-case
// 模板中: @select="handleSelect"

// ❌ 禁止
const session_id = ref('')                  // Python 风格，前端不用
function fetch_sessions() { ... }           // 同上
```

### 7.4 Git 分支命名

```
feat/api-session-crud          # A: 后端会话 CRUD
feat/api-chat-sse              # A: 对话 SSE
feat/api-generate-task         # A: 课件生成异步任务
feat/ui-chat-page              # B: 对话页面
feat/ui-file-upload            # B: 文件上传
feat/ai-intent-analyzer        # C: 意图分析
feat/ai-rag-retriever          # C: RAG 检索
feat/gen-ppt-engine            # D: PPT 生成
feat/gen-pdf-parser            # D: PDF 解析
fix/file-upload-encoding       # 修 Bug
```

---

## 八、每日集成检查清单

> 每天 21:00 站会前，每个成员确认以下事项：

| 检查项 | A | B | C | D | E |
|--------|---|---|---|---|---|
| `python backend/main.py` 不报错 | ✅ | — | — | — | — |
| `import ai` 不报错 | ✅ | ✅ | ✅ | ✅ | — |
| `import gen` 不报错 | ✅ | — | — | ✅ | — |
| `pnpm dev` 前端可启动 | — | ✅ | — | — | — |
| Swagger `/docs` 可访问 | ✅ | ✅ | — | — | ✅ |
| `poetry run pytest` 全绿 | ✅ | — | ✅ | ✅ | — |
| 代码已 push 到远端 feat 分支 | ✅ | ✅ | ✅ | ✅ | — |
| 看板任务状态已更新 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 今日产出的文件/截图已发群 | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 九、协议变更流程

1. 发现现有协议不满足需求 → 在团队群提出变更（附理由）
2. PM（E）评估影响范围（谁依赖这个接口？）
3. A 更新本文档 → 更新 `backend/schemas.py`
4. C/D/B 同步更新自己的代码
5. PM 验证变更后的接口两端对齐

> **严禁**在未更新本文档的情况下私下修改函数签名或 API 格式。

---

*本文档版本 v1.0 | 2026-07-16 | 任何变更必须经 A 和 E 共同确认。*
