---
name: skill-to-github
description: 把本地 Skill 推送到 GitHub 仓库并自动更新 README。Make sure to use this skill whenever the user says 上传 skill、推 skill 到 GitHub、publish skill、sync skill、add skill to repo、update skills repo、把这个 skill 发到 GitHub、把刚才的 skill 推到仓库, or references a specific skills collection repo (like Zero-Skills) and wants to add a new SKILL.md. Trigger when the user finishes creating a new SKILL.md and wants to publish it, or wants to batch-sync multiple skills. Even if the user doesn't name this skill explicitly — if the intent is "把 skill 文件送进 GitHub 仓库并保证 README 反映新内容", use this skill.
---

# Skill-to-GitHub

把本地 SKILL.md 推送到 GitHub 仓库,自动维护仓库目录结构和 README 索引。

## 元信息层

- 视角名称:Skill 发布工作流
- 核心问题:怎么把一个本地 Skill 干净、可追溯地送进 GitHub 仓库?
- 适用场景:个人/团队 skill 集合仓库的发布、版本管理、README 维护
- 关联工具:git、gh CLI
- 适用对象:单文件 SKILL.md 或含 references/、assets/、scripts/ 的 skill 目录

## 核心定义层

### 什么是 skill-to-github

不是简单的 git push,也不是单独的 gh repo create。它把发布 skill 拆成"确认输入→仓库状态→目录结构→README 维护→提交推送→验证交付"六个闸,每闸都有显式判断和兜底。

### 核心概念

- **Skill 即包**:每个 skill 是一个目录(SKILL.md 为入口,可附带 references/、assets/、scripts/),不是单文件
- **仓库即目录**:skill 在仓库中的位置固定为 `skills/<slug>/`,README 按类别分小节索引
- **README 即入口**:README 是 skill 集合的发现入口,新增 skill 必须更新 README 才算完成发布
- **可重入**:重复执行同一 skill 不会破坏仓库(幂等),只在文件实际变化时产生新 commit

## 操作工序层

### 第一步:确认输入

**必填参数**:
- `SKILL_PATH` — 本地 SKILL.md 绝对路径,或 skill 目录路径(含 SKILL.md)
- `REPO` — 目标仓库,格式 `<owner>/<repo>`,例如 `jeekeagle/Zero-Skills`
- `SLUG` — skill 短名,kebab-case,默认从 SKILL.md 文件名推断

**可选参数**:
- `CATEGORY` — 类别(视角/工具/方法论/路由器/其他),默认从 SKILL.md 元信息层推断
- `LOCAL_CLONE` — 本地已 clone 的仓库路径,默认自动克隆到临时目录
- `COMMIT_PREFIX` — commit 前缀,默认 `feat`
- `BRANCH` — 默认 `main`
- `VISIBILITY` — `public` / `private`,默认 `public`(只在新建仓库时生效)

**输出**:
```json
{
  "skill_path": "C:/.../taxonomy-perspective/SKILL.md",
  "skill_name": "taxonomy-perspective",
  "repo": "jeekeagle/Zero-Skills",
  "category": "视角",
  "branch": "main"
}
```

### 第二步:解析 SKILL.md 关键信息

**说明**:从目标 skill 的 frontmatter 和正文提取 README 索引需要的字段。

**方法**:
- 读 SKILL.md frontmatter,提取 `name`、`description`
- 读"元信息层"小节,提取"核心问题""关键特点"(如有)
- 截取 description 的前 80 字符作为 README 一句话描述

**输出**:
```json
{
  "name": "taxonomy-perspective",
  "short_desc": "从分类学视角分析事物的分类原则、分类方法及其背后的认知逻辑...",
  "core_question": "事物是如何被分类的?分类背后的原则和逻辑是什么?"
}
```

### 第三步:检查仓库状态

**说明**:判断目标仓库是否存在,以及本地是否已 clone。

**方法**:
- 仓库是否存在:`gh repo view <REPO> --json name,defaultBranchRef`
- 不存在 → 询问用户后用 `gh repo create <REPO> --<visibility> --description "<仓库描述>"` 创建
- 本地是否已 clone:`ls <LOCAL_CLONE>/.git` 存在 → 复用;否则 `git clone https://github.com/<REPO>.git <LOCAL_CLONE>`

**输出**:
```json
{
  "repo_exists": true,
  "local_clone": "E:/Zero/Zero-Agent/提示词处理/Zero-Skills",
  "default_branch": "main"
}
```

### 第四步:构建目录结构

**说明**:在仓库的 `skills/<slug>/` 位置放置 skill。

**方法**:
- 确认目标目录:`<LOCAL_CLONE>/skills/<slug>/`
- 复制 SKILL.md → `skills/<slug>/SKILL.md`
- 若源是目录,同步整个目录(保留 references/、assets/、scripts/ 子目录)
- 不要复制 .git、node_modules、workspace 之类杂质

**输出**:
```
skills/<slug>/
├── SKILL.md       (12 KB)
├── references/    (如有)
├── assets/        (如有)
└── scripts/       (如有)
```

