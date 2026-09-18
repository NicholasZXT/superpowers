# Superpowers 源码研读笔记

> 对象仓库：`obra/superpowers`（SDD = Spec-Driven Development 工具，v6.3.0）
> 笔记来源：对 `skills/`、`hooks/`、`scripts/`、各 harness 专用目录、`docs/` 与 `RELEASE-NOTES.md` 的通读
> 关键参考文档：`docs/porting-to-a-new-harness.md`（理解整个架构的钥匙）、`CLAUDE.md`（贡献者/AI 规则）、`.github/PULL_REQUEST_TEMPLATE.md`

---

## 0. TL;DR 核心结论

| 问题                                 | 结论                                                                                                               |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| Superpowers 本质是什么               | 一套**内容（skills）+ 投递层（bootstrap 注入）** 的开发方法论，不是命令集                                    |
| 内容载体                             | `skills/` 是唯一真源，跨 harness **逐字共享**                                                              |
| 生效条件                             | **两条腿**：① bootstrap 在每次会话开始时注入 ② harness 能发现/invoke skill。缺任一条 → skill 文件是"死重" |
| 让 skill 活起来的是什么              | `using-superpowers` 的 bootstrap 注入，而非 skill 本身                                                           |
| 是否需要 slash command               | **否**，v5.1.0 起已彻底删除；靠 skill description 语义匹配自动触发                                           |
| 能否直接拷贝`skills/` 到目标目录用 | 可作为实验起点，但**达不到可用标准**（丢 bootstrap、丢工具映射、丢 `superpowers:` 命名空间）               |
| 规范文件生产者                       | 只有`brainstorming`（spec）和 `writing-plans`（plan）产出"进 git 的文档"                                       |

---

## 1. 仓库架构

### 1.1 三层结构（`docs/porting-to-a-new-harness.md` Part 1）

```mermaid
flowchart TB
    subgraph Core["① harness 无关的内容（唯一真源）"]
        S["skills/<skill>/SKILL.md<br/>只描述动作，不写具体工具名"]
        R["② skills/using-superpowers/references/<harness>-tools.md<br/>动作 → 该 harness 真实工具名"]
    end
    subgraph Per["③ 每 harness 的投递层（薄适配）"]
        H["hooks/（shell hook 型）"]
        E[".opencode/ .pi/ .hermes-plugin/<br/>（进程内插件型）"]
        C["GEMINI.md + gemini-extension.json<br/>（指令文件型）"]
    end
    P["各 harness 的 manifest<br/>.claude-plugin/ .codex-plugin/ .cursor-plugin/ ..."]
    S --> P
    R --> P
    P --> H
    P --> E
    P --> C
```

三层职责对应关系：

| #  | 组件                                 | 位置                                                       | 职责                                                                                                      |
| -- | ------------------------------------ | ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| ① | **Skills**（harness 无关）     | `skills/`                                                | 唯一真源，跨 harness**逐字共享**；只描述"动作"，从不点名工具                                        |
| ② | **Tool mapping**（每 harness） | `skills/using-superpowers/references/<harness>-tools.md` | 把动作词汇翻译成该 harness 的真实工具名；也可**内联进 bootstrap 注入器**                            |
| ③ | **Bootstrap**（每 harness）    | `hooks/`、`.opencode/`、`.pi/` 等投递层              | 会话开始时把`using-superpowers/SKILL.md` 全文包进 `<EXTREMELY_IMPORTANT>`，连同 ② 一起送进模型上下文 |

> **"The bootstrap is the entire integration."**
> 没有它：skill 文件存在于磁盘，但永远不会在正确时机被调用。

### 1.2 两条不可违反的规则

1. **Skills 只写动作，不写工具名。** 移植时**禁止**改 `skills/*/SKILL.md` 正文去适配工具名——修复属于"工具映射"。
   （唯一例外：`SKILL.md` 的 *Platform Adaptation* 指针清单可以加一行。）
2. **一切通过 harness 自己的安装机制投递，绝不改用户文件。** 不允许写 `~/.gemini/GEMINI.md`、`settings.json`、`~/.bashrc` 等。若安装机制确实无法承载 bootstrap，应**上报为限制**，而不是手改用户配置。

---

### 1.3 `hooks/` —— shell-hook 型 harness 的 bootstrap 注入器

| 文件                        | 作用                                                                                                                                                                                                                                                                              |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `hooks/hooks.json`        | **Claude Code** hook 配置。`SessionStart`，matcher = `startup\|clear\|compact`（**刻意不含 `resume`**），命令 `"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd" session-start`，`"shell": "bash"`，`async: false`（必须同步，否则首轮可能在注入前就开始 — #571） |
| `hooks/hooks-cursor.json` | **Cursor** 变体，schema 完全不同：`{"version":1,"hooks":{"sessionStart":[{"command":"./hooks/run-hook.cmd session-start"}]}}` —— 小写 `sessionStart`、相对路径、无 `matcher`/`type`/`async`                                                                     |
| `hooks/run-hook.cmd`      | **polyglot 分发器**（同一文件既是合法 batch 又是合法 bash）。Windows 上依次尝试 `C:\Program Files\Git\bin\bash.exe` → `Program Files (x86)` → PATH 上的 `bash`；找不到 bash 则 `exit /b 0` 静默退出（harness 仍可用，仅无注入）                                   |
| `hooks/session-start`     | **真正的注入逻辑**。**无扩展名是刻意的**：Claude Code 在 Windows 会给含 `.sh` 的命令自动前置 `bash`，会造成重复调用                                                                                                                                               |

#### `session-start` 的三分支输出（最易踩的坑）

| 判断条件                                             | 输出形状                                                                                     | 平台                   |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------- |
| `CURSOR_PLUGIN_ROOT` 已设置                        | `{ "additional_context": "…" }`                                                           | Cursor                 |
| `CLAUDE_PLUGIN_ROOT` 设置且 `COPILOT_CLI` 未设置 | `{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "…" } }` | Claude Code            |
| 其他                                                 | `{ "additionalContext": "…" }`                                                            | Copilot CLI / SDK 标准 |

> **为什么必须严格只输出一个字段**：Claude Code 会同时读 `additional_context` 和 `hookSpecificOutput` 且**不去重**，同时输出会**双重注入**。

其他实现细节：

- 用 `${s//old/new}` 参数替换做 JSON 转义（曾用逐字符循环，在 Git Bash 上耗时 60s+）
- 用 `printf` 而非 heredoc（规避 bash 5.3+ heredoc hang，#571）
- 每个 `printf` 通过 `cat` 吸收 broken pipe（Windows 曾每会话报写错误，#1612）

### 1.4 `scripts/` —— 仓库维护脚本（与运行时无关）

