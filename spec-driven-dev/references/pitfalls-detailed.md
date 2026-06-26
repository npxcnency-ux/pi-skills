# 八大坑详解（含反例 + 修复）

按踩坑频率从高到低排列。每条配真实代码层面的反例。

---

## 坑 1：过度抽象 — Spec 用通用语言

**表现**：Spec 描述用 "the system should record decision events"，LLM 各种"创意"实现：有的写 JSON 有的写 YAML，有的写文件有的写 stdout。

**反例**：
```markdown
The hook SHOULD log its decision for later analysis.
```

LLM 读到这句话可能：
- 写到 stderr（违反 fail-silent）
- 写到 `~/.local/log/`（违反 per-project）
- 用 Python logging（引入外部依赖）
- 调 `print()`（污染 hook stdout）

**修复**：把每个不变量字面化到约束 + example。

```markdown
The hook MUST append decision events to:
  <project_root>/.dsbot/logs/run_trace.jsonl

Each event MUST be a single-line JSON object matching schema in §7.1.
Example line:
  {"schema_version": 1, "event_id": "...", "decision": "fail", ...}

The hook MUST NOT write to stderr/stdout from telemetry path (see §7.6).
```

---

## 坑 2：任务过大 — 一个 task 改 8 个文件

**表现**：T-001 "实现整个 trace 系统"，LLM 上下文溢出 → 偷工减料 / 跳过测试 / 漏改文件。

**反例**：
```markdown
### T-001: 实现 Run Trace V0
全部实现：lib + hook + analysis script + tests + docs
```

**修复**：拆到单 session 颗粒度，明确依赖：

```markdown
T-001: 实现 lib/run_trace.py（无依赖，~150 行 Python）
T-002: 为 lib/run_trace.py 写单测（depends on T-001）
T-003: 改造 hooks/node_transition_hook.py（depends on T-001）
T-004: 写 e2e 测试（depends on T-003, T-005）
T-005: 实现 scripts/trace_stats.py（无依赖）
T-006: 为 trace_stats.py 写单测（depends on T-005）
T-007: 写用户文档（depends on T-003, T-005）
T-008: 更新 README / AGENTS（depends on T-007）
```

每个 task 都能在 1-3 小时单 session 内完成。

---

## 坑 3：隐藏假设 — "exit code 当然是 2"

**表现**：Spec 写 "Hook 在 fail 分支输出阻断 JSON 并 exit code 2"，但读源码发现 `output_block` 只 print，从不 sys.exit。LLM 按 spec 实施可能引入新的 sys.exit 逻辑，破坏原行为。

**反例 spec 措辞**：
```markdown
Hook 已通过 stdout 写好阻断 JSON + 设好 exit code 2，与改造前完全一致
```

**实际源码**：
```python
def output_block(continue_val: bool = True, ...):
    print(json.dumps(result, ensure_ascii=False))
    # 没有 sys.exit
```

**修复**：reverse engineering pass 时验证每个"现状描述"。改写为：

```markdown
Hook 通过 stdout 输出 `{"continue": false, "stopReason": "..."}` 由 Claude Code
runtime 解释为阻断信号；Python 进程本身按 exit code 0 退出。Telemetry 注入
MUST NOT 引入任何 sys.exit 调用。
```

---

## 坑 4：示例与规则脱节 — Example 展示了 body 没写的规则

**表现**：你今天的 Bug 4——示例里写 "pass 时附带显示，按 §7.5.2 不计入 top-K 排序权重"，但翻 §7.5.2 body 只说"按 fail/skip 降序"，**完全没提附带展示规则**。

**反例**：
```markdown
## §7.5.2
5. Top 10 reason_code — 按 fail / observable-skip 出现次数降序

## §8.2 示例输出
| check_n6_output | 2 (pass) | n6-output 制品文件 cp 完成（pass 时附带显示，按 §7.5.2 不计入 top-K 排序权重）|
```

实施 T-005 的 LLM 按 §7.5.2 读，根本不会展示 pass count——和示例对不上。

**修复**：示例注解的每条规则都必须能在 spec body grep 到。

