# 成员D · AI工程师（课件生成 + 多模态解析）— 任务说明

> **你的职责：** 系统"双手"。两个方向：① 输入 `GenerationInstruction` → 生成 .pptx / .docx / .html 课件文件 ② 解析教师上传的参考资料（PDF/Word/图片/视频）→ 提取文本供 C 的融合模块使用。
>
> **代码获取：** 具体代码片段见 `docs/08_代码速查手册_Code_Cheatsheet.md` 第四节（D · 课件生成代码）。
>
> **关键约束：** 你的函数被 A 和 C 直接 import。函数签名必须严格按协议文档第 4.2 节。

---

## 你需要创建的全部文件

```
gen/
├── pptx/
│   ├── generator.py          ← gen_ppt(instruction, template, output_dir) → Path
│   ├── slides.py             ← 4 种页面模板：add_cover / add_toc / add_content / add_summary
│   └── layout.py             ← 样式常量：颜色/字体/字号/边距
├── docx/
│   ├── generator.py          ← gen_doc(instruction, output_dir) → Path
│   └── sections.py           ← 6 个教案板块模板
├── anim/
│   └── generator.py          ← gen_animation(instruction, anim_type, output_dir) → Path
├── parse/
│   ├── pdf.py                ← extract_pdf_text(pdf_path) → str
│   ├── doc.py                ← extract_doc_text(docx_path) → str
│   ├── image.py              ← describe_image(image_path, api_key) → str
│   └── video.py              ← extract_keyframes + summarize_video(video_path, api_key) → str
├── prompts/
│   ├── ppt_outline.txt       ← PPT 大纲生成 Prompt
│   ├── ppt_content.txt       ← PPT 单页内容 Prompt
│   ├── doc_plan.txt          ← 教案内容生成 Prompt
│   └── anim_idea.txt         ← 动画/游戏创意 Prompt
└── test_gen.py               ← PPT 生成自测脚本
```

---

## Phase 1：环境搭建与解析验证（Day 1-3）

---

### Day 1 — 依赖验证 + PDF 解析 + 测试 PPT

#### 任务 1.1：验证所有依赖

**要求：** 逐个 import 验证以下库，全部通过后截图：
- `from pptx import Presentation` → python-pptx
- `from docx import Document` → python-docx
- `import fitz` → PyMuPDF（PDF 解析）
- `import cv2` → OpenCV（视频处理）
- `from PIL import Image` → Pillow（图片处理）

还需要安装 FFmpeg（视频处理依赖）：从 ffmpeg.org 下载，加入系统 PATH。

---

#### 任务 1.2：PDF 解析

**要求：** `gen/parse/pdf.py`
- `extract_pdf_text(pdf_path)` → `str`：用 PyMuPDF 打开 PDF → 逐页 `get_text("text")` → 过滤空页 → 用 `\n\n` 拼接返回
- `extract_pdf_images(pdf_path, output_dir)` → `list[str]`：逐页提取嵌入图片 → 保存为 PNG → 返回路径列表
- 用 C 找的教材 PDF 测试：打印前 500 字 → 中文正常、段落结构保留

**验证：** 教材 PDF 能提取出正确的中文文本，截图发群。

---

#### 任务 1.3：手写测试 PPT

**要求：** `gen/test_gen.py`
- 手写一个 3 页 PPT（不用 LLM，纯 python-pptx 代码）：
  - 第 1 页：封面（标题 "Python程序设计入门" + 副标题）
  - 第 2 页：目录（4 个条目）
  - 第 3 页：内容页（标题 + 4 个要点）
- 设置 16:9 尺寸（`Inches(13.333)` × `Inches(7.5)`）
- 用 PowerPoint 打开确认中文正常（不是方框□□□）
- 如果中文变方框 → 字体问题，改 `layout.py` 中的 `FONT_TITLE = "微软雅黑"` → 如果系统没有微软雅黑 → 换成 `"SimHei"` 或 `"宋体"`

