# 04 — 成员D · AI 工程师（课件生成 + 多模态解析）

> **你是系统的"双手"。C 产出指令集，你把它变成真实的 .pptx / .docx / .html 文件。**
> 核心职责边界：**输入 GenerationInstruction → 输出可直接下载的标准格式课件文件。**

---

## 你需要产出的文件（完整清单）

```
gen/
├── __init__.py
├── pptx/
│   ├── __init__.py
│   ├── generator.py              # gen_ppt() 函数 — A 直接 import
│   ├── slides.py                 # 4 种页面模板函数
│   └── layout.py                 # 配色、字体、间距常量
├── docx/
│   ├── __init__.py
│   ├── generator.py              # gen_doc() 函数 — A 直接 import
│   └── sections.py               # 6 个教案板块模板
├── anim/
│   ├── __init__.py
│   └── generator.py              # gen_animation() 函数 — A 直接 import
├── parse/
│   ├── __init__.py
│   ├── pdf.py                    # extract_pdf_text() — C 会 import
│   ├── doc.py                    # extract_doc_text() — C 会 import
│   ├── image.py                  # describe_image() — C 会 import
│   └── video.py                  # summarize_video() — C 会 import
├── prompts/
│   ├── ppt_outline.txt           # PPT 大纲生成 Prompt
│   ├── ppt_content.txt           # PPT 单页内容生成 Prompt
│   ├── doc_plan.txt              # 教案内容生成 Prompt
│   └── anim_idea.txt             # 动画/游戏创意 Prompt
└── test_gen.py                   # 你自用的测试脚本
```

---

## 你与 A 和 C 的精确对接

```
A 的 backend/services/orchestrator.py 会这样 import 你的代码:

  from gen.pptx.generator import gen_ppt          ← 你必须提供这个函数
  from gen.docx.generator import gen_doc           ← 你必须提供这个函数
  from gen.anim.generator import gen_animation     ← 你必须提供这个函数

C 的 ai/fusion/fusion.py 会这样 import 你的代码:

  from gen.parse.pdf import extract_pdf_text      ← 你必须提供这个函数
  from gen.parse.doc import extract_doc_text      ← 你必须提供这个函数
  from gen.parse.video import summarize_video     ← 你必须提供这个函数
  from gen.parse.image import describe_image      ← 你必须提供这个函数

你和 A 和 C 共享同一个数据模型:

  from backend.schemas import GenerationInstruction
  # ⚠️ gen_ppt / gen_doc / gen_animation 的 instruction 参数就是这个类型
```

---

## 你的技术清单

| 需求 | 安装 |
|------|------|
| Python 依赖 | `poetry install`（A 的 pyproject.toml 已包含你的全部依赖） |
| FFmpeg | 从 ffmpeg.org 下载，加入 PATH（视频处理需要） |
| DeepSeek API Key | 和 C 共用同一个 Key（从 `.env` 读取） |

---

## Phase 1：环境搭建 + 解析验证（Day 1-3）

### Day 1 — 今天必须完成

**任务 1.1：验证依赖安装 + 创建目录结构（20 分钟）**

```bash
cd "C:\Users\kaipie\Desktop\省服务外包大赛\多模态AI教学系统"
poetry install
mkdir -p gen/pptx gen/docx gen/anim gen/parse gen/prompts

# 验证每个关键依赖
poetry run python -c "from pptx import Presentation; print('python-pptx OK')"
poetry run python -c "from docx import Document; print('python-docx OK')"
poetry run python -c "import fitz; print('PyMuPDF OK')"
poetry run python -c "import cv2; print('OpenCV OK:', cv2.__version__)"
poetry run python -c "from PIL import Image; print('Pillow OK')"
```

**任务 1.2：PDF 解析验证（15 分钟）**

