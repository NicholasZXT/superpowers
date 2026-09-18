# 将 Superpowers 移植到新的宿主环境（Harness）

本指南介绍了如何为新的宿主环境（即 IDE、CLI 或 Agent 运行器，而非 Claude Code）添加支持，使 Superpowers 技能能够像在原生环境中一样自动触发。

文档分为两个层次。**第 1-3 部分**解释了系统的工作原理以及如何判断某个宿主环境是否受支持；在动手修改任何内容之前，请务必阅读这些部分。**第 4-8 部分**是为 Agent（在人类伙伴监督下）执行端到端移植直至分发的规定流程。附录索引了当前的参考集成实现，以便你复制最接近的版本。

不同宿主环境的集成机制各不相同，且会持续变化。本指南特意教授那些**不变量**——无论机制如何都必须成立的条件——并指向一个实时的参考实现供你复制。当本指南与代码不一致时，以代码为准；请修正指南。

## 开始之前

添加宿主环境是本仓库中风险最高的贡献类型。在编写任何内容之前：

- 完整阅读 `CLAUDE.md` 和 `.github/PULL_REQUEST_TEMPLATE.md` —— 贡献者规则和新宿主环境 PR 要求不是可选项。
- 搜索**已打开和已关闭**的 PR，查看是否有人尝试过该宿主环境。如果存在，请在开始自己的尝试之前了解其停滞的原因。

------

## 第 1 部分 — Superpowers 在不同宿主环境中的工作原理

Superpowers 的内容在任何地方都是相同的。随宿主环境变化的是那个薄层，它负责将内容传递给模型，并将指令转换为宿主环境的原生工具。包含三个组件：

1. **技能（与宿主环境无关）。** `skills/` 中的所有内容是唯一事实来源，被每个宿主环境逐字共享。技能旨在描述*动作*——“调用技能”、“读取文件”、“分发子代理”、“创建待办事项”——绝不命名特定工具。这使得同一个技能主体无需编辑即可在 Claude Code、Codex、Gemini、pi 等平台上运行。
2. **工具映射（特定于宿主环境）。** 每个宿主环境都需要将动作词汇转换为其真实的工具名称。该转换位于 `skills/using-superpowers/references/<harness>-tools.md` 和/或内联在宿主环境的引导注入器中（见第 5 部分）。例如，它规定：“*分发子代理* → 使用 `subagent_type` 参数调用 `task`。”
3. **引导（Bootstrap，特定于宿主环境）。** 在每次会话开始时，完整的 `skills/using-superpowers/SKILL.md` 会被注入到模型的上下文中，包裹在 `<EXTREMELY_IMPORTANT>` 标签内，并附带工具映射。正是这个注入的技能教会模型技能的存在，以及它在行动前必须检查是否有相关技能。**引导就是整个集成。** 没有它，技能文件就是惰性的——存在于磁盘上，但永远不会被调用。

### 使其生效的两条规则

**1. 技能命名动作，而非工具。** **不要**为了适配你的宿主环境而编辑技能主体。移植只是添加一个工具映射参考和一个引导注入器；它绝不会深入 `skills/*/SKILL.md` 去替换工具名称。（项目的贡献者指南将技能内容视为精心调整的行为塑造代码；为了“合规”而重写它会被直接拒绝。）

**2. 所有内容都通过宿主环境自身的安装机制发布。绝不编辑用户的文件。** 引导、技能和工具映射都作为*宿主环境安装内容的一部分*进行交付——插件、扩展、市场条目、扩展捆绑的上下文文件。移植**绝不能**触及用户的全局或个人配置（`~/.gemini/config/AGENTS.md`、`settings.json`、`trustedFolders.json`、手动编辑的 `~/.bashrc` 等）来注入任何内容。宿主环境拥有其加载内容的控制权；你的安装产物是你唯一能写入的东西。如果安装机制确实无法承载引导，这是一个需要暴露的限制（第 6 部分）——绝不是手动编辑用户配置的许可。（Shape C *不是*例外：Gemini 的上下文文件没问题，因为它随*已安装的扩展一起发布*，并由清单的 `contextFileName` 声明——宿主环境加载的是扩展自己的文件，而不是你在用户主目录中编辑的文件。）

------

## 第 2 部分 — 该宿主环境是否受支持？

只有当宿主环境能够完成以下所有操作时，才能支持 Superpowers。在编写代码之前检查这些条件——如果第一个失败，请立即停止。

### 硬性要求：自动会话启动注入

宿主环境必须允许你在**每次会话开始时**将文本注入模型的上下文，**无需人类伙伴进行按次选择加入**。这是唯一不可协商的能力。它可以采取任何形式：

- 一个**钩子/事件系统**，在会话启动时运行 shell 命令并读取其标准输出（Claude Code, Cursor, Copilot CLI），或
- 一个**进程内插件/扩展**，具有会话启动或消息生命周期回调，可以修改消息数组（OpenCode, pi），或
- 一个**指令文件**约定，宿主环境加载一个由*你安装的扩展提供并声明*的上下文文件（例如 Gemini 的 `contextFileName` 指向扩展自己的 `GEMINI.md`）——而不是你在用户主目录中编辑的文件。

如果让 Superpowers 出现在模型面前的唯一方法是让人类伙伴在每次会话中选择加入（粘贴提示词、运行命令、启用模式），则该宿主环境**无法**得到适当支持。第 3 部分的验收测试将会失败，PR 将被关闭。这是“移植”并非真正移植的最常见原因。

### 其余能力检查清单

| 能力                                 | 为什么需要它                                                 | 如果缺失                                                     |
| :----------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **技能发现 + 调用**                  | 模型必须能够按需加载技能的完整内容                           | 如果没有原生技能工具，认可的后备方案是直接 `read` 相关的 `SKILL.md` ——见第 5 部分。既没有技能工具也没有文件读取能力的宿主环境无法工作。 |
| **文件读取 / 写入 / 编辑**           | 几乎每个技能都会操作文件                                     | 必不可少。无变通方法。                                       |
| **运行 Shell 命令**                  | TDD、验证、git 工作流                                        | 必不可少。                                                   |
| **子代理 / 任务分发**                | `dispatching-parallel-agents`, `subagent-driven-development` | 可降级：如果不可用，这些特定技能会告诉模型内联完成工作或报告缺失的能力——*绝不要*凭空编造一个 `Task` 调用。某些宿主环境通过配置标志来控制此功能（例如 Codex 需要启用多代理）。 |
| **待办事项 / 任务跟踪**              | 多个技能中的进度跟踪                                         | 可降级：退回到计划文件或 `TODO.md`。                         |
| **网页获取 / 搜索**                  | 少数技能                                                     | 可降级。                                                     |
| **Shell 或多语言脚本执行 (Windows)** | 仅适用于 shell-hook 形态，且仅当你需要 Windows 支持时        | 见第 7 部分。进程内插件宿主环境完全避开此问题。              |

