# Remote Shadow Development Skill

这是一个用于“本地编辑、远端验证/运行”的 agent skill。它适合接手同时包含本地工作区、SSH 远端机器、远端影子目录、长时间任务或本地到远端同步流程的项目。

## 适用场景

- 本地改代码，远端服务器跑训练、评测、批处理或长任务。
- 需要明确区分本地源码、远端 shadow workspace、远端输出目录和日志目录。
- 需要在正式长跑任务前先做 bounded smoke test。
- 需要用可复现的命令、PID 文件和日志路径交接远端任务。
- 项目里有不能覆盖或同步的 `data/`、`outputs/`、`logs/`、checkpoint、模型缓存等目录。

## 基本用法

把本仓库作为 skill 安装到目标 agent 的 skills 目录，并在对话中触发：

```text
Use $remote-shadow-development to take over this project with a local workspace and SSH remote shadow directory.
```

触发后，agent 会优先确认本地路径、当前分支、远端 SSH 入口、远端项目路径、环境激活命令、同步方向、禁止同步目录和验收标准；然后按 `SKILL.md` 中的流程执行本地修改、远端同步、smoke test 和长任务交接。

## 适用平台

- Codex / OpenAI agent 环境：可放入 Codex skills 目录，例如 `~/.codex/skills/remote-shadow-development`。仓库中的 `agents/openai.yaml` 提供 OpenAI/Codex 展示元数据。
- Claude Code：可放入 `~/.claude/skills/remote-shadow-development`，通过 skill 名称触发。
- Gemini CLI 或其他支持 agent skills / `SKILL.md` frontmatter 的环境：复制整个目录，并按对应平台的 skill 激活方式使用。

## 文件结构

```text
remote-shadow-development/
  SKILL.md
  agents/
    openai.yaml
```

`SKILL.md` 是主说明文件；`agents/openai.yaml` 是 OpenAI/Codex 相关的界面元数据。
