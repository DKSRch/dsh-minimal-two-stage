# DSH Minimal Two-Stage（极简二阶模式）

## 这是什么？

- 一款把 `dsh-web-ui` 中「梁神模式」的两阶段锚定引导，移植到「极简模式（Git Bash）」基础上的 DeepSeek Harness agent preset。
- **阶段一（锚定）**：模型只看到 `bash` + `str_replace_editor` 两个工具、一行 persona、无运行时上下文，且只放行用户的直接消息。
- **阶段二（晋升）**：只开放完整工具目录，**不注入任何新提示词**——persona 保持同一行（`complete: true` + `includeRuntimeContext: false`），系统提示词晋升前后逐字节一致。
- 与姊妹 preset `dsh-minimal-plus` 的关键区别：minimal-plus 晋升后恢复完整提示词组装并切换 Code Mode（PTC）；本 preset 晋升只换工具列表（native 呈现），并且**不设阶段一输出 token 上限**。

---

## 文件结构

- `agent.cordis.yml` — preset 核心 composition（全部 plugin 行）
- `tool-bootstrap.mjs` — 两阶段锚定引导器（本地扩展，含 `workspaceLine: false` 选项）
- `gitbash-executor.mjs` — Windows Git Bash 子进程 shell 执行器
- `preset.yml` — 元数据（name/description/order）
- `NOTICE` — 版权声明
- `LICENSE` — MIT

## 上游来源