“可降级”意味着：技能已经为缺失的工具准备了后备措辞。你在工具映射中的工作是：当真实工具存在时指向它，当不存在时复用该后备措辞。

### 你可能根本不需要新目录

一些“新宿主环境”实际上只是现有集成的不同安装器。例如，Factory 的 Droid 通过其自身的 `plugin install` 命令使用 Claude Code 插件，这里不需要新文件。在构建之前，检查宿主环境是否可以简单地加载现有清单。除了 README 中的一段话外不向本仓库添加任何内容的移植，也是一个完全合理的结果。

------

## 第 3 部分 — 完成定义

当满足以下**所有**条件时，移植才算完成：

1. `using-superpowers` 引导在每次会话开始时加载，无需按次选择加入。

2. 存在针对该宿主环境的工具映射（在 `references/<harness>-tools.md` 中、内联在引导中，或两者皆有——依据第 5 部分）。

3. 技能实际上可以被调用——原生地，或通过记录在案的读取 `SKILL.md` 后备方案——并且模型会遵循它们。

4. **验收测试通过。** 在一个干净的会话中，用户消息：

   > Let's make a react todo list

   必须在**编写任何代码之前**自动触发 `brainstorming` 技能。捕获完整的对话记录——PR 需要提供它。

5. 测试覆盖了集成（第 5 部分）并通过。

6. 真实用户可以通过宿主环境自身的机制安装它（而不是手动复制文件），并且在适用的情况下版本在 `.version-bump.json` 中被跟踪（第 6 部分）。注意，某些安装程序在安装时会重写或剥离清单（有的将其精简为仅 `{"name": …}`），因此“*已安装*的文件报告仓库版本”并不总是可实现的目标——在源清单处跟踪版本，不要将重写的已安装清单视为失败。

在完整验收测试之前的快速冒烟检查：启动一个会话并要求模型描述其 superpowers。如果引导已注入，它就会知道自己拥有这些能力。（OpenCode 的安装文档使用 `opencode run --print-logs "hello" 2>&1 | grep -i superpowers` 通过不同的机制达到相同目的——日志 grep 而非询问模型；`2>&1` 很重要，因为日志输出到 stderr。找到你的宿主环境的等效方法。）

------

## 第 4 部分 — 选择你的集成形态

有三种结构形态，区别在于*你如何将引导呈现在模型面前*。选择与你的宿主环境暴露的方式相匹配的一种，然后复制该参考实现。形态决定了第 5 部分中的几乎所有内容——以下步骤据此分支。

### 如何判断你属于哪种形态

在路由之前，了解宿主环境的*实际*机制——不要假设它有完善的文档，或者它的行为与其派生自的宿主环境相同。

**找到接口表面：**

- **在网上搜索该宿主环境的文档**（扩展 / 插件 / 钩子 / 技能 / MCP / “上下文文件” / “规则文件”）。供应商工具变化很快；请搜索而非依赖训练知识。
- **查找并阅读该宿主环境现有的第三方扩展/插件。** 一个真实可用的例子胜过文档——它展示了清单格式、安装命令以及宿主环境实际加载的组件。
- 检查宿主环境在启动时加载什么：设置文件？扩展目录？每项目或全局指令文件（`AGENTS.md`, `<NAME>.md`）？

**如果文档不全，通过经验进行逆向工程**（真正的移植者不得不做过以下每一项）：

- 对二进制文件使用 `strings` / 在安装树中 grep 钩子事件名称、配置路径及其读取的指令文件。
- **要求运行中的模型列举其自身的工具名称**——例如，“列出你可以调用的每个工具的精确机器名称”。这是获取工具名称而不凭空捏造的权威方法（见步骤 4）。
- 用**唯一标记测试**证明每一个假设：通过你认为有效的机制注入一个无意义的令牌，启动一个新会话，并确认该令牌确实到达了模型。

**派生版本不会继承其父版本的行为。** 从另一个宿主环境派生的宿主环境（例如 Gemini 派生的 CLI）可能暴露父版本的清单字段和 `@`-include 语法，但*仍以不同的方式处理它们*。用标记验证；永远不要假设父版本的方案可以直接迁移。

然后路由到一种形态：

- 会话启动时运行 shell 命令并读取其 stdout → **Shape A**。
- 带有生命周期回调的插件/扩展模块，你可以在其中运行代码 → **Shape B**。
- 仅有一个始终开启的指令文件，没有钩子也没有代码插件 → **Shape C**。

**形态可以组合——它们不是互斥的。** *技能发现*机制和*引导*机制不必是同一形态——但**两者都必须依赖安装机制**（规则 2）。分别决定两个问题：*技能在哪里被发现？* 以及 *引导如何在每次会话中到达模型？* 一个宿主环境可能通过插件安装技能，但需要通过另一种安装发布的方式传递引导（扩展声明的上下文文件，或者——见下文——由宿主环境在会话启动时展示已安装的 `using-superpowers` 技能自身的描述）。如果有多个安装机制接口自动注入，优先选择最可靠的。你**不能**做的是通过编辑用户的全局配置来弥补缺口。

### Shape A — Shell-hook

宿主环境有一个钩子系统，在会话启动时运行 shell 命令并从其 stdout 读取 JSON。配置的命令运行 `run-hook.cmd`，这是一个多语言包装器，仅用于定位 bash 并分发指定的脚本；脚本（`hooks/session-start` 或特定于宿主环境的变体）负责读取 `using-superpowers/SKILL.md` 并打印一个 JSON 对象，其**字段名和嵌套结构因宿主环境而异**。

