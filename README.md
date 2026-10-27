# CC 点将台 · cc-dianjiangtai

**Claude Code 内部的轻量子 Agent 编排：Opus 做主控，Sonnet 做实现，一个全新 Sonnet 统一审核与测试。**

纯 Markdown：一个 Skill、两个子 Agent 定义。没有 MCP 服务、跨端调度、SDK、常驻进程或自建状态数据库。

## 它解决什么问题

把“理解需求与拆单”和“编码执行”分开；复杂任务可以并行交给多个实现者，最后只做一次整合后的独立审核与整体测试。适合日常开发、Debug、升级和重构，而不是为了小修改建立一套调度系统。

```text
Opus 主控：需求、范围、契约、拆单
    ↓
一个 Sonnet 实现者 / 多个 Sonnet 分模块实现
    ↓
指定其中一个实现者整合（多实现者时）
    ↓
全部停止写入
    ↓
一个全新 Sonnet：统一审核 + 整体测试
    ↓
简报；获准返修时交回对应实现者
```

多个实现者使用同一个 `djt-worker` 定义；“两个子 Agent 定义”不是只允许启动两个实例。小任务只开一个 worker；仅对契约明确、写入互不重叠的模块并行。

## 文件

| 文件 | 作用 |
| --- | --- |
| `.claude/skills/cc-dianjiangtai/SKILL.md` | 主控职责与完整编排规则，选择 `opus` |
| `.claude/agents/djt-worker.md` | 实现、局部测试、整合，选择 `sonnet` |
| `.claude/agents/djt-reviewer.md` | 最终独立审核及整体测试，选择 `sonnet` |

两个子 Agent 文件只集中声明模型、工具和职责，没有运行时代码。也可以将规则写进单个 Skill 并使用通用子 Agent；本仓库保留两个短定义，减少重复派单说明，让角色更明确。

## 使用：不向目标项目复制文件

先克隆本仓库，例如在 PowerShell 中：

```powershell
git clone https://github.com/xiangshazaosha/cc-dianjiangtai.git "$HOME\cc-dianjiangtai"
```

切换到你真正要开发的项目目录，启动 Claude Code：

```powershell
claude --model opus --add-dir "$HOME\cc-dianjiangtai"
```

然后输入：

```text
/cc-dianjiangtai 实现你的需求
```

额外目录用于加载本包的 Skill 与子 Agent，不会把它们复制进目标项目；同时它也会授予本次会话访问该目录的权限。工作目录仍应明确指向目标项目，而不是本仓库。

本方式需要支持从 `--add-dir` 加载 Skills 与子 Agent 的 Claude Code 版本。若当前版本不支持，可将上述 Skill 文件夹和两个 Agent 文件分别复制到用户级 `~/.claude/skills/`、`~/.claude/agents/`，再开启新会话；已有同名内容请先检查，不要直接覆盖。

## 模型路由

主控使用 `opus`，实现和审核测试使用 `sonnet`，沿用当前 Claude Code 的别名配置。Skill 不包含 API 地址、密钥或供应商切换逻辑。

作者环境的路由示例是 **Opus → Kimi K3.1 Pro，Sonnet → DeepSeek V4.1 Flash**。这只是自定义路由示例，不是官方模型名称等价关系，也不是对特定网关兼容性的保证。你可以使用实际 Claude 模型或自己的兼容路由。

启动命令同时指定 `--model opus`，让主会话后续回合保持主控模型；Skill 的模型覆盖只作用于调用回合。全局模型强制覆盖或模型不可用时，以实际环境为准，本包不擅自修改配置。

## 测试、审核与隔离

- **实现者可做局部测试；最终审核和整体测试由同一个新 reviewer 负责。** 不额外拆出一个测试 Agent。
- **默认同一工作区，不额外配置隔离。** 新上下文不等于文件系统隔离，reviewer 能运行命令，不能称为纯只读。
- 实现、整合和最终审核测试分阶段进行，避免边写边审。reviewer 不主动修实现，不修改测试源码；问题交回 worker。
- 测试可能写缓存、报告、快照或数据。必须使用测试资源；本包不提供沙箱或生产数据保护机制，也不能靠提示词保证命令无副作用。
- 用户要求“不测试”时，两种子 Agent 都跳过测试，明确标注未验证内容；代码审核仍可进行。用户要求“不审核”时不启动 reviewer，由 worker 承担允许的必要测试。
- 默认不自动返修。用户明确允许但未给次数时最多一轮；指定次数时遵守该上限。每次最终复核都用全新 reviewer。

本仓库仅提供编排指令，不保证模型一定遵守所有约束；实际执行仍受 Claude Code、工具能力、项目指令和用户权限控制。本次发布没有进行模型试跑或运行时测试。

## 来源与许可

编排思路来自同一维护者的[点将台 Codex → Claude Code 项目](https://github.com/xiangshazaosha/codex-cc-orchestrator)，本仓库面向 Claude Code 内部子 Agent 单独整理发布，不包含原 MCP 的代码或运行依赖。

本仓库采用 **[Zero-Clause BSD（0BSD）](LICENSE)**：允许商用、修改和再分发，不要求保留署名或以相同许可证公开修改。按现状提供，不作担保。此授权不改变原点将台项目或其他产品的许可；第三方模型、CLI 和服务另受其自身条款约束。

## 官方参考

- [Claude Code Skills](https://code.claude.com/docs/en/skills)：Skill 格式、模型覆盖和额外目录加载。
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)：子 Agent 定义、模型选择、工具与新实例/恢复。
- [OSI：0BSD](https://opensource.org/license/0bsd)：许可证原文与批准信息。
