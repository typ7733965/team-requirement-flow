# team-requirement-flow

> 需求驱动开发流程的 Claude Code 插件：**设计文档先行 → 分步实施 → 前端接入文档 → 文档即上下文恢复**。
>
> 把团队工作经验沉淀为 `docs/<需求名>/` 的结构化产物，让上下文跨人、跨时间、跨工具持久化。
>
> 仓库地址：<https://github.com/typ7733965/team-requirement-flow>

---

## 一、为什么需要这个插件

团队接到新需求时常见痛点：边做边想没记录、跨人接手靠口口相传、跨会话重述背景、前后端联调约定散落在脑子里。

本插件用「**docs/<需求名>/**」目录承载需求的 single source of truth：

```
docs/
└── <需求名>/
    ├── <需求名>设计方案.md     ← design-doc 产出
    └── 前端接入<功能名>步骤.md  ← frontend-handoff 产出
```

设计方案 + 前端文档 + 实施步骤拆分表 + git commit message = **完整可恢复的上下文**。

---

## 二、四个 Skill + 一个 Subagent

| 名称 | 类型 | 触发场景 |
|------|------|---------|
| `design-doc` | Skill | 接到新需求 → 写设计方案 |
| `implement-from-doc` | Skill | 设计敲定 → 按步骤推进开发 |
| `frontend-handoff` | Skill | 后端实施完 → 给前端写接入文档 |
| `resume-from-doc` | Skill | 新会话 / 接手他人需求 → 30 秒恢复上下文 |
| `doc-reviewer` | Subagent | 文档写完 → 对照源码审稿 |

### 典型工作流

```
新需求 ─► /design-doc ─► （review 调整） ─► /implement-from-doc ─► /frontend-handoff
                                                                         │
跨会话续做 ◄─ /resume-from-doc ◄────────────────── 提交后 doc-reviewer 审稿
```

---

## 三、安装

### 方式 1：从 GitHub 安装（推荐）

```
/plugin marketplace add typ7733965/team-requirement-flow
/plugin install team-requirement-flow@team-requirement-flow
```

第一条只是登记 marketplace，第二条才真正安装。也可输入 `/plugin` 进图形界面在 Browse 里点 Install。

### 方式 2：本地开发模式

```bash
claude --plugin-dir /absolute/path/to/team-requirement-flow/plugins/team-requirement-flow
```

修改后在会话内 `/reload-plugins` 热重载。

### 更新

```
/plugin marketplace update team-requirement-flow
/plugin update team-requirement-flow@team-requirement-flow
```

更新后建议开新会话以确保最新版本生效。

### 卸载

```
/plugin uninstall team-requirement-flow@team-requirement-flow
/plugin marketplace remove team-requirement-flow
```

### 验证

输入 `/`，命令面板应能看到带 `team-requirement-flow:` 前缀的 4 条命令。或试一句：

```
我要做一个新需求叫 测试需求
```

Claude 应自动调用 `design-doc`。

---

## 四、使用

### 命令式调用

```
/team-requirement-flow:design-doc <需求名> [--lang zh|en]
/team-requirement-flow:implement-from-doc <需求名> [step=N]
/team-requirement-flow:frontend-handoff <需求名> [--lang zh|en]
/team-requirement-flow:resume-from-doc <需求名>
```

### 自然语言触发

| 用户说 | Claude 调用 |
|-------|------------|
| "我要做一个新需求叫 xxx" / "先写个设计方案" | `design-doc` |
| "开始实施" / "按方案来" / "做下一步" | `implement-from-doc` |
| "给前端写接入步骤" | `frontend-handoff` |
| "继续上次做的 xxx" / "恢复 xxx 需求" | `resume-from-doc` |
| "审一下文档" / "review 一下设计方案" | `doc-reviewer` |

---

## 五、Skill 详解

### 5.1 design-doc

输入需求名，输出 `docs/<需求名>/<需求名>设计方案.md`，9 节：需求与目标 / 整体架构 / 数据模型 / 接口设计 / 错误码（字面量）/ 关键决策（候选 → 选择 → 理由）/ 实施步骤拆分 / 风险回滚 / 附录。

### 5.2 implement-from-doc

读设计方案 + git log 识别已完成步骤 → 输出进度判定 → 推进当前步骤（编码 → 编译验证 → 用户确认 → 下一步）。**不会主动 commit**，commit 必须由用户明确指示。

硬规则：先 Read 再 Edit、不跳步骤、错误码用设计方案字面量、跨仓库改动单独编译验证。

### 5.3 frontend-handoff

输出 `docs/<需求名>/前端接入<功能名>步骤.md`，10 节：全景流程图 / 阶段 A·B·C / 接口速查表 / 错误码与前端处理 / 硬编码约定（★ 必读）/ 联调 FAQ / Checklist 等。

质量保障：错误码字符串、success code、URL 必须 Grep 源码确认，**不抄设计方案**。

### 5.4 resume-from-doc

输出「需求恢复报告」：一句话摘要、关键决策回顾、涉及仓库、实施进度对账表（步骤 → 状态 → commit 证据）、进行中事项、下一步建议。**跨会话不再依赖记忆**。

### 5.5 doc-reviewer (Subagent)

「请用 doc-reviewer 审一下 docs/xxx」即触发。审稿维度：错误码字面量与源码 Grep 一致性、success code 数值、接口路径核对、章节完整性、跨仓库一致性、硬编码约定是否独立成节、TODO 占位是否闭合。

输出结构化审稿报告（阻塞 / 建议 / 优化），**不直接 Edit 文档**，修订由作者决定。

---

## 六、目录结构

```
team-requirement-flow/
├── .claude-plugin/marketplace.json
├── plugins/team-requirement-flow/
│   ├── .claude-plugin/plugin.json
│   ├── skills/
│   │   ├── design-doc/        SKILL.md + examples/
│   │   ├── implement-from-doc/SKILL.md
│   │   ├── frontend-handoff/  SKILL.md + examples/
│   │   └── resume-from-doc/   SKILL.md
│   └── agents/doc-reviewer.md
└── README.md
```

---

## 七、设计原则

- **文档先行但不重文档**：缺信息标 `<!-- TODO -->` 占位，别为流程而写
- **决策必有理由**：候选方案 + 选择 + 理由，不是「选了就选了」
- **只信源码不信文档**：错误码字符串必须 Grep 源码字面量
- **一步一验证**：每步独立 commit / 独立编译，便于 review 与回滚
- **跨仓库单独验证**：每个仓库改动独立 build，不积压
- **硬编码约定独立成节**：前端联调最常见翻车点，必须 ★ 标注

---

## 八、贡献

### 8.1 修改 Skill 描述

Skill 是否被自动调用，**完全取决于 description 字段是否覆盖用户语句**。修改 `SKILL.md` frontmatter `description` 时建议：

- 列出 3-5 个典型触发语句
- 描述 Skill 的输入与输出
- 描述何时**不应该**调用（避免误触发）

### 8.2 新增 Skill

1. 在 `plugins/team-requirement-flow/skills/<name>/` 新建 `SKILL.md`，frontmatter 含 `description` / `argument-hint` / `allowed-tools`
2. 提交 PR 时附带 1-2 个 examples

### 8.3 修改 Subagent

`agents/doc-reviewer.md` 的 frontmatter：`name` / `description` 决定是否被调用，`tools` 是工具白名单。

---

## 九、版本管理

遵循 SemVer：

- **MAJOR**：Skill 接口（参数 / 输出文档结构）破坏性变更
- **MINOR**：新增 Skill / Subagent
- **PATCH**：修复内部逻辑、更新 examples、文档优化

发版步骤：更新 `plugin.json` 的 `version` → 打 git tag → 团队 `/plugin marketplace update`。

---

## 十、FAQ

**Q：跟自带的 Slash Command 有什么区别？**
Slash Command 是单条指令；Skill 是可被自然语言唤起的能力包，含完整工作流 + 工具白名单 + 示例。

**Q：可以脱离插件单独用 docs/<需求名>/ 这种约定吗？**
可以。本插件只是把约定**自动化**——你完全可以手写设计方案，但写多了重复劳动。

**Q：Skill 自动调用太激进怎么办？**
用斜杠命令式调用即可：`/team-requirement-flow:design-doc xxx`，或在描述中加更精确的触发条件。

**Q：跨项目能用吗？**
可以。Skill 中的"项目结构探测"逻辑是通用的（先 Read README/CLAUDE.md/go.mod 等），不绑定特定语言。