```python
# gen/parse/pdf.py
import fitz  # PyMuPDF

def extract_pdf_text(pdf_path: str) -> str:
    """提取 PDF 全部文本，保留段落结构"""
    doc = fitz.open(pdf_path)
    texts = []
    for page in doc:
        text = page.get_text("text")
        if text.strip():
            texts.append(text.strip())
    doc.close()
    return "\n\n".join(texts)

def extract_pdf_images(pdf_path: str, output_dir: str) -> list[str]:
    """提取 PDF 中嵌入的图片，保存到 output_dir，返回路径列表"""
    import os
    os.makedirs(output_dir, exist_ok=True)
    doc = fitz.open(pdf_path)
    paths = []
    for i, page in enumerate(doc):
        for j, img in enumerate(page.get_images(full=True)):
            xref = img[0]
            pix = fitz.Pixmap(doc, xref)
            if pix.n < 5:  # GRAY or RGB
                path = f"{output_dir}/page{i+1}_img{j+1}.png"
                pix.save(path)
                paths.append(path)
            pix = None
    doc.close()
    return paths
```

```bash
# 测试 PDF 解析 — 用 C 找的那本教材 PDF 测试
poetry run python -c "
from gen.parse.pdf import extract_pdf_text
text = extract_pdf_text('knowledge-base/教材PDF/你的教材.pdf')
print(text[:500])
print(f'总字数: {len(text)}')
"
# 看到中文正常输出，截图发群
```

**任务 1.3：PPT 生成验证（15 分钟）**

```python
# gen/test_gen.py — Day 1 测试用
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.enum.text import PP_ALIGN
from pptx.dml.color import RGBColor

def quick_test_ppt():
    """手写一个 3 页 PPT，验证排版正常"""
    prs = Presentation()
    prs.slide_width  = Inches(13.333)   # 16:9
    prs.slide_height = Inches(7.5)

    # 封面
    slide = prs.slides.add_slide(prs.slide_layouts[6])  # blank
    from pptx.util import Inches, Pt
    left, top, width, height = Inches(1), Inches(2.5), Inches(11), Inches(1.5)
    txBox = slide.shapes.add_textbox(left, top, width, height)
    tf = txBox.text_frame
    tf.text = "Python程序设计入门"
    tf.paragraphs[0].font.size = Pt(44)
    tf.paragraphs[0].font.bold = True
    tf.paragraphs[0].font.color.rgb = RGBColor(0x1a, 0x73, 0xe8)

    # 目录
    slide2 = prs.slides.add_slide(prs.slide_layouts[6])
    items = ["1. Python简介", "2. 列表基础", "3. 字典基础", "4. 对比与实战"]
    txBox2 = slide2.shapes.add_textbox(Inches(2), Inches(1.5), Inches(9), Inches(4.5))
    tf2 = txBox2.text_frame
    tf2.text = "目  录"
    tf2.paragraphs[0].font.size = Pt(36)
    for item in items:
        p = tf2.add_paragraph()
        p.text = item
        p.font.size = Pt(24)
        p.space_after = Pt(12)

    # 内容页
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
    print("✅ test_output.pptx 已生成，用 PowerPoint 打开检查")

if __name__ == "__main__":
    quick_test_ppt()
```

```bash
poetry run python gen/test_gen.py
# 用 PowerPoint 打开 test_output.pptx → 中文正常、排版整齐 → 截图发群
```

---

### Day 2 — 多模态解析完善

**任务 2.1：Word 解析**

```python
# gen/parse/doc.py
from docx import Document

def extract_doc_text(docx_path: str) -> str:
    """提取 Word 文档文本和标题结构"""
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

**任务 2.2：图片解析**

```python
# gen/parse/image.py
import base64, os
from openai import OpenAI
from dotenv import load_dotenv
load_dotenv()

def describe_image(image_path: str, api_key: str = None) -> str:
    """调用 DeepSeek Vision（或 GPT-4V）描述图片内容"""
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

**任务 2.3：视频解析**

