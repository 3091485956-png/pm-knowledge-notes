今天我们把 **Agent 架构**系统讲透。

你前面已经学过 Agent 是什么、Agent 和 Workflow / Skill / MCP 的区别，也刚刚学了 Agent Badcase。今天不要再停留在“Agent = LLM + Tools + Memory”这种定义层面，而是要真正搞清楚：

> **一个 Agent 到底由哪些组件组成？这些组件之间怎么传递信息？Agent 为什么能自主执行任务？为什么会出错？产品经理应该怎么设计它？**

你以后面试遇到“设计一个 Agent”“讲讲 Agent 架构”“Agent 和 Workflow 有什么区别”，基本都可以从这套框架展开。

---

# 一、先建立一张完整的 Agent 架构图

先给你一张最重要的图。

```text
                         ┌─────────────────────┐
                         │       User          │
                         │ 目标 / 指令 / 约束    │
                         └──────────┬──────────┘
                                    ↓
                         ┌─────────────────────┐
                         │   Agent Interface   │
                         │ 输入理解 / 输出交互   │
                         └──────────┬──────────┘
                                    ↓
┌─────────────────────────────────────────────────────────┐
│                    Agent Runtime / Harness              │
│                                                         │
│  ┌──────────────┐       ┌──────────────────────────┐   │
│  │    Context   │──────→│          LLM             │   │
│  │  上下文管理   │       │  Reason / Plan / Decide  │   │
│  └──────────────┘       └────────────┬─────────────┘   │
│           ↑                          ↓                 │
│           │                 ┌─────────────────┐       │
│  ┌────────┴────────┐        │  Action / Tool  │       │
│  │      Memory     │        │    Selection    │       │
│  │ 短期/长期记忆    │        └────────┬────────┘       │
│  └─────────────────┘                 ↓                 │
│                              ┌─────────────────┐        │
│                              │      Tools      │        │
│                              │ API / DB / MCP  │        │
│                              │ Search / Code   │        │
│                              └────────┬────────┘        │
│                                       ↓                 │
│                              External Environment       │
│                                       ↓                 │
│                              Tool Observation            │
│                                       │                 │
│                                       └──────→ Context   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Guardrails / Policy / Permission / Validation   │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                                    ↓
                           ┌─────────────────┐
                           │   Final Result  │
                           └─────────────────┘
```

这里最关键的是：

**Agent 不是一个模型。**

它更像一个运行系统。

LLM 是其中最核心的“决策与推理引擎”，但 Agent 能不能真正可靠地工作，取决于：

> LLM + Context + Memory + Tools + State + Runtime + Guardrails + Evaluation

这也是为什么现在越来越多 Agent 工程问题，不再只是 Prompt Engineering，而是进入 **Agent Engineering / Harness Engineering**。

---

# 二、第一层：LLM——Agent 的“大脑”

最核心的组件当然还是 LLM。

但是在 Agent 中，LLM 和普通聊天机器人里的 LLM，使用方式不完全一样。

普通 Chat：

```text
User
 ↓
LLM
 ↓
Answer
```

Agent：

```text
User
 ↓
LLM
 ↓
决定下一步做什么
 ↓
Tool
 ↓
获得结果
 ↓
LLM
 ↓
决定下一步
 ↓
Tool
 ↓
……
 ↓
完成任务
```

所以 Agent 中的 LLM 不只是：

> “回答问题”

而是承担：

- 理解目标
- 分析当前状态
- 制定计划
- 选择工具
- 解释工具结果
- 决定下一步
- 判断是否需要继续
- 最终生成结果

这也是 Agent 和传统 LLM Chat 最大的区别之一。

---

# 三、第二层：Context——Agent 的“工作记忆”

这个非常重要。

LLM 本身并不会天然拥有一个完整的“当前任务状态”。

它每次做决策时，都需要系统把相关信息放进 Context。

例如用户说：

> 帮我策划一个 500 人的企业年会，预算 30 万，地点在北京，风格年轻化。

Agent 当前的 Context 可能包括：

