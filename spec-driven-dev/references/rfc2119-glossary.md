# RFC 2119 关键词速查 + 误用例子

源：[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)。Spec 用 RFC 2119 关键词是为了让 LLM（和读 spec 的人）**对"硬约束 vs 软建议"零歧义**。

---

## 五个关键词的精确语义

| 关键词 | 等价于 | 含义 | 违反后果 |
|---|---|---|---|
| **MUST** / REQUIRED / SHALL | 必须 | 绝对要求 | 不符合 spec |
| **MUST NOT** / SHALL NOT | 绝对禁止 | 绝对禁止 | 不符合 spec |
| **SHOULD** / RECOMMENDED | 应该 | 推荐做法，可有充分理由偏离 | spec-compliant 但需说明 |
| **SHOULD NOT** / NOT RECOMMENDED | 不应该 | 不推荐，可有充分理由偏离 | 同上 |
| **MAY** / OPTIONAL | 可以 | 完全可选 | 任意 |

---

## 用法判断流程

```
这条规则违反会导致下游 broken / 数据损坏 / 安全问题吗？
├─ Yes → MUST / MUST NOT
└─ No → 这是设计 best practice，但允许例外？
        ├─ Yes（应该这么做，但你有充分理由可以不） → SHOULD / SHOULD NOT
        └─ No（完全是品味选择） → MAY
```

---

## 常见误用

### 误用 1：本来应该是 MUST 写成了 SHOULD

❌ 反例：
> "Reason 字段 SHOULD 截断至 500 字符"

后果：LLM 实施时可能不截断，导致 JSONL 单行膨胀到 KB 级，分析时 IO 拖累。

✅ 正例：
> "Reason 长度 > 500 → MUST 截断至前 499 字符 + 末尾 '…'"

判断依据：不截断会导致**下游分析 broken**（性能 / 单行可读性），是硬约束。

---

### 误用 2：本来是 MAY 写成了 SHOULD

❌ 反例：
> "Top reason 样本 SHOULD 每 reason_code 展示 2 条 reason 原文"

后果：LLM 实施时机械地展示 2 条，但有时只有 1 条数据可用，要么报错要么造假。

✅ 正例：
> "Top reason 样本 — 每个高频 reason_code **MAY** 取 1-2 条 reason 原文展示"

判断依据：展示几条是品味选择，不影响数据正确性。

---

### 误用 3：本来是 SHOULD 写成了 MUST

❌ 反例：
> "Hook 总耗时增加 MUST ≤ 5ms"

后果：环境波动或临时磁盘慢导致 6ms 也算违规，无法 enforce。

✅ 正例（要么放宽，要么加限定）：
> "Hook 总耗时增加 **MUST** ≤ 5ms **on warm filesystem (近期 mount + 近期访问)**"

或：
> "Hook 总耗时增加 **SHOULD** ≤ 5ms"（如果允许偶尔超）

判断依据：MUST 必须能机器验证 + 环境无关；做不到就降级 SHOULD + 说明前置条件。

---

### 误用 4：用了模糊措辞替代关键词

❌ 反例：
- "尽量不要 ..."
- "建议 ..."
- "最好 ..."
- "应当避免 ..."

LLM 视这些为可选。

✅ 正例：
- "SHOULD NOT ..."
- "RECOMMENDED ..."
- "MAY avoid ..."
- "SHOULD NOT ..."

---

### 误用 5：双重否定 / 隐式条件

❌ 反例：
> "MUST NOT fail to write trace event unless filesystem is read-only"

LLM 读到：永远要写。但 "unless" 是例外，容易漏。

✅ 正例（拆开）：
> - "On writable filesystem: **MUST** write trace event"
> - "On read-only filesystem: **MUST** no-op silently"

---

## 在 spec 顶部声明

每份 spec 顶部 metadata 表 **MUST**（看，这就是 RFC 2119 用法）有这行：

```markdown
| Normative language | This document uses RFC 2119 keywords: **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** |
```

让 LLM 和读者第一眼就知道：粗体关键词不是修辞，是合同条款。

---

## 跟 Acceptance Criteria 的对应关系

| Spec 措辞 | Acceptance criteria 形式 |
|---|---|
| **MUST** X | `- [ ] 跑测试 verifies X` |
| **MUST NOT** Y | `- [ ] 跑测试 verifies NOT Y`（反向断言） |
| **SHOULD** Z | `- [ ] 默认情况下 verifies Z; 若偏离需 spec 文档说明` |
| **MAY** W | （不需要 acceptance criteria） |

如果 spec 写了 MUST X 但 task 的 acceptance criteria 没对应一条 verifies X → spec 与 task 脱节，必须补。