**验证：** 生成的 .pptx 在 PowerPoint 中打开，3 页完整，中文正常，截图发群。

---

### Day 2 — 多模态解析完善

#### 任务 2.1：Word 解析

**要求：** `gen/parse/doc.py`
- `extract_doc_text(docx_path)` → `str`
- 用 python-docx 读取段落 → 如果段落样式以 "Heading" 开头 → 用 `#` 标记标题层级 → 普通段落保留原文
- 段落间用 `\n\n` 分隔
- 测试：用 E 给的教案 Word 文件 → 打印输出 → 确认标题层级正确

---

#### 任务 2.2：图片描述

**要求：** `gen/parse/image.py`
- `describe_image(image_path, api_key=None)` → `str`
- 将图片 base64 编码 → 调 DeepSeek（或 GPT-4V）的视觉能力 → 返回图片描述
- Prompt 需要引导模型关注：图表、公式、教学结构、文字内容
- 如果 DeepSeek 不支持图片 → 降级方案：用 OCR（PaddleOCR）提取图片中文字 + 记录图片尺寸和文件名
- 测试：用一张含文字的截图 → 返回描述包含图中文字

---

#### 任务 2.3：视频解析

**要求：** `gen/parse/video.py`
- `extract_keyframes(video_path, interval_sec=10, output_dir="./frames")` → `list[str]`：
  - 用 OpenCV 读取视频 → 每隔 `interval_sec` 秒提取一帧 → 保存为 JPG → 返回路径列表
- `summarize_video(video_path, api_key)` → `str`：
  - 调 `extract_keyframes(interval_sec=30)` → 取前 6 帧 → 逐帧调 `describe_image` → 汇总描述 → 调 LLM 生成 200 字以内的视频摘要
  - 如果视频无法打开 → 返回 "无法解析视频内容"（不要抛异常）

---

### Day 3 — PPT 模板系统

#### 任务 3.1：样式常量

**要求：** `gen/pptx/layout.py`
- 定义画布尺寸：`SLIDE_W = Inches(13.333)` / `SLIDE_H = Inches(7.5)`（16:9）
- 定义 6 个颜色：主色（蓝 `#1A73E8`）、辅助（绿 `#34A853`）、强调（黄 `#FBBC04`）、正文（深灰 `#202124`）、副文（浅灰 `#5F6368`）、背景（白 / 封面蓝）
- 定义 5 个字号：封面标题 44pt / 封面副标题 24pt / 页面标题 32pt / 段落标题 28pt / 正文 18pt / 小字 14pt
- 定义边距：左 1.2inch / 右 1.2inch / 上 0.6inch / 内容宽度 10.8inch
- 字体统一用 `微软雅黑`

---

#### 任务 3.2：4 种页面模板

**要求：** `gen/pptx/slides.py`

**`add_cover(prs, title, subtitle, author)`：**
- 蓝色全幅背景 + 白色大标题（44pt 加粗）+ 副标题（24pt）+ 作者（14pt）
- 左对齐，标题垂直居中偏上

**`add_toc(prs, items)`：**
- 左侧蓝色竖条装饰 + "目 录"标题 + 编号目录项（01 xxx / 02 xxx ...）
- 每项 28pt，项间距适当

**`add_content(prs, title, bullet_points, image_path=None)`：**
- 顶部蓝色标题（32pt）+ 黄色分隔线 + 要点列表（18pt，项目符号 •）
- 如果有配图 → 右侧 4inch 宽放图，左侧 6.5inch 放文字
- 要点间 8pt 间距

**`add_summary(prs, key_points)`：**
- 顶部蓝色装饰条 + "📝 课堂小结"标题 + ✓ 标记的要点列表
- 最多展示 8 条

---

#### 任务 3.3：验证模板