```text
【用户目标】
策划企业年会

【硬约束】
人数：500
预算：≤30万
地点：北京

【偏好】
年轻化

【当前任务】
寻找适合500人的北京场地

【已经完成】
需求分析
预算分析

【工具结果】
场地A：容量600，租金12万
场地B：容量400，租金8万
```

LLM 根据这些信息判断：

> 场地 B 容量不足，所以应该排除；场地 A 可以进入下一步。

所以你可以把 Context 理解成：

**Agent 在当前这一刻“知道什么”。**

---

# 四、第三层：Memory——Agent 的“长期记忆”

Memory 和 Context 很容易混。

一个非常简单的判断方式：

> Context 解决“我现在需要知道什么”。  
> Memory 解决“我以前知道什么，而且未来可能还需要”。

比如用户长期告诉 Agent：

> 我比较喜欢简洁、年轻化的活动方案。

这个信息可以进入长期 Memory。

下一次用户说：

> 再帮我策划一个年会。

Agent 可以从 Memory 中取出：

```text
用户偏好：
- 简洁
- 年轻化
```

然后用于新的任务。

但这里就会出现我们刚才讲过的 **Memory Pollution**。

比如：

```text
这次活动预算：30万
```

这是一个 Task Context。

不能因为 Agent 记忆系统设计不好，就把它保存成：

```text
用户所有活动预算都不能超过30万
```

所以真正成熟的 Agent Memory，不是简单地：

```text
存东西 → 下次取出来
```

而需要：

```text
Memory
 ↓
分类
 ↓
重要性判断
 ↓
有效期
 ↓
来源
 ↓
适用范围
 ↓
冲突处理
 ↓
检索
```

这就是为什么 Memory 本身已经成为一个独立的 Agent Engineering 问题。

---

# 五、第四层：Tools——Agent 的“手”

这是 Agent 最关键的能力之一。

LLM 本身不能：

- 查数据库
- 发邮件
- 创建会议
- 搜索互联网
- 操作浏览器
- 执行 Python
- 修改 Excel
- 查询 CRM
- 调用企业 API

所以需要 Tools。

例如：

```text
Agent
 ├── Search Tool
 ├── Calculator
 ├── Database
 ├── Browser
 ├── Email
 ├── Calendar
 └── Code Interpreter
```

Agent 的核心变化就是：

**LLM 从“只能说话”变成“可以采取行动”。**

---

# 六、Tool Calling 到底发生了什么？

这个你一定要理解。

假设用户说：

> 帮我查一下明天北京天气，如果下雨就提醒我带伞。

Agent 并不是让 LLM 自己真的去查天气。

而是：

```text
User
 ↓
LLM
 ↓
判断：需要天气工具
 ↓
Tool Call
 ↓
Weather API
 ↓
返回：
明天北京有降雨
 ↓
LLM
 ↓
判断：
满足“下雨”
 ↓
执行提醒工具
 ↓
完成
```

因此：

**Tool Calling = LLM 决定调用什么工具 + 提供什么参数。**

工具真正执行事情。

这就是为什么工具设计非常重要。

---

# 七、MCP 在这里处于什么位置？

你之前学过 MCP，这里可以把它放回 Agent 架构里。

MCP 本质上解决的是：

> **Agent 如何以标准化方式发现和使用外部工具与数据资源。**

所以可以简单理解：

```text
Agent
  ↓
MCP
  ↓
┌──────────────┐
│ Tool         │
│ Resource     │
│ Prompt       │
└──────────────┘
```

例如：

```text
Agent
 ↓
MCP Server
 ↓
企业数据库
```

或者：

```text
Agent
 ↓
MCP Server
 ↓
GitHub
```

但注意：

**MCP ≠ Agent。**

MCP 是连接机制 / 协议。

Agent 是负责理解目标、决策和执行任务的系统。

---

# 八、第五层：Planning——Agent 的“规划器”

这是 Agent 与普通 LLM 最大的能力差异之一。

假设用户说：

> 帮我做一份北京三天旅游计划，预算 3000 元。

一个普通 LLM 可能直接生成：

```text
Day 1
Day 2
Day 3
```

但 Agent 可以先进行任务拆解：

