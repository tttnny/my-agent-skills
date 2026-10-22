# my-agent-skills

个人维护的 Agent Skills 合集，遵循 [Agent Skills 规范](https://agentskills.io)，可用于 CodeBuddy、Claude Code、Codex、Cursor 等 75+ 种 AI 编码代理。

## 技能列表

| 技能 | 目录 | 说明 |
|---|---|---|
| `clipping-to-note` | `skills/clipping-to-note` | 把本地网页剪藏提炼成排版好的中文笔记，原件不动，成品另存 |
| `goal-to-project` | `skills/goal-to-project-master` | 反向工程拆解职业/学习目标，输出项目式学习规划 `.md` |

## 安装

依赖 [skills CLI](https://github.com/vercel-labs/skills)，只需 `npx`，无需预先全局安装。要求 **Node.js ≥ 22.20**。

在任意目录执行（会交互式选择要安装的技能和目标代理）：

```bash
npx skills add tttnny/my-agent-skills -g
```

`-g` 表示装到用户目录（全局可用，如 `~/.codebuddy/skills/`）；去掉 `-g` 则只装到当前项目的 `.codebuddy/skills/`。

### 指定技能与代理（一条命令，无交互）

```bash
# 只装某一个技能，到 CodeBuddy 全局
npx skills add tttnny/my-agent-skills --skill clipping-to-note -a codebuddy -g -y

# 一次装多个
npx skills add tttnny/my-agent-skills --skill clipping-to-note --skill goal-to-project -a codebuddy -g -y

# 全部技能，装到所有已检测到的代理
npx skills add tttnny/my-agent-skills --skill '*' --agent '*' -g -y
```

常用参数：

| 参数 | 作用 |
|---|---|
| `-g, --global` | 装到 `~/<agent>/skills/`，跨项目可用；缺省为项目级 `./<agent>/skills/` |
| `-a, --agent <agents...>` | 指定目标代理，如 `codebuddy`、`claude-code`、`codex`、`cursor`；`'*'` 表示全部 |
| `-s, --skill <skills...>` | 按 SKILL.md 里的 `name` 指定技能；`'*'` 表示仓库内全部 |
| `-l, --list` | 只列出仓库里有哪些技能，不安装 |
| `--copy` | 复制文件而非软链接到代理目录 |
| `-y, --yes` | 跳过确认提示，适合脚本/CI |

### 先看看有哪些技能

```bash
npx skills add tttnny/my-agent-skills --list
```

### 手动安装

不想用 CLI 的话，把对应目录复制到代理的 skills 目录即可：

```bash
git clone https://github.com/tttnny/my-agent-skills.git
cp -R my-agent-skills/skills/clipping-to-note ~/.codebuddy/skills/
```

目录名可以随意，代理读取的是 `SKILL.md` 里的 `name` 字段。

## 查看、更新与卸载

```bash
npx skills list              # 已安装的技能
npx skills check             # 检查可更新项
npx skills update            # 更新
npx skills remove goal-to-project # 卸载
```

## 说明

- 各技能的 `SKILL.md` 里写了触发条件，装好后由代理按对话内容自动调用；标了 `disable-model-invocation: true` 的技能需要手动指定。
