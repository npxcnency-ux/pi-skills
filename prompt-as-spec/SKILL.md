---
name: prompt-as-spec
description: 把一次性大任务打包成单个高质量 LLM prompt 的方法论。当用户要给 LLM 一次性派活（实现完整功能 / app / 模块），或在 vibe coding 和正式 spec 之间纠结，或想评估自己写的 prompt 够不够"防跑偏"时使用。覆盖六大原则（决策前置、可量化验收、反例约束、领域知识注入、交付契约、时序闭合）、vibe coding 边界管理、翻车预警信号、prompt 自检清单。
---

# Prompt as Spec

把一次性大 prompt 当成**导出的 design doc** 来写——LLM 只负责把意图翻译成代码，不负责做决策。

与 `spec-driven-dev` 的关系：

| | spec-driven-dev | prompt-as-spec（本 skill） |
|---|---|---|
| 形态 | 正式 spec 文档（RFC/ADR） | 一次性 prompt |
| 体量 | ≥ 1 day 工作量、多 session | 单 session 能跑完 |
| 严肃度 | 团队级、要 review | 个人级、即用即弃 |
| 关键产物 | 长期可维护文档 | 一次跑出能用的代码 |

两者共享同一心智（**让 LLM 只翻译不决策**），但工具和颗粒度不同。

## 何时用本 skill

- 要给 LLM 一次性派一个"中等以上"任务（写一个完整 app / 模块 / 工具）
- 评估别人或自己写的 prompt 是否够稳
- vibe coding 走不动了，想切换姿势但又不想写完整 spec
- 同一个任务反复让 LLM 写，每次产出都不一样

## 核心心智

| 心智 | 含义 |
|---|---|
| Prompt 是导出的 design doc | 你脑子里那份设计，要"序列化"成文字喂给 LLM |
| LLM 不会替你做决策，只会猜 | 关键决策没写 = LLM 随机挑一个，你也分辨不出对错 |
| 反例约束 > 正例描述 | "不要 XX" 比 "要 XX" 更能压缩自由发挥空间 |
| 领域陷阱必须显式点破 | LLM 不会主动规避平台坑（输入法、权限、事件链），你不说它就踩 |
| 主观词必须翻译成数字 | "优雅"、"流畅"、"足够大" 在 LLM 眼里都是噪声 |

**判别 prompt 好不好的最简标准**：两个不同 LLM session 跑同一 prompt，输出是否实质等价？等价 → 合格。

---

## 六大原则（按重要性）

### 1. 决策前置（最关键）

所有 LLM 容易自由发挥的点，必须在 prompt 里钉死：

- **技术栈**：语言、框架、构建系统、版本
- **关键 API / 库**：用哪个，不用哪个
- **架构模式**：单文件 / 分层 / 状态机的具体形态
- **数据结构**：核心 schema 长什么样

❌ "做个 menubar 应用"
✅ "macOS 14+，Swift + SPM，LSUIElement 模式，悬浮窗用 NSPanel(nonactivatingPanel) + NSVisualEffectView(.hudWindow)"

判据：LLM 还能在哪些点上"自由发挥"？能想到 → 没钉够。

### 2. 可量化的验收点

把主观词翻译成数字。每个能量化的维度都量化：

❌ "波形动画要优雅"
✅ "5 根竖条 44×32px，权重 [0.5, 0.8, 1.0, 0.75, 0.55]，attack 40% release 15%，±4% 随机抖动"

❌ "弹窗要好看"
✅ "高 56px，圆角 28px，弹性宽度 160-560px，入场弹簧动画 0.35s"

判据：prompt 里数字密度。一段需求 0 个数字 → 大概率主观词没翻译。

### 3. 反例约束（"不要 XX"）

反例约束精准排除 LLM 的高概率错误路径，比正例描述更省字、更有效：

- "波形**不要用写死的假动画**"（堵偷懒）
- "LLM refine **绝对不要改写、润色或删除**"（防过度发挥）
- "**不要红绿灯和 titlebar**"（防默认 window 样式）
- "**不要加单元测试 / 不要重构现有代码 / 不要换库**"（典型 vibe 救命三连）

判据：写完 prompt 后想 LLM 最可能怎么跑偏，每条都补一句反例。

### 4. 领域知识注入（why，不只是 what）

不要干巴巴列 API，把"为什么这么做"嵌进去。LLM 拿到 why 之后能迁移到类似场景：

- "Fn 通过 CGEvent tap...**防止触发 emoji 选择器**"
- "切 ASCII...**防止中文输入法拦截 Cmd+V**"
- "LLM 纠错例子：**配森 → Python、杰森 → JSON**"

为什么必要：LLM 通用知识里**没有**平台特有陷阱，这些必须由你显式注入。

判据：prompt 里平台坑 / 业务约束 / 反直觉决策，是否都带了 why 或例子。

### 5. 完整交付契约

不只说功能，还说**交付物长什么样**：

