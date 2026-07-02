---
name: post-to-wujiblog
description: Publish a new post to jeekeagle/wuji-blog (Astro site at https://jeekeagle.github.io/wuji-blog/). Use when the user wants to write and publish a new article to the wuji-blog repo. Covers frontmatter schema, validation, scheduled-publish semantics, GitHub Actions auto-deploy, and the gotchas of working in this Astro-based blog with BASE_PATH=/wuji-blog and dynamic OG image generation.
---

# post-to-wujiblog

发布新文章到 `jeekeagle/wuji-blog` 仓库 → 自动部署到 GitHub Pages 项目页 `https://jeekeagle.github.io/wuji-blog/`。

> 这是**纯文档型 skill** —— 没有 Python 模块、没有脚本、没有测试。agent 读完这份契约就能正确地引导用户完成发布流程。

## 适用边界

**做这个**:
- 引导用户写一篇新的 markdown 文章
- 校验 frontmatter 合法性(必填字段、时间合法性、slug 命名)
- 帮用户 commit + push 到 main,触发 deploy
- 解释 deploy 状态、调试失败

**不做这个**:
- 写文章正文(那是用户的脑力活,agent 只在用户明确要求时帮润色)
- 改 site 设计 / 配置(那是另外的 task,不是"发文章")
- 不直接调 GitHub API 发布(本 skill 不带 publish 模块,走 `git push`)

## 仓库背景(写给 agent 看)

```
仓库:    jeekeagle/wuji-blog  (public, MIT, 默认分支 main)
线上:    https://jeekeagle.github.io/wuji-blog/
框架:    Astro 5 + Tailwind CSS 4 + Pagefind 索引
base:    /wuji-blog  (项目页部署,所有站内链接要带这个前缀)
作者:    theo  (SITE.author 当前硬编码)
字体:    霞鹜文楷 (LXGW WenKai)
强调色:  朱砂 #a8311a (设计 tokens 已定,不要改)
当前 tags 体系: 反思 / 命名 / 浪潮 / 站点 / 试笔 / 随笔 / AI行业
```

**关键事实**(必须记牢,错一个就翻车):

1. `pubDatetime` 必须是**过去时间**(有 scheduled-post 机制)
2. **正文从 h2 开始,不用 h1** — h1 是 frontmatter `title` 渲染的
3. base path = `/wuji-blog`,所有站点内链接必须用 `siteUrl()` helper,**不能直接写绝对路径**
4. 文章总数极少(2026-07-02 起 4 篇),`SITE.postPerIndex` 触发判别见下
5. `dynamicOgImage: true`,新文章上线后第一次访问 og.png 会有几秒渲染延迟

**两套目录并存**(2026-07-02 才发现 skill 没覆盖,新贡献者最容易混):

- `src/data/blog/` — **生产文章**,被 Astro content collection 收集并上线
- `sources/articles/<slug>/` — **素材/草稿**,2026-07-02 看到 3 篇中文长文,每篇挂 `materials/` 下的 lens 分解文件,**未挂到 content config,不会上线**

本 skill 只处理 `src/data/blog/`。要把 `sources/articles/` 里的文章上线,需要单独改 `src/content.config.ts`(那是另一档 task),不是「发文章」流程的一部分。

**`SITE.postPerIndex` 触发判别**(原文只说「考虑改回 4」,写明规则):

- 当前值:`3`
- 触发条件:当 `src/data/blog/` 总文章数 ≥ `postPerIndex`,首页「最近文章」区块**少展示 1 篇**(剩 N-1)
- 新增文章后该改多少:通常 = 新总数
- 例:第 4 篇发出后,`postPerIndex: 3` 让首页只展示最近 3 篇(缺 1 篇);改成 `4` 让首页展示最近 4 篇
- 决策时机:**文章 push 后,看一眼首页是否缺文章**,再决定改不改
- 改的位置:`src/config.ts` 里的 `SITE.postPerIndex`

## 发文章的标准流程(7 步)

### 1 · 想清楚 slug

- 文件名 = URL slug
- 文件路径:`src/data/blog/<slug>.md`
- 线上路径:`https://jeekeagle.github.io/wuji-blog/posts/<slug>/`
- 命名建议:**kebab-case 中文意译**(例 `why-wuji.md` / `surfing-imagery.md`),不要纯拼音不要拼音带空格
- 不要重名:用 `gh api repos/jeekeagle/wuji-blog/contents/src/data/blog/<slug>.md` 检查 404 才行,否则覆盖了别人

### 2 · 写 frontmatter

```markdown
---
title: "你的文章标题"           # 必填,字符串
author: theo                   # 必填,当前硬编码 theo
pubDatetime: 2026-07-01T10:00:00.000Z   # 必填,UTC,过去时间
modDatetime: 2026-07-01T10:00:00.000Z   # 选填,改稿时同步
featured: true                 # 选填, true 进首页"甄选"
tags:                          # 选填但推荐
  - 随笔
  - 命名
description: "SEO / 卡片预览,150 字以内"   # 选填但强烈推荐
lang: zh                       # 选填,默认 en
draft: false                   # 选填; true 会被 getStaticPaths 过滤
canonicalURL: "https://..."    # 选填,翻译文章加这个防 SEO 惩罚
---

## 从 h2 开始,不用 h1
...
```

**字段详解**:

| 字段 | 必填 | 类型 | 规则 |
|---|---|---|---|
| `title` | ✓ | string | 显示在卡片 + h1,会拼到 Layout title 里(`title \| 无记`) |
| `author` | ✓ | string | 当前必须 `theo`,改 SITE.author 是另一档事 |
| `pubDatetime` | ✓ | ISO 8601 UTC | **必须过去时间**,否则走定时发布藏着 |
| `modDatetime` | × | ISO 8601 UTC | 改稿时同步更新 |
| `featured` | × | boolean | true 进首页"甄选 · Featured"区块(默认 3 篇都标 true) |
| `tags` | × | string[] | 当前有 7 个,新 tag 要想清楚再用 |
| `description` | × | string | 影响 og:description + 卡片预览,150 字以内 |
| `lang` | × | string | 默认 en,中文站建议 `zh` |
| `draft` | × | boolean | true 不进 prod(被 getStaticPaths 过滤) |
| `canonicalURL` | × | URL | 翻译文章必填 |

### 3 · 时间转换坑(最容易出错)

- `pubDatetime` **必须是 UTC**
- 北京时间 5:00 = UTC 前一天 21:00
- 例子:北京时间 2026-07-02 10:00 → `2026-07-02T02:00:00.000Z`
- `SITE.scheduledPostMargin: 15 * 60 * 1000` = 15 分钟缓冲
- 含义:你写"5 分钟后发"也会被压住,要过去 15 分钟才显示

### 4 · 验证(必做,推送前自己过一遍)

- [ ] frontmatter 8 个字段都在该在的位置
- [ ] `pubDatetime` 是过去时间,且**早于 now() - 15min**
- [ ] 没有 h1 标题(从 h2 开始)
- [ ] 文件名是 kebab-case
- [ ] tags 不是新造词(优先复用现有 7 个)
- [ ] description 长度 ≤ 150 字
- [ ] 没有内嵌 image 用错路径(图片放 `public/images/posts/<slug>/`,markdown 引用 `/images/posts/<slug>/xxx.png`)

### 5 · 本地预览(可选,强烈推荐)

```bash
cd wuji-blog
pnpm install              # 第一次
pnpm dev                  # http://localhost:4321/wuji-blog/
```

注意:本地访问要**带 `/wuji-blog/` 前缀**,因为 base path = `/wuji-blog`,不带会 404。

### 6 · commit + push

**首选路径**:
```bash
git add src/data/blog/<slug>.md
git commit -m "post: <title>"
git push origin main
```

**如果 `git push` 撞墙**(本机装了 Cloudflare WARP / 其他劫持 443 的代理时常见):走 Contents API 降级,完整步骤、副作用、对齐做法见 `references/github-pages-deploy-pitfalls.md` 第 7 节。降级之后必须做一次 `git fetch origin main && git reset --hard origin/main` 让本地 main 跟远端对齐,否则下次 push 会冲突。

### 7 · 等 deploy,验证上线(这一步是硬纪律,不是"建议")

- push 后立即触发 `deploy.yml`(GitHub Actions)
- URL: https://github.com/jeekeagle/wuji-blog/actions
- 约 60~120s 跑完
- 跑完访问 `https://jeekeagle.github.io/wuji-blog/posts/<slug>/`

**报告"已部署"的硬条件**(`head_sha` + `status` + `conclusion` 三者必须同时满足):
```bash
gh api repos/jeekeagle/wuji-blog/actions/runs?per_page=1 \
  --jq '.workflow_runs[0] | "\(.head_sha[0:8]) \(.status) \(.conclusion)"'
```
- `.head_sha[0:8]` = 当前 main HEAD 前 8 位 ✓
- `.status` = `completed` ✓
- `.conclusion` = `success` ✓

**三项都满足才能告诉用户"已部署"**。任何一项不满足,要么等、要么修、要么 rerun,**绝不允许**只看 `git log` 报"已部署"。

**如果 deploy 失败**,常见原因:
- `pnpm run lint` 失败:markdown 没引好外部图 / 格式不合规
- `pnpm run format:check` 失败:Prettier 风格不一致
- `pnpm run build` 失败:frontmatter 解析错(最常见是`pubDatetime` 写了无效时间字符串)
- **`concurrency.cancel-in-progress: true`** 把新 deploy 挤掉:此时访问会看到旧版,需 rerun

**坑细节 / 全部判别流程见 `references/github-pages-deploy-pitfalls.md`**(踩过的:2026-07-01 v1.1.2 那次 deploy 被 cancel,代码上线了但线上没变)。

## 当前文章现状(2026-07-02)

| slug | pubDatetime | featured | tags |
|---|---|---|---|
| why-wuji | 2026-06-30T16:00:00Z | ✓ | 站点, 随笔, 命名 |
| test-note | 2026-06-30T21:00:00Z | ✓ | 试笔, 随笔, 站点 |
| surfing-imagery | 2026-05-18T16:00:00Z | ✓ | AI行业, 反思, 浪潮 |
| test-skill-flow | 2026-07-02T00:30:00Z | × | 试笔, 站点, AI行业 |

第 4 篇 `test-skill-flow` 是 skill 流程实测稿(2026-07-02),不进 featured。发完后需要去看首页,决定 `SITE.postPerIndex: 3` 是否要改成 `4`。详见上文「`SITE.postPerIndex` 触发判别」。

`sources/articles/` 下另有 3 篇中文长文(套路词 / 文理交叉 / 标签力学),**未上线**,不在本表。

## 反模式(明确不要做)

- ❌ 用 `gh api PUT /contents/...` 直接发文走默认路径(本 skill 默认走 git push;只有 `git push` 失败时才降级到 Contents API,见 step 6 降级路径)。
  - ⚠️ Contents API 降级要认识到副作用:会产生一个 GitHub 算的 blob/commit sha,跟本地 git 算的不一样,留 orphan。本地 main 跟远端 main divergent,推完后**必须** `git fetch origin main && git reset --hard origin/main` 对齐。这条纪律在 `references/github-pages-deploy-pitfalls.md` 第 7 节。
- ❌ 改 `src/config.ts` 的 `author`、`title`、`lang`(改这些不属于发文章)
- ❌ 把 `pubDatetime` 写成 `"2026-07-02"` 不带时间(会被 frontmatter parser 报错)
- ❌ 在 frontmatter 里写 `h1` 标签(渲染时会变 h2,顺序错乱)
- ❌ 引用 `../../images/...` 相对路径(用 `/images/posts/<slug>/...` 绝对路径,因为 base path 会改相对路径基准)
- ❌ 改 `Base path` / `deploy.yml` —— 那是部署侧,跟发文章无关
- ❌ 直接 push 到 `gh-pages` 分支(仓库用的是 `main` + Actions,没有 `gh-pages` 分支)

## 协作姿态

- **agent 不要替用户写正文**:用户写,agent 校验 + 引导
- **润色请求才动笔**:用户说"帮我润色"才动,不要主动改稿
- **直接执行,不在 commit / push 前问**：(2026-07-02 用户纠正"你以后工作不要每次都问我，直接执行即可",覆盖本 skill 原「每个动作前问一句」纪律) 用户在 driver seat 上的偏好是放手干、事后看。commit 信息、push 动作都不需要逐条确认。**只在以下情况停下**:用户显式说 stop/wait,或动作有真实 trade-off(切其它分支、删 tag、force push)用 clarify 让用户选。
- **失败了诚实报告**:lint 报错 / build 失败 / og.png 慢,**不**掩饰

## 已知不修的痛点

- `Main.astro` 的 `data-backUrl` + sessionStorage `setItem` 仍是死代码(v1.1.1 BackButton 改用 `history.back()` 绕开,但写入端还在)。下一次重构可以考虑彻底删。
- og.png 第一次访问慢,satori 渲染。可以做"构建时预渲染"但需要重构 og.png.ts,v2 再议。
- `author: theo` 硬编码,多作者场景要扩 Astro content config schema,先不做。

## 参考命令速查

```bash
# 仓库可达性
gh api repos/jeekeagle/wuji-blog --jq .default_branch

# 检查 slug 不重复(2026-07-02 修正:gh api 不支持 curl 风格的 -o /dev/null,用 -i --silent 取 HTTP 状态行)
gh api repos/jeekeagle/wuji-blog/contents/src/data/blog/<slug>.md -i --silent | head -1
# 期望:HTTP/2 200 → 重名,换一个
# 期望:HTTP/2 404 → 可用

# 看最新 deploy
gh api repos/jeekeagle/wuji-blog/actions/runs?per_page=1 \
  --jq '.workflow_runs[0] | "\(.head_sha[0:8]) \(.status) \(.conclusion)"'

# rerun 一个 cancelled 的 deploy
gh api -X POST repos/jeekeagle/wuji-blog/actions/runs/<run_id>/rerun

# 看 sitemap 上线列表
curl -sL https://jeekeagle.github.io/wuji-blog/sitemap-0.xml | grep -oE '/posts/[^/]+/'
```

## Support files

- `references/github-pages-deploy-pitfalls.md` —— `concurrency.cancel-in-progress` 坑细节、强制纪律、判别流程、cancelled vs failure 区别、长期方案。**部署前 / 部署后必读**。