# 00 — 总执行手册 · Master Plan

> 版本 v2.0 | 2026-07-16 | 项目经理：成员E

---

## ⚠️ 必读文档

| 文档 | 用途 | 谁必须读 |
|------|------|----------|
| **`01_技术协议与接口规范_Technical_Protocol.md`** | 所有角色间的接口合同、数据模型、API 契约、函数签名 | **全队必读** |
| `00_总执行手册_Master_Plan.md`（本文档） | 架构、里程碑、工程规范 | 全队 |
| 各人执行文档 | 精确到文件路径的操作指令 | 按角色 |

> **协议文档是唯一的真相来源（Single Source of Truth）。任何接口变更必须先更新协议文档。**

---

## 一、架构与技术栈约束

### 1.1 总体架构

```
┌─────────────────────────────────────────────────────────┐
│                    前端 (Vue 3 + Vite)                    │
│  对话面板 │ 文件上传 │ 课件预览 │ 导出下载                │
└──────────────────────┬──────────────────────────────────┘
                       │ RESTful API (JSON) + WebSocket(对话流)
┌──────────────────────▼──────────────────────────────────┐
│                  后端 (FastAPI + Python 3.11)             │
│  用户会话 │ 文件管理 │ 任务队列 │ API 网关                │
└──┬──────────┬──────────┬──────────┬─────────────────────┘
   │          │          │          │
   ▼          ▼          ▼          ▼
┌──────┐ ┌──────┐ ┌──────────┐ ┌──────────────┐
│RAG   │ │意图  │ │课件生成  │ │多模态解析    │
│引擎  │ │理解  │ │PPT+Word  │ │PDF/视频/图片 │
└──┬───┘ └──┬───┘ └────┬─────┘ └──────┬───────┘
   │        │           │              │
   ▼        ▼           ▼              ▼
┌──────────────────────────────────────────────────────────┐
│                   模型层 (统一调用接口)                     │
│  DeepSeek API (主) │ Whisper (语音) │ 本地 Embedding      │
└──────────────────────────────────────────────────────────┘
```

### 1.2 技术选型（已确定，不可自行更换）

| 层级 | 技术 | 版本 | 理由 |
|------|------|------|------|
| 前端框架 | Vue 3 + Vite | 3.4+ | 学习曲线平缓，生态完善，Element Plus 中文文档好 |
| UI 组件库 | Element Plus | 2.7+ | 企业级组件，中文友好 |
| 后端框架 | FastAPI | 0.111+ | 异步高性能，自动生成 Swagger 文档，Python AI 生态无缝 |
| 数据库 | SQLite → PostgreSQL | — | 开发期用 SQLite 零配置；上线切 PostgreSQL |
| ORM | SQLAlchemy 2.0 | 2.0+ | Python 事实标准 |
| 向量数据库 | ChromaDB | 0.5+ | 嵌入式运行，无需单独部署，竞赛场景够用 |
| RAG 框架 | LangChain | 0.3+ | 社区最活跃，集成丰富 |
| LLM | DeepSeek-V3 API | — | 性价比高，中文能力强，教育内容生成质量好 |
| 语音识别 | faster-whisper | 1.0+ | 本地运行免费，中英文准确率够用 |
| PDF 解析 | PyMuPDF (fitz) | 1.24+ | 文本/图片提取稳定 |
| Word 生成 | python-docx | 1.1+ | 原生 .docx，支持样式 |
| PPT 生成 | python-pptx | 0.6+ | 原生 .pptx，支持图文 |
| 视频处理 | OpenCV + FFmpeg | 4.9+ / 6.0+ | 帧提取 + 摘要 |
| 包管理 | Poetry | 1.8+ | 依赖锁定，团队环境一致 |
| IDE | PyCharm / IntelliJ IDEA / VS Code | — | 按角色选用 |

### 1.3 开发环境统一要求

- **Python**：3.11.x（必须统一，防止兼容问题）
- **Node.js**：20 LTS
- **Poetry**：用于后端依赖管理
- **pnpm**：用于前端依赖管理
- **Git**：所有代码托管于团队仓库

---

## 二、全局里程碑 (WBS)

### Phase 1 — 环境搭建与接口定义（Day 1 ~ Day 3，共 3 天）

| # | 交付物 | 负责人 | 验收标准 |
|---|--------|--------|----------|
| 1.1 | 后端项目骨架跑通，`/health` 返回 200 | A | Postman 截图 |
| 1.2 | 前端项目骨架跑通，首页渲染 | B | 浏览器截图 |
| 1.3 | API 接口文档 v1.0（Swagger 可访问） | A | E review 通过 |
| 1.4 | 数据库 ER 图 + 建表脚本 | A | 脚本可在 SQLite 执行 |
| 1.5 | LLM API Key 申请 + 测试调用通 | C | 控制台输出"你好，世界" |
| 1.6 | ChromaDB 安装 + 基础 CRUD 验证 | C | 脚本跑通 |
| 1.7 | 参考资料解析 PoC：读取 PDF 文本 | D | 打印前 200 字 |
| 1.8 | 知识库资料收集清单确认 | E | 清单文档 |

