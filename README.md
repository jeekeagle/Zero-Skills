# Zero-Skills

Zero Agent 的 Skill 集合。每个 skill 都是一个分析视角（perspective lens）、方法论框架或写作工具。

## 设计原则

- **视角化**：每个 skill 提供一个独特的认知框架，不是任务清单
- **理论锚定**：核心观点都引用学科经典理论，标注原文与出处
- **工序可复用**：每步操作有方法 + 工具 + 引用 + 输出格式四要素
- **判据自检**：附判断闸门，使用前先过判据避免误用

## 目录结构

```
skills/
├── perspective/             # 分析视角类
│   └── <skill-slug>/
│       └── SKILL.md
├── methodology/             # 方法论框架类
│   └── <skill-slug>/
│       ├── SKILL.md
│       └── examples/        # 示范输出
└── tool/                    # 工具类
    ├── <skill-slug>/        # 通用工具
    │   └── SKILL.md
    └── wuji/                # wuji-博客系列专用工具
        └── <skill-slug>/
            ├── SKILL.md
            └── references/  # 长文档 / 操作手册
```

每个 skill 放在 `skills/<category>/<slug>/` 下，`<category>` 是 skill 的类别，`<slug>` 是 kebab-case 短名。
当前支持类别：
- `perspective/` — 分析视角（认知框架、理论锚定）
- `methodology/` — 方法论框架（做事流程、思维工序）
- `tool/` — 工具（发布、转换、维护）

未来可扩展：`router/`（视角路由器）等。

## 已收录

### 视角类（perspective/）

- **taxonomy-perspective** — 分类学视角
  - 核心问题：事物是如何被分类的？分类背后的原则和逻辑是什么？
  - 七步工序：识别对象 / 分析原则 / 追溯演化 / 揭示权力 / 评估效果 / 提出替代 / 反思局限
  - 经典理论锚点：涂尔干-莫斯、林奈、科学分类学四原则、郭绍虞文体分类学

### 方法论类（methodology/）

- **initialization-framework** — 六维做事穿透工具
  - 核心问题：如何把一个项目/任务从战略意图到迭代机制一次性想透？
  - 六层穿透：价值判断 / 话语体系 / 实现路径 / 验证标准 / 调度与编排 / 扩展与演化
  - 反坍缩闸：六条错误自检（把价值判断当目标、把话语体系当技术方案、把验证标准当里程碑、把调度编排当分工表、把扩展演化当总结、把六层当检查清单）
  - 配套示例：`examples/示例-做事分析-营销科研Skill手册.md`（用框架分析"做一本营销科研 Skill 手册"）

### 工具类（tool/）

- **post-to-wujiblog** — 发布文章到 jeekeagle/wuji-blog（含 frontmatter 校验、UTC 时间换算、Deploy 三件套、git push 降级路径）
- **skill-to-github** — Skill 发布工作流
  - 核心问题：怎么把一个本地 Skill 干净、可追溯地送进 GitHub 仓库？
  - 七步工序：确认输入 / 解析 SKILL.md / 检查仓库状态 / 构建目录 / 更新 README / 提交推送 / 验证交付
  - 关键能力：自动维护 README 索引、四条判据自检、六条反坍缩闸兜底

## 使用方式

每个 `SKILL.md` 都遵循同一规范：
1. YAML frontmatter（`name` + `description`）
2. 元信息层 / 核心定义 / 操作工序 / 判据 / 结构判断 / 写作规范 / 输出
3. 触发关键词与场景写在 description 中

将 `SKILL.md` 复制到 Claude Code 的 skills 目录即可使用：
- 项目级：`.claude/skills/<name>/SKILL.md`
- 用户级：`~/.claude/skills/<name>/SKILL.md`

## 贡献

新增 skill 请保持：
- 单文件 SKILL.md 优先，超过 500 行再拆 `references/`
- frontmatter description 包含触发关键词（"pushy"风格）
- 引用经典理论时标注原文与出处
- 路径遵循 `skills/<category>/<slug>/` 规范

## License

MIT