```text
Goal：
北京三日游

↓
Step 1：确认预算
Step 2：查交通
Step 3：查酒店
Step 4：查景点
Step 5：计算整体成本
Step 6：检查是否超预算
Step 7：生成最终方案
```

这就是 Planning。

---

# 九、Planning 有哪些不同方式？

这个非常适合面试。

### 1. Fixed Planning

提前规定：

```text
Step1
 ↓
Step2
 ↓
Step3
 ↓
Step4
```

本质上接近 Workflow。

优点：

- 稳定
- 可预测
- 容易测试

缺点：

- 灵活性低。

---

### 2. Dynamic Planning

让 LLM 根据当前任务动态决定下一步。

```text
Goal
 ↓
LLM
 ↓
Plan
 ↓
执行
 ↓
观察
 ↓
重新规划
 ↓
执行
```

灵活性更高，但可靠性更差。

---

### 3. Replanning

这是现在 Agent 很重要的能力。

例如：

```text
原计划：

找场地
 ↓
找供应商
 ↓
生成方案

执行找场地
 ↓
发现预算不足
 ↓
重新规划

找更便宜场地
 ↓
调整供应商
 ↓
重新计算预算
 ↓
生成方案
```

这就是：

> **Plan → Act → Observe → Replan**

而不是：

> Plan → 一条路走到底。

---

# 十、Agent 的核心运行循环：ReAct

你一定要掌握这个概念。

经典 Agent Loop 可以抽象成：

```text
Reason
 ↓
Act
 ↓
Observe
 ↓
Reason
 ↓
Act
 ↓
Observe
 ↓
……
```

也就是：

**思考 → 行动 → 观察 → 再思考。**

例如：

```text
用户：
帮我找北京适合500人开年会的场地。

        ↓

Agent：
需要查询场地信息。

        ↓

Action：
调用搜索工具。

        ↓

Observation：
找到3个场地。

        ↓

Agent：
A容量不足，B价格过高，C符合要求。

        ↓

Action：
进一步查询C的交通和设备。

        ↓

Observation：
C满足要求。

        ↓

Agent：
生成最终方案。
```

这就是 Agent 的基本闭环。

---

# 十一、为什么 Agent 能“自主”？

这里有一个非常重要的误区。

很多人认为：

> Agent 自主 = 模型自己想。

其实不是。

Agent 的自主性来自：

**系统允许 LLM 在多个行动之间进行选择。**

例如：

```text
当前状态
   ↓
LLM
   ↓
┌──────────────┐
│ Search       │
│ Database     │
│ Calculator   │
│ Browser      │
│ Ask User     │
│ Finish       │
└──────────────┘
```

LLM 选择：

> 下一步做什么？

这就是 Agent 的“自主决策”。

---

# 十二、所以 Agent 的核心其实是 State

这是我特别希望你理解的一个概念。

很多人学习 Agent 时，会一直关注：

> Prompt 怎么写？

> Tool 怎么接？

但真正复杂的 Agent，核心问题之一其实是：

**State Management。**

因为 Agent 执行几十步之后，必须知道：

```text
我是谁？
我要完成什么？
现在做到哪一步？
已经做了什么？
接下来要做什么？
哪些信息已经确认？
哪些信息还不确定？
哪些操作已经执行？
哪些操作失败？
```

所以可以把 Agent 看成：

```text
Goal
 ↓
State
 ↓
Decision
 ↓
Action
 ↓
Observation
 ↓
State Update
 ↓
Decision
 ↓
……
```

这才是真正的 Agent Loop。

---

# 十三、State 到底保存什么？

可以把 State 拆成：

```text
Agent State
│
├── Goal
│   └── 最终目标
│
├── Constraints
│   └── 预算、时间、权限等
│
├── Plan
│   └── 当前计划
│
├── Progress
│   └── 已完成 / 未完成
│
├── Working Memory
│   └── 当前任务需要的信息
│
├── Tool History
│   └── 调用了哪些工具
│
├── Tool Results
│   └── 得到了什么结果
│
├── Errors
│   └── 哪些步骤失败
│
└── User Preferences
    └── 用户偏好
```

如果这些东西管理不好，就会出现你刚才问的那些 Badcase：

