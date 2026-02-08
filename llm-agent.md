---
title: Agent极简理解 
date: 2025-08-23 17:43:20
tags: [agent,llm,ai]
categories: llm 
---

本文旨在说明agent如何工作（各个组件LLM, Orchestrator/dify, tool等都做了什么），其中有一些设计上的不合理，例如多轮简单的请求其实可以合并、让LLM做汇总计算容易出错，暂忽略。

<!-- more -->

# 假设

现有tool:

- MCP tool-A: 查询公司每个月的支出列表。只返回一个列表，表项是expenseID（没有金额，没有任何细节）；并且每次只能查一个月，input是月份(Jan, Feb, ...)；
- HTTP API tool-B: 根据 expenseID 查询一个支出的金额；

假设基于dify开发一个 “支出统计助手” agent，用于统计年初（1月1日）至今的所有支出总额。

# 开发agent

## **步骤1**：定义 SystemPrompt（由你编写）

你在 Dify 中为 Agent 设置如下 SystemPrompt：

```
你是一个公司的财务助手，负责计算并报告从当年1月1日至今的所有支出总额。

你有两个工具可用：
- **get_monthly_expense_ids(month: str)**: 
    输入：月份英文缩写（如 "Jan", "Feb", ..., "Dec"）
    输出：该月所有支出的 expenseID 列表（仅ID，无金额）
- **get_expense_amount(expense_id: str)**:
    输入：一个 expenseID
    输出：该支出的金额（数字，单位：元）

工作流程：
1. 确定当前年份和当前月份。
2. 对从 "Jan" 到当前月份的每一个月，调用 get_monthly_expense_ids 获取该月所有 expenseID。
3. 对每个 expenseID，调用 get_expense_amount 获取金额。
4. 将所有金额相加，得到总支出。
5. 输出极简报告，格式：{"total_expenses": <总金额>}（保留两位小数）

注意：
- 必须逐月查询，不能跳过。
- 必须查询每一个 expenseID 的金额。
- 最终只输出 JSON，不要解释。
```

## **步骤2**：在 Dify 中注册两个工具

 - Tool-A：配置为 MCP 工具或 HTTP 工具，名称 get_monthly_expense_ids，输入参数 month
 - Tool-B：配置为 HTTP 工具，名称 get_expense_amount，输入参数 expense_id


# 使用agent

以3月份用户请求为例：

- 用户在2025-03-15日输入UserPrompt: “请报告今年至今的总支出。”
- Dify 做：
    - 构建完整prompt:
```
[SYSTEM]
你是一个公司的财务助手……（完整 SystemPrompt）

[USER]
请报告今年至今的总支出。
```
    - 发送给 LLM（如 GPT-4）
- LLM 做：
    - 理解任务：需查 Jan, Feb, Mar 三个月
    - 决定并行调用 Tool-A 三次
    - 输出：

```
{
  "tool_calls": [
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Jan"}},
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Feb"}},
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Mar"}}
  ]
}
```

- Dify 做：
    - 解析 LLM 输出，识别出 3 个工具调用
    - 并行调用外部服务：

```
调 Tool-A(month="Jan") → 返回 ["E001", "E002"]
调 Tool-A(month="Feb") → 返回 ["E003"]
调 Tool-A(month="Mar") → 返回 ["E004", "E005"]
```

- Dify 做：
    - 将工具返回结果追加到对话历史：
```
[SYSTEM]
你是一个公司的财务助手……（完整 SystemPrompt）

[USER]
请报告今年至今的总支出。

[ASSISTANT]
{
  "tool_calls": [
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Jan"}},
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Feb"}},
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Mar"}}
  ]
}

[TOOL_RESPONSE for get_monthly_expense_ids (Jan)]
["E001", "E002"]

[TOOL_RESPONSE for get_monthly_expense_ids (Feb)]
["E003"]

[TOOL_RESPONSE for get_monthly_expense_ids (Mar)]
["E004", "E005"]
```
    - 再次发送完整上下文给 LLM

- LLM 做：
    - 提取所有 ID：E001, E002, E003, E004, E005
    - 决定调用 Tool-B 五次
    - 输出：

```
{
  "tool_calls": [
    {"name": "get_expense_amount", "arguments": {"expense_id": "E001"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E002"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E003"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E004"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E005"}}
  ]
}
```

- Dify 做：并行调用 Tool-B 五次：
    - E001 → 1000 元
    - E002 → 2000 元
    - E003 → 1500 元
    - E004 → 800 元
    - E005 → 700 元

- Dify 做：
    - 将所有金额结果追加到上下文

```
[SYSTEM]
你是一个公司的财务助手……（完整 SystemPrompt）

[USER]
请报告今年至今的总支出。

[ASSISTANT]
{
  "tool_calls": [
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Jan"}},
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Feb"}},
    {"name": "get_monthly_expense_ids", "arguments": {"month": "Mar"}}
  ]
}

[TOOL_RESPONSE for get_monthly_expense_ids (Jan)]
["E001", "E002"]

[TOOL_RESPONSE for get_monthly_expense_ids (Feb)]
["E003"]

[TOOL_RESPONSE for get_monthly_expense_ids (Mar)]
["E004", "E005"]

[ASSISTANT]
{
  "tool_calls": [
    {"name": "get_expense_amount", "arguments": {"expense_id": "E001"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E002"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E003"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E004"}},
    {"name": "get_expense_amount", "arguments": {"expense_id": "E005"}}
  ]
}

[TOOL_RESPONSE for get_expense_amount (E001)]
1000.00

[TOOL_RESPONSE for get_expense_amount (E002)]
2000.00

[TOOL_RESPONSE for get_expense_amount (E003)]
1500.00

[TOOL_RESPONSE for get_expense_amount (E004)]
800.00

[TOOL_RESPONSE for get_expense_amount (E005)]
700.00
```

    - 再次发送完整上下文给 LLM


- LLM 做：
    - 计算总和：1000 + 2000 + 1500 + 800 + 700 = 6000.00 (注意：这里设计不合理，旨在说明各个组件的分工)
    - 严格按指令输出极简 JSON

```
{"total_expenses": 6000.00}
```

- Dify 做：将此结果作为最终回复返回给用户“3月总支出：6000.00 元”



# 总结

- LLM是聪明又没有记忆的大脑，它根本不知道agent的存在，只是 `output = LLM (prompt)`；
- Agent是外部Orchestrator(如dify)的概念；

另外：

- SystemPrompt：尽可能详细、明确、结构化；为 LLM 提供清晰的“操作手册”，像写 API 文档或员工培训手册——精确、无歧义、可执行。应包含：
    - 角色定义（你是谁？）
    - 可用工具及用法（你能做什么？参数格式？）
    - 工作流程（分步骤怎么做？）
    - 输出格式约束（必须返回 JSON / 不要解释 / 保留两位小数等）
    - 边界与限制（不能做什么？什么情况下跳过？）

- UserPrompt：尽量自然、简洁、友好，像同事在 Slack 上发消息——自然、高效、人性化。
    - 使用日常语言（如“帮我查一下…”、“今年花了多少钱？”）
    - 不需要知道内部工具或技术细节
    - 可以模糊、简略（LLM + System Prompt 会补全意图）
