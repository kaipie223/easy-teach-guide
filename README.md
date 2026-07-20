# 多模态AI互动式教学智能体

> 省服务外包大赛 - 智能计算 · 应用类 | 锐捷网络
> 开发周期：2026年7月22日 → 8月22日

## 项目简介

开发以教师教学思路为核心的 AI 互动式教学智能体，支持：
- **智能对话**：多轮对话深度理解教学意图，主动追问、确认
- **多模态解析**：解析 PDF/Word/视频/图片等参考资料
- **课件自动生成**：一键生成 PPT + Word 教案 + HTML5 互动游戏
- **迭代优化**：预览 → 反馈 → 再生成的闭环

## 技术栈

| 层级 | 技术 |
|------|------|
| 前端 | Vue 3 + Vite + Element Plus |
| 后端 | FastAPI (Python 3.11) |
| 数据库 | SQLite |
| LLM | DeepSeek-V3 |
| 向量库 | ChromaDB |
| RAG | LangChain |
| 语音 | faster-whisper |
| PPT/Word | python-pptx / python-docx |

## 快速启动

### 后端
```bash
# 安装依赖
poetry install

# 配置环境变量（复制 .env.example 为 .env，填入 API Key）
cp .env.example .env

# 启动后端
poetry run python -m backend.main
# → http://localhost:8000/docs
```

### 前端
```bash
cd frontend
pnpm install
pnpm dev
# → http://localhost:5173
```

### 知识库构建
```bash
# 将资料放入 knowledge-base/ 目录
poetry run python ai/build_kb.py
```

## 项目结构

```
├── backend/          # 后端 (FastAPI)
├── frontend/         # 前端 (Vue 3)
├── ai/               # AI 模块 - 意图理解 + RAG + 语音
├── gen/              # 生成模块 - PPT/Word/动画 + 多模态解析
├── knowledge-base/   # 知识库原始资料
├── docs/             # 全部文档
└── output/           # 生成的课件文件
```

## 团队

| 姓名 | 角色 |
|------|------|
| 姜文杰 | 后端工程师 |
| 潘卓然 | 前端工程师 |
| 陈澜 | AI工程师（意图理解 + RAG + 语音） |
| 赵钰洁 | AI工程师（课件生成 + 多模态解析） |
| 陶克钦 | PM / 测试 / 数据 |

## 文档索引

| 文档 | 用途 |
|------|------|
| `docs/00_总执行手册_Master_Plan.md` | 架构、里程碑、工程规范 |
| `docs/01_技术协议与接口规范_Technical_Protocol.md` | 接口合同 |
| `docs/09_功能模块划分与开发总览.md` | 7 个功能模块详细定义 |
| `docs/10_任务管理表_飞书导入.md` | 80 条任务，可导入飞书 |
