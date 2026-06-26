---
name: spec-driven-dev
description: 用 spec 驱动 LLM 写代码以提升质量、减少返工的最佳实践。当用户要写技术 spec / RFC / ADR / design doc，或要拆解大任务给 LLM 执行，或抱怨"LLM 输出每次都不一样 / 总跑偏 / 改完埋雷"时使用。覆盖 RFC 2119 规范语言、trade-off 决策表、Open Questions 协议、任务拆分到单 session 颗粒度、acceptance criteria checkbox、reverse engineering review、避坑清单与衡量指标。
---

# Spec-Driven LLM Development

把工程师脑中的隐式约定**显式化、可执行化、可验证化**——让 LLM 只做翻译，不做决策。

## 何时用本 skill

- 要写一份新 spec / RFC / design doc，给后续 LLM session 实施
- 已有 spec 但 LLM 实施总跑偏 / 输出不稳定 / 改完埋雷
- 要拆大任务（≥ 1 day 工作量）给多个 LLM session 协作
- 做 spec 一致性 audit / reverse engineering review

## 核心心智

| 心智 | 含义 |
|---|---|
| Spec 是合同，不是文档 | LLM 不会自动遵守"良好工程实践"，约束必须显式写出 |
| Spec ≠ PRD | PRD 写"做什么"；Spec 写"做的时候不能怎样、必须怎样、产出长什么样" |
| LLM 是 junior contractor，不是同事 | 不要假设它懂你的意图，意图必须**字面**化 |
| Single source of truth | Spec / 代码 / 测试三者矛盾时，按约定挑一个为准（通常是 spec），其他对齐 |

**判别"spec 写得好不好"最简标准**：给两个不同 LLM session 同一份 spec 跑同一个 task，输出是否实质等价？等价 → 合格。不等价 → 还有暗坑。

---

## 八大实践（按重要性）

### 1. RFC 2119 规范语言（MUST / SHOULD / MAY / MUST NOT）

LLM 对模糊措辞极敏感。"建议"、"最好"、"可能要" 在 LLM 眼里 = 可选。

| ❌ | ✅ |
|---|---|
| "尽量不要写 stderr" | "**MUST NOT** write to stderr" |
| "应该 ≤ 5ms" | "**MUST** ≤ 5ms on warm filesystem" |
| "失败可以重试" | "On failure **MUST** no-op; **MUST NOT** retry" |

判据：每条规则能不能写成可机器验证的断言。

### 2. 拒绝替代方案表（Trade-offs）

LLM 实现时很喜欢"我有个更好的主意"——因为它没看过你权衡过的语境。**把拒绝理由预先冻结**到 spec 里。

每行：Decision / 选 / 拒绝的替代 + 理由。模板见 `references/spec-template.md`。

判据：如果 LLM 想偏离决策，能在 spec 找到反驳依据吗？答不出 → 补 TD 行。

### 3. Open Questions 显式留白协议

Spec 永远不可能 100% 完备。**关键是别让 LLM 偷偷脑补**。

立规：
> Agent 在实施过程中如遇歧义，**MUST** 追加 OQ-XX，**MUST NOT** 自行决断

判据：审查时 OQ 增长 > 0 = 协议生效；永远空表 = LLM 在偷偷决断。

### 4. 任务拆分到单 LLM session 颗粒度

每个 task 必须能在 1-3 小时 + 单文件聚焦内完成。模板：

```
T-XXX:
  Goal: 一句话目标
  Required reading: 显式 spec 章节 + 现有文件清单（绝对路径）
  Inputs: 前置 task 产物
  Outputs: 新增/修改的文件清单
  Implementation guide: 关键步骤 + 伪代码（不是完整代码）
  Acceptance criteria: 可勾选 checkbox 列表
  Notes for executing agent: 边界提醒
  Depends on: T-YYY
```

反例："T-001 实现整个系统" → LLM 必然偷工减料。

### 5. Example > 抽象描述

LLM 对 schema 描述的执行不如对**完整 example** 的复现。

每个 schema / API / 数据结构 **MUST** 至少配 1 个完整 example：

```yaml
# ❌ 抽象
event_id: string  # 32-char hex
```
```json
// ✅ schema + 完整 example
{"event_id": "a1b2c3d4e5f6789012345678abcdef00", "ts": "2026-06-25T10:30:15.123Z"}
```

### 6. 代码锚点（Verbatim Source Mapping）

让 spec 显式映射到现有代码位置——防止 spec 描述和源码漂移。

```
| Branch | 代码行 | 触发条件 | 处理 |
| B7 | ~606-613 | check_fn 返 False | record_hook_decision(decision="fail") |
```

行号会漂没关系，作为"实施时校对"锚点足够。

判据：spec 里"代码长这样"的描述能不能 grep 到。