```python
# gen/parse/video.py
import cv2
import os

def extract_keyframes(video_path: str, interval_sec: int = 10, output_dir: str = "./frames") -> list[str]:
    """每 interval_sec 秒提取一帧，返回帧图片路径列表"""
    os.makedirs(output_dir, exist_ok=True)
    cap = cv2.VideoCapture(video_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    interval_frames = int(fps * interval_sec)

    paths = []
    frame_count = 0
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        if frame_count % interval_frames == 0:
            path = f"{output_dir}/frame_{frame_count:06d}.jpg"
            cv2.imwrite(path, frame)
            paths.append(path)
        frame_count += 1
    cap.release()
    return paths

def summarize_video(video_path: str, api_key: str) -> str:
    """提取关键帧 → 逐帧描述 → 汇总生成视频摘要"""
    frames = extract_keyframes(video_path, interval_sec=30)  # 每30秒一帧，减少帧数
    from gen.parse.image import describe_image

    descriptions = []
    for fp in frames[:6]:  # 最多取6帧
        try:
            desc = describe_image(fp, api_key)
            descriptions.append(desc)
        except:
            pass

    # 汇总
    if descriptions:
        from openai import OpenAI
        import os
        client = OpenAI(api_key=api_key, base_url=os.getenv("DEEPSEEK_BASE_URL"))
        resp = client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "user", "content": f"以下是教学视频中不同时间段的画面描述，请汇总为一个 200 字以内的视频摘要：\n" + "\n---\n".join(descriptions)}],
            max_tokens=300
        )
        return resp.choices[0].message.content
    return "无法解析视频内容"
```

---

### Day 3 — PPT 模板 + 布局常量

**任务 3.1：`layout.py` — 统一样式常量**

```python
# gen/pptx/layout.py
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN

# 画布尺寸
SLIDE_W = Inches(13.333)
SLIDE_H = Inches(7.5)

# 颜色
COLOR_PRIMARY   = RGBColor(0x1A, 0x73, 0xE8)   # 主色：蓝
COLOR_SECONDARY = RGBColor(0x34, 0xA8, 0x53)   # 辅助：绿
COLOR_ACCENT    = RGBColor(0xFB, 0xBC, 0x04)   # 强调：黄
COLOR_TEXT      = RGBColor(0x20, 0x21, 0x24)   # 正文：深灰
COLOR_SUBTEXT   = RGBColor(0x5F, 0x63, 0x68)   # 副文：浅灰
COLOR_BG        = RGBColor(0xFF, 0xFF, 0xFF)   # 背景：白
COLOR_BG_DARK   = RGBColor(0x1A, 0x73, 0xE8)   # 封面背景：蓝

# 字体
FONT_TITLE = "微软雅黑"
FONT_BODY  = "微软雅黑"

# 字号
SIZE_COVER_TITLE    = Pt(44)
SIZE_COVER_SUBTITLE = Pt(24)
SIZE_SLIDE_TITLE    = Pt(32)
SIZE_SECTION_TITLE  = Pt(28)
SIZE_BODY           = Pt(18)
SIZE_BODY_SMALL     = Pt(14)

# 边距
MARGIN_LEFT   = Inches(1.2)
MARGIN_RIGHT  = Inches(1.2)
MARGIN_TOP    = Inches(0.6)
CONTENT_WIDTH = Inches(10.8)

# 行距
LINE_SPACING = Pt(30)
```

**任务 3.2：`slides.py` — 4 种页面模板**

