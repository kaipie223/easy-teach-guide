# GitHub 使用手册 — Easy-Teach

> 仓库地址：`https://github.com/kaipie223/easy-teach.git`
> 适用团队：陶克钦(PM) + 姜文杰 + 潘卓然 + 陈澜 + 赵钰洁

---

## 一、陶克钦（PM / 仓库 Owner）操作指南

### 1.1 首次推送（本地已有代码，推到 GitHub）

```bash
# 在项目根目录执行
cd "C:/Users/kaipie/Desktop/多模态AI教学系统"

# 如果还没登录 GitHub，先配置凭据
git config --global user.name "kaipie223"
git config --global user.email "你的邮箱"

# 推送（可能需要浏览器登录 GitHub）
git push -u origin main
```

> 如果弹窗让你登录 GitHub → 选 "Sign in with your browser" → 输入 GitHub 账号密码。

### 1.2 添加团队成员

1. 打开 https://github.com/kaipie223/easy-teach → Settings
2. 左侧菜单 → **Collaborators**
3. 点击 **Add people** → 输入每个人的 GitHub 用户名或邮箱
4. 4 个人都会收到邀请邮件 → 让他们点击 **Accept invitation**

### 1.3 创建 dev 分支（开发主分支）

```bash
# 基于 main 创建 dev 分支
git checkout -b dev
git push -u origin dev
```

之后所有功能分支都从 `dev` 切出，PR 也合并到 `dev`。

### 1.4 日常管理操作

**每天站会后更新 Sprint Board 对应的 Git 状态：**

```bash
# 拉取最新代码
git pull origin main

# 查看每个人的分支和提交情况
git branch -a                    # 看所有分支
git log --oneline --all --graph  # 看提交历史图

# 查看最近谁提交了什么
git log --oneline --since="2026-07-22" --author="姜文杰"
```

**合并 PR（在 GitHub 网页操作）：**

1. GitHub 仓库 → Pull requests → 点开 PR
2. 检查代码变更（Files changed 标签页）
3. 确认至少 1 人已 Review 并点了 Approve
4. 点击 **Merge pull request** → **Confirm merge**
5. 合并后删除功能分支（GitHub 会自动提示）

**如果本地需要更新到最新：**

```bash
git checkout dev
git pull origin dev

git checkout main
git pull origin main
```

### 1.5 版本发布（Phase 节点打 Tag）

```bash
# Phase 1 完成时
git checkout main
git pull origin main
git tag v0.1-phase1 -m "Phase 1 完成：环境搭建 + 接口定义"
git push origin v0.1-phase1

# Phase 2 完成时
git tag v0.2-phase2 -m "Phase 2 完成：核心逻辑联调"
git push origin v0.2-phase2

# 最终交付
git tag v1.0.0 -m "最终交付版本"
git push origin v1.0.0
```

---

## 二、姜文杰 / 潘卓然 / 陈澜 / 赵钰洁（开发者）操作指南

### 2.1 第一步：克隆仓库

```bash
# 在你自己电脑上，选一个目录
cd ~/Desktop

# 克隆仓库
git clone https://github.com/kaipie223/easy-teach.git
cd easy-teach
```

### 2.2 第二步：配置 Git 身份

```bash
git config --global user.name "你的姓名拼音"
git config --global user.email "你的GitHub邮箱"
```

### 2.3 第三步：切换到 dev 分支，创建自己的功能分支

```bash
# 获取最新分支
git fetch origin

# 切换到 dev
git checkout dev

# 拉取最新
git pull origin dev

# 基于 dev 创建你的功能分支（按模块命名）
# 姜文杰：
git checkout -b feat/m1-chat-api
# 潘卓然：
git checkout -b feat/m1-chat-ui
# 陈澜：
git checkout -b feat/m1-intent
# 赵钰洁：
git checkout -b feat/m2-pdf-parser
```