### 7. Acceptance Criteria = 可勾选测试列表

每条都是可执行断言：

```
- [ ] python3 -c "from lib.run_trace import record_hook_decision" 不抛错
- [ ] reason_code="RANDOM" 时 event.reason_code == "_invalid_reason_code"
- [ ] tmp_path 只含 project.yaml 时项目根仍可被找到
```

判据：每条 criteria 能不能跑成一个测试用例。不能 → 改写。

### 8. Reverse Engineering Pass

Spec 完稿后，让**没参与撰写的 LLM session** 拿着 spec 读现有代码，找：

| 检查项 | 例子 |
|---|---|
| 隐藏假设是否成立 | "spec 假设 output_block 会 sys.exit，实际不会" |
| Schema 覆盖所有真实分支吗 | "spec 说 5 个分支，源码其实有 9 个" |
| 命名一致吗 | "spec 用 current_step，源码用 step" |
| 示例与规则脱节吗 | "example 注解了规则但 spec body 没写" |

修完一轮，再让另一 session 复核。

---

## 工作流（典型 spec-driven 循环）

```
1. Spec v0.1: MUST/SHOULD + Trade-offs + Example + OQ + Task 拆分
              ↓
2. Reverse engineering review: LLM 读 spec + 现有代码 → 列不一致 → 修订到 v0.2
              ↓
3. 实施: 按 task 顺序, 每 task 一个 LLM session
        ├ Session 启动喂 spec + 该 task Required reading
        ├ LLM 实现 → 自检 acceptance criteria
        └ 遇歧义 → 追加 OQ 而非自行决断
              ↓
4. 一致性 audit: Spec 内部交叉引用自洽 / Example 与 schema 一致 / 修复后回归扫描
              ↓
5. Spec 演进: schema_version 从 day 1 留好, V1+ 演进路径预先勾勒
```

---

## 避坑清单（按踩坑频率从高到低）

| # | 坑 | 对策 |
|---|---|---|
| 1 | 过度抽象（spec 用通用语言）| 强制 example + 代码锚点 |
| 2 | 任务过大 | 拆到单 LLM session 颗粒度（1-3h，单文件聚焦） |
| 3 | 隐藏假设（"exit code 当然是 2" 但代码不是）| Reverse engineering pass |
| 4 | 示例与规则脱节（example 展示了 body 没写的规则）| 任何 example 注解都回链到 body |
| 5 | 缺反向断言（"B6 写入"测了，"B1 不写入"没测）| Acceptance criteria 加反向 case |
| 6 | MAY/SHOULD/MUST 混用 | 严格 RFC 2119 |
| 7 | 无 OQ 协议 | 显式立规"必须追加 OQ 而非自行决断" |
| 8 | 不留演进空间（V1 加字段就要 migration）| schema_version + extra/optional 字段从 day 1 留好 |

---

## 衡量"spec-driven 是否真起作用"

| 指标 | 健康 | 病态 |
|---|---|---|
| 单 task 返工次数 | ≤ 1 | ≥ 3 |
| OQ 增长率 | 实施中持续增长（spec 有缝但被显式捕获）| 0（LLM 偷偷决断）|
| Spec 行数 / 代码行数 | 0.3 ~ 1.0 | < 0.1（spec 太薄）/ > 3.0（过度规范）|
| LLM 首次通过 acceptance criteria 比例 | ≥ 80% | < 50% |
| 两个独立 session 同 task 输出等价率 | ≥ 90% | < 60% |

---

## References（按需加载）

- [`references/spec-template.md`](references/spec-template.md) — 完整 spec 骨架，含 §1-§14 标准章节，可直接 copy-paste
- [`references/checklists.md`](references/checklists.md) — 写 spec 前 / 实施前 / review 时各自的 checkbox 清单
- [`references/rfc2119-glossary.md`](references/rfc2119-glossary.md) — MUST/SHOULD/MAY 等关键词的精确语义与误用例子
- [`references/pitfalls-detailed.md`](references/pitfalls-detailed.md) — 8 大坑的详细反例与修复方案

---

## 行业参考

| 实践 | 来源 | 借鉴点 |
|---|---|---|
| RFC 2119 | IETF | MUST/SHOULD/MAY 体系 |
| ADR | Michael Nygard | 每个决策含"拒绝替代方案 + 理由" |
| OpenAPI spec-first | OpenAPI 社区 | schema 先于代码 |
| Anthropic Skills / Claude Code | Anthropic | spec 即 prompt，可被 LLM 直接消费 |
| Cursor `.cursorrules` / Aider conventions | Cursor / Aider | 把工程约束做成可被 LLM mount 的 context |
| Spec-First TDD | Kent Beck 改造版 | 先写 spec、再写测试、最后写代码 |