```python
# gen/pptx/slides.py
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.enum.text import PP_ALIGN
from .layout import *

def add_cover(prs: Presentation, title: str, subtitle: str = "", author: str = ""):
    """封面页：大标题 + 副标题 + 作者"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])  # blank layout

    # 蓝色背景
    bg = slide.background
    fill = bg.fill
    fill.solid()
    fill.fore_color.rgb = COLOR_BG_DARK

    # 标题
    txBox = slide.shapes.add_textbox(MARGIN_LEFT, Inches(2.2), CONTENT_WIDTH, Inches(1.8))
    tf = txBox.text_frame
    tf.word_wrap = True
    p = tf.paragraphs[0]
    p.text = title
    p.font.size = SIZE_COVER_TITLE
    p.font.bold = True
    p.font.color.rgb = COLOR_BG
    p.font.name = FONT_TITLE
    p.alignment = PP_ALIGN.LEFT

    # 副标题
    if subtitle:
        p2 = tf.add_paragraph()
        p2.text = subtitle
        p2.font.size = SIZE_COVER_SUBTITLE
        p2.font.color.rgb = RGBColor(0xE8, 0xF0, 0xFE)
        p2.font.name = FONT_BODY
        p2.space_before = Pt(12)

    # 作者
    if author:
        txBox2 = slide.shapes.add_textbox(MARGIN_LEFT, Inches(6.0), CONTENT_WIDTH, Inches(0.6))
        tf2 = txBox2.text_frame
        tf2.paragraphs[0].text = author
        tf2.paragraphs[0].font.size = SIZE_BODY_SMALL
        tf2.paragraphs[0].font.color.rgb = RGBColor(0xA8, 0xC7, 0xFA)

    return slide


def add_toc(prs: Presentation, items: list[str]):
    """目录页"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])

    # 左侧色条
    shape = slide.shapes.add_shape(1, Inches(0), Inches(0), Inches(0.3), SLIDE_H)
    shape.fill.solid()
    shape.fill.fore_color.rgb = COLOR_PRIMARY
    shape.line.fill.background()

    txBox = slide.shapes.add_textbox(MARGIN_LEFT, MARGIN_TOP, CONTENT_WIDTH, Inches(1))
    tf = txBox.text_frame
    p = tf.paragraphs[0]
    p.text = "目  录"
    p.font.size = SIZE_SLIDE_TITLE
    p.font.bold = True
    p.font.color.rgb = COLOR_PRIMARY

    txBox2 = slide.shapes.add_textbox(MARGIN_LEFT, Inches(1.5), CONTENT_WIDTH, Inches(5))
    tf2 = txBox2.text_frame
    for i, item in enumerate(items):
        if i == 0:
            tf2.paragraphs[0].text = f"0{i+1}  {item}"
            tf2.paragraphs[0].font.size = SIZE_SECTION_TITLE
        else:
            p = tf2.add_paragraph()
            p.text = f"0{i+1}  {item}"
            p.font.size = SIZE_SECTION_TITLE
            p.space_before = Pt(16)
        tf2.paragraphs[-1].font.color.rgb = COLOR_TEXT

    return slide


def add_content(prs: Presentation, title: str, bullet_points: list[str], image_path: str | None = None):
    """内容页：标题 + 要点 + 可选配图"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])

    # 标题
    txBox = slide.shapes.add_textbox(MARGIN_LEFT, MARGIN_TOP, CONTENT_WIDTH, Inches(0.8))
    tf = txBox.text_frame
    tf.paragraphs[0].text = title
    tf.paragraphs[0].font.size = SIZE_SLIDE_TITLE
    tf.paragraphs[0].font.bold = True
    tf.paragraphs[0].font.color.rgb = COLOR_PRIMARY

    # 分隔线
    line = slide.shapes.add_shape(1, MARGIN_LEFT, Inches(1.4), Inches(2), Pt(3))
    line.fill.solid()
    line.fill.fore_color.rgb = COLOR_ACCENT
    line.line.fill.background()

    # 要点
    body_width = CONTENT_WIDTH if not image_path else Inches(6.5)
    txBox2 = slide.shapes.add_textbox(MARGIN_LEFT, Inches(1.8), body_width, Inches(5))
    tf2 = txBox2.text_frame
    tf2.word_wrap = True

    for i, point in enumerate(bullet_points):
        if i == 0:
            tf2.paragraphs[0].text = f"• {point}"
        else:
            p = tf2.add_paragraph()
            p.text = f"• {point}"
            p.space_before = Pt(8)
        tf2.paragraphs[-1].font.size = SIZE_BODY
        tf2.paragraphs[-1].font.color.rgb = COLOR_TEXT

    # 配图
    if image_path:
        slide.shapes.add_picture(image_path, Inches(8.5), Inches(2), Inches(4), Inches(4))

    return slide


def add_summary(prs: Presentation, key_points: list[str]):
    """总结页"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])

    # 顶部装饰条
    shape = slide.shapes.add_shape(1, Inches(0), Inches(0), SLIDE_W, Pt(6))
    shape.fill.solid()
    shape.fill.fore_color.rgb = COLOR_PRIMARY
    shape.line.fill.background()

    txBox = slide.shapes.add_textbox(MARGIN_LEFT, Inches(0.8), CONTENT_WIDTH, Inches(1))
    tf = txBox.text_frame
    tf.paragraphs[0].text = "📝 课堂小结"
    tf.paragraphs[0].font.size = SIZE_SLIDE_TITLE
    tf.paragraphs[0].font.bold = True
    tf.paragraphs[0].font.color.rgb = COLOR_PRIMARY

    txBox2 = slide.shapes.add_textbox(MARGIN_LEFT, Inches(2), CONTENT_WIDTH, Inches(4.5))
    tf2 = txBox2.text_frame
    for i, point in enumerate(key_points):
        if i == 0:
            tf2.paragraphs[0].text = f"✓ {point}"
        else:
            p = tf2.add_paragraph()
            p.text = f"✓ {point}"
            p.space_before = Pt(10)
        tf2.paragraphs[-1].font.size = SIZE_BODY
        tf2.paragraphs[-1].font.color.rgb = COLOR_TEXT

    return slide
```