- 构建系统（Make / npm scripts / SPM）
- 入口点（main 函数 / Makefile target / CLI 命令）
- 产物形态（.app bundle / 单文件 / Docker image）
- 运行模式（daemon / one-shot / interactive）
- 签名 / 权限 / 配置文件位置

判据：LLM 写完，能直接 `make build && make run` 跑起来吗？不能 → 契约不全。

### 6. 端到端时序闭合

复杂功能不要孤立列需求，把**状态流转**显式串起来：

✅ "松开 Fn → 检查 LLM 是否启用 → 已启用则悬浮窗显示 Refining... → 等 LLM 返回 → 注入最终文本 → 退场动画"

判据：从用户触发到结束，每个状态切换和分支是否都覆盖。

---

## Vibe Coding 边界管理

prompt-as-spec 不是要消灭 vibe coding，而是知道**什么时候 vibe、什么时候不能 vibe**。

### 适用场景对照

| 场景 | Vibe | Prompt-as-Spec | 正式 Spec |
|------|------|----------------|-----------|
| 探索想法、原型 | ✅ | — | — |
| 一次性脚本 / glue code | ✅ | — | — |
| 学新框架 | ✅ | — | — |
| 一个完整功能 / 小 app | ❌ | ✅ | — |
| 涉及平台陷阱 | ❌ | ✅ | ✅ |
| 多文件 / 多模块协作 | ❌ | ⚠️ | ✅ |
| 团队协作 / 长期维护 | ❌ | ❌ | ✅ |

### Vibe 时也该做的最低限度

哪怕想保持 vibe 的轻松感，这四件事成本极低收益极高：

1. **30 秒决策**：开聊前一句话说清技术栈 + 2-3 个关键决策
2. **反例约束**：临时加 "不要 XX"，比反复纠正高效
3. **关键数字别让 LLM 猜**：尺寸 / 时长 / 数量一句话钉死
4. **每个能跑的小特性就 commit**：留回退点

### 翻车预警信号

出现这些信号 → 立刻从 vibe 切到 prompt-as-spec（或更上一级到正式 spec）：

| 信号 | 含义 |
|------|------|
| 同一个 bug 改 3 次还没好 | 上下文已乱，停下来重新组织 prompt |
| LLM 开始加无关功能 | 约束不够，明确反例 |
| 出现平台/框架陷阱 | 领域知识没注入 |
| 代码量 > 500 行 | 单 session 颗粒度到顶，该拆 |
| 自己读不懂生成的代码 | 已经在埋雷 |
| 跨多个文件来回改 | 缺整体设计，需要正式 spec |

### 半结构化工作流（vibe-friendly 版）

```
1. 30 秒决策：技术栈 + 关键约束（2-3 句）
2. 写一个"轻量 prompt"：六大原则中至少覆盖 1、3、5
3. LLM 生成骨架 → 跑起来
4. vibe 迭代细节
5. 每个能跑的小特性 git commit
6. 出现预警信号 → 停下来 → 升级到完整 prompt-as-spec
```

---

## Prompt 自检清单

写完 prompt 后过一遍：

- [ ] 技术栈、框架、关键 API 是否都钉死
- [ ] 每段主观词（优雅/流畅/合理）是否都翻译成了数字
- [ ] 至少写了 3 条反例约束（"不要 XX"）
- [ ] 平台陷阱 / 反直觉决策是否带了 why 或例子
- [ ] 构建命令、运行方式、产物形态是否明确
- [ ] 复杂流程是否端到端串过一遍（从触发到结束）
- [ ] 想 LLM 还能在哪些点跑偏 → 每个都补一句
- [ ] 异步 / 并发 / 错误兜底 / 失败路径是否覆盖

最终自测：两个不同 LLM session 跑这份 prompt，结果会等价吗？

---

## 衡量"prompt-as-spec 是否真起作用"

| 指标 | 健康 | 病态 |
|---|---|---|
| LLM 一次跑通率 | ≥ 70% | < 30% |
| 跑出来代码能直接 build & run | ✅ | 需手动改才能跑 |
| 反例约束数量 | ≥ 3 | 0 |
| Prompt 里数字密度 | 主观词都有数字配 | 全是形容词 |
| 两个 session 输出等价率 | ≥ 80% | < 50% |

---

## References（按需加载）

- [`references/voice-input-app.md`](references/voice-input-app.md) — 一个完整范例 prompt（macOS menubar 语音输入法），逐段标注体现的原则
- [`references/before-after.md`](references/before-after.md) — 几组"差 prompt → 好 prompt"对照
- [`references/checklist.md`](references/checklist.md) — 可打印的 prompt 自检 checkbox

---

## 与其他 skill 的协作

| 场景 | 该用哪个 skill |
|------|---------------|
| 一次性大 prompt | prompt-as-spec（本 skill） |
| 要写持久化文档、给团队 review | spec-driven-dev |
| 给 pi 写扩展 | pi-extension-dev |
| Vibe 完想沉淀经验 | prompt-as-spec → 升级到 spec-driven-dev |