### Phase 2 — 核心逻辑与模型联调（Day 4 ~ Day 12，共 9 天）

| # | 交付物 | 负责人 | 验收标准 |
|---|--------|--------|----------|
| 2.1 | RAG 检索链路：录入→向量化→检索 | C | 检索结果 top3 正确 |
| 2.2 | 多轮对话引擎：意图提取 + 追问 + 确认 | C | 3 轮对话后能输出结构化意图 JSON |
| 2.3 | 知识融合模块：意图 + RAG + 参考资料 → 指令集 | C | 输出结构化课件指令 JSON |
| 2.4 | PPT 生成引擎：封面/目录/内容/总结页 | D | 用样例指令集生成 10 页 .pptx |
| 2.5 | Word 教案生成引擎 | D | 生成包含教学目标的 3 页 .docx |
| 2.6 | 动画/互动游戏生成（二选一） | D | HTML5 页面可交互 |
| 2.7 | 文件上传 API（PDF/Word/图片/视频） | A | 4 种格式上传成功 |
| 2.8 | 对话 API（SSE 流式） | A | Postman 验证流式输出 |
| 2.9 | 课件生成 API（异步任务 + 轮询） | A | 返回任务 ID → 轮询 → 获取下载链接 |
| 2.10 | 语音输入 API（audio → text） | C | 上传 .wav → 返回文本 |

### Phase 3 — 前端接入与联调测试（Day 10 ~ Day 16，共 7 天）

| # | 交付物 | 负责人 | 验收标准 |
|---|--------|--------|----------|
| 3.1 | 对话页面：文字输入 + 流式回复 | B | 完整多轮对话 |
| 3.2 | 语音输入组件 | B | 录音 → 上屏 |
| 3.3 | 文件上传 + 预览组件 | B | 拖拽上传 + 缩略图 |
| 3.4 | 课件预览页面（PPT/Word 在线预览） | B | 翻页浏览 |
| 3.5 | 修改反馈面板 + 再生按钮 | B | 输入修改意见 → 课件更新 |
| 3.6 | 导出下载功能 | B | 下载 .pptx / .docx |
| 3.7 | 全链路联调 + Bug 修复 | A+B+C+D | E 按测试用例逐条过 |

### Phase 4 — 文档与交付（Day 17 ~ Day 19，共 3 天）

| # | 交付物 | 负责人 | 验收标准 |
|---|--------|--------|----------|
| 4.1 | 项目概要介绍、方案文档 | E (收集) | 按竞赛要求格式 |
| 4.2 | 演示视频录制 | E + 全队 | 10 分钟内 |
| 4.3 | 本地知识库资料整理打包 | E | 文件清单 |
| 4.4 | 项目简介 PPT | E | 15 页以内 |
| 4.5 | 最终联调 + 演习 | 全队 | 完整走一遍 Demo 流程 |

> **总工期：19 天（含缓冲），实际开发 16 天。**

---

## 三、交互流转协议

### 3.1 核心数据流

```
[前端]                          [后端]                        [AI 模块]
  │                               │                              │
  │  POST /session/create         │                              │
  │  ──────────────────────────►  │                              │
  │  {teacher_name, subject}      │  返回 session_id             │
  │                               │                              │
  │  POST /chat/{session_id}      │                              │
  │  {text} 或 {audio_file}       │  ──► 转发意图理解模块 ──►    │
  │                               │      提取意图 + 生成追问     │
  │  ◄── SSE stream ──────────── │  ◄─────────────────────────  │
  │  {type: "question"/"confirm"} │                              │
  │                               │                              │
  │  POST /upload/{session_id}    │                              │
  │  multipart: file + ref_desc   │  ──► 存入文件系统 ───────►   │
  │                               │      多模态解析模块处理      │
  │                               │                              │
  │  POST /generate/{session_id}  │                              │
  │  (触发课件生成)               │  ──► 组装指令集 ──────────►  │
  │                               │      ├─ RAG 检索             │
  │                               │      ├─ 意图 + 参考融合      │
  │                               │      └─ 课件生成引擎         │
  │  ◄── {task_id} ───────────── │                              │
  │                               │                              │
  │  GET /task/{task_id}/status   │                              │
  │  ──────────────────────────►  │  {status, progress}          │
  │                               │                              │
  │  GET /download/{file_id}      │                              │
  │  ◄── .pptx / .docx / .html ──│                              │
```

### 3.2 数据格式规范

**意图理解输出 → 课件生成引擎（JSON）：**

```json
{
  "session_id": "s_001",
  "teaching_goal": "理解Python中列表与字典的区别",
  "target_audience": "大一计算机基础课",
  "duration_minutes": 45,
  "knowledge_points": [
    {"order": 1, "title": "列表定义", "difficulty": "basic", "key_points": ["..."]},
    {"order": 2, "title": "字典定义", "difficulty": "basic", "key_points": ["..."]}
  ],
  "logic_flow": ["概念引入", "对比分析", "代码演示", "课堂练习"],
  "style": {"tone": "生动", "case_preference": "校园生活"},
  "reference_materials": [
    {"file_id": "f_01", "type": "pdf", "use_segment": "第3章", "usage": "仿照例题格式"}
  ]
}
```