**要求：** 用 4 个模板函数改写 `test_gen.py` → 生成新 PPT → PowerPoint 打开确认排版正确 → 截图发群。

---

## Phase 2：课件生成引擎（Day 4-12）

---

### Day 4 — PPT 大纲生成 + Prompt 模板

#### 任务 4.1：PPT 大纲 Prompt

**要求：** `gen/prompts/ppt_outline.txt`
- 输入：`{instruction_json}`（GenerationInstruction 的 JSON）
- 输出：每页的结构定义 JSON `{"slides": [{"type": "content"/"section"/"example", "title": "...", "points": ["..."]}, ...]}`
- 页数规则：按 `duration_minutes` 计算，每 5 分钟 ≈ 1-2 页内容
- 必须覆盖 instruction 中所有 knowledge_points

---

#### 任务 4.2：PPT 单页内容 Prompt

**要求：** `gen/prompts/ppt_content.txt`
- 输入：某页的 title + knowledge_point 详情
- 输出：该页的 bullet_points（3-5 个要点，每个 15-30 字，适合 PPT 展示）

---

#### 任务 4.3：PPT Generator 框架

**要求：** `gen/pptx/generator.py`
- 调 LLM 生成大纲 JSON → 创建 Presentation → 封面 → 目录 → 内容页（循环）→ 总结页 → 保存
- 文件名：`{teaching_goal[:30]}.pptx`（截断到 30 字符，避免路径过长）
- 输出目录默认 `./output`，不存在则自动创建

**测试：** 用假 `GenerationInstruction`（Python 列表课，45 分钟，3 个知识点）→ 生成 .pptx → 确认 ≥ 10 页 → 用 PowerPoint 打开检查内容。

---

### Day 5 — PPT 生成完善

#### 任务 5.1：PPT 生成稳定性

**要求：**
- 同一指令生成 3 次 → 3 次都成功（不能有偶发失败）
- 中文不出现方框 → 如果出现，将字体 fallback 设为 SimHei
- 内容页要点不超出文本框 → 如果 LLM 返回的要点太长 → 截断到 30 字/条
- LLM 返回 JSON 格式错误 → 重试 3 次 → 仍失败则返回错误信息

---

#### 任务 5.2：添加页码

**要求：**
- 每页右下角添加页码（格式：`01 / 15`）
- 封面和目录不显示页码

---

### Day 6 — 交付 gen_ppt + 开始 Word 教案

#### 任务 6.1：gen_ppt 正式交付给 A（关键交付节点）

**要求：**
- 确认函数签名正确：`gen_ppt(instruction: GenerationInstruction, template: str = "default", output_dir: str = "./output") -> Path`
- 确认返回值是 `.pptx` 文件的 `Path` 对象
- 让 A 跑 `from gen.pptx.generator import gen_ppt` → 确认能 import
- 用 A 那边真实的 `GenerationInstruction` 测试一次 → 确认生成成功

---

#### 任务 6.2：Word 教案 Prompt

**要求：** `gen/prompts/doc_plan.txt`
- 输入：`{instruction_json}`
- 输出：6 个板块的内容 JSON：
  - objectives（教学目标，三维目标格式）
  - key_difficulties（教学重难点）
  - procedure（教学过程，按 logic_flow 详细展开）
  - methods（教学方法）
  - activities（课堂活动设计）
  - homework（课后作业与评价）

---

#### 任务 6.3：Word Generator 框架

**要求：** `gen/docx/generator.py`
- 调 LLM 生成教案内容 JSON → 创建 Document → 标题（居中）→ 6 个板块（一级标题 + 正文）→ 保存
- `gen/docx/sections.py`：每个板块的默认文本模板和样式
- 文件名：`{teaching_goal[:30]}_教案.docx`

---

### Day 7 — Word 教案完善 + 交付 gen_doc

#### 任务 7.1：Word 教案完善