| 脚本                                         | 作用                                                                                                                                                                                                                         |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bump-version.sh` + `.version-bump.json` | 按声明表同步所有 manifest 的 version（支持`plugins.0.version` 嵌套路径），并做仓库级漂移审计                                                                                                                               |
| `lint-shell.sh`                            | ShellCheck +`shfmt`，默认只查改动过的 shell 文件                                                                                                                                                                           |
| `sync-to-codex-plugin.sh`                  | 把受版本控制的插件文件 rsync 到外部 fork（`prime-radiant-inc/openai-codex-plugins`）的 `plugins/superpowers/`，提交并开 PR。带**锚定**的 EXCLUDES 列表（防止一个 harness 的 dotdir 泄漏到另一个 harness 的分发物） |
| `package-codex-plugin.sh`                  | 生成 Codex portal 上传用的确定性归档（归一化时间戳、保留执行权限、校验每个 skill 带 OpenAI 元数据、拒绝脏工作区）                                                                                                            |

### 1.5 `skills/` —— 内容本体

- **14 个 skill 目录**，**扁平命名空间**
- frontmatter **只有两个必需字段**：`name` + `description`（上限 1024 字符；其余字段见 agentskills.io 规范）
- `description` 规则（关键）：**只写触发条件，绝不概括流程**（见 §3.2）
- 支撑文件通过 SKILL.md 的**相对路径**引用，例：
  - `test-driven-development/SKILL.md` → `[writing-good-tests.md](writing-good-tests.md)`
  - `requesting-code-review/SKILL.md` → `[code-reviewer.md](code-reviewer.md)`
  - `subagent-driven-development/SKILL.md` → `./implementer-prompt.md`、`./task-reviewer-prompt.md`、`./re-review-prompt.md`、`../requesting-code-review/code-reviewer.md`
- **`skills/using-superpowers/`** 是入口，同时存放 `references/<harness>-tools.md`（现有 5 份：`codex`、`pi`、`gemini`、`antigravity`、`hermes`）
- 隐式仓库根依赖（拷贝时需注意）：
  - `skills/brainstorming/scripts/server.cjs` 的 `readSuperpowersVersion()` 向上三级读 `package.json`，读不到再退 `.codex-plugin/plugin.json`
  - `skills/brainstorming/SKILL.md:250` 用的是**仓库根相对路径** `skills/brainstorming/visual-companion.md`（不是 `./`）
  - `skills/subagent-driven-development/scripts/*` 依赖 `git rev-parse --show-toplevel`

### 1.6 各 harness 专用目录

先看**集成形态**（Part 4 把形态分为 A/B/C 三类）：

| 目录 / 文件                                                     | Harness      | 形态                  | 关键内容                                                                                                                                                                                                                                                         |
| --------------------------------------------------------------- | ------------ | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.claude-plugin/plugin.json`                                  | Claude Code  | **A**           | **刻意不写 `skills`/`hooks` 字段** —— Claude 按约定自动发现 `skills/` 和 `hooks/hooks.json`。同目录 `marketplace.json` 为开发用 marketplace（`source: "./"`）                                                                                |
| `.cursor-plugin/plugin.json`                                  | Cursor       | **A**           | 显式`"skills": "./skills/"` + `"hooks": "./hooks/hooks-cursor.json"` —— **Shape A 的参考模板**                                                                                                                                                       |
| `.codex-plugin/plugin.json`                                   | Codex        | 无 hook               | 有`"skills": "./skills/"`；`"hooks": {}` 是**必须的空对象**：缺 `hooks` 字段时 Codex 会 fallback 去自动发现 `hooks/hooks.json`（即 Claude 的 SessionStart hook）。`[]`、缺字段、空内联列表都会 collapse 回 fallback，必须是 `{}`               |
| `.agents/plugins/marketplace.json`                            | Codex        | 分发                  | Codex marketplace 要求在 marketplace 根有此文件，否则能命名 marketplace 却找不到可安装条目                                                                                                                                                                       |
| `.opencode/plugins/superpowers.js` + `.opencode/INSTALL.md` | OpenCode     | **B**           | `config` hook 把 skills 目录 push 进 `config.skills.paths`（无需 symlink）；`experimental.chat.messages.transform` 把 bootstrap **作为 user message unshift 到首条用户消息**，用 `EXTREMELY_IMPORTANT` 标记去重，模块级缓存                        |
| `.pi/extensions/superpowers.ts`                               | pi           | **B**           | `resources_discover` → `skillPaths`；`session_start`/`session_compact` 置 `injectBootstrap=true`，`agent_end` 清掉；`context` 事件插入 user message 且**跳过开头的 compactionSummary**；模块级缓存。**无原生 skill 工具时的最佳参考** |
| `gemini-extension.json` + 根 `GEMINI.md`                    | Gemini CLI   | **C**           | manifest 用`"contextFileName": "GEMINI.md"`（**不写 skills 字段**，Gemini 自动发现随扩展打包的 `skills/`）；`GEMINI.md` 内容就两行 `@`-include                                                                                                     |
| `.kimi-plugin/plugin.json`                                    | Kimi Code    | 声明式                | `"skills": "./skills/"` + `"sessionStart": {"skill": "using-superpowers"}` + 内联 `skillInstructions`（工具映射）。无需 hook、无需拷贝、无 symlink                                                                                                         |
| `.devin-plugin/plugin.json`                                   | Devin CLI    | 无 hook               | 只有元数据；Devin 自动发现**同目录的** `skills/`，并在 system prompt 列出每个 skill 的 name+description，用原生 `skill` 工具调用                                                                                                                       |
| `.hermes-plugin/`（`plugin.yaml` + `__init__.py`）        | Hermes Agent | **B**（Python） | `provides_hooks: [pre_llm_call]`；`register(ctx)` 逐个 `ctx.register_skill(name, Path(skill_md))` —— **必须传 `pathlib.Path`**，传 `str` 会让 Hermes 静默禁用整个插件；`is_first_turn` 时 `return {"context": bootstrap}`                  |
| 根`package.json`                                              | pi           | B 的元数据            | `main` 指向 OpenCode 插件；`pi.extensions` / `pi.skills` 声明 pi 侧资源；`pi-package` keyword                                                                                                                                                            |
| `.version-bump.json`                                          | 全仓         | 发布                  | 列出所有带版本的 manifest 及其字段路径。**新 harness 若不登记就会发旧版本**                                                                                                                                                                                |

**两种注入策略的差异（Shape B 的坑）**：

- OpenCode 是**每 agent step** 都触发 transform → 靠**每次调用的去重 guard**
- pi 是**每 turn** 触发 → 靠**lifecycle flag + `agent_end` 重置**
- 把一方的去重策略套到另一方会破坏注入

---

## 2. 集成形态与"能否直接拷贝"

### 2.1 硬需求（一票否决）

> **automatic session-start injection**：能在**每次会话开始**把文本注入模型上下文，且**不需要用户每会话 opt-in**。

可接受形式：hook/event 系统读 stdout；进程内插件/扩展的生命周期回调改消息数组；扩展**自带并声明**的 instructions 文件。

不可接受：需要用户粘贴 prompt / 跑命令 / 开启某模式。→ 验收测试必然失败。

### 2.2 三种形态

| 形态                                       | 特征                                                | 参考实现                                                                              |
| ------------------------------------------ | --------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **A — Shell-hook**                  | hook 系统跑 shell 命令并读 stdout JSON              | Cursor（`hooks/session-start` + `hooks/hooks-cursor.json` + `.cursor-plugin/`） |
| **B — In-process plugin/extension** | JS/TS 模块暴露生命周期回调，代码里改消息数组        | OpenCode（`.opencode/`）；**无原生 skill 工具则抄 pi**                        |
| **C — Instructions-file**           | 只有始终加载的上下文文件（扩展自带、manifest 声明） | Gemini（`gemini-extension.json` + `GEMINI.md` + `references/gemini-tools.md`）  |

补充要点：

- **形态可组合**：skill 发现机制与 bootstrap 机制可以不同形态，但**两者都必须走安装机制**
- Codex/Devin 属"**surfaced skill index**"软路径：靠 harness 在会话开始暴露每个 skill 的 name+description 触发，**只 bootstrap 触发、不 bootstrap 工具映射**，且无结构保证
- "无 skill 系统"的 harness 需自备发现路径：生成 skill 索引（会过期）或指示运行时列举 `skills/*/SKILL.md`（推荐，永不过期）

### 2.3 能否直接把 `skills/` 拷到目标 Agent 的 skills 目录？

**可以当实验起点，但达不到可用标准**，会坏在四处：

1. **没 bootstrap ⇒ 不自动触发（最致命）。** `using-superpowers/SKILL.md` **不枚举**其他 skill，模型不知道有哪些、何时用
2. **工具映射缺失。** skill 只写动作；映射表在 `references/<harness>-tools.md`，而 `SKILL.md` 的 Platform Adaptation 只登记 Codex/Pi/Antigravity/Hermes
3. **命名空间与路径。** skill 间大量用 `superpowers:test-driven-development` 这类前缀（来自插件命名空间），拷到个人 skills 目录后失效；且存在仓库根相对路径
4. **配套脚本静默降级。** `server.cjs` 版本读到 `unknown`；`sdd-workspace` 等依赖 git 仓库根；`.version-bump.json` 版本一致性失效

**欲实验请按序补齐**：整棵树保结构复制 → 写目标 harness 的映射表 → **保证每次会话自动送达 bootstrap**（分水岭）→ 用验收提示验证。

**验收测试（PR 强制）**：干净会话发送

> Let's make a react todo list

`brainstorming` 必须在**任何代码之前**自动触发，并附完整 transcript。PR 模板明确列出的"不算真集成"：手工拷贝 skill 文件 / 用 `npx skills` 之类运行时 shim / 任何需要用户每会话 opt-in 的做法。

---

## 3. `using-superpowers` 描述的原则

该 skill 极短（约 80 行），作用是**元规则**：规定"如何决定做什么"。

### 3.1 唯一核心规则

> **Invoke relevant or requested skills BEFORE any response or action** — including clarifying questions, exploring the codebase, or checking files.

"Including" 后的三项是刻意的：模型最常用的三个逃避借口被逐个点名封死。

### 3.2 强制强度：1% 规则

```
If you think there is even a 1% chance a skill might apply to what you are doing,
you ABSOLUTELY MUST invoke the skill.
IF A SKILL APPLIES TO YOUR TASK, YOU DO NOT HAVE A CHOICE. YOU MUST USE IT.
```

### 3.3 优先级裁定

- **process skills 优先于 implementation skills**（前者定方法，后者才执行）
- `"Let's build X"` → 先 `brainstorming`，再 implementation skills
- `"Fix this bug"` → 先 `systematic-debugging`，再 domain skills
- **Plan mode 拦截**：进入 plan mode 前若未 brainstorming，必须先 brainstorming

### 3.4 Red Flags 表（反合理化装置）

不列"要做什么"，而是**预先列出模型会想到的作弊念头并逐条反驳**：

| 模型的想法                | 反驳                                               |
| ------------------------- | -------------------------------------------------- |
| "这只是个简单问题"        | Questions are tasks. Check for skills.             |
| "我需要先多点上下文"      | Skill check comes BEFORE clarifying questions.     |
| "让我先探索代码库"        | Skills tell you HOW to explore. Check first.       |
| "我可以快速查 git/文件"   | Files lack conversation context. Check for skills. |
| "让我先收集信息"          | Skills tell you HOW to gather information.         |
| "这不需要正式 skill"      | If a skill exists, use it.                         |
| "我记得这个 skill"        | Skills evolve. Read current version.               |
| "这不算一个 task"         | Action = task. Check for skills.                   |
| "这个 skill 有点小题大做" | Simple things become complex. Use it.              |
| "我先只做这一件事"        | Check BEFORE doing anything.                       |
| "这感觉很有成效"          | Undisciplined action wastes time.                  |
| "我知道那是什么意思"      | Knowing the concept ≠ using the skill. Invoke it. |

> **这套 "rationalization table" 是 Superpowers 的通用写作范式**：`brainstorming`、`systematic-debugging`、`using-git-worktrees`、`verification-before-completion`、`subagent-driven-development`、`receiving-code-review` 里都有同构的表。`CLAUDE.md` 把这类内容定义为 "carefully-tuned behavior-shaping code"，改动需要 eval 证据 —— **它们不是文档，是行为补丁**。

### 3.5 两条兜底条款

- **`<SUBAGENT-STOP>`**：若自己是被派来干具体活的 subagent，忽略此 skill（避免 subagent 又触发一遍全流程）
- **User Instructions 优先级最高**：`CLAUDE.md`/`AGENTS.md`/`GEMINI.md` 等指令文件与用户直接要求 > skills > 默认行为

### 3.6 它如何"指导 Agent 使用其他 skill"

它**不列举**其他 skill，也不写流程，只做三件事：① 宣告"你有一批 skill 存在"；② 给出"动手前必须查"的强制规则与顺序；③ 给出冲突优先级与豁免条件。

"有哪些 skill、何时适用"完全交给 harness 的 **skill 发现机制**。

**因此 `writing-skills` 规定：`description` 只写触发条件，绝不概括流程。**

> 真实教训：`subagent-driven-development` 的 description 曾写成 "code review between tasks"，结果模型只做**一次** review，而正文流程图要求**两次**（spec compliance + code quality）。改成纯触发条件描述后，模型才去读流程图并正确执行。
> **陷阱本质**：概括流程的 description 会给模型一条捷径，使 skill 正文变成被跳过的文档。

### 3.7 完整链路（以 Claude Code 为例）

```mermaid
flowchart LR
    A["session-start hook 注入<br/><EXTREMELY_IMPORTANT> + using-superpowers 全文"] --> B["模型知道：动手前必须查 skill"]
    C["harness 在 system prompt 暴露<br/>每个 skill 的 name+description"] --> D["模型按当前任务做语义匹配"]
    B --> D
    D --> E["模型调用 Skill 工具加载命中的 SKILL.md"]
    E --> F["announce：Using X to Y，然后按正文执行"]