---

## Phase 2：课件生成引擎（Day 4-12）

### Day 4-6：PPT 生成 `generator.py`

```python
# gen/pptx/generator.py
import json, os
from pathlib import Path
from pptx import Presentation
from openai import OpenAI
from backend.schemas import GenerationInstruction
from .layout import SLIDE_W, SLIDE_H
from .slides import add_cover, add_toc, add_content, add_summary

def gen_ppt(
    instruction: GenerationInstruction,
    template: str = "default",
    output_dir: str = "./output"
) -> Path:
    os.makedirs(output_dir, exist_ok=True)

    # 1. 调 LLM 生成大纲
    client = OpenAI(
        api_key=os.getenv("DEEPSEEK_API_KEY"),
        base_url=os.getenv("DEEPSEEK_BASE_URL")
    )

    outline_prompt = _load_prompt("ppt_outline.txt")
    prompt = outline_prompt.format(instruction_json=instruction.model_dump_json(indent=2))
    resp = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=2000,
        response_format={"type": "json_object"}
    )
    outline = json.loads(resp.choices[0].message.content)
    # outline 结构: {"slides": [{"type": "content", "title": "...", "points": [...]}, ...]}

    # 2. 创建 PPT
    prs = Presentation()
    prs.slide_width = SLIDE_W
    prs.slide_height = SLIDE_H

    # 3. 封面
    add_cover(prs, instruction.teaching_goal, f"授课对象：{instruction.target_audience} | {instruction.duration_minutes}分钟")

    # 4. 目录
    toc_items = [s["title"] for s in outline["slides"] if s["type"] == "content"]
    add_toc(prs, toc_items)

    # 5. 内容页
    for slide_info in outline["slides"]:
        if slide_info["type"] == "content":
            add_content(prs, slide_info["title"], slide_info.get("points", []))

    # 6. 总结
    all_keys = []
    for kp in instruction.knowledge_points:
        all_keys.extend(kp.key_points)
    add_summary(prs, all_keys[:8])  # 最多8条

    # 7. 保存
    filename = f"{instruction.teaching_goal[:30]}.pptx"
    filepath = Path(output_dir) / filename
    prs.save(str(filepath))
    return filepath


def _load_prompt(name: str) -> str:
    base = os.path.dirname(os.path.dirname(__file__))  # gen/
    with open(f"{base}/prompts/{name}", encoding="utf-8") as f:
        return f.read()
```

### Day 7-8：Word 教案生成