- 忘记目标
- 重复调用工具
- 重复执行操作
- 任务提前结束
- 无限循环
- 记忆污染

所以：

> **Agent 可靠性很大程度上就是 State 管理能力。**

---

# 十四、第六层：Runtime——Agent 的“操作系统”

这个概念你之前学过 Harness，但现在可以把它放进整体架构里理解。

Runtime / Harness 负责：

> **让 Agent 能够真正运行起来。**

比如：

```text
Agent Runtime
│
├── LLM 调用
├── Tool 调用
├── Context 管理
├── Memory 管理
├── State 管理
├── 权限管理
├── 重试
├── 超时
├── 并发
├── 日志
├── Trace
├── 错误恢复
└── Termination
```

所以：

**LLM 是大脑，Tools 是手脚，Memory 是长期记忆，Context 是工作记忆，State 是任务状态，而 Runtime/Harness 是整个身体的运行系统。**

这个比喻不是严格的技术定义，但非常适合你面试时快速建立整体认知。

---

# 十五、第七层：Guardrails——给 Agent “装刹车”

这是 Agent 产品非常重要的一层。

因为 Agent 能行动之后，风险就比 Chat 高很多。

普通聊天：

```text
模型说错一句话
```

可能只是体验不好。

Agent：

```text
模型判断错误
 ↓
调用支付工具
 ↓
真的付款
```

性质完全不同。

所以 Agent 需要 Guardrails。

可以分成四层：

### 输入 Guardrail

检查：

```text
用户输入
 ↓
是否恶意？
是否越权？
是否包含敏感信息？
是否存在 Prompt Injection？
```

### Action Guardrail

检查：

```text
Agent准备调用工具
 ↓
是否允许？
参数是否正确？
是否需要人工确认？
```

### Output Guardrail

检查：

```text
Agent准备输出
 ↓
格式正确吗？
有没有敏感信息？
有没有违反业务规则？
```

### Runtime Guardrail

检查：

```text
运行时间
调用次数
Token
成本
循环次数
权限
```

---

# 十六、Human-in-the-loop 在哪里？

这个也非常重要。

并不是所有 Agent 都应该完全自主。

可以设置：

```text
低风险操作
Agent 自动执行

中风险操作
Agent 执行 + 自动校验

高风险操作
Agent 提议
      ↓
Human Confirm
      ↓
Agent 执行
```

比如你的 Vibe Helper_skill 里面设计的 **4 个强制人工确认节点**，其实就是很典型的 Agent Architecture 设计。

它解决的是：

> **把方向性决策交给人，把机械执行交给 Agent。**

这是一个非常好的产品设计思想。

---

# 十七、为什么不能让 Agent 什么都自己做？

因为 Agent 的自主性和可靠性之间存在天然张力。

可以粗略理解成：

```text
自主性 ↑
    │
    │       Agent
    │      ／
    │    ／
    │  ／
    │／
    └────────────→ 可靠性
```

不是说自主性越高越好。

比如：

**自动生成活动方案**

可以高度自主。

**自动修改企业数据库**

需要更严格控制。

**自动转账**

必须有极强的权限控制和人工确认。

所以产品经理要设计的是：

> **哪些决策交给 Agent，哪些决策必须由人掌握。**

---

# 十八、Agent 架构中最容易混淆的几个概念

这个你以后面试很容易被问。

### Agent vs Workflow

Workflow：

```text
A → B → C → D
```

流程提前定义。

Agent：

```text
A
 ↓
根据状态决定 B/C/D
 ↓
执行
 ↓
重新决定
```

所以：

**Workflow 强调确定性，Agent 强调动态决策。**

实际产品里经常是：

```text
Workflow
  +
Agent
```

而不是二选一。

---

### Agent vs Skill

Skill 更像：

> Agent 可以使用的一项能力。

例如：

```text
Agent
 ├── 搜索 Skill
 ├── 写作 Skill
 ├── 数据分析 Skill
 └── PPT Skill
```

Skill 可以封装一组具体操作。

---

### Agent vs MCP

MCP 是标准化连接外部工具和数据的协议。

```text
Agent
 ↓
MCP
 ↓
Tools / Resources
```

所以：

**Agent 是行动者，MCP 是连接机制。**