```

---

## 4. 工作流

**存在一条强制主干，但不是硬编码状态机，而是由 skill 互相点名串起来的。**

```mermaid
flowchart TB
    B["brainstorming<br/>设计探索 + 用户批准"] -->|architectural 路径| S["写 spec<br/>docs/superpowers/specs/"]
    S --> W["writing-plans<br/>拆成 2-5 分钟粒度的任务"]
    W -->|subagent 可用| SDD["subagent-driven-development"]
    W -->|subagent 不可用| EP["executing-plans"]
    B -->|bounded/spike 路径| IMPL["直接进入正常开发流程"]
    SDD --> TDD["test-driven-development<br/>每任务内强制 RED-GREEN-REFACTOR"]
    EP --> TDD
    SDD --> RCR["requesting-code-review<br/>每任务一次 + 分支末一次"]
    TDD --> FIN["finishing-a-development-branch<br/>验证测试 → 呈现选项 → 清理"]
    SDD --> FIN
    RCR --> FIN
```

**关键接缝（源码实际措辞）**：

- `brainstorming` 的**唯一合法出口**：*"Terminal states are path-bound. Architectural: the ONLY skill you invoke after brainstorming is writing-plans — never frontend-design, mcp-builder, or any other implementation skill."*
- `writing-plans` 结尾强制 offer 执行方式二选一，给出 `REQUIRED SUB-SKILL`
- `executing-plans` 与 `subagent-driven-development` 都以 `REQUIRED SUB-SKILL: Use superpowers:finishing-a-development-branch` 收尾
- `systematic-debugging` Phase 4 点名 `superpowers:test-driven-development` 与 `superpowers:verification-before-completion`

### 4.1 brainstorming 的三条路径（v6.3.0 #2063「Ceremony now scales to the task」）

先分类，且**必须把分类说出口**让用户可否决：

| 路径                    | 触发                                                     | 终点                                   | 是否产出文档                       |
| ----------------------- | -------------------------------------------------------- | -------------------------------------- | ---------------------------------- |
| **Spike**         | "能不能做到"类可行性问题                                 | 一份建议                               | 无（产出是答案，代码标记为一次性） |
| **Bounded**       | 现有代码里的小改动（**仓库里已有要改的流程可读**） | 聊天里的短设计 → 用户批准 → 直接实现 | 无 spec、无 plan                   |
| **Architectural** | 新项目 / 新子系统 / 重构组件关系                         | spec 文件 → plan → SDD/EP            | **有**                       |

配套硬约束：

- **`<HARD-GATE>`**：无论哪条路径，**任何实现动作前都必须先呈现意图并获批准**。
  *"What scales with simplicity is the artifact, never the approval."*
- **单向棘轮**：中途发现隐藏复杂度只能升级路径，不能降级。*"When in doubt between two paths, take the heavier one."*
- **Bounded 的判定依据是仓库而非熟悉度**：*"Bounded measures the repo, not your familiarity. A new project has no existing flow — it is architectural."*

### 4.2 每个 skill 的"仪式感"：announcement

四个 skill 显式要求开工前宣告（**这是判断 Agent 是否真加载了 skill 的可见信号**）：

```
"I'm using the writing-plans skill to create the implementation plan."
"I'm using the executing-plans skill to implement this plan."
"I'm using the using-git-worktrees skill to set up an isolated workspace."
"I'm using the finishing-a-development-branch skill to complete this work."
```

`using-superpowers` 通用要求 announce `"Using [skill] to [purpose]"`，并对 checklist 类 skill 要求"每项建一个 todo"。

> **注意**：这是**约定**而非结构保证。所以用 transcript 验收集成时，应看**实际行为**（如 `brainstorming` 是否真的先于代码触发），而不是只看有没有 announce 字串。

---

## 5. 其他 skill 核心内容

### 5.1 流程主干（process skills）

| Skill                                    | 核心原则 / Iron Law                                                          | 关键机制                                                                                                                                                                                                                                                                  |
| ---------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **brainstorming**                  | `<HARD-GATE>`：实现前必须先获批准                                          | spike/bounded/architectural 三路径；一次只问一个问题；提 2-3 个方案带 trade-off；YAGNI ruthless；设计分段呈现并逐段确认；Spec Self-Review 四项（placeholder / 一致性 / 范围 / 歧义）；用户复核 gate                                                                       |
| **writing-plans**                  | 假设执行者是"零上下文、品味可疑但技术熟练的工程师"                           | 任务粒度 2-5 分钟一步；每任务必须有**Files:**（Create/Modify/Test 精确路径）与 **Interfaces:**（Consumes/Produces 精确签名）；**No Placeholders** 清单；Self-Review 三项（spec 覆盖 / placeholder / 类型一致性）；DRY + YAGNI + TDD + frequent commits  |
| **executing-plans**                | 批判性读 plan，然后严格执行                                                  | 先`using-git-worktrees` 建隔离工作区；有疑问先提；每步按 plan 验证；四类情况必须停下来问；结束交 `finishing-a-development-branch`。**v5.1.0 起取消"批次执行 + 人类检查点"旧模式**，改为连续执行                                                                 |
| **subagent-driven-development**    | Fresh subagent per task + task review（spec + quality）+ 分支末 broad review | 见 §5.2                                                                                                                                                                                                                                                                  |
| **using-git-worktrees**            | 先检测已有隔离 → 优先原生工具 → 再退 git。Never fight the harness          | Step 0 用`GIT_DIR != GIT_COMMON` 检测（含 submodule guard）；**建 worktree 前先征求同意**；目录优先级：指令声明 > `.worktrees/` > `worktrees/` > 默认 `.worktrees/`；必须 `git check-ignore` 验证；创建后跑依赖安装 + 干净基线测试                        |
| **finishing-a-development-branch** | 验证测试 → 检测环境 → 呈现选项 → 执行 → 清理                             | 测试不绿就不给菜单；三态菜单（普通仓库 / 命名 worktree = 3 选项，detached HEAD = 2 选项）；**"Discard" 不再是菜单项**，只在用户显式要求时走且需输入 `discard` 确认；清理只对 `.worktrees/` 下的做；`git worktree remove` 被拒时**绝不自行 `--force`** |

#### writing-plans 的 plan 文件强结构

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [一句话]
**Architecture:** [2-3 句]
**Tech Stack:** [关键技术]
**Spec:** [本 plan 实现的 spec 路径 — plan 从 spec 论证而来，所以 spec 随 plan 同行]

## Global Constraints

[spec 里的项目级要求：版本下限、依赖限制、命名与文案规则、平台要求 —
 逐行一条，值从 spec 逐字照抄。每个 task 的要求隐含包含本节。]

---

### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [使用前面 task 的东西 — 精确签名]
- Produces: [后面的 task 依赖的东西 — 精确函数名、参数与返回类型]

- [ ] **Step 1: Write the failing test**
- [ ] **Step 2: Run test to verify it fails**
      Run: `pytest tests/path/test.py::test_name -v`
      Expected: FAIL with "function not defined"
- [ ] **Step 3: Write minimal implementation**
- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Commit**
```