**要求：**
- 6 个板块必须全部有内容（如果 LLM 某字段返回空 → 用默认模板填充）
- 字体：标题 16pt 加粗 / 正文 12pt / 行距 1.5 倍
- 页边距：上下 2.54cm / 左右 3.18cm（标准 A4）
- 测试：用假指令生成教案 → Word 打开 → 确认结构完整、排版正常

---

#### 任务 7.2：交付 gen_doc 给 A

**要求：**
- 确认函数签名：`gen_doc(instruction: GenerationInstruction, output_dir: str = "./output") -> Path`
- 让 A 跑 import 确认
- 让 A 用真实指令测试一次

---

### Day 8 — 动画/游戏生成 + 交付 gen_animation

#### 任务 8.1：动画 Prompt

**要求：** `gen/prompts/anim_idea.txt`
- 输入：`{instruction_json}` + `{anim_type}`（"quiz_game" 或 "concept_anim"）
- 输出：`{"title": "...", "html_code": "<!DOCTYPE html>..."}`
- quiz_game：知识点闯关游戏（单选题，答对前进，答错提示）
- concept_anim：概念可视化动画（如列表 vs 字典的内存模型对比动画）
- html_code 必须是完整的、自包含的 HTML（不依赖外部 CDN，或只用常用 CDN）

---

#### 任务 8.2：动画 Generator

**要求：** `gen/anim/generator.py`
- 调 LLM 生成 HTML 代码 → 写入 .html 文件 → 返回路径
- LLM 参数：`max_tokens=4000`（HTML 代码较长）
- 文件名：`{teaching_goal[:20]}_互动.html`
- 生成后用浏览器打开测试：确认可交互、样式正常、中文不乱码

---

#### 任务 8.3：交付 gen_animation 给 A

**要求：**
- 确认函数签名：`gen_animation(instruction: GenerationInstruction, anim_type: str = "quiz_game", output_dir: str = "./output") -> Path`
- 让 A 跑 import 确认
- 这是最后一个需要交付给 A 的函数，交付后 A 的 orchestrator 完整

---

### Day 9 — 课件质量打磨

#### 任务 9.1：PPT 模板扩展

**要求：**
- 增加第 2 套配色方案：`template="academic"`（学术蓝）vs `template="colorful"`（活力橙）
- 在 `layout.py` 中增加 COLOR_ACCENT_2、COLOR_BG_DARK_2 等变量
- 两套方案各生成一份对比 PPT → 截图发给 E 选优

---

#### 任务 9.2：Word 教案字数自适配

**要求：**
- `duration_minutes` ≤ 30 分钟 → 教案简洁版（3-4 页）
- 30-60 分钟 → 标准版（5-7 页）
- > 60 分钟 → 详细版（8-10 页）
- 通过在 Prompt 中指定字数控制

---

#### 任务 9.3：动画兼容性

**要求：**
- 生成 3 个不同主题的动画 → 分别在 Chrome / Edge 中打开测试
- 确认不依赖被墙的 CDN（如 Google Fonts）
- 如果动画用了外部库 → 改用国内 CDN（如 bootcdn）或内联

---

### Day 10 — 解析模块稳定性 + 课件自动适配

#### 任务 10.1：解析模块稳定性

**要求：**
- PDF 解析：测试 5 本不同来源的 PDF（教材/论文/扫描版/加密/超大文件）→ 记录每种情况的结果
- 扫描版 PDF 会返回空文本 → 在函数中检测并提示 "该PDF为扫描版，无法提取文字"
- 视频解析：测试 3 种格式（MP4/AVI/MOV）→ 确认都能提取关键帧

---

#### 任务 10.2：课件自动适配

**要求：**
- PPT 页数 = `duration_minutes / 5 + 2`（封面+总结），最少 6 页，最多 30 页
- 动画类型自动选择：`logic_flow` 含 "练习" → quiz_game；含 "演示" → concept_anim
- Word 教案中如果 `reference_materials` 有教材 → 在 "教学过程" 板块引用教材章节

