# Spec-Driven Dev Checklists

三阶段 checkbox 清单：**写 spec 前 / 实施前 / Review 时**。

---

## A. 写 spec 前自检

- [ ] 我能用一句话说出 spec 要解决什么"工程师不写就 LLM 必踩"的坑吗？
- [ ] 有没有现成的 trade-off 我"懒得写出来"——这条往往就是后续 LLM 偏离的根因
- [ ] 我打算让多少个 LLM session 协作？> 1 个的话必须有 Required reading 显式引用机制
- [ ] V0 范围是不是过大？能否砍到 "1 个 hook + 1 个 lib + 1 个分析脚本" 颗粒度？
- [ ] V1+ 的演进有没有预留？schema_version / extra 字段 / 命名空间

---

## B. Spec 写完后自检（结构层）

### 必填章节

- [ ] §1 Background（数据/价值分层）
- [ ] §2 Goals（含可验证目标表，每行都有 Verification 列）
- [ ] §3 Principles（P1, P2, ... 每条引用具体 §X）
- [ ] §4 Glossary
- [ ] §5 Architecture Overview（**含 ASCII 架构图**）
- [ ] §6 MVP Scope
- [ ] §7 Detailed Design（schema + 完整 example + 字段约束 + error handling + 性能 + 并发 + 隐私）
- [ ] §8 Examples（**每个使用场景一个端到端 walkthrough**）
- [ ] §9 Trade-offs（含拒绝替代方案 + 理由）
- [ ] §10 Future Scope
- [ ] §11 Task List（每 task 都有 Required reading / Inputs / Outputs / Implementation guide / Acceptance criteria）
- [ ] §12 Validation Strategy
- [ ] §13 Open Questions（即使空表也保留章节）
- [ ] §14 References

### 规范语言

- [ ] 所有约束都用了 MUST / SHOULD / MAY / MUST NOT 之一
- [ ] 没有 "尽量"、"最好"、"可能要" 这类模糊措辞
- [ ] 顶部 metadata 表声明了 "This document uses RFC 2119 keywords"

### 示例与代码锚点

- [ ] 每个 schema / API / 数据结构至少 1 个完整 example
- [ ] Example 中的字段值与 schema 描述 100% 一致（容易在改 schema 时漏改 example）
- [ ] 对应到现有代码的地方有 grep-able 锚点（函数名 / 文件路径 / 行号近似）

### 任务可执行性

- [ ] 每个 T-XXX 任务能在 1-3 小时内单 LLM session 完成
- [ ] 每个 task 都有 Required reading 显式指向 spec 章节 + 现有文件路径
- [ ] 每个 acceptance criteria 都是可机器验证的断言
- [ ] 复杂 task 在 Implementation guide 给了**伪代码骨架**（不是完整代码！）

### Trade-off 防止偏离

- [ ] 每个关键设计决策都有对应 TD-X 行
- [ ] TD 表每行至少列 1 个被拒绝的替代方案
- [ ] Spec 顶部立了规则："Agent 如想偏离 TD，MUST 加 OQ，不得自行决断"

---

## C. Spec 完稿后 reverse engineering review

让一个**没参与撰写的 LLM session** 拿着 spec 读现有代码，对照本清单：

### 隐藏假设

- [ ] Spec 里所有"显然"的假设，源码里都验证过吗？
  - 例："output_block 会 sys.exit" → 实际只是 print
  - 例："tool_input 总是 dict" → 实际可能是 str
- [ ] 错误处理路径在源码中真的存在吗？

### Schema 覆盖率

- [ ] Spec 列举的"所有 N 个分支"是不是真的所有？
- [ ] 用 grep / AST 扫一遍现有代码，分支数与 spec 一致吗？

### 命名一致

- [ ] Spec 用的变量名 / 字段名 / 函数名与源码一致
- [ ] 不一致的地方在 spec 里有显式映射说明

### 示例与规则脱节

- [ ] Spec body 写的每条规则，example 都有体现
- [ ] Example 注解里引用的规则，body 都能找到对应章节
  - 反例（你的真实案例）：example 注解 "按 §X 不计入排序权重"，但 §X body 没写"附带展示"规则

### Acceptance criteria 反向覆盖

- [ ] "MUST 做什么" 有正向测试
- [ ] "MUST NOT 做什么" 有反向断言测试
  - 例：MUST 写 trace event → 正向 IT-01
  - 例：MUST NOT 在 cwd 不在 dsproject 时写文件 → 反向 IT-07

---

## D. 实施过程中（每个 task 启动时）

- [ ] LLM session 启动时喂了完整 spec（或至少 task 的 Required reading 全部）
- [ ] task 状态字段（Pending / In Progress / Done）实时更新
- [ ] LLM 遇到歧义时是追加 OQ 还是自行决断？前者 OK，后者必须打断
- [ ] 实施完后逐条勾选 acceptance criteria，未通过的明示原因

---

## E. 多 task 完成后一致性 audit

- [ ] Spec 内部交叉引用都成立（章节号没漂、§X 真的存在）
- [ ] Example 与 schema 各处一致
- [ ] Trade-off 表里被拒绝的方案，没有在 Implementation guide 偷偷复活
- [ ] 同一规则在 §7.1.2 / §7.5 / §7.6 这种多处出现的地方口径一致
- [ ] OQ 表里所有 P0 / P1 的疑问都已经有答案

---

## F. Spec 演进时

- [ ] V0 → V1 没破坏 forward compat？（不删既有字段、不重命名）
- [ ] 重大变更 bump 了 schema_version？
- [ ] Migration 策略写清了？
- [ ] 旧 reader 看到新 event 时的行为是定义过的？