**No Placeholders 禁止清单**（这些被称为 "plan failures"）：
`TBD`/`TODO`/"implement later"/"fill in details"；"add appropriate error handling"/"handle edge cases"；"Write tests for the above"（不给实际测试代码）；**"Similar to Task N"（必须重复代码，因为执行者可能乱序阅读）**；只描述"做什么"而不给代码块的步骤；引用任何 task 里都没定义的类型/函数。

> `Interfaces` 块的存在理由：**每个 implementer 只看得到自己那一个 task**，Consumes/Produces 是他了解邻接 task 用什么名字和类型的唯一途径。

### 5.2 subagent-driven-development 详述（最复杂的 skill）

- **为什么用 subagent**：隔离上下文。subagent"**绝不该继承你 session 的历史**"——你精确构造它所需的一切，同时保住自己的上下文用于协调。
- **Ledger（进度账本）** —— 最关键的工程化设计。原因写在正文里：*"Conversation memory does not survive compaction"*；真实 session 里曾有 controller 因上下文被压缩而**重新派发了整批已完成的 task**。
- **Pre-flight 冲突扫描**：派发 Task 1 前扫一遍 plan，输出一张**表**而非一句"扫描干净"——每一对共享文件/接口的 task 一行，每个 task 自洽性一行。发现了就记 `Ruling: <决定> — <理由> — <错了的代价>`。
- **只有四类才停机**：不可逆/破坏性操作；安全敏感操作；worktree 之外的副作用（merge/push/发布）；plan 烂到"每个方向都是猜"。**其他一切冲突由 controller 裁定并 ledger**，不许卡住等人类。
- **Fix loop 上限 5 轮**：R1-3 resume 原 implementer（上下文还在），R4-5 换更强模型的新 implementer（*"loop 撑过三次 resume 通常意味着 implementer 看不到自己的问题"*）；每轮后必须 **scoped re-review**（只验本次 fix diff，不许发散）；第 5 轮后触发 breaker，逐条裁定并 ledger。
- **禁止 worker 自派 reviewer**：真实 session 里发现"worker 派的每个 reviewer 都和 controller 后来派的重复了一遍，每个 task 多花一整份 review 成本"。
- **Model Selection**：区分"机械实现 / 集成判断 / 架构设计 / 审查"四档；*"Turn count beats token price"*（最便宜的模型常因多轮往返反而更贵）；派发**必须显式指定 model**，否则默默继承 session 最贵的那个。
- **产物全部走文件**：理由——*"你粘贴进 dispatch 的每个字都会在剩余 session 里常驻并在每一轮被重读"*。
- **结束时必须汇总所有 Ruling**：*"Before you delete anything, collect every ledger line containing 'Ruling:'"* —— *"A ruling that dies with the workspace was a decision made in secret."*
- **不许中途汇报**：除四类停机条件外，controller 不得停下来问"要继续吗"（*"Should I continue?" prompts and progress summaries waste their time*）。
- **Continuous execution**、**批量小同形任务**（多个同形小改动用 ONE dispatch），**等待子 agent 用有界长等待**（5-10 分钟一段）而非短轮询。