```python
# gen/docx/generator.py
import json, os
from pathlib import Path
from docx import Document
from docx.shared import Pt, Inches, RGBColor
from docx.enum.text import WD_ALIGN_PARAGRAPH
from openai import OpenAI
from backend.schemas import GenerationInstruction

def gen_doc(instruction: GenerationInstruction, output_dir: str = "./output") -> Path:
    os.makedirs(output_dir, exist_ok=True)

    client = OpenAI(
        api_key=os.getenv("DEEPSEEK_API_KEY"),
        base_url=os.getenv("DEEPSEEK_BASE_URL")
    )

    prompt = _load_prompt("doc_plan.txt").format(instruction_json=instruction.model_dump_json(indent=2))
    resp = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=3000,
        response_format={"type": "json_object"}
    )
    content = json.loads(resp.choices[0].message.content)

    doc = Document()

    # 标题
    title = doc.add_heading(instruction.teaching_goal, level=0)
    title.alignment = WD_ALIGN_PARAGRAPH.CENTER

    # 六大板块
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

### Day 9-10：动画/游戏生成

```python
# gen/anim/generator.py
import json, os
from pathlib import Path
from openai import OpenAI
from backend.schemas import GenerationInstruction

def gen_animation(
    instruction: GenerationInstruction,
    anim_type: str = "quiz_game",
    output_dir: str = "./output"
) -> Path:
    """
    anim_type: "quiz_game" (知识点闯关) | "concept_anim" (概念动画)
    """
    os.makedirs(output_dir, exist_ok=True)

    client = OpenAI(
        api_key=os.getenv("DEEPSEEK_API_KEY"),
        base_url=os.getenv("DEEPSEEK_BASE_URL")
    )

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
    # data 包含: {"title": "...", "html_code": "<!DOCTYPE html>..."}

    html = data["html_code"]
    filename = f"{instruction.teaching_goal[:20]}_互动.html"
    filepath = Path(output_dir) / filename
    filepath.write_text(html, encoding="utf-8")
    return filepath
```

---

## Phase 3-4：联调 + 交付（Day 10-19）

### 与 A 和 C 联调的 Checklist

```
□ extract_pdf_text() 能被 C 正常 import 且返回非空文本
□ summarize_video() 能被 C 正常 import 且返回合理摘要
□ gen_ppt() 能被 A 正常 import，输入 GenerationInstruction 生成 .pptx 文件
□ gen_doc() 能被 A 正常 import，输入 GenerationInstruction 生成 .docx 文件
□ gen_animation() 能被 A 正常 import，生成 .html 文件可在浏览器打开
□ A 的 orchestrator.py 完整链路: intent → RAG → fusion → gen_ppt/gen_doc/gen_anim 无报错
```

### 你需要交给其他人的东西

| 交给谁 | 是什么 | 何时 |
|--------|--------|------|
| A | `gen_ppt()` / `gen_doc()` / `gen_animation()` | Day 6 (PPT) / Day 8 (Word) / Day 10 (Anim) |
| C | `extract_pdf_text()` / `extract_doc_text()` / `summarize_video()` / `describe_image()` | Day 2 晚 |
| E | 3 套样例课件（入门课/进阶课/复习课）做 Demo 展示 | Day 19 |

### 你需要从谁那里拿到什么

| 从谁 | 是什么 | 何时 |
|------|--------|------|
| A | `backend/schemas.py` → `GenerationInstruction` 类型 | Day 2 晚 |
| C | DeepSeek API Key（.env 文件） | Day 1 |
| E | 课件样式偏好说明 + 测试用指令集 | Day 8 |

---

## 今天（Day 1）立刻做这三件事

**① 验证所有依赖 + 创建目录 + 跑通 PDF 解析**

```bash
cd "C:\Users\kaipie\Desktop\省服务外包大赛\多模态AI教学系统"
poetry install
mkdir -p gen/pptx gen/docx gen/anim gen/parse gen/prompts

# 逐个验证 import
poetry run python -c "import fitz, cv2; from pptx import Presentation; from docx import Document; print('ALL OK')"
```

**② 用 C 找的教材 PDF 测试 PDF 解析**

把上面 `gen/parse/pdf.py` 的代码贴进去，然后用教材 PDF 测试：
```bash
poetry run python -c "
from gen.parse.pdf import extract_pdf_text
text = extract_pdf_text('knowledge-base/教材PDF/教材名.pdf')
print(text[:300])
"
# 中文正常 → 截图发群
```

**③ 手写一个 3 页测试 PPT**

把 `gen/test_gen.py` 的代码贴进去运行，用 PowerPoint 打开确认中文排版正常，截图发群。