```markdown
## §7.5.2
5. Top 10 reason_code — 按 fail / observable-skip 出现次数降序（reason_code
   聚合，避免自然语言变体打散）。同一 reason_code 的 pass count MAY 在该行附带
   展示用于 pass rate 参考，但 MUST NOT 计入 top-K 排序权重
```

---

## 坑 5：缺反向断言

**表现**：测试只覆盖了 "MUST 做什么"，没覆盖 "MUST NOT 做什么"。LLM 实施时可能把"什么场景都写一条"当 feature，污染数据。

**反例 acceptance criteria**：
```markdown
- [ ] B6 / B7 / B8 / B9 各 instrument 一次（observable 分支写入）
```

只验证了写入。没验证 B1-B5 expected skip 不写入。LLM 可能写出"all branches always write"，把噪音灌满 jsonl。

**修复**：

```markdown
- [ ] B6 / B7 / B8 / B9 各 instrument 一次（正向）
- [ ] B1-B5 expected skip 分支 MUST NOT 写入 jsonl（反向断言）
  - IT-03: current_step ∉ [02,04,05] → jsonl 0 行新增
  - IT-04: TodoWrite 没标 completed → jsonl 0 行新增
  - IT-07: cwd 不在 dsproject → jsonl 不存在
```

---

## 坑 6：MAY / SHOULD / MUST 混用

参见 `rfc2119-glossary.md`。

**最坏情况**：spec 全篇用"应该 / 建议 / 推荐"，没有任何 RFC 2119 关键词。LLM 把所有规则都当成可选。

---

## 坑 7：无 OQ 协议 — LLM 自作主张埋雷

**表现**：Spec 没立"遇歧义必须追加 OQ"规则。LLM 遇到不确定就脑补：
- "spec 没说 reason 字段编码，那我用 utf-8 with BOM 吧"
- "spec 没说默认 reason_code，那我空字符串吧"

埋下与 spec 隐式假设冲突的雷。

**修复**：spec 顶部立规：

```markdown
## §3 Principles
> Agent 实施时遇到本 spec 未覆盖的决策点，**MUST** 在 §13 Open Questions 追加
> OQ-XX 并暂停，**MUST NOT** 自行决断。

## §13 Open Questions
| OQ ID | 问题 | 状态 |
|---|---|---|
| OQ-01 | <实施中发现的歧义点> | 待定 |
```

---

## 坑 8：不留演进空间 — V1 加字段就要 migration

**表现**：V0 schema 没有 schema_version 字段、没有 extra 扩展点。V1 想加新字段就要写 migration 脚本兼容旧 jsonl。

**反例 V0 schema**：
```yaml
event:
  ts: string
  decision: string
  reason: string
```

V1 想加 `step` / `node` 字段 → 旧 jsonl 没有 → 分析脚本爆 KeyError。

**修复**：day 1 就留好演进位：

```yaml
event:
  schema_version: integer   # MUST == 1 此 spec 范围
  ts: string
  decision: string
  reason: string
  extra: object             # MAY 含任意附加键值
```

V1 加字段策略：
1. 新字段 MUST 为可选 + 有默认值
2. 不删除 / 不重命名既有字段
3. 重大不兼容变更才 bump schema_version 到 2
4. 分析脚本 MUST 检查 schema_version，未知版本计入 unknown_schema_versions

---

## 总览：从坑 → 修复机制

| 坑 | 防御机制 |
|---|---|
| 1. 过度抽象 | Example + 代码锚点 |
| 2. 任务过大 | 拆到单 session 颗粒度 |
| 3. 隐藏假设 | Reverse engineering pass |
| 4. 示例与规则脱节 | Spec 完稿后做"示例 ↔ body" grep 互查 |
| 5. 缺反向断言 | Acceptance criteria 强制正反双面覆盖 |
| 6. RFC 2119 混用 | rfc2119-glossary.md 速查 |
| 7. 无 OQ 协议 | spec §3 立规则 + §13 留模板 |
| 8. 不留演进空间 | schema_version + extra 字段从 day 1 |