- 参考：`hooks/session-start`、`hooks/run-hook.cmd` 以及每个宿主环境的钩子配置 `hooks/hooks.json` (Claude Code) 和 `hooks/hooks-cursor.json` (Cursor)。
- 清单：`.cursor-plugin/plugin.json` 是 Shape A 的清单示例，它将宿主环境指向 `./skills/` 和正确的 `hooks-*.json`。Claude Code 的 `.claude-plugin/plugin.json` 未设置这两个字段——它按约定自动发现 `skills/` 和 `hooks/hooks.json`。**不要**为 Shape A 复制 Codex 的 `.codex-plugin/plugin.json`：它特意声明了一个空的 `hooks` 对象以抑制 Codex 的 `hooks/hooks.json` 自动发现，因为 Codex 原生展示技能且不运行会话启动钩子。

> **钩子系统不等于会话启动事件。** 一个宿主环境可能有 `hooks.json` 机制——甚至在其二进制文件中包含字面字符串 `SessionStart`——但却没有在会话启动时触发并能注入上下文的钩子事件。（一个真实的宿主环境仅暴露了工具前/后和停止事件；`SessionStart` 字符串仅用于遥测。）在承诺使用 Shape A 之前，确认你需要的*特定事件*存在并且可以写入模型上下文。如果不能，引导应放在指令文件（Shape C）中。

### Shape B — 进程内插件 / 扩展

宿主环境加载一个暴露生命周期回调的 JS/TS 模块。你通过宿主环境的 API 注册技能目录，并通过在代码中修改消息数组来注入引导。

- 参考：`.opencode/plugins/superpowers.js` (JavaScript) 和 `.pi/extensions/superpowers.ts` (TypeScript)。对于任何**没有原生技能工具**的宿主环境，pi 是最接近的参考。

### Shape C — 指令文件

宿主环境既没有 shell 钩子也没有代码插件——其会话启动接口是一个由*你安装的扩展提供并由清单声明*的上下文文件（例如 Gemini 的 `contextFileName` → 扩展自己的 `GEMINI.md`）。你不能运行代码或修改消息；扩展的上下文文件指向引导。没有注入器来组装字符串或剥离 frontmatter——宿主环境按原样加载引用的内容。**这之所以有效，完全是因为该文件是已安装扩展的一部分**——绝不要用“编辑用户的全局 `GEMINI.md`/`AGENTS.md`”来替代发布你自己的文件（规则 2）。

- 参考：`gemini-extension.json`（清单，包含 `contextFileName`）、`GEMINI.md`（两个 `@`-include——引导技能和工具映射参考）、`skills/using-superpowers/references/gemini-tools.md`。
- 注意：`@`-include 是 Gemini 的特性。如果你的宿主环境加载指令文件但没有 include 语法，你必须将引导内容内联到文件中。
- **不要相信 `@`-include 真的会被展开——要证明它。** 一个 Gemini *派生*的宿主环境可能接受 `@./path` 语法，但将其视为*模型可以选择读取的提示*（它会发出一个文件读取工具调用），而不是保证的内联展开。这就是引导在每次会话中可靠存在与模型可能读取它之间的区别。运行唯一标记测试：如果标记在没有工具调用的情况下不在上下文中，**内联内容**而不是 `@`-include 它。

### 路由表

| 如果宿主环境…                                                | 使用形态           | 复制自                                                       |
| :----------------------------------------------------------- | :----------------- | :----------------------------------------------------------- |
| 在会话启动时运行 shell 命令并读取其 stdout                   | A (shell-hook)     | Cursor (`hooks/session-start` + `hooks/hooks-cursor.json` + `.cursor-plugin/`) |
| 是具有会话/消息生命周期回调的 JS/TS 插件宿主                 | B (进程内)         | OpenCode (`.opencode/`) — 或 pi (`.pi/`) 如果它没有原生技能工具 |
| 提供一个它始终加载的扩展声明上下文文件                       | C (指令文件)       | Gemini (`gemini-extension.json` + `GEMINI.md` + `references/gemini-tools.md`) |
| 具有插件安装命令和安装程序保留的清单 `contextFileName`（或等效项） | C 通过插件安装程序 | Antigravity (`.antigravity-plugin/` — `agy plugin install` 发布生成的上下文文件；验证安装程序是否保留它 — 第 6 部分) |

大多数真实的宿主环境清晰地符合其中一行；最后一行是混合情况（规则 2 仍然适用——引导依赖安装机制，绝不是用户配置编辑）。

------

## 第 5 部分 — 移植流程

### 步骤 1 — 研究最接近的参考实现

打开第 4 部分中为你的形态命名的文件，从头到尾阅读。下面的模式是总结；代码才是规范。

### 步骤 2 — 创建清单 / 入口点

创建宿主环境用于识别插件的任何内容。在精神上匹配现有的：

- **Shape A:** 一个 `*-plugin/plugin.json`（见 `.cursor-plugin/plugin.json`），包含 `name`、`version`、`description`、作者/许可证/关键词、`"skills": "./skills/"` 和 `"hooks": "./hooks/hooks-<harness>.json"`。加上 `hooks-<harness>.json` 本身，注册一个会话启动钩子，其命令调用 `run-hook.cmd`。

- Shape B: 宿主环境加载的模块（例如 `.<harness>/plugins/*.js`）加上被发现所需的任何包元数据。提交的包元数据是

  仓库根目录的 `package.json`:`main` 指向 OpenCode 插件，`pi` 字段（`pi.extensions`, `pi.skills`）加上 `pi-package` 关键字声明 pi 扩展。特定于宿主环境的本地清单和锁文件不进 git——`.opencode/.gitignore` 排除了 `node_modules`、`package.json` 和锁文件。对你的宿主环境的本地安装产物也这样做，以免污染仓库——但绝不要 gitignore 仓库根目录的 `package.json`，它是被跟踪的唯一事实来源。

  - **构建/依赖检查。** 决定宿主环境如何加载你的模块：它是直接运行源代码（pi 的 `.ts` 在 `package.json` 中按原样引用；OpenCode 提供纯 `.js`），还是需要转译/构建步骤？Superpowers 是零运行时依赖的。pi 的 `import type { ExtensionAPI }` 之所以有效，是因为宿主环境直接运行 `.ts`，在加载时提供该类型，且仓库从不在 CI 中对该文件进行类型检查——该导入甚至未被声明为依赖。如果*你的*宿主环境确实对插件进行类型检查或打包，这会中断：未声明的类型导入会失败，且 PR 规则仅为新宿主环境免除*运行时*依赖，不包括开发/类型包。如果遇到这种情况，请与维护者确认方法，而不是悄悄添加依赖。将任何构建输出排除在 git 之外并记录命令。

