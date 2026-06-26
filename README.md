# pi-skills

Personal skill bundles for [pi coding agent](https://github.com/earendil-works/pi).

pi 启动时会自动扫描 `~/.pi/agent/skills/` 下所有 `SKILL.md` 并按 description 匹配触发。

## Skills

| Skill | 用途 |
|---|---|
| [pi-extension-dev](pi-extension-dev/) | 开发 pi 扩展时避坑用的核心知识。涉及 tool/command/event 注册、ctx.newSession、subagent 协作、OpenAI-compatible provider 对接、扩展加载错误调试。 |
| [spec-driven-dev](spec-driven-dev/) | 用 spec 驱动 LLM 开发，提升代码质量、减少返工。涵盖 RFC 2119 规范语言、trade-off 决策表、Open Questions 协议、任务拆分、acceptance criteria、reverse engineering review 等八大实践。 |

## 安装

```bash
git clone git@github.com:npxcnency-ux/pi-skills.git ~/.pi/agent/skills
```

或挂载到其他 pi skill 路径见 [pi skills 文档](https://github.com/earendil-works/pi/blob/main/docs/skills.md)。

## 添加新 skill

每个 skill 是一个目录 + `SKILL.md`（含 frontmatter 的 markdown），最小结构：

```
my-skill/
├── SKILL.md              # 必需，含 name / description frontmatter
├── scripts/              # 可选，helper 脚本
├── references/           # 可选，按需加载的详细文档
└── assets/               # 可选，模板等资源
```

详见 pi 官方文档。
