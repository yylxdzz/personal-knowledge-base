# 工作流详细参考

> 本文件包含四大核心流程的详细参考描述。SKILL.md 中已有操作指令，本文件供 Agent 在遇到边界情况或需要理解设计意图时查阅。

---

## 一、Ingest 详细流程

### 1.1 三路判断决策树

```
[发现新来源]
  → 读取 index.md + schema.md RESOLVER
  → 解析来源内容
  → 匹配已有知识单元（含 aliases）？
       ├─ 无匹配 → Ingest-A：新建知识单元
       ├─ 直接相关 + 新内容可融入 → Ingest-B：更新既有单元
       └─ 相关但独立 → Ingest-C：新建 + [[双链]] 关联
```

### 1.2 别称识别流程

| 步骤 | 操作 | 结果 |
|:----:|------|------|
| 1 | 语义识别：分析来源，判断是否存在别称 | 确定→关联 aliases |
| 2 | 联网搜索验证（如工具可用） | 确定→关联 aliases |
| 3 | 仍不确定 | 标注 `aliases_pending`，用户审核确认 |

### 1.3 替代型 vs 累积型判断（Ingest-B）

| 类型 | 判断条件 | 操作 |
|------|---------|------|
| 替代型 | 新数据使旧数据失去参考价值 | 上层覆盖重写，Timeline 追加 |
| 累积型 | 新旧数据共存、互为补充 | 上层追加新数据，Timeline 追加 |
| 模糊 | Agent 无法确定 | 用户审核时提供选择题 |

### 1.4 审核通知信息

```
📚 N 篇草稿待审核：
1. [主题名].md | 新增/更新 | domain: [值] | tags: [...] | OT: [N] | [freshness]
   frontmatter 变更: domain [旧]→[新] | tags: +tag1
Harness 验证: ✅ 全部通过 / ⚠️ [具体问题]
需关注: aliases_pending / domain 待确认 / ...

回复"通过"或"全部通过"以确认入库，
或"阻拦：[具体问题]"要求重新编译。
```

---

## 二、Delete 详细流程

### 2.1 触发方式

| 触发 | 说明 |
|------|------|
| 定时扫描 | 每2周对比 state.json，发现 sources/ 中文件被删除 |
| 用户指令 | "删除 xxx 文件"、"检查删除" |
| AGENTS.md 自检 | 启动时检查 last_delete_check 时间戳 |

### 2.2 定位机制

通过 `state.json` 中 `sources` → `ingested_to` 反向索引直接定位受影响的知识单元，无需逐文件扫描。

### 2.3 统一框架

```
[发现 source 被删除]
  → 查 ingested_to → 定位知识单元
  → 判断 sources_count
       ├─ >1 → 收集可靠来源（其他来源文件 + Git 旧版本参考）
       │        → 重新编译 → Harness → 草稿 → 审核 → 入库
       └─ ==1 → 用户决策：删除 / 保留(orphan) / 暂缓(unstable)
```

### 2.4 安全防护（依赖 Git）

| 步骤 | 操作 | 说明 |
|------|------|------|
| 写入新版本 | 覆盖原知识单元 | 基于可靠来源重新编译 |
| Git commit | 提交变更 | 保留完整历史 |
| 验证写入 | 检查文件存在 + frontmatter 完整 | 确保成功 |
| 失败回滚 | `git revert HEAD` | 回滚到上一版本 |

---

## 三、Lint 详细流程

### 3.1 第一层：结构验证（快筛）

| # | 检查项 | 说明 |
|:-:|--------|------|
| 1 | 字段完整性 | 8 个必填字段是否齐全 |
| 2 | 值合法性 | domain/confidence/freshness 枚举值 |
| 3 | 合约一致性 | sources_count == 正文来源列表条数 |
| 4 | 合约一致性 | open_threads_count == 正文 Open Threads 条数 |

### 3.2 第二层：内容质量

| # | 检查项 | 说明 |
|:-:|--------|------|
| 1 | 孤儿知识单元 | confidence=orphan |
| 2 | 缺失交叉引用 | 无入站 [[链接]] |
| 3 | 来源文件存续 | 来源是否还在 sources/ |
| 4 | 不稳定单元 | confidence=unstable 是否需要处理 |
| 5 | 知识单元长度 | 主观判断是否过长（不设硬性阈值） |
| 6 | Open Threads 长期未解决 | Agent 根据上下文判断，超 180 天提醒 |
| 7 | 疑似同义 tags | LLM 判断是否存在同义 tags |
| 8 | aliases_pending | 存在待验证别称的单元 |

> ⚠️ Lint **不检查** stale pages（这是 Query 的职责）。

---

## 四、Query 详细流程

### 4.1 搜索机制

遍历 index.md，匹配：主题名 + aliases + tags + 一句话摘要 + domain。返回所有匹配，不排序，由查询方整合。

### 4.2 组件化返回表

| 组合 | 返回内容 |
|------|---------|
| 快速判断 | frontmatter（8 字段） |
| 标准查询 | frontmatter + 速览 |
| 深度研究 | frontmatter + 速览 + 详细内容 + OT + 来源 |
| 历史追溯 | 全部 + Timeline |

### 4.3 stale 判断

`freshness=time-sensitive` + `last_ingested` 距今超过 90 天 → `stale_warning: true`

---

## 五、定时任务配置

| 任务 | Cron | 类型 | Message |
|------|------|:----:|---------|
| Ingest | `0 2 * * 1` | agent | 执行 Ingest 扫描 |
| Delete | `0 2 1,15 * *` | agent | 执行 Delete 扫描 |
| Lint | `0 2 1 * *` | agent | 执行 Lint 检查 |

> 兜底自检：每次新会话启动时，检查 state.json 时间戳和 .draft/ 待审核草稿。