- **Shape C (指令文件):** 一个小清单（见 `gemini-extension.json`：`name`, `description`, `version`, `contextFileName`）加上上下文文件本身（`GEMINI.md` 只是两个 `@`-include：引导技能和工具映射参考）。Gemini 清单没有 `skills` 字段——Gemini 自动发现安装在扩展中的 `skills/` 目录。如果你的宿主环境有原生技能工具但没有清单字段来注册目录，你必须找到其发现约定（阅读其扩展文档），然后通过经验验证：连接后，要求模型列出其可用技能——如果捆绑的技能没有出现，说明发现尚未生效。

### 步骤 3 — 连接引导注入

这是移植的核心。共同目标：在会话启动时，将 `using-superpowers` 技能内容（包裹在 `<EXTREMELY_IMPORTANT>` 标签中）加上宿主环境的工具映射呈现在模型面前，并注明该技能已激活，以免模型尝试再次加载它。*如何*做到这一点——以及你组装什么与宿主环境原始加载什么——完全取决于你的形态。**不要**将一种形态的方案应用于另一种。

**Shape A — 脚本读取 `SKILL.md` 并打印宿主环境的 JSON。** 分发的脚本 (`hooks/session-start`) `cat` 整个 `SKILL.md`（包括 frontmatter——这没问题；它是逐字发出的），用“You have superpowers… for all other skills use the Skill tool”前言包裹它，进行转义，并打印宿主环境的 JSON 格式。Shape A 的工具映射**不**在这里内联——它位于 `references/<harness>-tools.md`（步骤 4）。确保 JSON 输出格式完全正确。`hooks/session-start` 从环境变量检测宿主环境并打印*三种*格式之一：

- Cursor (`CURSOR_PLUGIN_ROOT` 已设置): `{ "additional_context": "…" }`
- Claude Code (`CLAUDE_PLUGIN_ROOT` 已设置, `COPILOT_CLI` 未设置): `{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "…" } }`
- Copilot CLI / SDK 标准 (其他): `{ "additionalContext": "…" }`

这是一个陷阱。发出错误的字段或多余的字段意味着引导要么从不注入，要么注入两次（Claude Code 同时读取 `additional_context` 和 `hookSpecificOutput` 且不去重，因此同时发出两者会导致双重注入）。找到你的宿主环境期望的确切字段、嵌套和事件匹配器值。然后决定：在 `hooks/session-start` 中添加第四个分支，或者——如果宿主环境需要不同的引导消息或环境契约——添加一个专用的 `hooks/session-start-<harness>` 脚本。如果你添加了一个分支，而你的宿主环境*也*设置了先前分支所依赖的环境变量（某些宿主环境也设置 `CLAUDE_PLUGIN_ROOT`），请将你的分支排在否则会遮蔽它的分支之前。匹配宿主环境自身的事件匹配器字符串（Claude Code 使用 `startup|clear|compact`，Cursor 使用 `sessionStart`）；错误的匹配器意味着钩子静默地永不触发。

**钩子配置模式本身因宿主环境而异**——不要假设 Claude Code 的格式是通用的。比较 `hooks/hooks.json` 和 `hooks/hooks-cursor.json`：Cursor 的使用 `"version": 1`，小写的 `sessionStart` 键，相对的 `./hooks/run-hook.cmd` 命令，并省略了 Claude Code 使用的 `matcher`/`type`/`async` 字段。将你的 `hooks-<harness>.json` 匹配到最接近的现有文件，而不是单一的规范模板。

钩子**命令字符串引用宿主环境提供的插件根变量**，其名称因宿主环境而异：`hooks.json` 使用 `${CLAUDE_PLUGIN_ROOT}`，`hooks-cursor.json` 使用相对路径。使用你的宿主环境导出的任何内容。（`session-start` 脚本本身通过 `dirname` 重新推导根目录，因此脚本主体不依赖于此——但清单中的命令依赖于此。）

**发现宿主环境的契约。** 上述三个事实——环境变量、JSON 字段/嵌套、匹配器字符串——是宿主环境的契约，不是 Superpowers 的，所以你必须获取它们。阅读宿主环境的钩子文档，或通过经验找出：注册一个一次性的会话启动钩子，转储其环境并发出标记，然后观察哪个环境变量标识了宿主环境，以及宿主环境是否/如何处理你的 stdout。在编写真正的分支之前确定这些。

**Shape B — 在代码中组装字符串，然后作为用户消息注入。** 在这里你自己构建引导：读取 `SKILL.md`，剥离其 YAML frontmatter，并组装 `<EXTREMELY_IMPORTANT>` + 简短的前言（说明技能已加载且不得重新调用）+ 剥离后的主体 + 内联工具映射 + `</EXTREMELY_IMPORTANT>`。参考资料中存在一个细微的分歧：OpenCode 的前言说“do NOT use the skill tool…”（假设存在 `skill` 工具），而 pi 只是说“do not try to load using-superpowers again”。如果你的宿主环境没有技能工具，请使用 pi 的措辞，而不是 OpenCode 的。

