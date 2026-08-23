# 设计原理与背景

> 本文件是知识库管理系统的设计依据，供 Agent 在遇到设计原则冲突时参考。
> 不包含操作指令，仅记录核心理念和权衡决策。

---

## 一、核心定位

基于 Karpathy LLM Wiki 模式的**编译型知识库系统**。核心理念：

> "Obsidian 是 IDE；LLM 是程序员；wiki 是代码库；**但人类是代码审查员。**"
> — Karpathy 原话 + 本方案的修正

### 与 Karpathy 原方案的关键差异

| 维度 | Karpathy | 本方案 | 选择原因 |
|------|---------|--------|---------|
| 更新方式 | str_replace 局部外科手术 | 重新编译 Compiled Truth | LLM 无法可靠精确定位；frontmatter 与正文是联动决策 |
| 错误防护 | Git + Lint 后置检查 | 草稿区 + 审核前置 + 程序验证 | 错误在入库前拦截，降低人工审查负担 |
| 历史记录 | 单一平面页面 | Compiled Truth + Timeline 两层 | 上层可重写，下层保留所有历史证据 |

## 二、两层存储模型

| 层 | 名称 | 特性 | 内容 |
|---|------|------|------|
| 上层 | **Compiled Truth** | 可重写 | frontmatter + 速览 + 详细内容 + Open Threads + 来源 |
| 下层 | **Timeline** | Append-only | 更新日志 + 来源记录 |

**核心价值**：上层放心重写（不需要担心信息丢失），下层保留所有历史证据。

## 三、Agent 架构

| Agent | 职责 | 边界 |
|-------|------|------|
| **知识库管理师** | 初始化、Ingest、Delete、Lint、Query | 从 sources/ 读 → 编译到 wiki/ |
| **知识问答与沉淀师** | 问答 + 学习笔记 + 写入日常 vault | 不写 sources/ 和 wiki/ |
| **用户** | 全权限（审核、指示、配置） | — |

## 四、三层内容返回模型

查询时按组件组合返回，而非全文倒灌：

| 组合 | 内容 | 场景 |
|------|------|------|
| 快速判断 | 仅 frontmatter | 判断是否有相关内容 |
| 标准查询 | frontmatter + 速览 | 大多数查询 |
| 深度研究 | frontmatter + 速览 + 详细内容 + OT | 深入研究 |
| 历史追溯 | 全部 + Timeline | 来源验证 |

## 五、风险防护清单（摘要）

| 风险 | 防护 |
|------|------|
| sources/ 被覆盖 | 只读铁律 + 写入权仅归用户 |
| LLM 误解来源 | 强制来源标注 + 后置验证 + 审核前置 |
| 自指漂移 | Ingest 时必须读 sources/ 原始来源 |
| Lint 报告膨胀 | schema 质量 + 审核前置 |
| freshness 误标 | 默认保守 + 用户修正 + Lint 复查 |
| frontmatter 不一致 | 后置程序验证（Harness 9 条） |

## 六、Karpathy 借鉴的关键失败模式

| 失败模式 | 应对 |
|---------|------|
| Silent Corruption | 强制来源标注 + 后置程序验证 + 审核前置 |
| Self-referential Drift | Ingest 时读 sources/，不依赖已有 wiki |
| Maintenance Ratchet | schema 质量 + 审核前置减轻审查负担 |

---

## 七、设计决策记录

| # | 决策 | 理由 |
|:-:|------|------|
| 1 | 以 v3.7 全流程设计为唯一核对基准 | 用户明确要求 |
| 2 | Skill 同时覆盖两套 Agent 逻辑 | 管理师 + 问答师需在同一知识域中协作 |
| 3 | Delete 安全防护依赖 Git | v3.7 §4.2 明确依赖 Git（commit→验证→revert） |
| 4 | 对话沉淀完全用户驱动 | v3.7 §8.1 明确"Agent 不主动建议" |
| 5 | 不写脚本，纯文档型 Skill | v3.7 的"后置程序验证"是 Agent 按规则执行；纯文档更轻、零维护 |
| 6 | state.json 对齐 v3.7 字段 | last_scan/last_lint/last_delete_check，无 created |
| 7 | MCP 演进（v3.7 §16）不进入 v1 | 属 Phase 2 技术演进，v1 保留 chat_with_agent 通道 |

---

## 八、后续演进方向

- MCP 工具（kb_query / kb_info / kb_update_vector）
- 向量索引（vectors.json + 语义搜索）
- Web UI 审核界面扩展
- 多知识库支持（当前只支持 1 个）