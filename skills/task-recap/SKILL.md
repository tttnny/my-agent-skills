---
name: task-recap
description: Summarize the previous AI task's complete resolution process, including trial-and-error, into a concise structured summary. Triggered when the user explicitly asks to recap/recap/summarize/review the previous task, e.g. "总结一下刚才的任务", "复盘", "刚才做了什么", "recap last task", "写个总结".
agent_created: true
---

# Task Recap

## What This Skill Does

When the user asks for a recap (e.g. "recap what you just did", "总结一下"), produce a concise but complete summary of everything the AI did in response to the **user's previous message**. That's one "task" — a full round-trip of user request → AI work → completion.

The summary is for the user's own reference. Keep it brief but cover everything, including mistakes and dead ends.

## Process

### Step 1: Define the task boundary

Identify the user's **previous message** (the one before the recap request). The task is everything the AI did in response to that message, up to the current point.

**边缘情况 — 无实质工作:** 如果上一条消息是闲聊或无实质内容（"好的"、"知道了"等），不要立即输出复盘。向前追溯对话，找到最近一条触发了实质工作（工具调用、文件创建、分析等）的消息，复盘那个任务。如果整个对话都没有实质工作，简短回复："本轮没有实质工作，无需复盘。"

### Step 2: Extract the narrative

Walk through the task chronologically, identifying only externally visible events — what was done, not what was thought:

- What the user asked for
- Every tool-related action taken, in order
- Which actions failed: error messages, dead ends, tool failures — and how you pivoted
- What finally worked
- What files were created or modified

Do NOT include internal reasoning, decision rationale, or thought process. The recap is about what happened, not why you thought it was a good idea.

**信息来源（按优先级）：**
1. **对话上下文** — 主要来源，优先使用当前上下文中已有的内容。
2. **每日日志与任务列表** — 补充来源。自行判断：如果对话上下文不完整（工具输出被截断、细节折叠、后台任务结果缺失），读取每日日志（`{workspace}/.workbuddy/memory/YYYY-MM-DD.md`）和/或使用 TaskList 补充。如果上下文完整，则不需要。
3. **不完整标注** — 如果使用所有来源后叙述仍有缺口，在输出中显式注明："注意：受上下文限制，部分细节可能不完整。"

### Step 3: Output the summary

Output directly in the conversation. Do not save to a file. Use this structure:

```markdown
# 任务复盘

## 目标
{一句话：用户要做什么。}

## 执行过程

{用 bullet 列表，按时间顺序写。每条写一个关键节点——做了什么、结果如何、失败了怎么转向的。}

示例格式：
- 先做了 X → 顺利完成
- ~~尝试了 Y → 遇到错误 Z，放弃~~
- 转向方案 A → 跑通
- 最后通过 B 和 C 完成全部工作

## 产出物
{创建或修改的文件，一行一个，附简短说明。}

## 结果
{最终状态。能用吗？有什么遗留问题？后续步骤？}
```

**Language:** The output MUST be in Chinese. All section headings, labels, and body text.

### Guidelines

- **bullet 列表** — 按时间顺序用 `-` 逐条列出关键节点
- **视觉区分** — 成功的步骤不做任何标记。失败的步骤（死胡同、被放弃的方案）用 `~~划掉~~` 包裹，一眼就能分辨哪些是最终路径
- **只看外部事件** — 只记录工具调用、错误、转向、产出物。不写思考链、推理过程、决策理由
- **试错要诚实但精简** — 所有失败尝试都记录，但每个只用一两句话
- **不要编造** — 如果没有试错或失败，就直接写「过程顺利，没有遇到阻碍」
- **严格单轮** — 只覆盖上一条用户消息到当前消息之间的工作（除非触发"无实质工作"的追溯逻辑）