将结果作为**用户角色消息注入，而非系统消息**——系统消息在每轮重复时会膨胀 token (#750)，且多个系统消息会使某些模型崩溃 (#894)。你必须复制三件事：

- **去重保护。** 生命周期回调可能反复触发（OpenCode 的 transform 在*每个* agent 步骤运行；pi 的 `context` 每轮触发）。在注入之前，检查引导标记是否已存在，如果存在则跳过。（参考资料选择了不同的标记——pi 使用自定义字符串，OpenCode 使用 `EXTREMELY_IMPORTANT` 标签；匹配标签更稳健，因为它不需要宿主环境特定的常量。）在模块级别缓存引导内容，这样你就不必在每次调用时重新读取和解析 `SKILL.md` (#1202)。
- **压缩（Compaction）。** 如果宿主环境压缩/总结历史，请在之后重新注入。pi 在 `session_start` 和 `session_compact` 上设置 `injectBootstrap` 标志，在 `agent_end` 上清除它，并在任何前导压缩摘要消息*之后*插入消息。OpenCode 依赖其每步重新注入加上去重保护。
- **消息对象格式是特定于宿主环境的——发现你的，不要照搬字面量。** 两个参考资料使用了*不兼容*的格式：pi 构建 `{ role, content: [{ type, text }], timestamp }`；OpenCode 操作 `message.info.role` 和 `message.parts[]`。从其 API 中找到你的宿主环境的消息格式；逐字复制参考资料的對象字面量将会静默失败。

**Shape C — 将你扩展的上下文文件指向引导；无需组装。** 没有注入器，所以你*不*剥离 frontmatter 或构建包裹的字符串。你扩展提供的上下文文件（由清单声明——*不是*用户自己的全局文件）引入两样东西：`using-superpowers` 技能和宿主环境的工具映射参考。`GEMINI.md` 通过两个 `@`-include 做到这一点（`@./skills/using-superpowers/SKILL.md` 和 `@./skills/using-superpowers/references/<harness>-tools.md`）；宿主环境原始加载它们，包括 frontmatter，且 `SKILL.md` 内部已经携带了自己的 `<EXTREMELY-IMPORTANT>` 块。如果你的宿主环境没有 include 语法，请将内容内联到指令文件中。Gemini **不提供**“已加载，请勿重新调用”的前言——对于 `@`-include 宿主环境，内容是活动的指令集，而不是模型会重新加载的技能。如果你发现你的宿主环境确实尝试重新调用，请将该注释作为字面行添加到指令文件中（你没有代码以其他方式添加它）。

### 步骤 4 — 编写工具映射

将动作词汇转换为宿主环境的真实工具。涵盖以下每一个动作（仅省略真正不适用的）：

- 读取文件
- 创建 / 编辑 / 删除文件（一个 `apply_patch` 风格的工具，还是单独的 write/edit？）
- 运行 shell 命令
- 搜索文件内容 / 按名称查找文件（grep, glob）
- 获取 URL / 网络搜索
- **分发子代理**，包括如何传递代理类型——以及启用它所需的任何配置标志
- **创建 / 更新待办事项**（将旧的 `TodoWrite` 引用视为此动作）
- **调用技能**——见步骤 5

**从宿主环境获取真实的工具名称；绝不发明它们。** 如果文档没有列出它们，权威来源是宿主环境本身：在实时会话中，要求模型“列出你可以调用的每个工具的精确机器名称，每行一个”，并使用它报告的内容。

**宿主环境如何找到 `skills/` 目录本身也是特定于宿主环境的**——确认它，不要假设。可能性包括：清单 `skills` 路径字段（Codex 的 `"skills": "./skills/"`）；宿主环境自动扫描的*并列* `skills/`（其中路径字段被**忽略**——一个真实的宿主环境只扫描位于 `plugin.json` 旁边的 `skills/`）；API/注册调用（OpenCode, pi）；或者你准备一个安装目录，将清单与**指向仓库 `skills/` 的符号链接**配对，并将安装程序指向该暂存目录（验证安装程序是否*解引用*符号链接并复制真实文件——在依赖之前用 `agy plugin validate`/`install` 或等效命令确认）。`skills` 路径字段*不是*可移植的。

映射的位置取决于形态：

- **Shape A:** 将其放在 `skills/using-superpowers/references/<harness>-tools.md` 中。Agent 从引导中访问它——`SKILL.md` 的“Platform Adaptation”部分链接了每个宿主环境的参考文件。（Shape A 宿主环境没有指令文件；映射*不*内联在钩子输出中。）
- **Shape B:** 映射通常内联在你注入的引导字符串中（见 `superpowers.js` 中的 `toolMapping` 常量）。pi 将其保存在*两个*地方——`piToolMapping()` 内联**以及** `references/pi-tools.md`。如果你在两个地方维护它，请同时更新，否则移植只做了一半。
- **Shape C:** 将其放在 `references/<harness>-tools.md` 中并将其拉入始终加载的指令文件（例如 `GEMINI.md` `@`-includes `gemini-tools.md`）。

你还可以在 `SKILL.md` 的“Platform Adaptation”部分添加一行指向你的宿主环境的指针，以便阅读引导的 Agent 知道其映射在哪里。这是移植可以对 `SKILL.md` 进行的唯一编辑——而且仅因为该部分是指针列表，不是行为塑造内容。这不违反“不要编辑技能主体”规则（第 1 部分）；不要触碰任何技能中的其他内容。（该列表是便利指针，不是详尽的注册表——并非每个宿主环境都被列出。）

### 步骤 5 — 处理没有原生技能工具的宿主环境

`using-superpowers/SKILL.md` 告诉模型*永远不要使用文件工具手动读取技能文件——始终使用你平台的技能加载机制。* 重点是“不要绕过机制”，而不是“永远不要使用文件读取”。什么算作“你平台的机制”取决于宿主环境——对于没有技能工具的宿主环境，记录在案的机制*就是*读取 `SKILL.md`。因此，在那里读取它是遵守规则而非破坏规则。区分三种情况：

1. **原生 `Skill` 风格工具**（Claude Code, Copilot CLI, Gemini 的 `activate_skill`）：将映射指向该工具。

2. **原生技能\*发现\*但没有 `Skill` 工具**（pi, Antigravity）：宿主环境可以找到并列出技能，但模型无法调用工具来加载一个。将技能安装在宿主环境扫描的位置（pi 通过 `resources_discover` → `skillPaths` 注册；OpenCode 通过其 `config` 钩子；`agy plugin install` 复制它们），并告诉模型在技能适用时通过**使用文件读取工具读取其 `SKILL.md`** 来加载技能——这是此处认可的机制，正如 `references/pi-tools.md` 所述。

   **对于引导本身，优先使用声明的上下文文件（第 6 部分）。** 如果宿主环境有 `contextFileName` 风格的清单字段——如 Antigravity 那样——通过安装程序发布生成的上下文文件：它保证被加载，并同时携带 `using-superpowers` 内容和工具映射。这是强有力的首选路径。

   **后备方案——展示的技能索引。** 如果没有上下文文件字段，但宿主环境在会话启动时展示每个已安装技能的名称 + 描述，你*既不*需要构建索引*也不*需要运行时列表指令——宿主环境就是索引，`using-superpowers` 自身展示的描述可以触发模型加载它。这比声明的上下文文件更弱；与上下文文件 / 钩子 / 进程内注入器相比，它**不能**给你两样东西——需兼顾两者：

   - **它引导\*触发\*，而非\*工具映射*。** 注入器在每次会话中将 `<harness>-tools.md` 与 `using-superpowers` 一起前置。这里没有什么注入映射——模型只看到技能*描述*，并且在需要工具名称时必须*读取*你的 `references/<harness>-tools.md`。它之所以有效，是因为技能命名动作（模型在行动时读取映射），但它比注入更弱。确保映射可从模型加载的内容中访问——例如从 `SKILL.md` 的 Platform Adaptation 部分链接并与技能一起安装——而不仅仅是放在仓库中。
   - **没有结构性保证触发会发生。** 没有 `<EXTREMELY_IMPORTANT>` 包装器，没有去重，压缩后没有重新注入——触发取决于模型选择对其在索引中看到的描述采取行动。这正是为什么验收测试在此处是强制性的：它是*唯一的*保证，所以在用户实际使用的模型上运行它，而不仅仅是在最强的模型上。

3. **完全没有技能系统：** 没有什么可注册的，*唯一的*机制是模型按需读取 `SKILL.md`。但模型无法读取它找不到的东西：`using-superpowers/SKILL.md` **不会**枚举可用技能，因此仅靠它自己，模型不知道存在哪些技能或其触发条件。你必须提供发现路径。两个选项，它们的持久性不同：(a) 生成一个技能索引（每个 `skills/*/SKILL.md` 的 `name` + `description` frontmatter）并将其放置在 `<EXTREMELY_IMPORTANT>` 包装器*内部*与工具映射一起（上述 Shape B 方案），使其受到去重保护——但构建时索引会随着技能添加而过时；或 (b) 指示模型在运行时列出 `skills/*/SKILL.md` 并读取其 frontmatter 以找到匹配项——较慢但永不过时。除非你有理由不这样做，否则优先选择 (b)。如果没有这两者，无技能系统的移植会加载引导但静默地从不触发任何其他技能。

在情况 2 和 3 中，在你的工具映射中明确说明读取 `SKILL.md` 是认可的路径，这样模型就不会认为它违反了“永远不要读取技能文件”规则。不要在没有技能系统的宿主环境中寻找 `skillPaths` 风格的注册 API——情况 3 没有。

### 步骤 6 — 添加测试

匹配现有的每个宿主环境的测试风格：

- **Shape A:** 断言钩子的 stdout 具有你的宿主环境消耗的确切 JSON 格式，并且包含引导。见 `tests/hooks/test-session-start.sh`，它验证每个宿主环境的输出格式。
- **Shape B:** 一个单元测试，伪造宿主环境的插件 API 并断言生命周期处理程序已注册、引导注入一次、去重保护有效，以及（如果相关）压缩重新注入有效。见 `tests/pi/test-pi-extension.mjs`。按照 `tests/opencode/` 的风格添加隔离安装集成检查。
- 如果引导被缓存，测试当文件丢失时缓存的行为（见 OpenCode 缓存测试）。

这些自动化测试覆盖了连接；步骤 7 中的实时 tmux 运行才是证明集成实际触发技能的关键。

### 步骤 7 — 本地安装，然后驱动实时实例进行验证

你无法通过阅读代码来确认移植有效。你必须运行加载了你进行中移植的宿主环境并观察真实会话——这也是你生成 PR 所需对话记录的方式。

**本地安装。** 将宿主环境的*本地*实例指向你的工作树，而不是已发布的构建：

- **Shape A / C:** 从本仓库的本地路径安装插件/扩展（或将其目录符号链接到宿主环境查找的任何位置）。在其文档中找到宿主环境的“从本地目录 / git checkout 安装”路径。
- **Shape B:** 注册本地模块——例如 `opencode.json` 的 `plugin` 条目指向本地路径，或 pi 从仓库解析 `package.json` 字段。

每次更改后重新安装并重启宿主环境，因为引导在启动时加载。

**使用 tmux 驱动它。** 大多数宿主环境是无法通过管道 stdin 驱动的交互式 REPL/TUI，因此在分离的 tmux 会话中运行宿主环境并使用 `send-keys` / `capture-pane` 控制它。宿主环境可能宣传非交互式的“运行一个提示”模式（例如 `opencode run "..."`）——尝试它进行快速冒烟检查，但**不要依赖它**：这些模式经常不稳定、受认证限制或受信任限制（一个真实宿主环境的 `--print` 模式每次都挂起并超时且无输出）。准备好通过 tmux 做*所有事情*，包括冒烟检查。

**先清除障碍，否则 tmux 会静默停滞。** 许多宿主环境在首次运行时会在入门引导、“你信任此文件夹吗？”提示、沙盒模式或权限门上阻塞——分离的 tmux 会话只会静静地坐在那里等待而没有错误。在运行之前，预先信任你的临时目录（在宿主环境的设置/配置中），或者准备好通过 `send-keys` 回答这些提示，并在你的第一个 `sleep` 中考虑宿主环境的启动时间。

```bash
# 1. 在一个一次性项目目录中分离启动宿主环境
mkdir -p /tmp/port-smoke
tmux new-session -d -s port-test -c /tmp/port-smoke '<harness-launch-command>'

# 2. 让它初始化——真实的 TUI 比你想象的要长（带模型握手需 10 秒以上）；调整此值。
#    然后在输入提示之前捕获并清除任何阻塞模态框：首次运行入门和“信任此文件夹？”是模态的，
#    在此期间发送的按键会选择菜单项而不是输入你的提示。
sleep 12
tmux capture-pane -t port-test -p          # 入门 / 信任提示？先通过 send-keys 回答
# (例如 tmux send-keys -t port-test Enter   # 接受信任提示——在假设之前先检查)

# 3. 冒烟检查：模型知道它有 superpowers 吗？
#    将文本和 Enter 作为单独的 send-keys 发送，中间有间隔——
#    一起发送在某些 TUI 上会有竞争（Enter 在文本落地之前到达）。
tmux send-keys -t port-test 'What are your superpowers?'; sleep 0.4; tmux send-keys -t port-test Enter
sleep 5
tmux capture-pane -t port-test -p          # 回复应显示它知道自己的技能

# 4. 验收测试：确切提示（注意转义的撇号），新会话
tmux send-keys -t port-test 'Let'\''s make a react todo list'; sleep 0.4; tmux send-keys -t port-test Enter
# 轮询直到回合结束——每隔几秒重新捕获，不要只捕获一次
sleep 8
tmux capture-pane -t port-test -p          # 通过 = brainstorming 在任何代码之前触发

# 5. 保存对话记录以供 PR 使用，然后清理
tmux capture-pane -t port-test -p > /tmp/port-smoke/transcript.txt
tmux kill-session -t port-test
```

此处容易踩坑的 tmux 注意事项：启动后等待再进行第一次捕获；将提示文本和 `Enter` 作为*单独*的 `send-keys` 调用发送，中间有短暂的 `sleep`（一起发送在某些 TUI 上会有竞争），且 `Enter` 是键名而非 `\n`；Agent 的回合需要时间，所以**循环轮询 `capture-pane`** 而不是只捕获一次；`capture-pane` 仅显示可见窗格，因此对于长对话，使用宿主环境自身的对话记录/日志文件作为真实记录；完成后务必 `kill-session`。

如果冒烟检查显示模型*不知道*它有 superpowers，说明引导未加载——在费心验收测试之前先修复这个问题。

------

## 第 6 部分 — 分发与发布

在本仓库中工作的集成直到真实用户可以安装它才算可用。分发因宿主环境生态系统而异——找到你的：

| 渠道                          | 示例                        | 你需要做的                                                   |
| :---------------------------- | :-------------------------- | :----------------------------------------------------------- |
| 原生插件市场                  | Claude Code                 | 在 `.claude-plugin/marketplace.json` 中注册；用户 `/plugin install`。外部 `superpowers-marketplace` 仓库是用户安装的唯一事实来源——见 `CLAUDE.md` 中的发布步骤。 |
| 外部市场分支，通过脚本同步    | Codex                       | `scripts/sync-to-codex-plugin.sh` 将跟踪的插件文件 rsync 到一个单独的分支仓库并打开 PR。阅读其包含/排除列表，以便你发布正确的树（它特意丢弃仓库内部目录和其他宿主环境的点目录）。 |
| Git-URL 扩展安装              | Gemini, Kimi Code, OpenCode | 用户从 git URL 安装（`gemini extensions install …`；Kimi Code `/plugins install …`；`opencode.json` `plugin` 数组条目）。记录确切的命令。 |
| 包清单字段                    | pi                          | 通过仓库根目录 `package.json` 中的字段声明；用户通过宿主环境的包命令安装。 |
| 本地安装程序 (plugin install) | Antigravity (`agy`)         | 一个小的 `install.sh`，针对包含清单、技能和生成的 `contextFileName` 上下文文件（引导）的暂存目录运行宿主环境自身的 `agy plugin install`。一切都通过安装机制到达——*不是*通过编辑用户配置（见下文）。 |

然后：

- **插件安装程序可能会静默剥离未声明的文件——所以使引导成为安装程序识别的文件，绝不是用户配置编辑。** `plugin install` 通常只复制它知道的组件（skills/agents/commands/mcp/hooks/context）并丢弃其他任何东西，因此清单未声明的上下文文件会从安装中消失。修复方法**不是**放弃并写入用户配置（**规则 2**）——而是将引导声明为已识别的组件。按升级顺序：
  - **发布清单声明的上下文文件。** 如果宿主环境有 `contextFileName` 风格的字段（它每次会话加载的扩展声明文件），那是强力的干净引导：声明它，安装程序会保留它*并且*宿主环境会加载它。在安装时从实时的 `using-superpowers/SKILL.md` + 工具映射（包裹在 `<EXTREMELY_IMPORTANT>` 中）生成它，这样已安装的引导永远不会漂移。这就是 `.antigravity-plugin/install.sh` 所做的——`agy plugin install` 报告 `✔ context : ANTIGRAVITY.md`，且干净会话读取 `using-superpowers` 的 SKILL.md，加载 `brainstorming`，并在任何代码之前进入头脑风暴流程。**用标记验证**安装程序保留了文件且宿主环境加载了它：一位移植者错误地认为它不能，因为他们发布了文件*但没有*声明 `contextFileName`，结果它被作为未识别项剥离了。
  - **否则依赖已安装的 `using-superpowers` 技能本身。** 如果宿主环境在会话启动时展示每个已安装技能的名称 + 描述，`using-superpowers` 的描述（“Use when starting any conversation…”）可以提示模型加载它——安装技能*就是*引导。更弱（没有保证的包装器；它携带触发但不携带工具映射——见步骤 5），所以在可用时优先使用声明的上下文文件。
  - 如果两者都不行，则该宿主环境目前无法被干净地支持——**如实说明**并提出问题，而不是手动编辑用户配置。
- **编写安装文档。** 一个 `docs/README.<harness>.md` 和/或 `.<harness>/INSTALL.md`（见 `docs/README.opencode.md` 和 `.opencode/INSTALL.md`），加上顶层 `README.md` 中的安装部分。唯一支持的安装操作是**运行宿主环境自身的安装命令**（`agy plugin install`, `gemini extensions install`, `/plugin install` 等）。手动复制技能文件和编辑用户的全局/个人配置都是*禁区*（规则 2 / PR 规则）。如果宿主环境根本没有安装命令——其唯一接口是用户拥有的配置文件——那么它违反了“通过安装机制交付”规则，你应该提出这个问题，而不是发布一个编辑用户文件的安装程序。
- **注册版本。** 如果你的宿主环境引入了一个*新的*带版本清单，将其路径和版本字段添加到 `.version-bump.json`，以便 `scripts/bump-version.sh` 保持同步（阅读该文件以查看当前跟踪的内容）。未在那里注册的新清单将发布过时的版本。如果你的宿主环境反而依赖于已跟踪的文件——pi 在仓库根目录 `package.json` 中声明自己，该文件已被列出——则无需添加新内容。
- **如果没有现有渠道适合，你正在建立一个新的。** 四行中可能没有一行匹配你的宿主环境。如果它需要 Codex 风格的外部分支同步，`scripts/sync-to-codex-plugin.sh` 是要克隆的模板（注意其锚定的包含/排除列表及其 PR 自动化）。每当你添加一个新的每个宿主环境目录时，将其添加到*其他*宿主环境的同步排除列表中（例如 `sync-to-codex-plugin.sh` 中的 EXCLUDES 列表），以免你的点目录泄漏到它们的分发中。

------

## 第 7 部分 — 跨平台 / Windows

仅与 shell-hook 形态相关。`hooks/run-hook.cmd` 是多语言的：单个文件既是有效的 Windows 批处理脚本又是 Unix shell 脚本。

在 Windows 上，`cmd.exe` 运行批处理部分，定位 `bash`（Git for Windows，然后是 PATH 上的 `bash`）并运行命名的钩子脚本；如果找不到 bash，它会干净退出，宿主环境仍然工作，只是没有注入。

在 Unix 上，前导 `:` 使批处理块成为空操作，shell 直接运行脚本。

这强制执行了两条规则，你必须遵守：

- **钩子脚本是无扩展名的**（`session-start`，而非 `session-start.sh`）。Claude Code 的 Windows 处理会给任何包含 `.sh` 的命令加上 `bash` 前缀，这将导致双重调用。命名你的钩子脚本时不带扩展名。
- 不要编写钩子脚本的特定操作系统变体。一个无扩展名的 bash 脚本加上多语言包装器涵盖了所有三个平台。

`hooks/run-hook.cmd` 本身是权威实现——阅读它。有关调度器模式的背景和原理，请参阅 `docs/windows/polyglot-hooks.md`。

------

## 第 8 部分 — 提交 PR

- 目标分支为 **`dev`**。每个 PR 一个宿主环境。
- 填写 PR 模板的 **“New harness support”** 部分，并粘贴完整的验收测试对话记录（“Let's make a react todo list” 会话显示 `brainstorming` 自动触发）。没有此证明的 PR 将被关闭。
- Superpowers 是零依赖插件。不要添加第三方运行时依赖。添加新宿主环境是贡献者规则允许的唯一例外，即便如此，也要将其限制在集成严格需要的范围内——编译时消除的纯类型导入是可以的；运行时包不行。
- 不要触碰技能主体（第 1 部分）。如果你发现自己编辑了 `SKILL.md` 以使移植工作，修复应在你的工具映射中。

------

## 附录 A — 参考集成（当前）

将此作为实时索引；如有疑问，阅读文件而非此表。

| 宿主环境    | 入口点                                                       | 引导机制                                                     | 工具映射                                                 | 测试                                       | 分发                                         |
| :---------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :------------------------------------------------------- | :----------------------------------------- | :------------------------------------------- |
| Claude Code | `.claude-plugin/plugin.json` + `hooks/hooks.json`            | shell hook → `hooks/session-start` (`hookSpecificOutput.additionalContext`) | 原生 `Skill` 工具；无需适配器文件                        | `tests/hooks/`                             | 市场                                         |
| Codex       | `.codex-plugin/plugin.json` (声明空 `hooks`)                 | 原生技能发现 (无会话启动钩子)                                | `references/codex-tools.md`                              | `tests/codex/`, `tests/codex-plugin-sync/` | 分支同步 (`scripts/sync-to-codex-plugin.sh`) |
| Cursor      | `.cursor-plugin/plugin.json` + `hooks/hooks-cursor.json`     | shell hook → `hooks/session-start` (`additional_context`)    | 无需 (兼容 Claude Code 的工具接口)                       | `tests/hooks/`                             | 手写                                         |
| Copilot CLI | (共享 Claude Code 钩子路径; `COPILOT_CLI` 环境变量)          | shell hook → `hooks/session-start` (`additionalContext`)     | 无需 (兼容 Claude Code 的工具接口)                       | `tests/hooks/`                             | —                                            |
| Gemini CLI  | `gemini-extension.json` + `GEMINI.md`                        | 指令文件 `@`-includes 引导 + 映射                            | `references/gemini-tools.md`                             | —                                          | `gemini extensions install`                  |
| Kimi Code   | `.kimi-plugin/plugin.json`                                   | 清单 `sessionStart.skill` 加载 `using-superpowers`           | 清单中内联 `skillInstructions`                           | `tests/kimi/`                              | 市场或 `/plugins install` GitHub URL         |
| OpenCode    | `.opencode/plugins/superpowers.js` (通过根 `package.json` `main` 声明) | 进程内: `config` 钩子注册技能目录; `experimental.chat.messages.transform` 注入用户消息 | 内联在 `superpowers.js` 中                               | `tests/opencode/`                          | `opencode.json` 插件 git URL                 |
| pi          | `.pi/extensions/superpowers.ts`                              | 进程内: `resources_discover` 注册技能; `context` 事件注入用户消息; 生命周期标志 + 压缩感知 | `piToolMapping()` 内联 **以及** `references/pi-tools.md` | `tests/pi/`                                | 仓库根 `package.json` 字段                   |

## 附录 B — 移植者曾遇到的陷阱

- **选择加入不是移植。** 如果你的人类伙伴必须在每次会话中做任何事情才能获得 Superpowers，验收测试就会失败。重读第 2 部分。
- **错误的 JSON 字段 → 静默失败或双重注入。** 仅限 Shape A。确认确切的字段/嵌套；Claude Code 读取两个字段且不去重。
- **钩子配置模式因宿主环境而异。** Shape A。Cursor 的 `hooks-cursor.json` 与 Claude Code 的完全不同（`version`、小写 `sessionStart`、相对命令、无 `matcher`/`type`/`async`）。匹配最接近的现有文件。
- **插件根环境变量因宿主环境而异。** Shape A。钩子命令使用 `${CLAUDE_PLUGIN_ROOT}` (Claude) 或相对路径 (Cursor)。使用你的宿主环境导出的内容；脚本本身会重新推导根目录。
- **系统消息注入。** Shape B 故意注入*用户*消息 (#750, #894)。不要将其“修复”为系统消息。
- **每步 vs 每轮回调。** OpenCode 每步触发（每次调用去重保护）；pi 每轮触发（生命周期标志 + `agent_end` 重置）。将一个宿主环境的去重策略复制到另一个的回调频率上会破坏注入。
- **消息对象格式是特定于宿主环境的。** Shape B。pi 和 OpenCode 使用不兼容的格式；发现你的，不要复制参考资料的对象字面量。
- **寻找不存在的技能注册 API。** 没有技能系统（不仅仅是没有 `Skill` 工具）的宿主环境没有什么可注册的——模型按需读取 `SKILL.md`。不要假设存在 `skillPaths` 等价物。
- **映射在两个地方。** 对于进程内插件，映射可能同时存在于内联和 `references/` 文件中 (pi)。两者都要更新。
- **“永远不要读取技能文件”这一行。** 它的意思是“不要绕过你平台的技能加载机制”，而不是“永远不要使用文件读取”。在没有技能工具的宿主环境上，该机制*就是*读取 `SKILL.md`——在映射中明确说明这一点（第 5 部分）。
- **Windows 上的 `.sh`。** 保持钩子脚本无扩展名（第 7 部分）。
- **未注册的版本。** 未添加到 `.version-bump.json` 的新清单会发布过时版本（第 6 部分）。
- **编辑技能以适应宿主环境。** 绝不。修复应在工具映射中。