# team-requirement-flow

> 需求驱动开发流程的 Claude Code 插件：**设计文档先行 → 分步实施 → 前端接入文档 → 文档即上下文恢复**。
>
> 把团队工作经验沉淀为 `docs/<需求名>/` 的结构化产物，让上下文跨人、跨时间、跨工具持久化。

---

## 一、为什么需要这个插件

团队接到新需求时，常见痛点：

- ❌ 直接开干 → 边做边想，决策无据可查
- ❌ 跨人接手 → 新人靠口口相传补背景
- ❌ 跨会话续做 → 每次都要从零给 Claude 讲一遍背景
- ❌ 后端 ↔ 前端联调 → 接口/错误码/硬编码约定散落在脑子里，反复沟通

本插件用「**docs/<需求名>/**」目录承载需求的 single source of truth：

```
docs/
└── 用户登录组件/
    ├── 用户端登录认证组件设计方案_Claude.md   ← design-doc Skill 产出
    └── 前端接入登录组件步骤.md                ← frontend-handoff Skill 产出
```

> 设计方案 + 前端文档 + 实施步骤拆分表 + git commit message = **完整可恢复的上下文**。

---

## 二、四个 Skill + 一个 Subagent

| 名称 | 类型 | 何时调用 |
|------|------|---------|
| `design-doc` | Skill | 接到新需求 → 写设计方案 |
| `implement-from-doc` | Skill | 设计方案敲定 → 按步骤推进开发 |
| `frontend-handoff` | Skill | 后端实施完 → 给前端写接入文档 |
| `resume-from-doc` | Skill | 新会话 / 接手他人需求 → 30 秒恢复上下文 |
| `doc-reviewer` | Subagent | 文档写完 → 对照源码审稿 |

### 典型工作流

```
新需求
   │
   ▼
/design-doc 用户登录组件
   │  ⇒ docs/用户登录组件/设计方案.md
   ▼
（用户 review，调整设计方案）
   │
   ▼
/implement-from-doc 用户登录组件
   │  ⇒ 逐步推进：编码 → 编译验证 → 用户确认 → commit → 下一步
   ▼
/frontend-handoff 用户登录组件
   │  ⇒ docs/用户登录组件/前端接入步骤.md
   ▼
（doc-reviewer 审稿 → 修订）
   │
   ▼
前端联调
```

下次有新需求 / 新会话 / 新人接手时：

```
/resume-from-doc 用户登录组件
   ⇒ 输出"已完成步骤 + 进行中事项 + 下一步建议"
```

---

## 三、安装

### 方式 1：从本 Marketplace 安装（推荐）

```
/plugin marketplace add <your-team-git-url>
/plugin install team-requirement-flow@team-requirement-flow
```

### 方式 2：本地开发模式

```bash
# 在 ~/.claude/settings.json 中添加：
{
  "extraKnownMarketplaces": {
    "team-requirement-flow": {
      "source": {
        "source": "local",
        "path": "/absolute/path/to/team-requirement-flow"
      }
    }
  }
}
```

然后：

```
/plugin install team-requirement-flow@team-requirement-flow
```

### 验证安装

进入任何项目，输入 `/`：

- 应能看到 `/team-requirement-flow:design-doc` 等 4 个命令
- 输入「我要做一个新需求」，Claude 应能自动联想到 `design-doc` Skill

---

## 四、使用

### 命令式调用

```
/team-requirement-flow:design-doc 用户登录组件
/team-requirement-flow:design-doc 用户登录组件 --lang en
/team-requirement-flow:implement-from-doc 用户登录组件
/team-requirement-flow:implement-from-doc 用户登录组件 step=3
/team-requirement-flow:frontend-handoff 用户登录组件
/team-requirement-flow:resume-from-doc 用户登录组件
```

### 自然语言触发

各 Skill 配置了 description 字段，Claude 会根据用户语句自动判断调用哪个 Skill：

| 用户说 | Claude 调用 |
|-------|------------|
| "我要做一个新需求叫 xxx" | `design-doc` |
| "先写个设计方案" | `design-doc` |
| "开始实施" / "按方案来" | `implement-from-doc` |
| "做下一步" | `implement-from-doc` |
| "给前端写接入步骤" | `frontend-handoff` |
| "继续上次做的 xxx" / "恢复 xxx 需求" | `resume-from-doc` |
| "审一下文档" / "review 一下设计方案" | `doc-reviewer` subagent |

---

## 五、Skill 详解

### 5.1 design-doc

**输入**：需求名（必填）+ 语言（可选 `--lang zh|en`）

**输出**：`docs/<需求名>/<需求名>设计方案_Claude.md`，包含 9 节：
1. 需求与目标（含范围 / 不在范围）
2. 整体架构（ASCII 时序图 + 组件表 + 调用链路）
3. 数据模型（DB / Redis / MQ）
4. 接口设计（RPC / HTTP / SDK）
5. 错误码（**字符串字面量**）
6. 关键设计决策（候选 → 选择 → 理由）
7. 实施步骤拆分（每步可独立 commit）
8. 风险与回滚
9. 附录（名词表 + 参考资料）

**示例**：见 `plugins/team-requirement-flow/skills/design-doc/examples/`

### 5.2 implement-from-doc

**输入**：需求名 + 可选 `step=N`

**行为**：
1. Read 设计方案 + git log 识别已完成步骤
2. 输出"进度判定"
3. 推进当前步骤：编码 → 编译验证 → 用户确认 → 下一步
4. **不会主动 commit**：commit 必须由用户明确指示

**硬规则**：
- 不读不改（先 Read 再 Edit）
- 不跳步骤，不批量做多步
- 错误码必须用设计方案中定义的字面量
- 跨仓库改动每个仓库单独编译验证

### 5.3 frontend-handoff

**输入**：需求名 + 可选 `--lang zh|en`

**输出**：`docs/<需求名>/前端接入<功能名>步骤.md`，10 节：
1. 总体顺序与全景流程图
2-4. 阶段 A/B/C 分解
5. 接口速查表
6. 错误码对照表（**前端处理建议**）
7. 硬编码约定（★ 必读）
8. 联调 FAQ
9. Checklist
10. 答疑联系

**质量保障**：错误码字符串、success code、URL 必须 Grep 源码确认，**不抄设计方案**。

**示例**：见 `plugins/team-requirement-flow/skills/frontend-handoff/examples/`

### 5.4 resume-from-doc

**输入**：需求名（不传则 `ls docs/` 让用户选）

**输出**：「需求恢复报告」，含：
- 需求一句话摘要
- 关键设计决策回顾
- 涉及仓库与模块
- 实施进度（对账表：步骤 → 状态 → commit hash 证据）
- 进行中事项细节
- 下一步建议
- 需要用户确认的事项

**核心价值**：跨会话 / 跨人 / 跨时间，**不再依赖会话历史**。

### 5.5 doc-reviewer (Subagent)

**调用方式**：让 Claude 「请用 doc-reviewer 审一下 docs/xxx」

**审稿维度**：
- 错误码字符串 vs 源码 Grep 一致性
- success code 数值对照（项目实际可能是 0 / 200 / 20000）
- 接口路径 / RPC 方法签名核对
- 文档完整性（章节缺失检测）
- 跨仓库一致性（git log 校验）
- 硬编码约定章节（前端接入翻车最常见点）
- TODO 占位闭合检查

**输出**：结构化审稿报告（阻塞性问题 / 建议性问题 / 优化建议 / 修订建议）

**约束**：只输出报告，**不直接 Edit 文档**。修订由作者决定。

---

## 六、目录结构

```
team-requirement-flow/
├── .claude-plugin/
│   └── marketplace.json                ← Marketplace 元数据
├── plugins/
│   └── team-requirement-flow/
│       ├── .claude-plugin/
│       │   └── plugin.json             ← Plugin 元数据
│       ├── skills/
│       │   ├── design-doc/
│       │   │   ├── SKILL.md
│       │   │   └── examples/
│       │   ├── implement-from-doc/
│       │   │   └── SKILL.md
│       │   ├── frontend-handoff/
│       │   │   ├── SKILL.md
│       │   │   └── examples/
│       │   └── resume-from-doc/
│       │       └── SKILL.md
│       └── agents/
│           └── doc-reviewer.md         ← Subagent
└── README.md                           ← 本文件
```

---

## 七、设计原则

本插件秉持以下原则，团队成员贡献时请遵守：

### 7.1 文档先行，但不重文档

- 设计方案不是为了"应付流程"而写，而是**对齐共识 + 沉淀决策 + 后续上下文**
- 不要为了写文档而写文档：缺信息就标 `<!-- TODO: 待 xxx 同学补充 -->` 占位

### 7.2 决策必有理由

设计方案的「关键设计决策」章节是灵魂：
- ❌ 只写"我们选了 Redis 单实例"
- ✅ 写"候选 Redis 单实例 vs Redis Cluster；选 单实例；理由：QPS 预估 5k 内单实例足够，运维成本低"

### 7.3 只信源码不信文档

特别是写前端接入文档时：
- ❌ 错误码字符串从设计方案抄
- ✅ 错误码字符串必须 Grep 源码字面量
- 设计方案可能与实现漂移，源码是唯一真相

### 7.4 一步一验证

- ❌ 一次写完所有改动，最后统一编译
- ✅ 每个步骤独立 commit，独立编译，便于 review 与回滚

### 7.5 跨仓库改动单独验证

跨仓库需求最容易出现"改 A 仓库时 B 仓库已经过期"的问题：
- 每个仓库改动后单独 build
- 不要积压跨仓库的改动一起验证

### 7.6 硬编码约定必须独立成节

最常见的前端联调翻车点：
- `country_code = +86` 这类约定散落在某段
- 应在前端接入文档单独成节并标 ★ 必读

---

## 八、贡献

### 8.1 修改 Skill 描述

Skill 是否被自动调用，**完全取决于 description 字段是否覆盖用户语句**。

修改 `SKILL.md` 的 frontmatter `description` 时，建议：
- 列出 3-5 个典型触发语句（如"我要做一个新需求"/"先写个设计方案"）
- 描述 Skill 的输入与输出
- 描述何时**不应该**调用（避免误触发）

### 8.2 新增 Skill

1. 在 `plugins/team-requirement-flow/skills/` 下新建目录
2. 新增 `SKILL.md`，遵循其他 Skill 的 frontmatter 结构
3. 在 `plugins/team-requirement-flow/.claude-plugin/plugin.json` 的 `skills` 数组中注册
4. 提交 PR 时附带 1-2 个 examples

### 8.3 修改 Subagent

`agents/doc-reviewer.md` 的 frontmatter 字段：
- `name` / `description`：Claude 据此判断是否调用
- `tools`：subagent 可用工具列表
- `model`：建议保持 `inherit` 即用主会话模型

---

## 九、版本管理

遵循 SemVer：

- **MAJOR**：Skill 接口（参数 / 输出文档结构）破坏性变更
- **MINOR**：新增 Skill / Subagent
- **PATCH**：修复 Skill 内部逻辑、更新 examples、文档优化

每次发版：
1. 更新 `plugins/team-requirement-flow/.claude-plugin/plugin.json` 的 `version`
2. 在 `CHANGELOG.md`（如有）记录变更
3. 打 git tag（如 `v1.0.0`）
4. 团队 Marketplace 拉取：`/plugin marketplace update team-requirement-flow`

---

## 十、FAQ

### Q1：跟 Claude Code 自带的 Slash Command 有什么区别？

- Slash Command 是「单条指令」
- Skill 是「可被自然语言唤起的能力包」，含完整工作流 + 工具白名单 + 示例

### Q2：可以脱离本插件单独用 docs/<需求名>/ 这种约定吗？

可以。本插件只是把约定**自动化**——你完全可以手写设计方案，但写多了重复劳动。

### Q3：Skill 自动调用太激进怎么办？

- 用斜杠命令式调用即可：`/team-requirement-flow:design-doc xxx`
- 或在 Skill 描述中加更精确的触发条件

### Q4：跨项目能用吗？

可以。Skill 中的"项目结构探测"逻辑是通用的（先 Read README/CLAUDE.md/go.mod 等），不绑定特定语言。

---

## 十一、致谢

本插件流程提炼自 game-operation 项目「用户端登录认证组件」需求开发过程，在该需求中验证了：
- 设计方案 → 8 步实施 → 前端接入文档 的完整链路
- 跨仓库（proto / 业务 / 网关）协同改动
- 错误码源码与文档对账机制

详见 `plugins/team-requirement-flow/skills/*/examples/` 中的真实产出。

---

## 十二、License

MIT