---

### Agent vs Multi-Agent

一个 Agent：

```text
一个大脑
 ↓
多个工具
```

Multi-Agent：

```text
Orchestrator
   ↓
┌───────┬───────┬───────┐
Agent A Agent B Agent C
```

例如：

```text
研究 Agent
   ↓
写作 Agent
   ↓
审核 Agent
```

但是 Multi-Agent 不一定更好。

增加 Agent 就意味着：

- 更多通信
- 更多状态同步
- 更多错误传播
- 更高 Token 成本
- 更复杂的 Debug

所以：

> **能用一个 Agent 解决的问题，不要为了“高级”强行 Multi-Agent。**

---

# 十九、Agent Architecture 可以进一步分成三种

这是面试比较有用的一层。

## 1. Single Agent

```text
User
 ↓
Agent
 ├── Tool A
 ├── Tool B
 ├── Tool C
 └── Memory
```

适合：

- 任务相对通用
- 工具数量有限
- 决策链不复杂

优点是简单。

---

## 2. Agent + Workflow

```text
Workflow
 ├── Step 1
 ├── Agent
 ├── Step 3
 ├── Agent
 └── Step 5
```

这是实际产品里非常常见的一种。

因为：

> **确定性的部分交给 Workflow，不确定性的部分交给 Agent。**

这其实是一个非常重要的架构思想。

---

## 3. Multi-Agent

```text
                    Orchestrator
                         ↓
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
 Research Agent     Analysis Agent    Execution Agent
       ↓                 ↓                 ↓
       └─────────────────┼─────────────────┘
                         ↓
                    Final Agent
```

适合：

- 任务复杂
- 专业领域不同
- 不同 Agent 需要不同工具 / Prompt / 权限
- 可以并行执行

但架构复杂度也明显提高。

---

# 二十、Agent 的“可靠性”应该怎么设计？

现在把你上一节学的 Badcase 和今天的架构串起来。

一个成熟 Agent 至少应该有：

```text
                 Agent
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
      Input      State       Tools
      Guard      Manager     Guard
        │          │          │
        └──────────┼──────────┘
                   ↓
                 LLM
                   ↓
              Decision
                   ↓
               Action
                   ↓
             Observation
                   ↓
             Validation
                   ↓
        ┌──────────┴──────────┐
        ↓                     ↓
      Continue               Finish
```

其中最重要的是三个东西：

**State + Guardrails + Validation。**

因为单纯让 LLM 更聪明，并不能解决所有 Agent 问题。

---

# 二十一、再往深一点：Agent 为什么会失败？

你现在可以用一个非常好的思维模型：

> **Agent Failure = Reasoning Failure + Context Failure + Tool Failure + State Failure + Environment Failure**

### Reasoning Failure

模型想错了。

例如：

> 明明应该搜索，却决定直接回答。

### Context Failure

模型没有拿到正确的信息。

例如：

> 用户之前说预算 30 万，但当前 Context 里没有了。

### Tool Failure

工具本身或者工具调用出问题。

例如：

> API 返回 500。

### State Failure

Agent 对任务状态理解错误。

例如：

> 已经发送过邮件，却认为没有发送。

### Environment Failure

外部世界出问题。

例如：

> 网络断了、权限没了、数据库挂了。

这五类归因思路，你以后分析 Agent Badcase 会非常好用。

---

# 二十二、一个完整例子：招聘 Agent

现在把整个架构串起来。

用户：

> 帮我从这 1000 份简历里筛选出最适合 AI 产品经理岗位的 Top 20。

Agent 收到目标。

### Step 1：Goal Understanding

```text
目标：
筛选1000份简历

输出：
Top 20

岗位：
AI产品经理
```

### Step 2：获取岗位知识

```text
RAG / Knowledge Base
        ↓
岗位能力模型
        ↓
产品能力
AI技术理解
项目经验
沟通能力
……
```

### Step 3：Planning

```text
读取简历
 ↓
结构化提取
 ↓
岗位匹配
 ↓
评分
 ↓
排序
 ↓
生成Top20
 ↓
人工审核
```

### Step 4：Tools

Agent 使用：