### 5.3 质量与纪律

| Skill                                    | Iron Law                                                     | 要点                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **test-driven-development**        | `NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST`          | 写了代码才补测试？**删掉重来**，连"留作参考"都不许。Verify RED / Verify GREEN 都是 MANDATORY，必须真跑命令看输出。GREEN 阶段禁止加特性/重构/"顺手优化"。反模式用 `<Good>`/`<Bad>` 代码对照呈现（坏例是"测试了 mock 而不是代码"）。附 `writing-good-tests.md`                                                                   |
| **systematic-debugging**           | `NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST`          | 四阶段：① 根因调查（读错误、稳定复现、查最近改动、**多组件系统在各边界插桩取证**、沿数据流回溯）② 模式分析（找可工作的对照）③ 假设与最小验证（一次一个变量）④ 实现（先写失败测试）。**3 次修复失败 → 停下质疑架构**，不许试第 4 次。附 `root-cause-tracing.md`、`defense-in-depth.md`、`condition-based-waiting.md` |
| **verification-before-completion** | `NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE` | Gate Function：IDENTIFY（什么命令能证明）→ RUN（完整跑）→ READ（完整输出 + 退出码）→ VERIFY → 才能声称。禁词：`should`/`probably`/`seems to`/`Great!`/`Done!`。特别点名"信任 subagent 的成功报告"是违规 —— 必须查 VCS diff                                                                                                 |
| **receiving-code-review**          | 技术严谨性 > 社交舒适                                        | 禁语：`"You're absolutely right!"`、`"Great point!"`、任何感谢。要求：复述技术需求、先验证再实现、一次一项并各自测试。**多条反馈中只要有一条不清楚，就停下全部不要实现**（"items may be related"）。对"implementing properly"类建议先 `grep` 检查是否真有人用，无人用则按 YAGNI 拒绝                                           |
| **requesting-code-review**         | Review early, review often                                   | 强制时机：SDD 每任务后、大特性完成后、合入 main 前。取`BASE_SHA`/`HEAD_SHA`，用 `code-reviewer.md` 模板派 subagent。Rationalization 表明确禁止"我自己 inline review 就行"——因为*"你是协调者，inline review 会烧掉你继续驱动工作所需的上下文"*                                                                                      |
| **dispatching-parallel-agents**    | 每个独立问题域派一个 agent                                   | 判定流程图（多个失败？→ 相互独立？→ 能并行？）。**同一响应里发出多个 dispatch 才是并行**；一次一个就是串行。给了"任务太宽泛 / 无上下文 / 无约束 / 输出模糊"四个常见错误的正反例                                                                                                                                                    |

### 5.4 元技能

| Skill                    | 内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **writing-skills** | Iron Law：`NO SKILL WITHOUT A FAILING TEST FIRST`（对新 skill **和修改已有 skill** 都适用）。核心比喻：**写 skill 就是 TDD 应用到流程文档上**（压力场景 = 测试用例，SKILL.md = 生产代码，基线失败 = RED，agent 遵守 = GREEN，补漏洞 = REFACTOR）。含 Skill Discovery Optimization、Token 效率目标（常驻 skill < 200 词、其他 < 500 词）、交叉引用规范（用 `**REQUIRED SUB-SKILL:** Use superpowers:x`，**禁止 `@` 链接**因为会强制加载烧 200k 上下文）、flowchart 使用边界。附 `anthropic-best-practices.md`、`persuasion-principles.md`、`testing-skills-with-subagents.md`、`render-graphs.js` |

---

## 6. 规范文件产出

**分两类性质完全不同的文件**：① 提交进 repo 的"规格资产"；② git-ignored 的"运行时草稿"。

```mermaid
flowchart TB
    subgraph Committed["提交进 git 的规格资产"]
        SPEC["docs/superpowers/specs/<br/>YYYY-MM-DD-topic-design.md"]
        PLAN["docs/superpowers/plans/<br/>YYYY-MM-DD-feature.md"]
        GI[".worktrees/ 目录 + .gitignore 改动"]
    end
    subgraph Scratch["git-ignored 运行时草稿"]
        SDD[".superpowers/sdd/plan-basename/<br/>progress.md / task-N-brief.md /<br/>task-N-report.md / review-*.diff"]
        BS[".superpowers/brainstorm/id/<br/>content/ + state/"]
    end
    subgraph Scripts["脚本与工具"]
        CJS["brainstorm server.cjs / helper.js /<br/>frame-template.html"]
        SH["sdd-workspace / task-brief / review-package"]
        FP["find-polluter.sh"]
        RG["render-graphs.js / graphviz-conventions.dot"]
    end
    B["brainstorming"] --> SPEC
    B --> BS
    B --> CJS
    W["writing-plans"] --> PLAN
    G["using-git-worktrees"] --> GI
    S["subagent-driven-development"] --> SDD
    S --> SH
    D["systematic-debugging"] --> FP
    WS["writing-skills"] --> RG
```

### 6.1 提交进 git 的规格资产

