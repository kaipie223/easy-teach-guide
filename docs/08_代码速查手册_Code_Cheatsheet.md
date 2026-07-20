# 08 — 代码速查手册 · Code Cheatsheet

> **本文档是纯代码参考。** 不要从头读，任务文档告诉你"用第 X 节的代码"时再来看。
> 所有代码都是粘贴即用的，不要自己改结构。

---

## 一、A · 后端代码

### 1.1 `backend/config.py`

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    DEEPSEEK_API_KEY = os.getenv("DEEPSEEK_API_KEY", "sk-placeholder")
    DEEPSEEK_BASE_URL = os.getenv("DEEPSEEK_BASE_URL", "https://api.deepseek.com")
    WHISPER_MODEL = os.getenv("WHISPER_MODEL", "small")
    CHROMA_PERSIST_DIR = os.getenv("CHROMA_PERSIST_DIR", "./chroma_data")
    DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./data.db")
    UPLOAD_DIR = os.getenv("UPLOAD_DIR", "./backend/uploads")
    OUTPUT_DIR = os.getenv("OUTPUT_DIR", "./output")

config = Config()
```

### 1.2 `backend/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import uvicorn

app = FastAPI(title="EduAgent API", version="0.1.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
def health():
    return {"status": "ok", "version": "0.1.0"}

if __name__ == "__main__":
    uvicorn.run("backend.main:app", host="0.0.0.0", port=8000, reload=True)
```

### 1.3 `.env` 模板

```bash
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxx
DEEPSEEK_BASE_URL=https://api.deepseek.com
WHISPER_MODEL=small
CHROMA_PERSIST_DIR=./chroma_data
DATABASE_URL=sqlite:///./data.db
UPLOAD_DIR=./backend/uploads
OUTPUT_DIR=./output
```

### 1.4 `.gitignore`

```gitignore
__pycache__/
*.pyc
.env
*.db
backend/uploads/*
!backend/uploads/.gitkeep
output/*
!output/.gitkeep
chroma_data/
node_modules/
dist/
.vscode/
.idea/
*.log
```

### 1.5 `backend/schemas.py`（完整版见协议文档第 2 节）

```python
from pydantic import BaseModel
from datetime import datetime
from enum import Enum

# --- 会话 ---
class SessionCreate(BaseModel):
    teacher_name: str = ""
    subject: str = ""

class SessionInfo(BaseModel):
    session_id: str
    teacher_name: str
    subject: str
    status: str
    created_at: datetime

# --- 消息 ---
class MessageRole(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"

class MessageType(str, Enum):
    TEXT = "text"
    QUESTION = "question"
    CONFIRM = "confirm"

class ChatRequest(BaseModel):
    message: str

class ChatEvent(BaseModel):
    event_type: MessageType
    content: str
    data: dict | None = None

# --- 文件 ---
class FileType(str, Enum):
    PDF = "pdf"
    WORD = "word"
    PPT = "ppt"
    IMAGE = "image"
    VIDEO = "video"

class FileInfo(BaseModel):
    file_id: str
    original_name: str
    file_type: FileType
    size_kb: float
    upload_time: datetime
    ref_description: str | None = None

# --- 任务 ---
class TaskStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

class GenerateRequest(BaseModel):
    session_id: str

class OutputFile(BaseModel):
    file_id: str
    file_type: str
    file_name: str
    size_kb: float
    download_url: str

class TaskInfo(BaseModel):
    task_id: str
    session_id: str
    status: TaskStatus
    progress: int
    outputs: list[OutputFile] | None = None
    error: str | None = None

# --- 知识点 ---
class KnowledgePoint(BaseModel):
    order: int
    title: str
    difficulty: str
    key_points: list[str]
    examples: list[str] = []
    estimated_minutes: int = 5

# --- 意图分析结果 ---
class IntentResult(BaseModel):
    teaching_goal: str
    target_audience: str
    duration_minutes: int
    knowledge_points: list[KnowledgePoint]
    logic_flow: list[str]
    style_preference: str
    missing_info: list[str] = []
    follow_up_question: str | None = None
    confirm_summary: str | None = None
    is_complete: bool = False

# --- RAG 检索结果 ---
class RAGDocument(BaseModel):
    content: str
    source: str
    score: float

# --- 参考资料解析结果 ---
class ReferenceMaterial(BaseModel):
    file_id: str
    file_type: FileType
    extracted_text: str
    key_topics: list[str]
    format_notes: str | None = None

# --- 课件生成指令集 ---
class GenerationInstruction(BaseModel):
    session_id: str
    teaching_goal: str
    target_audience: str
    duration_minutes: int
    knowledge_points: list[KnowledgePoint]
    logic_flow: list[str]
    style_preference: str
    rag_context: list[RAGDocument]
    reference_materials: list[ReferenceMaterial]
    extra_requirements: str = ""
```

### 1.6 `backend/models/session.py`

```python
from sqlalchemy import Column, String, DateTime, Text, create_engine
from sqlalchemy.orm import declarative_base, sessionmaker
from datetime import datetime, timezone
import uuid

Base = declarative_base()

def gen_id(prefix: str) -> str:
    return f"{prefix}_{uuid.uuid4().hex[:8]}"

class Session(Base):
    __tablename__ = "sessions"
    session_id = Column(String, primary_key=True, default=lambda: gen_id("s"))
    teacher_name = Column(String, default="")
    subject = Column(String, default="")
    status = Column(String, default="active")
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))

class ChatMessage(Base):
    __tablename__ = "chat_messages"
    id = Column(String, primary_key=True, default=lambda: gen_id("msg"))
    session_id = Column(String, nullable=False)
    role = Column(String, nullable=False)
    content = Column(Text, nullable=False)
    msg_type = Column(String, default="text")
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
```

### 1.7 `backend/models/file.py`

```python
from .session import Base, gen_id
from sqlalchemy import Column, String, Float, DateTime
from datetime import datetime, timezone

class FileRecord(Base):
    __tablename__ = "files"
    file_id = Column(String, primary_key=True, default=lambda: gen_id("f"))
    session_id = Column(String, nullable=False)
    original_name = Column(String, nullable=False)
    file_type = Column(String, nullable=False)
    stored_path = Column(String, nullable=False)
    size_kb = Column(Float, default=0)
    ref_description = Column(String, nullable=True)
    upload_time = Column(DateTime, default=lambda: datetime.now(timezone.utc))
```

### 1.8 `backend/models/task.py`

```python
from .session import Base, gen_id
from sqlalchemy import Column, String, Integer, DateTime, JSON
from datetime import datetime, timezone

class Task(Base):
    __tablename__ = "tasks"
    task_id = Column(String, primary_key=True, default=lambda: gen_id("task"))
    session_id = Column(String, nullable=False)
    status = Column(String, default="pending")
    progress = Column(Integer, default=0)
    outputs = Column(JSON, nullable=True)
    error = Column(String, nullable=True)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
```

### 1.9 `backend/models/__init__.py`

```python
from .session import Base, Session, ChatMessage, gen_id
from .file import FileRecord
from .task import Task
```

### 1.10 Session 路由（Day 3 内存版 → Day 4 改造为 DB 版）

```python
# backend/routers/session.py (Day 3 内存版)
from fastapi import APIRouter, HTTPException
from backend.schemas import SessionCreate, SessionInfo
from backend.models.session import gen_id

router = APIRouter(prefix="/api/v1/sessions", tags=["sessions"])
_sessions = {}

@router.post("", status_code=201)
def create_session(req: SessionCreate):
    sid = gen_id("s")
    _sessions[sid] = {**req.model_dump(), "session_id": sid, "status": "active"}
    return _sessions[sid]

@router.get("/{session_id}")
def get_session(session_id: str):
    if session_id not in _sessions:
        raise HTTPException(404, detail="会话不存在")
    return _sessions[session_id]
```

### 1.11 Chat SSE 路由（Day 5-6 完整版）

```python
# backend/routers/chat.py
import json
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from backend.schemas import ChatRequest, ChatEvent, MessageType

router = APIRouter(prefix="/api/v1/sessions", tags=["chat"])
intent_analyzer = None  # 在 main.py lifespan 中初始化

def _sse_event(event: ChatEvent) -> str:
    return f"data: {event.model_dump_json()}\n\n"

async def chat_stream(session_id: str, message: str):
    messages = load_history(session_id)
    messages.append({"role": "user", "content": message})
    result = intent_analyzer.analyze(session_id, messages)

    if not result.is_complete:
        event = ChatEvent(
            event_type=MessageType.QUESTION,
            content=result.follow_up_question,
            data={"missing": result.missing_info}
        )
        yield _sse_event(event)
    else:
        event = ChatEvent(
            event_type=MessageType.CONFIRM,
            content=result.confirm_summary,
            data=None
        )
        yield _sse_event(event)
    yield "data: [DONE]\n\n"

@router.post("/{session_id}/chat")
async def chat(session_id: str, req: ChatRequest):
    return StreamingResponse(
        chat_stream(session_id, req.message),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"}
    )
```

### 1.12 `backend/services/orchestrator.py`

```python
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
        self.intent_analyzer = IntentAnalyzer(config.DEEPSEEK_API_KEY, config.DEEPSEEK_BASE_URL)
        self.rag_retriever = RAGRetriever(config.CHROMA_PERSIST_DIR)
        self.fuser = KnowledgeFuser(config.DEEPSEEK_API_KEY, config.DEEPSEEK_BASE_URL)
        self.config = config

    def run_generation(self, session_id: str) -> TaskInfo:
        # Step 1: 锁定意图
        intent = self.intent_analyzer.lock_intent(session_id)

        # Step 2: RAG 检索
        query = f"{intent.teaching_goal} {' '.join(kp.title for kp in intent.knowledge_points)}"
        rag_docs = self.rag_retriever.search(query, top_k=5)

        # Step 3: 解析参考资料
        references = []
        for f in session_files(session_id):
            if f.file_type == FileType.PDF:
                text = extract_pdf_text(f.path)
            elif f.file_type == FileType.WORD:
                text = extract_doc_text(f.path)
            elif f.file_type == FileType.VIDEO:
                text = summarize_video(f.path, self.config.DEEPSEEK_API_KEY)
            else:
                continue
            references.append(ReferenceMaterial(
                file_id=f.file_id, file_type=f.file_type,
                extracted_text=text, key_topics=[],
                format_notes=f.ref_description
            ))

        # Step 4: 知识融合
        instruction = self.fuser.fuse(intent, rag_docs, references)

        # Step 5: 生成课件
        pptx_path = gen_ppt(instruction, output_dir=self.config.OUTPUT_DIR)
        docx_path = gen_doc(instruction, output_dir=self.config.OUTPUT_DIR)
        html_path = gen_animation(instruction, output_dir=self.config.OUTPUT_DIR)

        # Step 6: 注册输出文件
        return self._build_task_result(pptx_path, docx_path, html_path)
```

---

## 二、B · 前端代码

### 2.1 `frontend/src/main.js`

```javascript
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { createPinia } from 'pinia'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import zhCn from 'element-plus/dist/locale/zh-cn.mjs'

const app = createApp(App)
app.use(router)
app.use(createPinia())
app.use(ElementPlus, { locale: zhCn })
app.mount('#app')
```

### 2.2 `frontend/src/App.vue`

```vue
<template>
  <router-view />
</template>
```

### 2.3 `frontend/src/router/index.js`

```javascript
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  { path: '/', name: 'Home', component: () => import('../views/Home.vue') },
  { path: '/chat/:id', name: 'Chat', component: () => import('../views/Chat.vue') },
  { path: '/preview/:taskId', name: 'Preview', component: () => import('../views/Preview.vue') },
  { path: '/download/:taskId', name: 'Download', component: () => import('../views/Download.vue') },
]

export default createRouter({ history: createWebHistory(), routes })
```

### 2.4 `frontend/src/types/index.ts`

```typescript
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

### 2.5 `frontend/src/utils/request.js`

```javascript
import axios from 'axios'

const api = axios.create({
  baseURL: 'http://localhost:8000/api/v1',
  timeout: 30000,
})

api.interceptors.response.use(
  res => res.data,
  err => {
    const msg = err.response?.data?.detail || err.message || '网络错误'
    console.error('[API Error]', msg)
    return Promise.reject(err)
  }
)

export default api
```

### 2.6 `frontend/src/utils/sse.js`

```javascript
export function createChatStream(sessionId, message, onEvent, onDone, onError) {
  const controller = new AbortController()

  fetch(`http://localhost:8000/api/v1/sessions/${sessionId}/chat`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message }),
    signal: controller.signal,
  }).then(async response => {
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    let buffer = ''

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split('\n')
      buffer = lines.pop() || ''

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6).trim()
          if (data === '[DONE]') { onDone(); return }
          try { onEvent(JSON.parse(data)) }
          catch (e) { console.warn('SSE parse error:', e) }
        }
      }
    }
  }).catch(onError)

  return controller
}
```

### 2.7 `frontend/src/views/Home.vue`

```vue
<template>
  <div class="home-container">
    <h1>📚 多模态AI教学智能体</h1>
    <p class="subtitle">告诉我你想备什么课，我来帮你做课件</p>
    <div class="start-section">
      <el-input v-model="subject" placeholder="例如：Python程序设计入门" size="large" @keyup.enter="startSession" />
      <el-button type="primary" size="large" @click="startSession" :loading="creating">开始设计</el-button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import api from '../utils/request'

const router = useRouter()
const subject = ref('')
const creating = ref(false)

async function startSession() {
  if (!subject.value.trim()) return
  creating.value = true
  const res = await api.post('/sessions', { teacher_name: '', subject: subject.value.trim() })
  router.push(`/chat/${res.session_id}`)
}
</script>
```

### 2.8 `frontend/src/stores/session.js`

```javascript
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useSessionStore = defineStore('session', () => {
  const sessionId = ref('')
  const teacherName = ref('')
  const subject = ref('')
  const messages = ref([])

  function addMessage(msg) {
    messages.value.push(msg)
  }

  function clearMessages() {
    messages.value = []
  }

  return { sessionId, teacherName, subject, messages, addMessage, clearMessages }
})
```

### 2.9 `frontend/src/stores/task.js`

```javascript
import { defineStore } from 'pinia'
import { ref } from 'vue'
import api from '../utils/request'

export const useTaskStore = defineStore('task', () => {
  const taskId = ref('')
  const status = ref('')
  const progress = ref(0)
  const outputs = ref([])
  const error = ref(null)

  async function pollTask(id) {
    taskId.value = id
    return new Promise((resolve, reject) => {
      const timer = setInterval(async () => {
        const task = await api.get(`/tasks/${id}/status`)
        status.value = task.status
        progress.value = task.progress
        outputs.value = task.outputs || []
        error.value = task.error

        if (task.status === 'completed') { clearInterval(timer); resolve(task) }
        if (task.status === 'failed') { clearInterval(timer); reject(task.error) }
      }, 2000)
    })
  }

  return { taskId, status, progress, outputs, error, pollTask }
})
```

### 2.10 `frontend/src/components/chat/FileUploader.vue`（核心逻辑）

```javascript
async function uploadFile(file) {
  const form = new FormData()
  form.append('file', file)
  form.append('ref_description', refDesc.value)

  const res = await api.post('/files/upload', form, {
    headers: { 'Content-Type': 'multipart/form-data' },
    onUploadProgress: (e) => { progress.value = Math.round((e.loaded / e.total) * 100) }
  })
  uploadedFiles.value.push(res)
  emit('uploaded', res)
}
```

### 2.11 `frontend/src/components/chat/VoiceInput.vue`（核心逻辑）

```javascript
async function startRecord() {
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
  mediaRecorder = new MediaRecorder(stream)
  const chunks = []

  mediaRecorder.ondataavailable = (e) => chunks.push(e.data)
  mediaRecorder.onstop = async () => {
    const blob = new Blob(chunks, { type: 'audio/webm' })
    const form = new FormData()
    form.append('audio', blob, 'recording.webm')
    const res = await api.post('/speech/transcribe', form)
    emit('transcribed', res.text)
  }

  mediaRecorder.start()
}
```

### 2.12 `frontend/src/views/Preview.vue`（轮询逻辑）

```javascript
import { ref, onMounted, onUnmounted } from 'vue'
import { useRoute } from 'vue-router'
import api from '../utils/request'

const route = useRoute()
const task = ref(null)
let timer = null

onMounted(() => {
  timer = setInterval(async () => {
    task.value = await api.get(`/tasks/${route.params.taskId}/status`)
    if (task.value.status === 'completed' || task.value.status === 'failed') {
      clearInterval(timer)
    }
  }, 2000)
})

onUnmounted(() => clearInterval(timer))
```

---

## 三、C · AI 代码

### 3.1 `ai/test_llm.py`

```python
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
```

### 3.2 `ai/test_chroma.py`

```python
import chromadb
client = chromadb.PersistentClient(path="./chroma_data")
collection = client.get_or_create_collection("test")
collection.add(
    documents=["Python是一种解释型、面向对象的高级编程语言。"],
    metadatas=[{"source": "test"}],
    ids=["doc_001"]
)
results = collection.query(query_texts=["Python是什么"], n_results=1)
print(results["documents"])
```

### 3.3 `ai/prompts/intent_extract.txt`

```text
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

### 3.4 `ai/prompts/follow_up.txt`

```text
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

### 3.5 `ai/prompts/confirm.txt`

```text
根据以下教学意图，生成一份结构化的"需求确认书"。

意图信息：
{intent_json}

请生成：
{
  "summary": "格式化的确认文本，包含课程名称、对象、时长、知识点列表、教学风格、教学流程"
}
```

### 3.6 `ai/prompts/fusion.txt`

```text
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

输出严格的 JSON 格式（GenerationInstruction 结构）
```

### 3.7 `ai/intent/analyzer.py`

```python
import json, os
from openai import OpenAI
from backend.schemas import IntentResult, KnowledgePoint

class IntentAnalyzer:
    def __init__(self, api_key: str, base_url: str):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.sessions = {}
        self._load_prompts()

    def _load_prompts(self):
        base = os.path.dirname(os.path.dirname(__file__))
        with open(f"{base}/prompts/intent_extract.txt", encoding="utf-8") as f:
            self.prompt_extract = f.read()
        with open(f"{base}/prompts/follow_up.txt", encoding="utf-8") as f:
            self.prompt_followup = f.read()
        with open(f"{base}/prompts/confirm.txt", encoding="utf-8") as f:
            self.prompt_confirm = f.read()

    def analyze(self, session_id: str, messages: list[dict], uploaded_files: list[str] = []) -> IntentResult:
        formatted = "\n".join([f"{m['role']}: {m['content']}" for m in messages])
        prompt = self.prompt_extract.format(messages=formatted)

        resp = self.client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=1000,
            response_format={"type": "json_object"}
        )
        raw = json.loads(resp.choices[0].message.content)

        if not raw.get("is_complete") and raw.get("missing_info"):
            follow_up = self._gen_follow_up(raw)
            raw["follow_up_question"] = follow_up["question_text"]
            raw["confirm_summary"] = None
        else:
            summary = self._gen_confirm(raw)
            raw["confirm_summary"] = summary["summary"]
            raw["follow_up_question"] = None
            raw["is_complete"] = True

        self.sessions[session_id] = raw
        return IntentResult(**raw)

    def lock_intent(self, session_id: str) -> IntentResult:
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

### 3.8 `ai/rag/loader.py`

```python
import os
from pathlib import Path

def load_documents(kb_dir: str) -> list[dict]:
    docs = []
    kb_path = Path(kb_dir)
    for file_path in kb_path.rglob("*.pdf"):
        from gen.parse.pdf import extract_pdf_text
        text = extract_pdf_text(str(file_path))
        docs.append({"content": text, "source": file_path.name, "type": "pdf"})
    for file_path in kb_path.rglob("*.txt"):
        text = file_path.read_text(encoding="utf-8")
        docs.append({"content": text, "source": file_path.name, "type": "txt"})
    for file_path in kb_path.rglob("*.docx"):
        from gen.parse.doc import extract_doc_text
        text = extract_doc_text(str(file_path))
        docs.append({"content": text, "source": file_path.name, "type": "docx"})
    return docs
```

### 3.9 `ai/rag/splitter.py`

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

def split_documents(docs: list[dict], chunk_size: int = 500, overlap: int = 50) -> list[dict]:
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

### 3.10 `ai/rag/embedder.py`

```python
import chromadb
from chromadb.config import Settings

def build_index(chunks: list[dict], persist_dir: str, collection_name: str = "knowledge_base"):
    client = chromadb.PersistentClient(path=persist_dir, settings=Settings(anonymized_telemetry=False))
    collection = client.get_or_create_collection(collection_name)
    for i, chunk in enumerate(chunks):
        collection.add(
            documents=[chunk["content"]],
            metadatas=[{"source": chunk["source"], "chunk_index": chunk["chunk_index"]}],
            ids=[f"chunk_{i}"]
        )
    print(f"已入库 {len(chunks)} 个文档块")
    return collection
```

### 3.11 `ai/rag/retriever.py`

```python
import chromadb
from backend.schemas import RAGDocument

class RAGRetriever:
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

### 3.12 `ai/fusion/fusion.py`

```python
import json, os
from openai import OpenAI
from backend.schemas import IntentResult, RAGDocument, ReferenceMaterial, GenerationInstruction

class KnowledgeFuser:
    def __init__(self, api_key: str, base_url: str):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        base = os.path.dirname(os.path.dirname(__file__))
        with open(f"{base}/prompts/fusion.txt", encoding="utf-8") as f:
            self.prompt_fusion = f.read()

    def fuse(self, intent: IntentResult, rag_docs: list[RAGDocument], references: list[ReferenceMaterial]) -> GenerationInstruction:
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
        return GenerationInstruction(**data)
```

### 3.13 `ai/build_kb.py`

```python
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

### 3.14 `ai/test_intent.py`

```python
import os
from dotenv import load_dotenv
load_dotenv()

from ai.intent.analyzer import IntentAnalyzer

analyzer = IntentAnalyzer(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url=os.getenv("DEEPSEEK_BASE_URL")
)

result = analyzer.analyze("test_session", [
    {"role": "user", "content": "我想备一节Python列表的课"}
])

print(f"is_complete: {result.is_complete}")
print(f"teaching_goal: {result.teaching_goal}")
print(f"missing_info: {result.missing_info}")
print(f"follow_up: {result.follow_up_question}")
```

### 3.15 `ai/speech/transcriber.py`

```python
from faster_whisper import WhisperModel

class SpeechTranscriber:
    def __init__(self, model_size: str = "small"):
        self.model = WhisperModel(model_size, device="cpu", compute_type="int8")

    def transcribe(self, audio_path: str) -> str:
        segments, _ = self.model.transcribe(audio_path, language="zh")
        return "".join([seg.text for seg in segments])
```

---

## 四、D · 课件生成代码

### 4.1 `gen/parse/pdf.py`

```python
import fitz

def extract_pdf_text(pdf_path: str) -> str:
    doc = fitz.open(pdf_path)
    texts = []
    for page in doc:
        text = page.get_text("text")
        if text.strip():
            texts.append(text.strip())
    doc.close()
    return "\n\n".join(texts)
```

### 4.2 `gen/parse/doc.py`

```python
from docx import Document

def extract_doc_text(docx_path: str) -> str:
    doc = Document(docx_path)
    lines = []
    for para in doc.paragraphs:
        if para.style.name.startswith("Heading"):
            level = para.style.name.replace("Heading ", "")
            prefix = "#" * int(level) + " "
            lines.append(prefix + para.text)
        elif para.text.strip():
            lines.append(para.text)
    return "\n\n".join(lines)
```

### 4.3 `gen/parse/image.py`

```python
import base64, os
from openai import OpenAI
from dotenv import load_dotenv
load_dotenv()

def describe_image(image_path: str, api_key: str = None) -> str:
    key = api_key or os.getenv("DEEPSEEK_API_KEY")
    base = os.getenv("DEEPSEEK_BASE_URL", "https://api.deepseek.com")
    client = OpenAI(api_key=key, base_url=base)

    with open(image_path, "rb") as f:
        img_b64 = base64.b64encode(f.read()).decode()

    resp = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": "请描述这张图片的内容，重点关注与教学相关的图表、公式、结构。"},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}}
            ]
        }],
        max_tokens=500
    )
    return resp.choices[0].message.content
```

### 4.4 `gen/parse/video.py`

```python
import cv2, os

def extract_keyframes(video_path: str, interval_sec: int = 10, output_dir: str = "./frames") -> list[str]:
    os.makedirs(output_dir, exist_ok=True)
    cap = cv2.VideoCapture(video_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    interval_frames = int(fps * interval_sec)
    paths = []
    frame_count = 0
    while True:
        ret, frame = cap.read()
        if not ret: break
        if frame_count % interval_frames == 0:
            path = f"{output_dir}/frame_{frame_count:06d}.jpg"
            cv2.imwrite(path, frame)
            paths.append(path)
        frame_count += 1
    cap.release()
    return paths

def summarize_video(video_path: str, api_key: str) -> str:
    frames = extract_keyframes(video_path, interval_sec=30)
    from gen.parse.image import describe_image
    descriptions = []
    for fp in frames[:6]:
        try:
            desc = describe_image(fp, api_key)
            descriptions.append(desc)
        except: pass

    if descriptions:
        from openai import OpenAI
        client = OpenAI(api_key=api_key, base_url=os.getenv("DEEPSEEK_BASE_URL"))
        resp = client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": f"以下是教学视频中不同时间段的画面描述，请汇总为一个200字以内的视频摘要：\n" + "\n---\n".join(descriptions)}],
            max_tokens=300
        )
        return resp.choices[0].message.content
    return "无法解析视频内容"
```

### 4.5 `gen/pptx/layout.py`

```python
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor

SLIDE_W = Inches(13.333)
SLIDE_H = Inches(7.5)

COLOR_PRIMARY   = RGBColor(0x1A, 0x73, 0xE8)
COLOR_SECONDARY = RGBColor(0x34, 0xA8, 0x53)
COLOR_ACCENT    = RGBColor(0xFB, 0xBC, 0x04)
COLOR_TEXT      = RGBColor(0x20, 0x21, 0x24)
COLOR_SUBTEXT   = RGBColor(0x5F, 0x63, 0x68)
COLOR_BG        = RGBColor(0xFF, 0xFF, 0xFF)
COLOR_BG_DARK   = RGBColor(0x1A, 0x73, 0xE8)

FONT_TITLE = "微软雅黑"
FONT_BODY  = "微软雅黑"

SIZE_COVER_TITLE    = Pt(44)
SIZE_COVER_SUBTITLE = Pt(24)
SIZE_SLIDE_TITLE    = Pt(32)
SIZE_SECTION_TITLE  = Pt(28)
SIZE_BODY           = Pt(18)
SIZE_BODY_SMALL     = Pt(14)

MARGIN_LEFT   = Inches(1.2)
MARGIN_RIGHT  = Inches(1.2)
MARGIN_TOP    = Inches(0.6)
CONTENT_WIDTH = Inches(10.8)
LINE_SPACING = Pt(30)
```

### 4.6 `gen/pptx/slides.py`（4 种页面模板）

贴原文档 `04_成员D` 的 slides.py 全部代码（add_cover / add_toc / add_content / add_summary）

### 4.7 `gen/pptx/generator.py`

```python
import json, os
from pathlib import Path
from pptx import Presentation
from openai import OpenAI
from backend.schemas import GenerationInstruction
from .layout import SLIDE_W, SLIDE_H
from .slides import add_cover, add_toc, add_content, add_summary

def gen_ppt(instruction: GenerationInstruction, template: str = "default", output_dir: str = "./output") -> Path:
    os.makedirs(output_dir, exist_ok=True)
    client = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"), base_url=os.getenv("DEEPSEEK_BASE_URL"))

    outline_prompt = _load_prompt("ppt_outline.txt")
    prompt = outline_prompt.format(instruction_json=instruction.model_dump_json(indent=2))
    resp = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=2000,
        response_format={"type": "json_object"}
    )
    outline = json.loads(resp.choices[0].message.content)

    prs = Presentation()
    prs.slide_width = SLIDE_W
    prs.slide_height = SLIDE_H

    add_cover(prs, instruction.teaching_goal, f"授课对象：{instruction.target_audience} | {instruction.duration_minutes}分钟")
    toc_items = [s["title"] for s in outline["slides"] if s["type"] == "content"]
    add_toc(prs, toc_items)

    for slide_info in outline["slides"]:
        if slide_info["type"] == "content":
            add_content(prs, slide_info["title"], slide_info.get("points", []))

    all_keys = []
    for kp in instruction.knowledge_points:
        all_keys.extend(kp.key_points)
    add_summary(prs, all_keys[:8])

    filename = f"{instruction.teaching_goal[:30]}.pptx"
    filepath = Path(output_dir) / filename
    prs.save(str(filepath))
    return filepath

def _load_prompt(name: str) -> str:
    base = os.path.dirname(os.path.dirname(__file__))
    with open(f"{base}/prompts/{name}", encoding="utf-8") as f:
        return f.read()
```

### 4.8 `gen/docx/generator.py`

```python
import json, os
from pathlib import Path
from docx import Document
from docx.enum.text import WD_ALIGN_PARAGRAPH
from openai import OpenAI
from backend.schemas import GenerationInstruction

def gen_doc(instruction: GenerationInstruction, output_dir: str = "./output") -> Path:
    os.makedirs(output_dir, exist_ok=True)
    client = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"), base_url=os.getenv("DEEPSEEK_BASE_URL"))

    prompt = _load_prompt("doc_plan.txt").format(instruction_json=instruction.model_dump_json(indent=2))
    resp = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=3000,
        response_format={"type": "json_object"}
    )
    content = json.loads(resp.choices[0].message.content)

    doc = Document()
    title = doc.add_heading(instruction.teaching_goal, level=0)
    title.alignment = WD_ALIGN_PARAGRAPH.CENTER

    sections = [
        ("一、教学目标", content.get("objectives", "")),
        ("二、教学重难点", content.get("key_difficulties", "")),
        ("三、教学过程", content.get("procedure", "")),
        ("四、教学方法", content.get("methods", "")),
        ("五、课堂活动设计", content.get("activities", "")),
        ("六、课后作业与评价", content.get("homework", "")),
    ]
    for title_text, body_text in sections:
        doc.add_heading(title_text, level=1)
        doc.add_paragraph(body_text)

    filename = f"{instruction.teaching_goal[:30]}_教案.docx"
    filepath = Path(output_dir) / filename
    doc.save(str(filepath))
    return filepath
```

### 4.9 `gen/anim/generator.py`

```python
import json, os
from pathlib import Path
from openai import OpenAI
from backend.schemas import GenerationInstruction

def gen_animation(instruction: GenerationInstruction, anim_type: str = "quiz_game", output_dir: str = "./output") -> Path:
    os.makedirs(output_dir, exist_ok=True)
    client = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"), base_url=os.getenv("DEEPSEEK_BASE_URL"))

    prompt = _load_prompt("anim_idea.txt").format(
        instruction_json=instruction.model_dump_json(indent=2),
        anim_type=anim_type
    )
    resp = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=4000,
        response_format={"type": "json_object"}
    )
    data = json.loads(resp.choices[0].message.content)

    html = data["html_code"]
    filename = f"{instruction.teaching_goal[:20]}_互动.html"
    filepath = Path(output_dir) / filename
    filepath.write_text(html, encoding="utf-8")
    return filepath
```

### 4.10 `gen/test_gen.py`

```python
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor

def quick_test_ppt():
    prs = Presentation()
    prs.slide_width  = Inches(13.333)
    prs.slide_height = Inches(7.5)

    slide = prs.slides.add_slide(prs.slide_layouts[6])
    txBox = slide.shapes.add_textbox(Inches(1), Inches(2.5), Inches(11), Inches(1.5))
    tf = txBox.text_frame
    tf.text = "Python程序设计入门"
    tf.paragraphs[0].font.size = Pt(44)
    tf.paragraphs[0].font.bold = True
    tf.paragraphs[0].font.color.rgb = RGBColor(0x1a, 0x73, 0xe8)

    slide2 = prs.slides.add_slide(prs.slide_layouts[6])
    txBox2 = slide2.shapes.add_textbox(Inches(2), Inches(1.5), Inches(9), Inches(4.5))
    tf2 = txBox2.text_frame
    tf2.text = "目  录"
    tf2.paragraphs[0].font.size = Pt(36)
    for item in ["1. Python简介", "2. 列表基础", "3. 字典基础", "4. 对比与实战"]:
        p = tf2.add_paragraph()
        p.text = item
        p.font.size = Pt(24)
        p.space_after = Pt(12)

    slide3 = prs.slides.add_slide(prs.slide_layouts[6])
    txBox3 = slide3.shapes.add_textbox(Inches(1), Inches(0.5), Inches(11), Inches(1))
    tf3 = txBox3.text_frame
    tf3.text = "Python 列表基础"
    tf3.paragraphs[0].font.size = Pt(32)

    body_box = slide3.shapes.add_textbox(Inches(1), Inches(2), Inches(11), Inches(5))
    bf = body_box.text_frame
    bf.text = "• 列表是有序的可变序列"
    for point in ["• 使用方括号 [] 创建", "• 支持索引、切片操作", "• 常用方法：append、insert、pop"]:
        p = bf.add_paragraph()
        p.text = point
        p.font.size = Pt(20)
        p.space_after = Pt(8)

    prs.save("test_output.pptx")
    print("✅ test_output.pptx 已生成")

if __name__ == "__main__":
    quick_test_ppt()
```

---

## 五、E · PM 模板

### 5.1 测试用例表 `test-cases.xlsx` 列名

```
ID | 模块 | 用例标题 | 前置条件 | 操作步骤 | 预期结果 | 实际结果 | 状态(Pass/Fail) | 优先级(P0/P1/P2) | 备注
```

### 5.2 Bug 追踪表 `bug-tracker.xlsx` 列名

```
Bug ID | 标题 | 模块 | 严重度(P0/P1/P2) | 复现步骤 | 预期 | 实际 | 负责人 | 状态(Open/Fixed/Closed) | 发现日期 | 修复日期
```

### 5.3 Sprint Board 列名

```
任务ID | 任务名称 | 负责人 | 阶段 | 状态(待办/进行中/已完成) | 开始日 | 截止日 | 阻塞标记 | 备注
```

### 5.4 知识库资料清单 `知识库资料清单.xlsx` 列名

```
文件名 | 类型(PDF/Word/PPT/视频) | 来源 | 内容简介 | 适用学科 | 入库状态(已入库/未入库) | 入库日期 | 备注
```

---

*本文档是所有代码的唯一存放处。所有人写代码时从这里复制，不要自己重新写。*
*如果协议文档中的结构有更新，A 应同步更新本文档第 1.5 节。*
