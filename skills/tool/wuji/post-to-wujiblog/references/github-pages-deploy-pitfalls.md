# references/github-pages-deploy-pitfalls.md

> 关于 `jeekeagle/wuji-blog`(任何 GitHub Pages 项目页部署)的 deploy 行为细节、踩坑记录、强制纪律。

## 1 · `concurrency.cancel-in-progress: true` 会吞 deploy

**症状**:push main 之后,`Deploy to GitHub Pages` workflow run 状态 = **`cancelled`**(不是 `success` 也不是 `failure`)。此时:
- 仓库 main HEAD 已经更新
- 但**线上仍是上一版**
- 用户访问站点看不到新内容
- agent 如果只查 git HEAD 就报"已部署"是**错的**

**根因**:`deploy.yml` 里有:
```yaml
concurrency:
  group: pages
  cancel-in-progress: true
```
当上一次 deploy 还在跑、新一次 push 触发新 deploy,GitHub Actions 会**取消旧的**让新的接上。但如果在极短时间窗口内(rerun / 多次 push)触发了**第二次** deploy,**那一次也会被取消**。Cancel 不会回滚代码,代码已经在 main 上,只是 build artifact 没部署出去。

**修复**:找到那个 `cancelled` 状态的 run,rerun 即可:
```bash
gh api -X POST repos/jeekeagle/wuji-blog/actions/runs/<RUN_ID>/rerun
```

**真实案例**:2026-07-01 v1.1.2 那次 push 撞上 cancel。`RUN_ID = 28520970480`,rerun 后 v1.1.2 正常上线。

---

## 2 · 强制纪律:推完先查 run 状态再报告

**这条要刻在脑子里** —— 不是"建议",是"必须":

```
push 后报告"已部署"之前,先做:
  gh api repos/<owner>/<repo>/actions/runs?per_page=1 \
    --jq '.workflow_runs[0] | "\(.head_sha[0:8]) \(.status) \(.conclusion)"'
```

只有当:
- `.head_sha[0:8]` = 当前 main HEAD 的前 8 位
- `.status` = `completed`
- `.conclusion` = `success`

三者**同时**满足,才能报告"已部署"。

任何一项不对:
- head_sha 不匹配 → 还在排队 / 别的分支的 run
- status 不为 completed → 还在跑 / 队列里
- conclusion 为 failure / cancelled → 看上面的修复

**绝不允许**只看 `git log` 就报"已部署"。

---

## 3 · 验证线上真的变了(不是只看 main HEAD)

推完且 deploy 跑通之后,还要**主动验证**线上 HTML 真的反映了新代码,而不是 cache / 旧 artifact。

最小可行验证:
```bash
# 拿一个"新代码独有"的特征串去线上查
curl -sL https://jeekeagle.github.io/wuji-blog/<path> | grep -c "<独有字符串>"
# 或
curl -sL https://jeekeagle.github.io/wuji-blog/<path> | grep "<应有但还没出现的>"
```

举例:验证 v1.1.2 的"删大字"是否真生效:
```bash
curl -sL https://jeekeagle.github.io/wuji-blog/ | grep -c 'site-title__main'
# 期望:0  (没有 .site-title__main,意味着大字体 span 已删)
# 实际如果是 1,说明线上还是旧版
```

---

## 4 · `cancelled` 状态的隐藏陷阱

Actions UI 上 `cancelled` 跟 `failure` 视觉上都是红圈,容易看错。但:

- `failure` → workflow 里有 step 报错,需要修代码重试
- `cancelled` → workflow 被外部取消,代码本身没问题,rerun 就行

判别:点开 run 看详情,如果所有 step 都绿、只是 run 被标记 cancelled → 走 rerun。

---

## 5 · 长期方案(写在 SKILL.md "已知不修" 里)

`deploy.yml` 的 `concurrency.cancel-in-progress: true` 是合理的(避免新旧 build 互相覆盖的中间态),但跟快速连续 push 有冲突。如果未来频繁撞到这个坑,有两个解法:

**方案 A · 改 concurrency**
```yaml
concurrency:
  group: pages
  cancel-in-progress: false   # 新 deploy 等旧的跑完
```
代价:连续两次 push 要排队等,但保证都跑。

**方案 B · 用 gh CLI 自检 + 自动 rerun**

写一个 wrapper,推完自动:
1. 等 30s
2. 查 run 状态
3. 如果 cancelled → rerun
4. 验证结论 success 后再回报

本 skill 目前不带这个 wrapper(用户明确选 A = 纯文档型)。后续如果用户多次撞到,可以升级到 B 或 C 类 skill。

---

## 6 · 跟"author theo" 类的硬编码同样要警觉

任何时候看到 **deploy.yml 配了 `cancel-in-progress: true`**,心里要响起警报:**这个项目的 deploy 不是 fire-and-forget**,需要主动确认结果。

记一笔:这是 2026-07-01 第一次撞到,文档化在这里。后续 session 起手加载这个 skill,就能少踩一脚。

---

## 7 · `git push` 在某些网络环境会被掐,要会降级到 Contents API

**症状**:`git push origin main` 直接报:
```
fatal: unable to access 'https://github.com/<repo>.git/':
       Failed to connect to github.com port 443 after 21055 ms:
       Could not connect to server
```
或者 `Connection was reset`。`gh api` 同时却能用。`git fetch` 也可能间歇性通,推不推得动看天。

**根因**:这台机器的 `git.exe` 走裸 https 到 `github.com:443` 会被 Cloudflare WARP(198.18.0.17)周期性阻断。`gh` 客户端自己有内置的代理/直连策略,走不同的代码路径,所以能通。

**触发条件**:本机装了 Cloudflare WARP(或其他会劫持 443 的代理),并且 `git` 没有配 credential helper / 没有走 gh 的代理设置。验证方式:
```bash
gh auth status          # OK
git fetch origin        # 间歇性失败
gh api repos/<owner>/<repo> --jq .default_branch   # 永远 OK
```

**降级方案 · 用 Contents API 推一个文件**:
```bash
# 1. 把文件 base64 编码
B64=$(base64 -w 0 src/data/blog/<slug>.md)   # 或 certutil 在 Windows

# 2. PUT 到 GitHub
gh api -X PUT \
  -H 'Accept: application/vnd.github+json' \
  repos/jeekeagle/wuji-blog/contents/src/data/blog/<slug>.md \
  -f message="post: <title>" \
  -f content="$B64" \
  -f branch="main"
```
成功后返回的 JSON 含 `content.sha`,这是 GitHub 端算的 blob sha(跟本地 git 算的不一样)。

**降级方案带来的副作用 · 本地 main 跟远端 main 会 divergent**:
- 本地 commit(sha `1d7af26`)只存在于本地 `resource/`
- 远端 commit(sha `b39922b`)是 Contents API 创建的,内容相同但 sha 不同
- 直接 `git pull` 会冲突,因为 git 把它当两个不同 commit 处理
- 不能 fast-forward,`merge --ff-only` 失败

**修法**:对齐本地 HEAD 到远端,丢掉那个本地 commit(它对应的内容已经在远端 commit 里):
```bash
git fetch origin main
git reset --hard origin/main
```
**前提**:本地没有未推送的其它改动。如果还有别的 commit 要保留,先 `git stash` 或 cherry-pick。

**真实案例**:2026-07-02 发 `test-skill-flow.md` 那次:
- `git push` 撞墙,`gh auth setup-git` 改了 credential helper 也不通
- 走 Contents API 上去了,deploy run `28557856792` head `b39922b` success
- 本地 `1d7af26` reset 到 `b39922b`,本地 main 跟远端同步

**SKILL.md 的 step 6 该补**:git push 失败时降级到 Contents API,并明示「会跟本地 commit 产生 divergent,推送后做 `git reset --hard origin/main`」。当前 SKILL.md 的 step 6 只写了 `git push origin main`,没给降级路径。