---

### Day 11 — Demo 课件制作 + 稳定性测试

#### 任务 11.1：Demo 课件

**要求：** 准备 3 套完整 Demo 课件，对应的教学场景：
1. **入门课**：Python 列表基础 → 大一新生 → 45 分钟
2. **进阶课**：面向对象三大特性 → 大二 → 90 分钟
3. **复习课**：数据结构期末复习 → 大三 → 60 分钟

每套包含：.pptx（≥10页）+ .docx（≥3页）+ .html（可交互）→ 发给 E。

---

#### 任务 11.2：稳定性测试

**要求：**
- 同一指令生成 PPT 10 次 → 成功率 ≥ 95%
- 如果某次失败 → 记录失败原因（LLM 返回格式异常 / 文件写入失败 / 其他）→ 针对性修复
- PPT 文件大小优化：如果 > 10MB → 检查是否有超大图片 → 压缩或使用占位符

---

### Day 12 — Phase 2 收尾

**要求：**
1. 自查四个模块（PPT / Word / 动画 / 解析）全部稳定
2. 写 `gen/README.md`：各模块用途、调用方式、测试方法
3. 确保全部函数能被 A 正常 import
4. 3 套 Demo 课件最终确认 → 发给 E

---

## Phase 3：联调测试（Day 13-16）

### Day 13 — 课件质量回归

**要求：**
- 修复 E 测试中发现的所有 PPT/Word/动画问题
- PPT 文件大小优化（如超过 10MB → 压缩）
- 动画在所有目标浏览器中验证通过

---

### Day 14-15 — 彩排配合

**要求：**
- 配合 E 的彩排 → 确保课件生成稳定
- 准备备用课件（如果生成失败 → 用预先生成好的代替）
- 最终确认 3 套 Demo 课件
- 代码 freeze

---

### Day 16 — 最终检查

**要求：**
- 确认全部函数可被 A import
- 确认生成环境依赖完整（特别是 FFmpeg）
- 写课件模块交接文档

---

## Phase 4：文档与交付（Day 17-19）

### Day 17-18 — 提供文档素材

**要求：**
- 画课件生成流程图（指令集 → LLM大纲 → PPT/Word/动画生成 → 文件输出）
- 画多模态解析流程图（PDF/Word/图片/视频 → 各解析函数 → 文本输出）
- 提供 PPT/Word/动画的高质量截图
- 审查 E 写的课件技术章节

---

### Day 19 — 最终检查

**要求：**
- 代码全部 push
- 3 套 Demo 课件归档
- FFmpeg 路径等注意事项写在 README 中
- 参与最终评审

---

## 依赖关系

### 你需要从谁那里拿到什么

| 从谁 | 什么 | 何时 | 用于什么 |
|------|------|------|---------|
| C | DeepSeek API Key（.env） | Day 1 | 课件生成调用 LLM / 图片描述 / 视频摘要 |
| A | `backend/schemas.py` → `GenerationInstruction` | Day 2 晚 | gen_ppt / gen_doc / gen_animation 的输入类型 |
| E | 教案样例 Word / 教材 PDF | Day 2 | 解析函数测试素材 |
| E | 课件样式偏好 + 测试用指令集 | Day 8 | 课件生成测试 |
| E | Demo 演示场景需求 | Day 11 | 3 套 Demo 课件 |

### 你需要交给谁什么

| 交给谁 | 什么 | 何时 |
|--------|------|------|
| C | `extract_pdf_text` / `extract_doc_text` / `summarize_video` / `describe_image` | Day 2 晚 |
| A | `gen_ppt()` | Day 6 晚 |
| A | `gen_doc()` | Day 7 晚 |
| A | `gen_animation()` | Day 8 晚 |
| E | 3 套 Demo 课件 | Day 11 |
| E | 课件模块交接文档 | Day 19 |