| 产物                          | 路径与命名                                                       | **提供者 skill**                                                       | 出处                                                                                      |
| ----------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **设计规格 spec**       | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`          | **`brainstorming`**（唯一生产者，且仅 **architectural 路径**） | `SKILL.md` 第 100 行 checklist 第 6 项 + 第 206 行 "After the Design → Documentation"  |
| **实施计划 plan**       | `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`          | **`writing-plans`**（唯一生产者）                                    | `SKILL.md` 第 18 行 `**Save plans to:**`                                              |
| **隔离工作区目录**      | `.worktrees/<branch-name>/`（或 `worktrees/`、用户指定目录） | **`using-git-worktrees`**                                            | `SKILL.md` Step 1b 的 Directory Selection 三级优先级                                    |
| **`.gitignore` 改动** | 项目根`.gitignore` 追加 `.worktrees`                         | **`using-git-worktrees`**                                            | `SKILL.md` Step 1b "Safety Verification"：**未被 ignore 时必须先加进去并 commit** |

细节：

- **只有这两个 skill 产出"进 git 的文档"**，其余 12 个 skill 一个都不产出提交类文件
- 路径可被用户偏好覆盖：两处都写了 *"User preferences for spec/plan location override this default"*
- **命名刻意区分**：spec 以 `-design.md` 结尾，plan 不带后缀。本仓库自己是活样本（`docs/superpowers/specs/` 19 份全带 `-design.md`，另有 1 份 `...-eval-results.md` 衍生文件；`docs/superpowers/plans/` 15 份全无后缀）
- **spec 被 plan 反向引用**：plan header 的 `**Spec:**` 字段；SDD 在 Setup 阶段顺指针读 spec 作为冲突裁决的权威依据（the spec is the binding authority, the plan is its argument）

### 6.2 git-ignored 运行时草稿

#### A. `.superpowers/sdd/` —— 全部由 `subagent-driven-development` 提供

| 产物                  | 命名                                                                   | 具体生成者                                     | 说明                                                                     |
| --------------------- | ---------------------------------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------ |
| 工作区目录            | `<repo-root>/.superpowers/sdd/<plan-basename>/`                      | `scripts/sdd-workspace PLAN_FILE`            | slug = plan 文件名去`.md`；同时 `mkdir -p` 并写自忽略 `.gitignore` |
| 自忽略文件            | 上述目录的父级`.gitignore`                                           | 同上（`printf '*\n' > "$base/.gitignore"`）  | 让工作区不进`git status`、不被误提交                                   |
| **进度账本**    | `progress.md`，首行必须是 `# SDD ledger — plan: <plan file path>` | **controller 手工写**（无脚本）          | 用于跨上下文压缩恢复；Ruling 行全部靠手写                                |
| **任务简报**    | `task-<N>-brief.md`                                                  | `scripts/task-brief PLAN_FILE N`             | 用 awk 抽出整段`### Task N:` 正文                                      |
| **执行者报告**  | `task-<N>-report.md`                                                 | implementer subagent 写                        | 与 brief**同名配对**，路径由 controller 在 dispatch 里指定         |
| **评审包 diff** | `review-<base7>..<head7>.diff`                                       | `scripts/review-package PLAN_FILE BASE HEAD` | 含 commit 列表 +`--stat` + `-U10` 净 diff                            |

> `review-*.diff` 按 SHA 区间命名是刻意的：**fix 之后重新生成的评审包会得到新文件名**，不覆盖上一轮。

**位置演进原因**（v6.0.3）：从 `.git/sdd/` 迁出，因为"Claude Code 把 `.git/` 视为受保护路径，会拒绝 agent 写入"，导致 implementer 写报告被中断。
**已知坑**：工作区是 git-ignored 临时文件，`git clean -fdx` 会连 progress ledger 一起删，只能从 `git log` 恢复。

#### B. `.superpowers/brainstorm/` —— 全部由 `brainstorming` 的 visual companion 提供

| 产物                | 命名                                                                          | 具体生成者                                                       |
| ------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| 会话目录            | `<project>/.superpowers/brainstorm/<pid>-<unixtime>/`                       | `scripts/start-server.sh`（传 `--project-dir` 时）           |
| **屏幕 HTML** | `content/*.html`（**每次新文件名，绝不复用**，如 `layout-v2.html`） | 由 agent（模型）写                                               |
| 连接信息            | `state/server-info`（owner-only `0600`，内含 session key）                | `server.cjs` 启动时写                                          |
| 停机标记            | `state/server-stopped`                                                      | `stop-server.sh` 或 `server.cjs`                             |
| 用户交互            | `state/events`（每行一个 JSON；**推新屏幕时自动清空**）               | `server.cjs` 追加写                                            |
| 进程状态            | `state/server.pid`、`state/server.log`、`state/server-instance-id`      | `start-server.sh`                                              |
| 端口/密钥复用       | `.superpowers/brainstorm/.last-port`、`.last-token`                       | `start-server.sh`（让重启复用端口、已开的浏览器 tab 自动重连） |
| 静态资源            | `frame-template.html`、`helper.js`                                        | `server.cjs` 启动时读入并注入                                  |

**设计原因**：`content/` 与 `state/` 拆成**平级目录**，是为了"服务器状态和用户交互数据不再与对外服务的 HTML 放在一起，否则会通过 HTTP 暴露"。
不传 `--project-dir` 时落到 `/tmp/brainstorm-<id>/`；`stop-server.sh` 只 `rm -rf` `/tmp/*` 下的会话，持久目录保留。

#### C. `TODO.md` —— 不是 superpowers 的产物，是降级方案

`skills/using-superpowers/references/pi-tools.md`：pi 无原生 todo 工具时，"track tasks in the plan or `TODO.md`"。属 pi 侧工具映射给出的**退化路径**。

### 6.3 配套脚本与工具的提供者

| 文件                                                                  | 提供者                          | 用途                                                                                             |
| --------------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------ |
| `skills/brainstorming/scripts/server.cjs`                           | `brainstorming`               | 零依赖 Node WebSocket 静态服务器（只读`content/`，拒绝 symlink / dotfile / 路径逃逸 / 硬链接） |
| `skills/brainstorming/scripts/helper.js`                            | `brainstorming`               | 浏览器端 helper，注入到屏幕 HTML                                                                 |
| `skills/brainstorming/scripts/frame-template.html`                  | `brainstorming`               | 屏幕外壳模板（含品牌 logo 与遥测）                                                               |
| `skills/brainstorming/scripts/start-server.sh` / `stop-server.sh` | `brainstorming`               | 启停与 owner-PID 生命周期                                                                        |
| `skills/subagent-driven-development/scripts/sdd-workspace`          | `subagent-driven-development` | 解析 / 创建工作区                                                                                |
| `skills/subagent-driven-development/scripts/task-brief`             | `subagent-driven-development` | 抽任务正文                                                                                       |
| `skills/subagent-driven-development/scripts/review-package`         | `subagent-driven-development` | 生成评审包                                                                                       |
| `skills/systematic-debugging/find-polluter.sh`                      | `systematic-debugging`        | 二分定位"哪个测试污染了文件/状态"                                                                |
| `skills/writing-skills/render-graphs.js`                            | `writing-skills`              | 把 skill 里的 graphviz 流程图渲染成 SVG                                                          |
| `skills/writing-skills/graphviz-conventions.dot`                    | `writing-skills`              | graphviz 样式规范                                                                                |

### 6.4 prompt 模板的提供者（含一个已失效残留）