> **分支命名规则：** `feat/<模块>-<简述>`，如 `feat/m5-ppt-generator`、`feat/m3-rag-retriever`

### 2.4 日常开发流程（每天要做的事）

```bash
# ===== 每天开始工作前 =====
# 1. 切到 dev 拉取最新
git checkout dev
git pull origin dev

# 2. 切回你的功能分支，合并 dev 的最新代码
git checkout feat/m1-chat-api     # 换成你自己的分支名
git merge dev                      # 把 dev 的最新代码合并进来

# 3. 开始今天的开发
# ... 写代码 ...

# ===== 下班前 =====
# 4. 查看改了什么
git status
git diff

# 5. 添加改动
git add -A

# 6. 提交（commit message 格式：动作: 简述）
git commit -m "feat: 完成Session CRUD接口"
# 或
git commit -m "fix: 修复文件上传中文乱码问题"

# 7. 推到 GitHub
git push origin feat/m1-chat-api
```

### 2.5 Commit Message 规范

| 前缀 | 用途 | 示例 |
|------|------|------|
| `feat:` | 新功能 | `feat: 完成Chat SSE流式接口` |
| `fix:` | 修 Bug | `fix: 修复SSE消息粘包问题` |
| `refactor:` | 重构（不改变功能） | `refactor: 提取公共的文件校验逻辑` |
| `docs:` | 文档 | `docs: 更新接口文档的ChatEvent定义` |
| `test:` | 测试 | `test: 增加Session API测试用例` |
| `chore:` | 杂项（配置、依赖等） | `chore: 更新pyproject.toml依赖版本` |

### 2.6 提交 PR（功能开发完成时）

1. 确保你的分支已推到 GitHub：
   ```bash
   git push origin feat/m1-chat-api
   ```

2. 打开 https://github.com/kaipie223/easy-teach → **Pull requests** → **New pull request**

3. 设置：
   - **base:** `dev` ← 合并目标
   - **compare:** `feat/m1-chat-api` ← 你的分支

4. 标题写清楚做了什么，描述里写：
   - 这个 PR 做了什么
   - 怎么测试
   - 依赖谁的其他 PR（如果有）

5. 点击 **Create pull request**

6. 在群里 @陶克钦 和至少一个队友来 Review

### 2.7 解决冲突

如果 `git merge dev` 时报冲突：

```bash
# Git 会告诉你哪些文件冲突了
# 打开冲突文件，找到 <<<<<<< 和 >>>>>>> 标记
# 手动选择保留哪段代码，删除标记

# 修改完后：
git add .
git commit -m "merge: 合并dev到feat/m1-chat-api，解决schemas冲突"
git push origin feat/m1-chat-api
```

---

## 三、分支管理全景图

```
main ─────────────────────────────────────────────────────►
  │                                                          
  └── dev ──────────────────────────────────────────────►
        │                                                   
        ├── feat/m1-chat-api     (姜文杰) ──► PR ──┐
        ├── feat/m1-chat-ui      (潘卓然) ──► PR ──┤
        ├── feat/m1-intent       (陈澜)   ──► PR ──┤
        ├── feat/m2-pdf-parser   (赵钰洁) ──► PR ──┤
        ├── feat/m3-rag          (陈澜)   ──► PR ──┤
        ├── feat/m5-ppt-gen      (赵钰洁) ──► PR ──┤
        ├── feat/m6-preview      (潘卓然) ──► PR ──┤
        └── feat/m7-download     (潘卓然) ──► PR ──┘
                                                    │
                                                    ▼
                                                   dev
                                          (Review → Merge)
```

**规则：**
- `main` = 稳定版本，只从 `dev` merge，禁止直接 push
- `dev` = 日常集成，接受 PR
- `feat/*` = 每人每条功能一个分支，完成后提 PR 到 `dev`
- 至少 1 人 Review → 才能 Merge

---

## 四、每个人对应的分支建议

