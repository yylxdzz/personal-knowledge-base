# personal-knowledge-base

> **📦 v1.0.0** — 编译型个人知识库系统，基于 Karpathy LLM Wiki 模式，以知识单元为最小语义单位，LLM 编译而非拼接。

---

## 版本信息

| 字段 | 值 |
|------|-----|
| **版本号** | `v1.0.0` |
| **打包目录** | `personal-knowledge-base-v1.0.0` |
| **设计规范** | SemVer（x.y.z） |
| **变更记录** | `CHANGELOG.md` |
| **设计依据** | 知识库全流程设计.md v3.7 |
| **许可协议** | MIT |

### 版本号递增规则

| 变更类型 | 变化 | 示例 |
|---------|:----:|------|
| 破坏性变更（不兼容） | 主版本 +1 | v1.0.0 → v2.0.0 |
| 新增功能（向后兼容） | 次版本 +1 | v1.0.0 → v1.1.0 |
| Bug 修复 / 文档修正 | 修订版本 +1 | v1.0.0 → v1.0.1 |

### 发布流程

1. 更新 `SKILL.md` frontmatter `metadata.version`
2. 新增 `CHANGELOG.md` 条目
3. 重命名打包文件夹 `personal-knowledge-base-v{旧版本}` → `personal-knowledge-base-v{新版本}`
4. 更新本文件版本号徽章
5. 提交 Git commit（message 含版本号）

---

## 核心理念

| 原则 | 说明 |
|------|------|
| **编译而非拼接** | LLM 理解多个来源 → 用自己的语言重新组织为一个知识单元 |
| **双层架构** | Compiled Truth（当前可靠结论）+ Timeline（历史操作记录），可追溯 |
| **组件化查询** | 按需返回不同粒度的知识，而非全文倒灌 |
| **用户做选择题** | 所有变更先出草稿 → 用户确认/阻拦，不做编辑题 |
| **Git 做底** | wiki/ 是独立 Git 仓库，草稿不入主分支，可回滚 |

---

## 与 Karpathy 原始设计的差异

| 维度 | Karpathy 原版 | 本方案 |
|------|-------------|--------|
| 来源归因 | 语句级 | 主题级（LLM 不可靠） |
| 查询模式 | 按全文搜索 | 组件化返回（按需裁剪） |
| 审核机制 | 无 | 草稿审核前置（用户选择题） |
| 时效管理 | 无 | freshness + stale 提醒 |
| 版本控制 | 无 | Git 独立仓库 + 回滚 |

---

## 目录结构

### 本仓库文件

```
personal-knowledge-base/
├── SKILL.md                    ← 🎯 Skill 主文件
├── LICENSE                     ← MIT 开源许可
├── templates/
│   ├── config.yaml             ← 实例化配置模板（Agent ID / 路径 / cron）
│   ├── schema.md               ← 编译规范 + 判断规则卡 + Harness 9 条
│   ├── state.json              ← 状态跟踪文件模板
│   ├── index.md                ← 全局索引模板
│   ├── log.md                  ← 操作日志模板
│   ├── meta_log.md             ← 系统级日志模板
│   └── .gitignore              ← Git 忽略规则
├── references/
│   ├── design_principles.md    ← 设计原理、核心理念、风险防护
│   └── workflow_reference.md   ← 四大流程详细决策树和审核信息
└── README.md                   ← 本文件
```

### 初始化后的知识库目录

```
knowledge-base/
├── sources/                    ← 原始文件（只读，用户手动放入）
└── wiki/                       ← Obsidian vault + 独立 Git 仓库
    ├── .git/
    ├── .gitignore
    ├── index.md                ← 全局索引（按 domain 分组）
    ├── log.md                  ← 操作日志
    ├── .meta/
    │   ├── config.yaml         ← 实例化配置
    │   ├── schema.md           ← 工作手册
    │   ├── state.json          ← 文件状态跟踪
    │   └── log.md              ← 系统级操作日志
    └── .draft/                 ← 草稿区（审核前）
```

**关键设计**：`sources/` 不入 Git，只保留 `wiki/` 版本控制。原始文件归用户所有，编译产物才是知识库的核心资产。

---

## Git 存储原理

wiki/ 作为独立 Git 仓库，每次审核通过后的变更都 `git commit`。Git 会自动优化大文件存储（增量 diff 而非全量复制），一个 1MB 的知识单元每次修改可能只增加几 KB。

**核心设计**：草稿（.draft/）不入主分支，审核通过后才 commit。回滚通过 `git revert`（创建反向提交，不删除历史）。

---

## 快速上手

### 1. 安装 Skill

将本目录放在 agent 平台的 skills 目录中，Agent 即可自动识别。

### 2. 初始化知识库

告诉知识库管理师：
```
初始化知识库框架，路径为 /Users/xxx/knowledge-base
```

Agent 会自动：
- 创建目录结构
- 写入模板文件（schema.md / state.json / index.md / log.md / .gitignore）
- 初始化 Git 仓库 + 首次 commit

### 3. 放入来源

将笔记/文档/搜索结果放入 `sources/` 目录，文件名格式：
```
YYYYMMDD 主题 来源类型.md
```
例如：`20260503 RAG架构 笔记.md`

### 4. 执行 Ingest

```
入库
```
或等待每周一定时扫描。

Agent 会：读取来源 → 编译为知识单元 → 写入 .draft/ → 你审核 → 入库。

### 5. 日常问答

日常对话中，知识问答与沉淀师会自动查询知识库，回答你的问题。

---

## 定时任务

| 任务 | 频率 | 说明 |
|------|------|------|
| Ingest 扫描 | 每周一 02:00 | 扫描 sources/ 新文件 |
| Delete 扫描 | 每月 1、15 号 02:00 | 检查来源文件删除 |
| Lint 检查 | 每月 1 号 02:00 | 知识库健康检查 |

创建方式：通过 agent 平台的 cron 调度功能创建以下 3 个定时任务：

| 任务 | Cron | 类型 | 消息 |
|------|------|:----:|------|
| Ingest 扫描 | `0 2 * * 1` | agent | 执行 Ingest 扫描 |
| Delete 扫描 | `0 2 1,15 * *` | agent | 执行 Delete 扫描 |
| Lint 检查 | `0 2 1 * *` | agent | 执行 Lint 检查 |

---

## Obsidian 集成

wiki/ 目录本身就是 Obsidian vault，可直接用 Obsidian 打开。

**Dataview 配置**（可选，在 wiki/.obsidian/plugins/dataview/ 中）：

```dataview
TABLE domain, confidence, freshness, open_threads_count
FROM ""
SORT file.name
```

可配合 Dataview 插件实现知识单元的可视化统计和筛选。

---

## 文件清单

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Skill 主文件，Agent 加载后执行所有知识库操作 |
| `LICENSE` | MIT 开源许可 |
| `templates/config.yaml` | 实例化配置模板（Agent ID、路径、cron、embedding） |
| `templates/schema.md` | 编译规范 + 判断规则卡 + Harness 9 条验证规则 |
| `templates/state.json` | 状态跟踪文件模板 |
| `templates/index.md` | 全局索引模板 |
| `templates/log.md` | 操作日志模板 |
| `templates/meta_log.md` | 系统级日志模板 |
| `templates/.gitignore` | Git 忽略规则 |
| `references/design_principles.md` | 设计原理与背景 |
| `references/workflow_reference.md` | 四大流程详细参考 |
| `README.md` | 本文件 |