| 模板                                                    | 提供者                                    | 当前状态                                       |
| ------------------------------------------------------- | ----------------------------------------- | ---------------------------------------------- |
| `requesting-code-review/code-reviewer.md`             | **`requesting-code-review`**      | ✅ 活跃（也被 SDD 用作最终全分支评审模板）     |
| `subagent-driven-development/implementer-prompt.md`   | **`subagent-driven-development`** | ✅ 活跃                                        |
| `subagent-driven-development/task-reviewer-prompt.md` | **`subagent-driven-development`** | ✅ 活跃（规定 spec 合规 + 代码质量双 verdict） |
| `subagent-driven-development/re-review-prompt.md`     | **`subagent-driven-development`** | ✅ 活跃（fix loop 的 scoped 复审）             |
| `brainstorming/spec-document-reviewer-prompt.md`      | `brainstorming`                         | ⚠️**孤儿文件**                         |
| `writing-plans/plan-document-reviewer-prompt.md`      | `writing-plans`                         | ⚠️**孤儿文件**                         |

**孤儿文件的来龙去脉**（v5.0.6「Inline Self-Review Replaces Subagent Review Loops」）：

- 结论：派 subagent 评审 spec/plan 会**使执行时间翻倍（约 25 分钟开销）而无质量提升**——跨 5 个版本 × 各 5 次试验的回归测试显示质量分完全相同
- `brainstorming` 的 Spec Review Loop（subagent + 3 轮上限）→ **inline Spec Self-Review** 四项 checklist
- `writing-plans` 的 Plan Review Loop → **inline Self-Review** 三项 checklist
- 效果：自评在 ~30s 内抓到 3-5 个真实 bug，缺陷率与 subagent 方案相当
- **结果**：这两个 `*-document-reviewer-prompt.md` 现在不被任何 `SKILL.md` 引用（全量 grep 确认，唯一命中都在 `docs/superpowers/` 的历史设计文档里）。两处自评正文都明确写了 *"No need to re-review — just fix and move on"*，与模板的"派 subagent → 迭代到 approved"语义直接冲突

**因此现在的评审体系是两层不同性质的**：

| 层       | 评审对象    | 方式                                        | 提供者                                                      |
| -------- | ----------- | ------------------------------------------- | ----------------------------------------------------------- |
| 文档评审 | spec / plan | **inline 自评 checklist**（≈30 秒）  | `brainstorming`、`writing-plans`                        |
| 代码评审 | 实际 diff   | **派 subagent**（真的需要独立上下文） | `requesting-code-review`、`subagent-driven-development` |

理由：文档评审是"照清单扫自己刚写的东西"，同上下文更快更准；代码评审需要"独立于作者的眼光"，必须换 subagent。

### 6.5 SDD 账本行格式（controller 必须按此追加）

```
# SDD ledger — plan: docs/superpowers/plans/xxx.md          ← 首行身份行
Task <N>: complete (commits <base7>..<head7>, review clean)
Task <N>: complete (commits <base7>..<head7>, <K> parked)
Task <N>: minor (deferred): <one-liner>
Task <N>: fix round <R>/5 (<X> addressed, <Y> open — <finding 单行>; commits <a7>..<b7>)
Task <N>: parked — <finding> — Ruling: <为什么代码保留>
Task <N>: Ruling: <finding> — <决定与理由>
```

另有 pre-flight 扫描表（每对共享文件/接口的 task 一行，每个 task 自洽性一行）。

### 6.6 subagent 的固定输出格式（可解析契约，非建议）

**Spec / Plan reviewer**（现已不用，格式留档）：

```
## Spec Review
**Status:** Approved | Issues Found
**Issues (if any):** - [Section X]: [问题] - [为何影响规划]
**Recommendations (advisory, do not block approval):**
```

**Task reviewer**：`### Spec Compliance`（✅ / ❌ / ⚠️ Cannot verify from diff）→ `### Strengths` → `### Issues`（Critical / Important / Minor，每条含 file:line + 为什么重要 + 怎么修）→ `### Assessment`（`Task quality: Approved | Needs fixes`）。

**Implementer 报告**：正文写进 `task-N-report.md`，返回消息只允许 **<15 行**，含 `Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT`。

**Re-reviewer**：`### Finding Verdicts`（每条 ADDRESSED / NOT ADDRESSED + file:line，**"Attempted" 不算 addressed**）→ `### New Breakage in the Fix Diff` → `### Out-of-Scope Observations`（进 ledger 不延长循环）→ `### Verdict`。

---

## 7. 交互模型：无需 slash command

### 7.1 结论

**不需要任何 slash command，但"无感知"要拆开看 —— 触发是无感知的，过程有显式人类 gate。**

### 7.2 slash command 的历史

`RELEASE-NOTES.md` **v5.1.0（2026-04-30）**：