**课件生成引擎 → 文件系统 → 后端返回：**

```json
{
  "task_id": "task_001",
  "outputs": [
    {"type": "pptx", "file_id": "abc123", "pages": 12, "size_kb": 2048},
    {"type": "docx", "file_id": "def456", "words": 3000, "size_kb": 512},
    {"type": "html",  "file_id": "ghi789", "desc": "列表vs字典互动小游戏"}
  ]
}
```

### 3.3 API 命名约定

全部使用 RESTful 风格，名词复数，小写 + 下划线分隔：

```
GET    /api/v1/sessions/{id}        # 查询会话
POST   /api/v1/sessions             # 创建会话
POST   /api/v1/sessions/{id}/chat   # 对话（SSE）
POST   /api/v1/files/upload         # 上传文件
GET    /api/v1/files/{id}           # 获取文件信息
POST   /api/v1/generate             # 触发生成任务
GET    /api/v1/tasks/{id}/status    # 查询任务状态
GET    /api/v1/download/{file_id}   # 下载课件
```

---

## 四、团队工程规范（极重要）

### 4.1 代码命名铁律

> **原则：极简、专业、见名知意。禁止 AI 味冗长命名。**

| 场景 | ✅ 正确 | ❌ 错误（禁止） |
|------|---------|-----------------|
| 函数 | `parse_pdf()` | `parse_the_uploaded_pdf_file_and_extract_text()` |
| 函数 | `gen_ppt()` | `generate_and_return_the_complete_ppt_presentation()` |
| 变量 | `session_id` | `the_current_session_identifier_of_the_user` |
| 类 | `PPTGenerator` | `PowerPointPresentationGeneratingClass` |
| API 路径 | `/chat` | `/sendMessageToAIAndGetResponse` |
| 文件 | `parser.py` | `file_parsing_utility_functions.py` |
| 目录 | `rag/` | `retrieval_augmented_generation_engine/` |

**命名风格：**
- Python：`snake_case`（函数/变量）、`PascalCase`（类）
- Vue/JS：`camelCase`（变量/函数）、`PascalCase`（组件）
- 文件名：`snake_case.py` / `PascalCase.vue`
- 禁止拼音、禁止拼音英文混用

### 4.2 Git 分支策略

```
main                 # 生产分支，只接受 merge，禁止直接 push
├── dev              # 开发主分支，日常集成
│   ├── feat/api-xxx       # 后端功能分支
│   ├── feat/ui-xxx        # 前端功能分支
│   ├── feat/ai-xxx        # AI 功能分支
│   └── fix/xxx            # Bug 修复分支
```

**规则：**
1. 每人从 `dev` 切出自己的功能分支，命名：`feat/<模块>-<简述>`（如 `feat/api-chat-sse`）
2. 每天至少 `commit` 一次，`push` 到远端
3. 功能完成后提 PR 到 `dev`，至少 1 人 review 后才能 merge
4. **绝对禁止** `git push --force` 到共享分支
5. Commit message：`动作: 简述`（如 `feat: 完成对话SSE接口`、`fix: 修复文件上传中文乱码`）

### 4.3 沟通与阻塞升级机制

| 阻塞时长 | 动作 |
|----------|------|
| ≤ 30 分钟 | 自查文档 + Google + 问 LLM |
| 30 分钟 ~ 2 小时 | 在团队群抛出问题，@对应方向同学 |
| 超过 2 小时 | 立即找 PM（成员E），PM 组织快速对会（15 分钟内响应） |
| 超过半天 | 全队站会同步，调整任务分配 |

**每日站会：** 21:00（线上，15 分钟），每人三句话：昨天做了什么 / 今天做什么 / 有无阻塞。

---

## 五、角色分工总览

| 成员 | 角色 | 核心职责 | 技术栈 |
|------|------|----------|--------|
| A | 后端工程师 | API 服务、数据库、文件系统、系统集成 | FastAPI + SQLAlchemy + SQLite |
| B | 前端工程师 | 交互界面、对话、文件上传、预览、导出 | Vue 3 + Element Plus + Vite |
| C | AI 工程师（理解） | RAG 知识库、意图理解、知识融合、语音 | LangChain + ChromaDB + Whisper |
| D | AI 工程师（生成） | PPT/Word 生成、多模态解析、动画/游戏 | python-pptx + python-docx + PyMuPDF |
| E | PM / 测试 / 数据 | 进度管理、测试验收、数据收集、文档、Demo | — |

---

## 六、知识库资料清单（初始）

| 资料类型 | 内容示例 | 数量建议 | 负责人 |
|----------|----------|----------|--------|
| 教材 PDF | 大学计算机基础、Python 程序设计 | 2-3 本 | E |
| 教案 Word | 优秀教师教案样例 | 3-5 份 | E |
| 教学视频 | 精品课录播片段 | 2-3 段 | E |
| 课件 PPT | 获奖课件参考 | 3-5 个 | E |
| 试题库 | 选择题、编程题 | 20+ 题 | E |

---

*本手册由 PM（成员E）维护更新，任何变更需同步到团队群。*