### 第五步:更新 README 索引

**说明**:在 README 的"已收录"分类下追加新 skill 条目,保持现有结构。

**方法**:
- 读 `<LOCAL_CLONE>/README.md`
- 定位"已收录"小节,按 CATEGORY 找到对应子类(视角/工具/方法论/路由器/其他)
- 追加一行:
  ```
  - **<slug>** — <short_desc> | <核心问题或关键特点>
  ```
- 若 CATEGORY 子节不存在,新建
- 若 README 不存在,用仓库级模板创建

**输出**:README 中"已收录"小节新增一行,其余内容不动。

### 第六步:提交并推送

**说明**:单次 commit 包含 skill 文件和 README 更新。

**方法**:
- `git add skills/<slug>/ README.md`(精确 add,不把其他变更带进去)
- `git status --short` 复查
- `git commit -m "<prefix>: 收录 <slug> — <short_desc 一句话>"`
- `git push origin <BRANCH>`

**输出**:
```json
{
  "commit_sha": "abc1234",
  "branch": "main",
  "files_changed": ["skills/<slug>/SKILL.md", "README.md"]
}
```

### 第七步:验证交付

**说明**:通过 GitHub API 确认文件已落地。

**方法**:
- `gh api repos/<REPO>/contents/skills/<slug>` — 确认 SKILL.md 存在
- `gh api repos/<REPO>/commits/<BRANCH>` — 确认 commit 已推送
- 返回 GitHub 直链和原始文件 URL

**输出**:
```markdown
## ✅ 已发布 <slug>

**仓库**: https://github.com/<owner>/<repo>
**SKILL.md 直链**: https://github.com/<owner>/<repo>/blob/main/skills/<slug>/SKILL.md
**原始文件**: https://raw.githubusercontent.com/<owner>/<repo>/main/skills/<slug>/SKILL.md
**commit**: <sha>
```

## 判据层

在开始前过一遍这四条判据:

- **判据 1**:目标文件确实是 SKILL.md(有 YAML frontmatter,含 `name` 字段)
- **判据 2**:目标仓库可写(`gh auth status` 通过,REPO 在用户账号下或用户有写权限)
- **判据 3**:slug 在仓库中不重复(重复则警告并询问:覆盖 / 跳过 / 改 slug)
- **判据 4**:README 更新是增量,不会破坏现有结构(只在"已收录"对应类别下追加一行)

## 结构判断层

### 双闸判断

**闸 1:任务清晰度**
- 判断 1:用户是否提供了 SKILL.md 路径(或可推断的位置)?
- 判断 2:目标仓库名是否明确?
- 判断 3:推送权限是否具备?

**闸 2:可重入性**
- 判断 1:本次 push 是否会产生实际变更(无变更则跳过 commit)?
- 判断 2:远端是否已经是最新的 fetch 状态?
- 判断 3:本地工作区是否干净(无未提交改动)?

**判断逻辑**:
- 闸 1 全通 + 闸 2 全通 = 正常执行
- 闸 1 缺一项 = 询问用户补全
- 闸 2 有未提交改动 = 先 stash 或询问

### 反坍缩闸:避免常见错误

- **错误 1:误推杂质** — 把 workspace、eval、临时文件带进仓库
  - 对策:精确 `git add` 而不是 `git add .`,用 `.gitignore` 屏蔽测试工作区
- **错误 2:破坏 README** — 整段覆盖 README 导致现有条目丢失
  - 对策:用 Edit 而非 Write 修改 README,只在"已收录"对应小节追加
- **错误 3:覆盖现有 skill** — 同名 slug 静默覆盖
  - 对策:检测到重复时显式询问,让用户选择覆盖/改 slug/取消
- **错误 4:忘了更新 README** — 推了 SKILL.md 但 README 索引没动
  - 对策:把 README 更新作为必经工序,验证时确认 README 也有新 commit
- **错误 5:commit 信息太啰嗦** — 把整个 description 塞进 commit message
  - 对策:commit message 第一行 ≤ 72 字符,只保留一句话核心
- **错误 6:推送到错分支** — 仓库默认分支可能不是 main
  - 对策:用 `gh repo view --json defaultBranchRef` 查,推送用 `git push origin <branch>`

## 工具与依赖

- **git** — 版本控制
- **gh CLI** — GitHub API 调用,需 `gh auth login`
- **Bash** — 命令执行
- **Read / Write / Edit** — 文件操作

可选:
- **WebFetch** — 验证远端文件内容(API 不可用时的兜底)

## 输出格式

```markdown
## ✅ 已发布 <slug>

**仓库**: <repo-url>
**类别**: <category>
**SKILL.md 直链**: <github-blob-url>
**原始文件**: <raw-url>
**commit**: <sha>
**README 已更新**: 是 — 在"已收录 → <category>"小节新增一行
```

如果失败:
```markdown
## ❌ 发布失败

**原因**: <错误描述>
**建议**: <修复方式>
```