> **Legacy slash commands removed** — `/brainstorm`, `/execute-plan`, `/write-plan` are gone. They were deprecated stubs that did nothing but tell the user to invoke the corresponding skill. Invoke `superpowers:brainstorming`, `superpowers:executing-plans`, `superpowers:writing-plans` directly instead. (#1188)

更早的 v4.x 还曾给它们加过 `disable-model-invocation: true`（限制只能人工调用），最终整个 `commands/` 目录被删（`scripts/sync-to-codex-plugin.sh` 的 EXCLUDES 里还留着 `/commands/` 这条历史遗迹）。

v5.1.0 同时移除了 `superpowers:code-reviewer` 具名 agent —— 合并进 `requesting-code-review/code-reviewer.md` 作为自包含 Task-dispatch 模板。

设计意图总结（RELEASE-NOTES）：

> Skills are primarily invoked by describing what you want to do, not by running specific commands.

### 7.3 与 Spec-kit / OpenSpec 的根本差异

|                      | Superpowers                                                                          | Spec-kit / OpenSpec 类                                      |
| -------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| 阶段推进             | 模型根据`description` 语义匹配 + bootstrap 强制"动手前必须查"                      | 用户执行`/specify`、`/plan`、`/tasks`、`/implement` |
| 是否依赖用户记得命令 | 否                                                                                   | 是                                                          |
| 触发保障             | `<EXTREMELY_IMPORTANT>` 注入 + `using-superpowers` 的 1% 规则与 Red Flags 表     | 命令本身即状态转移                                          |
| 失败模式             | **触发失败**（bootstrap 未注入 / description 不匹配）→ 所以验收测试是唯一保障 | 用户忘了敲命令                                              |

### 7.4 "无感知"的四个例外（都是刻意的）

1. **`brainstorming` 的批准 gate**：无论哪条路径，实现前**必须**停下等一个明确的 "yes"。
   *"presenting the design and starting in the same breath is skipping the gate."*
2. **`writing-plans` 的执行方式选择**：plan 写完后问 Subagent-Driven 还是 Inline Execution
3. **`finishing-a-development-branch` 的三选一菜单**：merge / push+PR / keep as-is —— *"the integration decision is theirs"*
4. **`using-git-worktrees` 的建 worktree 同意门**：v5.1.0 起不再隐式创建（修复 #991：SDD 曾未经同意就 auto-create worktree）

反向约束：**除四类停机条件外，SDD controller 不许中途停下来汇报进度。**

### 7.5 模型会"出声"报告自己在用哪个 skill

因为 `announce at start` 约定 + `using-superpowers` 的 `"Using [skill] to [purpose]"` 要求，对话里会看到 `I'm using the brainstorming skill to...`。
更准确的描述：**零命令行、零阶段标识、零手工触发，但过程高度可见且有人类决策点。**

### 7.6 显式调用仍然可用

各 harness 原生 skill 机制保留显式入口：

- Claude Code：`superpowers:brainstorming` 这类插件命名空间下的 skill 名（插件提供的命令由 Claude Code 自动加命名空间以避免跨插件冲突）
- pi：`/skill:name`
- Codex / Devin：system prompt 里列出每个已安装 skill 的 name + description
- 自然语言点名："use the brainstorming skill" —— `tests/explicit-skill-requests/` 专测此路径（含 Haiku 等弱模型与多轮场景）

---

## 8. 速查表

### 8.1 关键路径速查

| 用途               | 路径                                                                              |
| ------------------ | --------------------------------------------------------------------------------- |
| spec 设计文档      | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`                           |
| plan 实施计划      | `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`                           |
| SDD 工作区         | `<repo-root>/.superpowers/sdd/<plan-basename>/`                                 |
| SDD 账本           | 上述目录下`progress.md`                                                         |
| brainstorm 会话    | `<project>/.superpowers/brainstorm/<pid>-<ts>/{content,state}/`                 |
| 工具映射           | `skills/using-superpowers/references/<harness>-tools.md`                        |
| bootstrap 内容来源 | `skills/using-superpowers/SKILL.md`                                             |
| 仓维护脚本         | `scripts/{bump-version,lint-shell,sync-to-codex-plugin,package-codex-plugin}.*` |
| 版本登记表         | `.version-bump.json`                                                            |
| 移植指南           | `docs/porting-to-a-new-harness.md`                                              |

### 8.2 关键常量速查

| 项                               | 值                                                               |
| -------------------------------- | ---------------------------------------------------------------- |
| `<EXTREMELY_IMPORTANT>`        | bootstrap 包装标签（也是去重标记）                               |
| `superpowers:`                 | skill 命名空间前缀（来自插件机制）                               |
| `startup\|clear\|compact`        | Claude Code 的 SessionStart matcher（**刻意不含 resume**） |
| `GIT_DIR != GIT_COMMON`        | 判断"已在 linked worktree 内"的检测式                            |
| `.worktrees/` > `worktrees/` | worktree 目录优先级（两者都有时前者胜）                          |
| 5 轮                             | SDD fix loop 上限                                                |
| 3 次                             | systematic-debugging 中"修复失败就质疑架构"的阈值                |
| ~30s vs ~25min                   | inline 自评 vs subagent 文档评审的耗时对比                       |

### 8.3 各 harness 集成速查（Appendix A）

| Harness      | Entry point                                                                 | Bootstrap 机制                                                                                       | Tool mapping                                                     | Tests                                          |
| ------------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------------- |
| Claude Code  | `.claude-plugin/plugin.json` + `hooks/hooks.json`                       | shell hook →`session-start`（`hookSpecificOutput.additionalContext`）                           | 原生`Skill` 工具，无需适配文件                                 | `tests/hooks/`                               |
| Codex        | `.codex-plugin/plugin.json`（声明空 `hooks`）                           | 原生 skill 发现（无 session-start hook）                                                             | `references/codex-tools.md`                                    | `tests/codex/`、`tests/codex-plugin-sync/` |
| Cursor       | `.cursor-plugin/plugin.json` + `hooks/hooks-cursor.json`                | shell hook →`session-start`（`additional_context`）                                             | 无需（Claude Code 兼容工具面）                                   | `tests/hooks/`                               |
| Copilot CLI  | 共用 Claude Code hook 路径（`COPILOT_CLI` env）                           | shell hook →`session-start`（`additionalContext`）                                              | 无需（同兼容面）                                                 | `tests/hooks/`                               |
| Gemini CLI   | `gemini-extension.json` + `GEMINI.md`                                   | instructions file`@`-include                                                                       | `references/gemini-tools.md`                                   | —                                             |
| Kimi Code    | `.kimi-plugin/plugin.json`                                                | `sessionStart.skill` 加载 `using-superpowers`                                                    | 内联`skillInstructions`                                        | `tests/kimi/`                                |
| OpenCode     | `.opencode/plugins/superpowers.js`（经根 `package.json` `main` 声明） | 进程内：`config` hook 注册 skills 目录；`experimental.chat.messages.transform` 注入 user message | 内联在`superpowers.js`                                         | `tests/opencode/`                            |
| pi           | `.pi/extensions/superpowers.ts`                                           | 进程内：`resources_discover` + `context` 事件；lifecycle flag + compaction-aware                 | `piToolMapping()` 内联 **且** `references/pi-tools.md` | `tests/pi/`                                  |
| Devin CLI    | `.devin-plugin/plugin.json`                                               | 原生 skill 发现 + system prompt 暴露 name/description                                                | 无需                                                             | `tests/devin/`                               |
| Hermes Agent | `.hermes-plugin/plugin.yaml` + `__init__.py`                            | `pre_llm_call` hook 首轮 `return {"context": bootstrap}`                                         | `references/hermes-tools.md`                                   | `tests/hermes/`                              |

### 8.4 移植时的 gotcha 清单（Appendix B 浓缩）

- **Opt-in 不算集成** —— 若用户每会话要做点什么才能用上，验收测试必失败
- **JSON 字段写错 → 静默失败或双重注入**（Shape A，仅此形态）
- **hook 配置 schema 每 harness 不同** —— Cursor 的与前者的完全不像，照最近的抄
- **插件根环境变量名每个 harness 不同** —— `${CLAUDE_PLUGIN_ROOT}` vs 相对路径
- **Shape B 注入 user 而非 system message** —— 刻意为之（#750、#894），别"修好"它
- **每 step vs 每 turn 的回调频率** —— 去重策略必须匹配频率，抄错会破坏注入
- **消息对象形状每 harness 不同** —— pi 与 OpenCode 用**不兼容**的结构
- **别去无 skill 系统的 harness 找 registration API** —— 它没有
- **映射表在两处** —— 进程内插件可能内联 + `references/` 各一份，需同时更新
- **"never read skill files" 的真实含义** —— 意为"别绕过你平台的 skill 加载机制"，而非"永不 file-read"。无 skill 工具的 harness 里，**读 `SKILL.md` 就是那个机制**，必须在映射里明说
- **Windows 上 hook 脚本不能带 `.sh` 扩展名**
- **新 manifest 不登记进 `.version-bump.json` 就会发旧版本**
- **永不为了适配 harness 去改 skill 正文** —— 修复属于工具映射

---

## 9. 待查证 / 后续可深入

- `evals/`（drill 评测框架）不在本仓库，需 clone `superpowers-evals` 才能真正跑 skill 行为评测；`docs/testing.md` 说明 `tests/` 与 `evals/` 的分工
- `docs/plans/` 与 `docs/superpowers/plans/` 是**两套不同年代的目录**（前者为历史遗留，v4.x 起新产物统一到 `docs/superpowers/`）
- `.antigravity-plugin/` 在 Appendix A 的行被移除，但 `docs/porting-to-a-new-harness.md` Part 6 仍引用 `.antigravity-plugin/install.sh`；可核实当前实现状态
- `research/`、`docs-learn/` 等目录的性质未逐一确认