- `agent.cordis.yml` 改编自 DeepSeek Harness 内置 Minimal 与 Standard preset（MIT），组合结构参考 dsh-liangshen 与 dsh-minimal-plus
- `tool-bootstrap.mjs` 来自 [xiaobright/dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard)（MIT），并包含 dsh-liangshen 的两阶段隔离扩展
- `gitbash-executor.mjs` 来自 [dsh-gitbash-preset](https://github.com/liceses/dsh-gitbash-preset)（MIT），每次命令以 `<git bash> -c <command>` 的即弃子进程执行，非持久化 PTY shell
- 完整上游归属链见 [NOTICE](./NOTICE)

本 preset 以 MIT 协议分发 — 详见 [LICENSE](./LICENSE)

## 行为契约（必读）

按 composition 头部的披露要求，以下行为与官方 shipped preset 不同，请知悉：

- **阶段一消息过滤**：永久丢弃所有非用户注入消息（goal 续轮、任务看板定时注入、时间上下文、子代理报告），只保留 `kind: 'user'` 的直接消息。一轮只含注入消息时会空转完成。
- **无环境上下文**：两个阶段模型都看不到 cwd / 沙箱 / 审批上下文，需自行发现工作区（如 `pwd`）。
- **bash 二进制门控**：gitbash 执行器按 danger-full-access 策略做二值拒绝——非完全访问下每条命令直接抛错，而非在受限 token 下运行（MSYS 无法在 Windows 沙箱中初始化）。完全访问 + 禁用审批时 bash 完全不受约束。
- **子代理**：`tool-subagent-fork` 为 one-shot（无 `send_message` 续聊），忠实于 liangshen 源预设；shipped preset 为 continuable。
- **锚定门**：`we` 探针仅英文生效，配合四步兜底；首轮无工具调用会在响应后自动晋升。
- **无阶段一输出上限**：刻意省略 `bootstrapMaxTokens`，阶段一按适配器默认上限运行，而非 liangshen / minimal-plus 的 1024 锚定预算。

## 前置条件

- `deepseek-harness` 本体
- Windows 上需安装 Git for Windows（提供 `bash.exe`）。`gitbash-executor.mjs` 自动按 `GIT_BASH` 环境变量 → Program Files / Program Files (x86) / LOCALAPPDATA 标准安装根 → PATH 的顺序查找，无需手动配置；若 Git 装在自定义位置，请设置 `GIT_BASH` 环境变量。
- `bash` 工具需要会话处于 danger-full-access（完全访问）沙箱策略，否则每条命令都会被二进制拒绝。

## 安装

将此目录复制或克隆到本地 agent-presets 根目录，然后重启 `dsh-web` 进程使 roster 重新扫描：

```sh
# Linux / macOS 默认根
git clone https://github.com/DKSRch/dsh-minimal-two-stage.git \
  "$HOME/.dsh/.agent-presets/dsh-minimal-two-stage"

# Windows (PowerShell) 默认根
git clone https://github.com/DKSRch/dsh-minimal-two-stage.git `
  "$env:USERPROFILE\.dsh\.agent-presets\dsh-minimal-two-stage"
```

重启后，preset 会出现在新建会话的预设选择器中。

## 验证挂载

在任意 session 中执行：

```js
await ctx.agentPresets.standingKeyFor('dsh-minimal-two-stage')
```

此调用会端到端组合 preset 的 plugin 子树，并拒绝以下情况：

- package 无法解析的行
- config 无效的行
- 从未激活的行
- 发布到根 realm 的 service

返回正常即表示 composition 挂载成功；然后启动一个真实 session 以确认：

1. 第一阶段工具目录仅为 `bash` + `str_replace_editor`，系统提示词只有一行 persona；
2. 晋升后完整工具目录以 native 方式呈现（无 Code Mode `run_code` PTC），系统提示词保持逐字节一致。

---

---

# DSH Minimal Two-Stage (English)

## What is this?

- A DeepSeek Harness agent preset that ports the two-phase anchor bootstrap of "Liangshen Mode (梁神模式)" from `dsh-web-ui` onto the "Minimal (Git Bash)" base.
- **Phase 1 (anchoring)**: the model sees only the `bash` + `str_replace_editor` tools, a one-line persona, no runtime contexts, and only direct user messages.
- **Phase 2 (promotion)**: only the full tool catalog opens — **no new prompt text is injected**. The persona keeps the exact one line (`complete: true` + `includeRuntimeContext: false`), so the system prompt stays byte-identical across promotion.
- Key difference from its sibling preset `dsh-minimal-plus`: minimal-plus restores the full prompt assembly and switches to Code Mode (PTC) after promotion; this preset only swaps the tool list (native presentation) and sets **no phase-1 output token cap**.

---

## Files

- `agent.cordis.yml` — preset composition (all plugin rows)
- `tool-bootstrap.mjs` — two-phase anchor bootstrap (local extension, adds the `workspaceLine: false` option)
- `gitbash-executor.mjs` — Windows Git Bash subprocess shell executor
- `preset.yml` — metadata (name/description/order)
- `NOTICE` — copyright notice
- `LICENSE` — MIT

## Upstream

- `agent.cordis.yml` adapted from the DeepSeek Harness builtin `minimal` and `standard` presets (MIT), with the composition structure referencing dsh-liangshen and dsh-minimal-plus
- `tool-bootstrap.mjs` from [xiaobright/dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard) (MIT), with the dsh-liangshen two-phase isolation extension
- `gitbash-executor.mjs` from [dsh-gitbash-preset](https://github.com/liceses/dsh-gitbash-preset) (MIT); each command runs as a throwaway `<git bash> -c <command>` subprocess, not a persistent PTY shell
- See [NOTICE](./NOTICE) for the full attribution chain

This preset is distributed under the MIT License — see [LICENSE](./LICENSE).

## Behavior contracts (must read)

As required by the composition header, the following behaviors differ from the official shipped presets:

- **Phase-1 message filter** permanently drops all non-user injected messages (goal continuations, task-board timer injections, time-context, subagent reports); only `kind: 'user'` messages pass. A turn carrying only injected messages completes empty.
- **No environment context**: the model never sees cwd / sandbox / approval context in either phase and must discover the workspace itself (e.g. `pwd`).
- **Binary bash gate**: the gitbash executor gates commands on the danger-full-access policy as a binary refusal — outside full access every command throws instead of running under a restricted token (MSYS cannot initialize in the Windows sandbox). Under full access with approvals disabled, bash is fully unconstrained.
- **Subagents**: `tool-subagent-fork` is one-shot (no `send_message` continuation), faithful to the liangshen source preset; shipped presets use continuable.
- **Anchor gate**: the `we` probe is English-only with a four-step fallback; a tool-less first response promotes automatically.
- **No phase-1 output cap**: `bootstrapMaxTokens` is deliberately omitted, so phase 1 runs at the adapter-default cap rather than the 1024 anchor budget used by liangshen / minimal-plus.

## Prerequisites

- A working `deepseek-harness` installation
- Git for Windows is required on Windows (provides `bash.exe`). `gitbash-executor.mjs` auto-discovers it in this order: the `GIT_BASH` environment variable → standard install roots (Program Files / Program Files (x86) / LOCALAPPDATA) → PATH. No manual configuration is needed; if Git lives in a custom location, set the `GIT_BASH` environment variable.
- The `bash` tool requires the session to run under the danger-full-access sandbox policy; otherwise every command is refused outright.

## Install

Copy or clone this directory into the local agent-presets root, then restart the `dsh-web` process so the roster re-scans:

```sh
# default root on Linux / macOS
git clone https://github.com/DKSRch/dsh-minimal-two-stage.git \
  "$HOME/.dsh/.agent-presets/dsh-minimal-two-stage"

# default root on Windows (PowerShell)
git clone https://github.com/DKSRch/dsh-minimal-two-stage.git `
  "$env:USERPROFILE\.dsh\.agent-presets\dsh-minimal-two-stage"
```

After restart, the preset appears in the new-session picker.

## Verify the mount

In any session, run:

```js
await ctx.agentPresets.standingKeyFor('dsh-minimal-two-stage')
```

This composes the preset's plugin subtree end-to-end and rejects:

- a row whose package does not resolve,
- a row with invalid config,
- a row that never activated,
- a service published into the root realm.

A clean return means the composition mounts; then start a real session on it to confirm:

1. the phase-1 tool catalog is `bash` + `str_replace_editor` only and the system prompt is just the one-line persona;
2. after promotion the full catalog opens with native presentation (no Code Mode `run_code` PTC) and the system prompt stays byte-identical.

---

## AI 作品声明 / AI Creation Statement

**中文：**

本仓库是 AI 辅助开发的作品。主要开发工作由 DeepSeek Harness 上的 AI 代理完成，人类协作者提供需求描述、设计方向和质量审查。

**本项目直接衍生自以下三个开源库，没有它们就没有本项目：**

- **[dsh-web-ui](https://github.com/zhu1090093659/dsh-web-ui)** — DSH Web GUI 插件全家桶，「梁神模式」两阶段锚定引导的来源。本项目的 plugin 架构、cordis 配置结构和 DSH 集成模式完全来自此库，本项目本质上是 dsh-web-ui 生态中的一个 preset 包。
- **[dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard)** — 两阶段工具引导器的实现来源，本 preset 的 `tool-bootstrap.mjs` 直接继承自此项目（含 dsh-liangshen 的阶段一隔离扩展）。
- **[dsh-gitbash-preset](https://github.com/liceses/dsh-gitbash-preset)** — Windows 上 Git Bash 子进程 shell 的实现来源，本 preset 的 bash 工具直接继承自此项目（每次命令执行 `<git bash> -c <command>`，非持久化 PTY shell）。

本项目的代码、文档和配置均由 AI 生成和整理，但所有创意决策、架构设计和质量保证均由人类协作者主导。

---

**English:**

This repository is an AI-assisted creation. The main development work was completed by AI agents running on DeepSeek Harness, with human collaborators providing requirement descriptions, design direction, and quality review.

**This project is directly derived from the following three open-source repositories. Without them, this project would not exist:**

- **[dsh-web-ui](https://github.com/zhu1090093659/dsh-web-ui)** — DSH Web GUI plugin suite and the source of the "Liangshen Mode" two-phase anchor bootstrap. The plugin architecture, cordis configuration structure, and DSH integration patterns of this project come entirely from this repository. This project is essentially a preset package within the dsh-web-ui ecosystem.
- **[dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard)** — The implementation source of the two-phase tool bootstrap. The `tool-bootstrap.mjs` of this preset is directly inherited from this project (with dsh-liangshen's phase-1 isolation extension).
- **[dsh-gitbash-preset](https://github.com/liceses/dsh-gitbash-preset)** — The implementation source of the Git Bash subprocess shell on Windows. The bash tool of this preset is directly inherited from this project (each command runs as `<git bash> -c <command>`; it is not a persistent PTY shell).

The code, documentation, and configuration of this project were all generated and organized by AI, but all creative decisions, architecture design, and quality assurance were led by human collaborators.
