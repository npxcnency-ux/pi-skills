# Spec 骨架模板

Copy 整段作为新 spec 起点。按 `<...>` 替换占位符。

---

# <Project / Feature> — Specification

| Field | Value |
|-------|-------|
| Spec ID | `<PROJECT-FEATURE-V0>` |
| Version | `0.1.0` |
| Status | Draft |
| Author | <team> |
| Last updated | <YYYY-MM-DD> |
| Supersedes | (none) |
| Implements | <brief one-liner> |
| Normative language | This document uses RFC 2119 keywords: **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** |

---

## Table of Contents

- [§1 Background](#1-background)
- [§2 Goals](#2-goals)
- [§3 Principles](#3-principles)
- [§4 Glossary](#4-glossary)
- [§5 Architecture Overview](#5-architecture-overview)
- [§6 MVP Scope](#6-mvp-scope)
- [§7 Detailed Design](#7-detailed-design)
- [§8 Examples](#8-examples)
- [§9 Trade-offs & Alternatives](#9-trade-offs--alternatives)
- [§10 Future Scope](#10-future-scope)
- [§11 Task List](#11-task-list)
- [§12 Validation Strategy](#12-validation-strategy)
- [§13 Open Questions](#13-open-questions)
- [§14 References](#14-references)

---

## §1 Background

<为什么需要这个？现状的痛点？数据价值分层？>

## §2 Goals

> 核心思想：<一句话>

#### 可验证目标

| ID | Goal | Verification |
|----|------|--------------|
| G1 | <可执行目标> | <如何验证> |
| G2 | <…> | <…> |

## §3 Principles

| # | Principle | 含义 | 体现位置 |
|---|---|---|---|
| P1 | <Principle 名> | <字面化定义> | §X |
| P2 | <…> | <…> | <…> |

> 这些 principle 是后续所有设计决定的根因。Agent 遇到与之矛盾的实现选择，**MUST** 在 §13 标记 OQ，**MUST NOT** 自行决断。

## §4 Glossary

| 术语 | 定义 |
|---|---|
| **<术语>** | <定义> |

## §5 Architecture Overview

### §5.1 组件关系

```
<ASCII 架构图>
```

### §5.2 数据流

1. <步骤 1>
2. <步骤 2>

## §6 MVP Scope

### §6.1 V0 范围

| 组件 | V0 状态 |
|---|---|
| <module> | 完整实现 / 部分 / 不实现 |

### §6.2 V0 完成的判定

参见 §12 Validation Strategy。

## §7 Detailed Design

### §7.1 Schema / API

```yaml
field_name: type    # MUST/SHOULD/MAY 说明
```

#### §7.1.1 完整 example

```json
{"field_name": "..."}
```

#### §7.1.2 字段约束

- `<field>` 不合法 → MUST <行为>
- `<field>` 超长 → MUST <行为>

### §7.2 Storage / 输出格式

### §7.3 Library API

```python
def public_api(...) -> None:
    """docstring 含 spec 路径引用"""
```

### §7.4 Integration Pattern

### §7.5 Error Handling

| Failure | Behavior |
|---|---|
| <错误类型> | <处理方式> |

### §7.6 Performance Constraints

- <操作> **MUST** ≤ <延迟>
- <频率> **SHOULD NOT** ><值>

### §7.7 Concurrency Model

### §7.8 Privacy & Security

## §8 Examples

每个使用场景给端到端 walkthrough。

### §8.1 Example E1 — <场景标题>

**场景**：<一句话>

**触发路径**：
1. <step>
2. <step>

**预期输出**：

```json
<完整 example>
```

## §9 Trade-offs & Alternatives

| # | Decision | 选 | 拒绝的替代 + 理由 |
|---|---|---|---|
| TD-1 | <决策点> | **<选项>** | ❌ <替代>：<理由><br>❌ <替代>：<理由> |

> LLM 实现时如果"觉得"某项决策应该改：**STOP**，去 §13 加 OQ，不要自己调整。

## §10 Future Scope (V1+)

### §10.1 Tier 2 — <扩展方向>

### §10.2 Tier 3 — <扩展方向>

## §11 Task List

### T-001: <任务标题>

| 属性 | 值 |
|---|---|
| Status | Pending |
| Type | lib / test / hook / script / doc / meta |
| Depends on | (none) |
| Required for MVP | Yes |

#### Goal
<一句话目标>

#### Required reading
- 本 spec §X (含具体子节)
- 现有文件: `<absolute path>` (理由)

#### Inputs
<前置 task 产物>

#### Outputs
<新增/修改文件清单>

#### Implementation guide

1. <步骤 1>
2. <步骤 2 + 伪代码（不是完整代码）>

#### Acceptance criteria
- [ ] <可机器验证的断言>
- [ ] <…>

#### Notes for executing agent
- 不要 <边界提醒>
- 如发现 §X 自相矛盾，追加到 §13 OQ，不要自行解决

---

### T-002: <…>

<同上结构>

## §12 Validation Strategy

### §12.1 测试金字塔

```
       ┌──────────┐
       │   E2E    │
       ├──────────┤
       │   Unit   │
       └──────────┘
```

### §12.2 测试命令

```bash
python3 -m pytest tests/ -v
```

### §12.3 决策门：是否进入下一 Tier

| 问题 | 通过条件 |
|---|---|
| Q1: <…> | YES = <…> |

## §13 Open Questions

| OQ ID | 问题 | 状态 |
|---|---|---|
| OQ-01 | <问题> | 待定。<当前倾向> |

## §14 References

### §14.1 项目内文档

- `<path>` — <说明>

### §14.2 源码

- `<path>` — <说明>

### §14.3 外部规范

- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)

### §14.4 Schema 演进政策

`schema_version` 字段命名空间属于本 spec。任何新增字段 **MUST** 兼容 V1 reader（不删除/重命名既有字段；新字段为可选）。重大不兼容变更 **MUST** bump `schema_version` 到 2。

---

**End of <PROJECT-FEATURE-V0> v0.1.0**