| 姓名 | 模块 | 建议分支名 |
|------|------|-----------|
| 姜文杰 | M1 Session/Chat API | `feat/m1-session-chat-api` |
| 姜文杰 | M2 文件上传 API | `feat/m2-file-upload-api` |
| 姜文杰 | M5 orchestrator | `feat/m5-orchestrator` |
| 姜文杰 | M7 下载 API | `feat/m7-download-api` |
| 潘卓然 | M1 对话 UI | `feat/m1-chat-ui` |
| 潘卓然 | M2 上传组件 | `feat/m2-upload-ui` |
| 潘卓然 | M6 预览页面 | `feat/m6-preview` |
| 潘卓然 | M7 下载页面 | `feat/m7-download-ui` |
| 陈澜 | M1 意图引擎 | `feat/m1-intent-analyzer` |
| 陈澜 | M3 RAG 全链路 | `feat/m3-rag` |
| 陈澜 | M4 语音识别 | `feat/m4-speech` |
| 陈澜 | M5 知识融合 | `feat/m5-fusion` |
| 赵钰洁 | M2 PDF/Word 解析 | `feat/m2-parsers` |
| 赵钰洁 | M2 图片/视频解析 | `feat/m2-vision` |
| 赵钰洁 | M5 PPT 生成 | `feat/m5-ppt-gen` |
| 赵钰洁 | M5 Word 生成 | `feat/m5-doc-gen` |
| 赵钰洁 | M5 动画生成 | `feat/m5-anim-gen` |

---

## 五、常见问题速查

### Q: `git push` 报 "Permission denied"？

**A:** 你可能还没接受 GitHub 邀请。检查邮箱 → 找 "Collaborator invitation" → Accept。或者让陶克钦确认你的 GitHub 用户名是否正确。

### Q: `git push` 报 "failed to push some refs"？

**A:** 远端有你没有的提交。
```bash
git pull origin dev --rebase
git push origin feat/m1-chat-api
```

### Q: 不小心在 `main` 或 `dev` 上直接改了代码？

**A:**
```bash
# 暂存你的改动
git stash

# 创建新分支
git checkout -b feat/你的分支名

# 恢复改动
git stash pop

# 正常 commit + push
git add -A
git commit -m "..."
git push origin feat/你的分支名
```

### Q: `git merge dev` 时冲突太多，想放弃？

**A:**
```bash
git merge --abort   # 回到 merge 前的状态
# 然后找陶克钦协调
```

### Q: 提交到了错误的分支？

**A:**
```bash
# 查看最近 3 条 commit
git log --oneline -3

# 假设最后 1 条是错的，回退（保留改动）
git reset --soft HEAD~1

# 切到正确分支
git checkout feat/正确的分支

# 重新提交
git add -A
git commit -m "..."
```

### Q: 怎么在 GitHub 网页上看队友的代码？

**A:** 仓库主页 → 点分支下拉框 → 选队友的分支 → 查看文件。或者在 Pull requests 里看每个 PR 的 Files changed。

---

## 六、快捷命令速查

```bash
# 看状态
git status

# 看提交历史（简洁版）
git log --oneline -10

# 看所有分支
git branch -a

# 切分支
git checkout feat/m1-chat-api

# 拉取 dev 最新
git checkout dev && git pull origin dev

# 合并 dev 到当前分支
git merge dev

# 添加所有改动
git add -A

# 提交
git commit -m "feat: 描述"

# 推送
git push origin 分支名

# 丢弃本地改动（谨慎！）
git checkout -- 文件名
```

---

## 七、陶克钦的每日检查清单

- [ ] 所有人今天 push 了吗？（看 GitHub 主页的 commit 时间线）
- [ ] 有没有人提了 PR 还没 Review？（看 Pull requests 标签页）
- [ ] `dev` 分支最新代码能跑通吗？（拉下来跑一下）
- [ ] 有没有 merge conflict 需要协调解决？
- [ ] Sprint Board 的任务状态和 Git 的 PR 状态一致吗？