```text
Resume Parser
Database
Embedding / Retrieval
Scoring Tool
```

### Step 5：State

保存：

```text
已处理：600 / 1000

成功提取：580

异常简历：20

当前阶段：
候选人评分
```

### Step 6：Loop

```text
读取简历
 ↓
分析
 ↓
更新状态
 ↓
继续
```

### Step 7：Validation

检查：

```text
是否真的处理1000份？
是否存在重复候选人？
是否有评分缺失？
Top20是否符合岗位硬性要求？
```

### Step 8：Human-in-the-loop

HR 最终确认：

```text
Top20候选人
 ↓
人工审核
 ↓
进入面试流程
```

这就是真正意义上的 Agent 系统，而不是：

> “把 1000 份简历丢给一个大模型，让它帮我选20个人。”

---

# 二十三、如果你现在去设计 Agent，我建议按照这个顺序

以后面试遇到：

> “请你设计一个 XX Agent。”

你可以直接按照：

**① Goal**

用户最终想完成什么？

↓

**② Environment**

Agent 能访问什么数据和系统？

↓

**③ Tools**

Agent 能做什么？

↓

**④ Context**

完成当前任务需要什么信息？

↓

**⑤ Memory**

哪些信息需要跨任务保存？

↓

**⑥ Planning**

任务如何拆解？哪些步骤固定？哪些需要动态决策？

↓

**⑦ State**

如何记录进度、结果和错误？

↓

**⑧ Action**

Agent 如何执行动作？

↓

**⑨ Validation**

怎么知道它真的做对了？

↓

**⑩ Guardrails**

哪些事情不能让 Agent 自己做？

↓

**⑪ Human-in-the-loop**

什么时候必须让人确认？

↓

**⑫ Evaluation**

最后怎么衡量 Agent 好不好？

这个框架比单纯背“Agent = LLM + Tools + Memory”高级很多。

---

# 二十四、最后给你一张真正应该记进笔记的总图

```text
                         USER
                           │
                           ↓
                  ┌─────────────────┐
                  │ Goal / Intent   │
                  └────────┬────────┘
                           ↓
┌─────────────────────────────────────────────────┐
│                 AGENT RUNTIME                   │
│                                                 │
│  ┌─────────────┐      ┌─────────────────────┐  │
│  │   Context   │◄────►│        State        │  │
│  │ 当前需要的信息 │      │ 目标/计划/进度/错误 │  │
│  └──────┬──────┘      └──────────┬──────────┘  │
│         │                         │             │
│         └──────────┬──────────────┘             │
│                    ↓                            │
│              ┌─────────────┐                    │
│              │     LLM     │                    │
│              │ Reason/Plan │                    │
│              │   Decide    │                    │
│              └──────┬──────┘                    │
│                     ↓                           │
│              ┌─────────────┐                    │
│              │   Action    │                    │
│              │ Tool Select │                    │
│              └──────┬──────┘                    │
│                     ↓                           │
│          ┌──────────────────────┐                │
│          │        TOOLS         │                │
│          │ API / DB / Search    │                │
│          │ Code / Browser / MCP │                │
│          └──────────┬───────────┘                │
│                     ↓                           │
│                Environment                     │
│                     ↓                           │
│                Observation                     │
│                     │                           │
│                     └──────────→ State Update   │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │ Guardrail / Permission / Validation       │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │ Memory：长期偏好 / 历史事实 / 跨任务信息 │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
                           │
                           ↓
                    FINAL RESULT
```

你现在应该把 Agent 架构理解成一句话：

> **Agent = 一个围绕 LLM 构建的、能够感知当前状态、进行决策、调用外部工具、根据环境反馈持续行动，并受到状态管理、权限、安全和验证机制约束的任务执行系统。**

而从 AI 产品经理角度，真正要设计的不是“让 Agent 更聪明”，而是：

**让它知道该做什么 → 给它正确的信息 → 给它合适的工具 → 让它知道现在做到哪了 → 限制它不能做什么 → 验证它到底有没有做对 → 出错后能够恢复。**

这几件事串起来，你对 Agent 的理解就从“会调用工具的大模型”进入到真正的 **Agent System Architecture** 了。
