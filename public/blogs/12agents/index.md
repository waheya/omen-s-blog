12 Factor Agent 分享

> Github: [https://github.com/humanlayer/12-factor-agents](https://github.com/humanlayer/12-factor-agents)


[文档目录]




## 前言
### 1.1 什么是Agent？
之前大家介绍了很多 Agent 框架，比如 Eino、CrewAI、LangChain、ADK-go 等等。但是否思考过：这些框架为什么要这么设计？它们背后的设计思想是什么？这些设计思想又是从哪里来的？

可以先从第一性原理重新思考一个根本问题：**什么是 Agent？**

**软件的演进**

**软件本质上是一个有向图（Directed Graph）**，这就是为什么我们曾用流程图来表示程序。大约 20 年前，Airflow、Prefect 等 DAG 编排器开始流行，它们在图模式基础上增加了可观测性、模块化、重试机制等工程化能力。

```
输入 → 处理 → 判断 → 输出
              ↓
           异常处理
```
而 Agent 最大的吸引力在于：可以把 DAG 扔掉了！只需给定目标和一组工具，让 LLM 实时决策执行路径。

**Agent 的本质**

**Agent = LLM in a for loop**，就是把 LLM 放在一个 for 循环里。

它的核心是一个三步循环（Agentic Loop）：LLM 决定下一步并输出结构化 JSON → 确定性代码执行工具调用 → 结果追加到上下文窗口 → 重复直到完成。

**现实的挑战**

然而，与 100 多位 SaaS 创始人交流后发现：大多数号称 "AI Agent" 的产品其实并没有那么 Agentic，它们主要是**确定性代码**，只**在关键节点使用 LLM **创造神奇体验。纯粹的 Agent 模式往往只能达到 70-80% 的质量，而这对生产环境远远不够。

所以真正好的 Agent 是**确定性代码与 LLM 智能决策的精心结合**。这正是 12-Factor Agents 要解决的问题。

### 1.2 背景
2011 年，Heroku 公司面对的是一个混乱的 Web 应用部署时代：**每个公司做法不同，本地能运行的代码到服务器上就不行，配置管理一团糟，应用扩展困难重重**。

为了解决这些问题，Heroku 提出了著名的 **12 Factor Apps** 原则，包括**一个代码库、显式声明依赖、配置存储在环境中、后端服务分离、构建/发布/运行分离**等 12 条工程实践。如今这套原则已经成为云原生应用的行业标准，被广泛采用，让应用变得更可靠、更易扩展。

作者观察到，今天的 AI Agent开发正处于类似的混乱阶段：**每个团队做法不同，Demo 能工作但生产环境跑不起来，Prompt 管理一团糟，成本控制困难，可靠性无法保证。**

这和 2011 年的 Web 应用何其相似。因此，作者借鉴 12 Factor Apps 的思路，针对 AI 应用的特点，提出了 **12 Factor Agents**。

> **一句话总结**：**12 Factor Agents 是一套构建可靠 AI Agent 的工程原则，就像建房子需要遵循设计规范一样。**
****

### 1.3 12 Factor 分类
12 个Factor分为 4 大类：

| **分类**             | **功能**                 | **具体Factor**                   | **一句话总结**                             | **关键词**         |
| -------------------- | ------------------------ | -------------------------------- | ------------------------------------------ | ------------------ |
| **输入与决策**       | 如何让 AI 理解和决策     | Factor 1: 自然语言 → 工具调用    | LLM 是翻译器，不是执行器                   | 工具、JSON、验证   |
|                      |                          | Factor 2: 拥有你的提示词         | 提示词是代码，需要版本控制                 | 可见、可测、可管理 |
|                      |                          | Factor 3: 拥有你的上下文窗口     | 主动管理，不要盲目追加                     | 成本、优化、缓存   |
| **执行与控制流**     | 如何执行和控制流程       | Factor 4: 工具就是结构化输出     | 工具调用 = JSON 生成 + 代码执行            | Schema、类型安全   |
|                      |                          | Factor 8: 拥有你的控制流         | 用代码控制流程，AI 参与决策                | Switch、循环       |
| **状态管理**         | 如何管理和保存状态       | Factor 5: 统一执行状态和业务状态 | 一切皆事件，状态从事件推导                 | Event Sourcing     |
|                      |                          | Factor 6: 启动/暂停/恢复         | 智能体是可暂停的进程                       | 生命周期、异步     |
|                      |                          | Factor 12: 无状态归约器          | 智能体 = 纯函数：state + event → new_state | Redux、可重放      |
| **人类协作与可靠性** | 如何与人类协作并保持可靠 | Factor 7: 用工具进行人机协同     | 人类是特殊的工具                           | HITL、批准         |
|                      |                          | Factor 9: 压缩错误到上下文       | 500 行堆栈 → 50 tokens 摘要                | 可恢复、可理解     |
|                      |                          | Factor 10: 小型、专注的智能体    | 3-10 步的微智能体 > 单体                   | 微服务、专注       |
|                      |                          | Factor 11: 灵活触发机制          | API、Slack、CLI、Web 都能用                | 多渠道、统一接口   |

**输入与决策**（Factor 1-3）解决的是** Agent 如何理解任务、如何做出决策的问题**。核心思想是：LLM 的职责是"翻译"自然语言为结构化指令，而不是直接执行；同时，Prompt 和上下文窗口都是需要精心管理的工程资产，而非随意堆砌的文本。

**执行与控制流**（Factor 4、8）解决的是**决策之后如何执行的问题**。工具调用本质上就是让 LLM 输出结构化 JSON，然后由确定性代码来执行。控制流应该掌握在开发者手中，而不是完全交给框架黑盒处理。

**状态管理**（Factor 5、6、12）借鉴了 Event Sourcing 和 Redux 的思想。Agent 的状态应该**从事件序列中推导出来，这样才能实现暂停、恢复、重放等能力**。把 Agent 设计成无状态的纯函数，输入当前状态和事件，输出新状态。

**人类协作与可靠性**（Factor 7、9、10、11）关注的是 **Agent 如何在真实世界中可靠运行**。人类审批被建模为一种特殊的工具调用；错误信息需要压缩后才能放入上下文；Agent 应该小而专注，而非试图做所有事情；同时要支持从多种渠道触发，而非绑定单一入口。



## 第一类：输入与决策
这一部分解决：**如何让 AI 正确理解用户意图并做出决策**

###  Factor 1: 自然语言转工具调用 （Natural Language to Tool Calls）
![](https://raw.githubusercontent.com/CaesarInCrypto/ImgStg/master/uPic/1765271832?token=A5SNFWB2H5RNVE2LYKRRNQDJG7UVO "")
##### **核心概念**
**问题**: AI理解了用户的话，但**不知道 或 不确定**该做什么；也**无法保证**AI做的是否正确；

**工具调用**: 调用**外部**函数、工具或服务来完成特定功能的能力；

**自然语言转工具调用**: 将用户的**自然语言请求**转换为**结构化的工具调用**





**常见错误： ** 让AI直接执行代码

```
# ❌ 危险的做法
user: "帮我删除所有测试数据"

# AI 生成并执行代码
ai_generated_code = """
import database
database.delete_all_where(environment='test')
"""
exec(ai_generated_code)  # 直接执行！危险！

# 问题：
# 1. AI 可能理解错误
# 2. 生成的代码可能有 bug
# 3. 没有人类审核
# 4. 无法回滚
```
**正确做法：**

```
# ✅ 安全的做法
user: "帮我删除所有测试数据"

# AI 生成工具调用
tool_call = {
    "name": "delete_test_data",
    "parameters": {
        "environment": "test",
        "confirm": False  # 需要确认
    }
}

# 系统处理
if not tool_call["parameters"]["confirm"]:
    return "请确认：你要删除测试环境的所有数据吗？(yes/no)"

# 用户确认后
tool_call["parameters"]["confirm"] = True

# 然后执行预定义的安全函数
safe_delete_test_data()
```
**实现这个Factor后，Agent系统可以：**

1. 验证：参数是否合法？是否有该环境？
2. 执行：调用实际的环境操作函数；
3. 审计：记录谁在什么时间操作了删除动作；



##### 实现示例
###### 示例 1：简单场景
```
# 定义工具的"格式"（Schema）
 deploy_tool_schema = {
     "name": "deploy",
     "description": "部署应用到指定环境",
     "parameters": {
         "version": {
             "type": "string",
             "description": "版本号，如 v2.1.0"
         },
         "environment": {
             "type": "string",
             "enum": ["staging", "production"],
             "description": "部署环境"
         }
     },
     "required": ["version", "environment"]
 }
 
 # 用户输入
 user_message = "部署 v2.1.0 到生产环境"
 
 # 调用 LLM
 response = llm.chat(
     messages=[{"role": "user", "content": user_message}],
     tools=[deploy_tool_schema]  # 告诉 LLM 可以用的工具
 )
 
 # LLM 返回
 tool_call = {
     "name": "deploy",
     "parameters": {
         "version": "v2.1.0",
         "environment": "production"
     }
 }
 
 # 验证
 assert tool_call["name"] == "deploy"
 assert tool_call["parameters"]["version"] == "v2.1.0"
 assert tool_call["parameters"]["environment"] in ["staging", "production"]
 
 # 执行
 result = execute_deployment(
     version=tool_call["parameters"]["version"],
     environment=tool_call["parameters"]["environment"]
 )
```




---

###### 示例 2：复杂场景（电商客服）
```
# 定义多个工具
 tools = [
     {
         "name": "查询订单",
         "description": "根据订单号查询订单详情",
         "parameters": {
             "订单号": {"type": "string"}
         }
     },
     {
         "name": "修改地址",
         "description": "修改订单的收货地址",
         "parameters": {
             "订单号": {"type": "string"},
             "新地址": {"type": "string"}
         }
     },
     {
         "name": "申请退款",
         "description": "为订单申请退款",
         "parameters": {
             "订单号": {"type": "string"},
             "原因": {"type": "string"}
         }
     }
 ]
 
 # 对话场景
 用户: "我的订单 12345 能改个地址吗？"
 
 # AI 理解并生成工具调用
 tool_call = {
     "name": "查询订单",  # 先查询订单状态
     "parameters": {
         "订单号": "12345"
     }
 }
 
 # 执行查询
 订单信息 = 查询订单("12345")
 # → {"订单号": "12345", "状态": "待发货", "可修改": True}
 
 # AI 继续
 if 订单信息["可修改"]:
     tool_call = {
         "name": "修改地址",
         "parameters": {
             "订单号": "12345",
             "新地址": "北京市朝阳区..."
         }
     }
```


















##### 要点总结
* LLM 的工作 = 理解意图 + 生成结构化指令
* 系统的工作 = 验证指令 + 执行代码
*  LLM 生成 JSON，不生成可执行代码
*  所有工具调用都要验证
*  实际执行由确定性代码完成
*  可以审计、回滚、测试



### Factor 2: 拥有你的提示词（ Own Your Prompts)
![](https://raw.githubusercontent.com/CaesarInCrypto/ImgStg/master/uPic/1765271833?token=A5SNFWDVFNLAFZPUE2X7TK3JG7UVS "")

**提示词**:  大模型提示词是用户输入给大模型的指令、问题或描述文本，用于引导大语言模型生成特定输出的关键信息。

**拥有你的提示词**:  将提示词视为代码的一等公民，而不是依赖框架抽象



**常见错误**: 

```
黑盒提示词的问题：

# 使用某个框架
agent = Framework.Agent(
    role="客服助手",
    goal="帮助客户",
    backstory="你是一个友好的助手..."
)

# 实际发送给 AI 的提示词是什么？无法确定
# 框架在背后做了什么转换？不清楚
# 如果 AI 表现不好，怎么调整？无从下手 or 修改框架
```


```
实际上框架可能生成这样的提示词：

System: You are a customer service agent.
Your role is: 客服助手
Your goal is: 帮助客户
Your backstory: 你是一个友好的助手...

[框架添加的一堆你不知道的指令]
- Always be polite
- Use the tools provided
- If you don't know, say so
- [还有 20 行你看不到的指令]

Now help the user with: [用户消息]
```
---

##### 实现示例
###### 示例 1: 显示提示词
```
# 完全控制提示词
system_prompt = """
你是一个部署管理助手。

你的职责：
1. 帮助用户部署应用到不同环境
2. 在部署到生产环境前，必须先确认
3. 记录所有部署操作

可用的工具：
- deploy_to_staging: 部署到测试环境
- deploy_to_production: 部署到生产环境（需要确认）
- check_status: 检查当前部署状态

重要规则：
- 部署到生产环境前，必须问用户"确认部署到生产环境吗？"
- 如果用户没有明确说"确认"或"yes"，不要部署
- 每次部署都要告诉用户部署的版本号

当前时间：{current_time}
当前用户：{user_name}
"""

# 使用
response = llm.chat(
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_message}
    ]
)
```
###### **示例 2: 可版本化的提示词** 
```
# 把提示词当作代码管理
# prompts/deploy_assistant_v1.txt

PROMPT_VERSION = "v1.0"

SYSTEM_PROMPT = """
你是一个部署管理助手 (版本 {version})

核心能力：
1. 部署应用
2. 回滚部署
3. 检查状态

安全规则：
1. 生产环境部署需要确认
2. 记录所有操作
3. 失败时提供清晰错误信息
"""

# 使用时
prompt = SYSTEM_PROMPT.format(version=PROMPT_VERSION)

# 好处：
# 1. 可以 git 追踪提示词的变化
# 2. 可以回滚到之前的版本
# 3. 可以 A/B 测试不同的提示词
# 4. 团队可以 code review 提示词
```
此外：提示词还可以：结构化、few-shot、调试；避免太模糊、太复杂;

##### 要点总结
提示词就是Agent的核心代码，必须主动拥有才能保证质量

* 不要依赖框架的黑盒提示词
* 提示词应该像代码一样被管理
* 使用 git 进行版本控制
* 可以测试和调试
* 可以 review 和改进

**衍生概念**: 

1. **提示词工程**: 提示词的开发和优化
    1. [https://www.promptingguide.ai/zh](https://www.promptingguide.ai/zh)
    2. [https://github.com/boundaryml/baml](https://github.com/boundaryml/baml)




### Factor 3: 拥有你的上下文窗口 （Own your context window）
**Token: ** 是大模型（LLM）用来表示自然语言文本的基本单位；

**上下文**:  模型在处理当前输入时能够"看到"和参考的所有信息范围；

**上下文窗口**：模型能处理的最大token数量。

**上下文的作用**: 

1. **信息串联与逻辑连贯**:  上下文让模型能够将零散的信息串联起来，形成连贯的逻辑。
2. **动态记忆与信息管理**:  上下文赋予模型“短期记忆”的能力，使其能够动态管理对话中的信息。
3. **提升理解与生成质量**:  上下文优化了模型的输入信息，使其能够更精准地理解用户意图并生成高质量的输出。

**上下文包含哪些内容**: 

1. **对话消息部分**：这包括用户发送的**所有提问、指令和补充说明**，以及**模型之前生成的所有回答内容**，这些历史对话记录构成了上下文的主体，让模型能够理解对话的来龙去脉。
2. **系统指令**：在**对话开始前或过程中设定的预设规则(Prompt)**，比如告诉模型"你是一个专业的法律顾问"或"请用简洁的语言回答"，这些指令会持续影响模型的回答风格和行为方式。
3. **用户上传的各类文件和素材**：包括PDF文档、Word文件、图片、代码文件、电子表格等，这些**外部材料**为模型提供了丰富的参考信息，使其能够基于具体内容进行分析和回答。
4. **工具/MCP调用产生的信息**：当模型使用网页搜索、代码执行或文件操作等功能时，这些**工具返回的结果**也会被纳入上下文，成为后续推理的依据。
5. **元信息和状态数据**：比如当前的日期时间、用户设置的偏好选项、对话所处的任务阶段等，这些辅助信息帮助模型更好地理解问答场景。·



##### 核心概念
**问题**:  LLM需要“记住”所有上下文，如果管理不当，会导致成本爆炸和性能下降。

**概念**:  通过合适的方式，显式地主动管理Agent的上下文； 



![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=4e1d19b59bf34de5b08d34536e294351&docGuid=RxuaqJOasFaCo3 "")
##### 实现示例
###### 示例1: 只保留相关信息
```
class SmartAgent:
    def __init__(self):
        self.full_history = []  # 完整历史（用于审计）
        self.system_prompt = "你是一个订单助手..."
    
    def build_context(self, user_message):
        """
        智能构建上下文
        """
        context = []
        
        # 1. 系统提示词（总是需要）
        context.append({
            "role": "system",
            "content": self.system_prompt
        })
        
        # 2. 最近的 3 轮对话（保持连贯性）
        recent = self.full_history[-6:]  # 3 轮 = 6 条消息
        context.extend(recent)
        
        # 3. 当前用户消息
        context.append({
            "role": "user",
            "content": user_message
        })
        
        return context
    
    def chat(self, user_message):
        # 构建优化的上下文
        context = self.build_context(user_message)
        
        # 调用 LLM
        response = llm.chat(messages=context)
        
        # 保存到完整历史
        self.full_history.append({"role": "user", "content": user_message})
        self.full_history.append({"role": "assistant", "content": response})
        
        return response

# 效果：
# 第 20 轮对话也只发送 ~1000 tokens
# 成本降低 80-90%
```
###### 示例2: 信息压缩
```
class CompressingAgent:
    def summarize_old_conversations(self, messages):
        """
        将旧对话压缩成摘要
        """
        if len(messages) < 10:
            return messages
        
        # 将前面的对话总结
        old_messages = messages[:-6]  # 除了最近 3 轮
        
        summary_prompt = f"""
            请总结以下对话的关键信息：
            {old_messages}
            
            格式：
            - 用户身份：
            - 主要需求：
            - 已解决的问题：
            - 待处理的问题：
        """
        
        summary = llm.chat([{"role": "user", "content": summary_prompt}])
        
        # 返回：摘要 + 最近对话
        return [
            {"role": "system", "content": f"对话摘要：{summary}"},
            *messages[-6:]  # 最近 3 轮
        ]
    
    def chat(self, user_message):
        # 压缩旧对话
        context = self.summarize_old_conversations(self.history)
        
        # 添加当前消息
        context.append({"role": "user", "content": user_message})
        
        response = llm.chat(messages=context)
        
        self.history.append({"role": "user", "content": user_message})
        self.history.append({"role": "assistant", "content": response})
        
        return responsev
```
###### 示例3: ⭐️分层上下文⭐️
```
class LayeredContext:
    """
    上下文分层管理
    """
    def __init__(self):
        self.permanent_context = []   # 永久信息（系统提示）
        self.session_context = []     # 会话信息（本次对话）
        self.temp_context = []        # 临时信息（当前任务）
    
    def build_full_context(self):
        """
        组合所有层级的上下文
        """
        return (
            self.permanent_context +    # 总是包含
            self.session_context[-10:] +  # 最近的会话
            self.temp_context           # 当前任务的临时信息
        )
    
    def add_permanent(self, content):
        """添加永久信息（如用户配置）"""
        self.permanent_context.append(content)
    
    def add_session(self, content):
        """添加会话信息（对话历史）"""
        self.session_context.append(content)
    
    def add_temp(self, content):
        """添加临时信息（如查询结果）"""
        self.temp_context.append(content)
    
    def clear_temp(self):
        """任务完成后清理临时信息"""
        self.temp_context = []

# 使用示例
context = LayeredContext()

# 永久信息
context.add_permanent({
    "role": "system",
    "content": "用户是 VIP 客户，优先处理"
})

# 会话信息
context.add_session({"role": "user", "content": "查订单"})
context.add_session({"role": "assistant", "content": "请提供订单号"})

# 临时信息（查询结果）
context.add_temp({
    "role": "system",
    "content": "订单信息：{...}"
})

# 构建上下文
full_context = context.build_full_context()

# 任务完成，清理临时信息
context.clear_temp()
```
###### 示例4: 预算管理
```
class ContextBudgetManager:
    """
    上下文令牌预算管理
    """
    def __init__(self, max_tokens=4000):
        self.max_tokens = max_tokens
        self.system_tokens = 500      # 系统提示词预留
        self.response_tokens = 1000   # 回复预留
        self.available_tokens = max_tokens - system_tokens - response_tokens
        # 实际可用于历史 = 2500 tokens
    
    def fit_context(self, messages):
        """
        确保上下文不超过预算
        """
        total_tokens = 0
        fitted_messages = []
        
        # 从后往前添加消息（最新的最重要）
        for msg in reversed(messages):
            msg_tokens = self.count_tokens(msg)
            
            if total_tokens + msg_tokens <= self.available_tokens:
                fitted_messages.insert(0, msg)
                total_tokens += msg_tokens
            else:
                break  # 预算用完
        
        return fitted_messages
    
    def count_tokens(self, message):
        """
        估算消息的 token 数
        简单估算：1 中文字 ≈ 2 tokens, 1 英文词 ≈ 1.3 tokens
        """
        content = message["content"]
        chinese_chars = sum(1 for c in content if '\u4e00' <= c <= '\u9fff')
        english_words = len(content.split())
        return chinese_chars * 2 + english_words * 1.3

# 使用
budget = ContextBudgetManager(max_tokens=4000)
fitted_context = budget.fit_context(full_history)
```
##### 要点总结
1. **信息密度**:  以“最大化LLM理解”的方式结构化信息；
2. **安全性**:  控制传递给LLM的信息，避免敏感数据；
3. **灵活性**:  主动决策在什么时候调整和控制上下文；
4. **Token效率**:  优化上下文以提高Token使用率和对LLM的理解；
    1. 只包含当前任务需要的信息
    2. 最近的信息 > 久远的信息
    3. 相关的信息 > 无关的信息
    4. 压缩历史，保留要点
    5. 监控和优化 token 使用
    6. 利用缓存机制降低成本

5. **相关上下文的比例：（经验）** 
    1. 系统提示词：10% - 20%
    2. 最近对话：10% - 20%
    3. 工具上下文：5% - 10%
    4. 回复预留： 50% - 75%




## 第二类**: 执行与控制流** 
这一部分解决**: 如何可靠地执行任务并控制整个流程**

### Factor 4: 工具就是结构化输出 （Tools are just structured outputs）
![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=39b355497e9e49358081658df8704a88&docGuid=RxuaqJOasFaCo3 "")


##### 核心概念
**工具**:  模型可以调用的外部功能模块或程序接口，赋予模型执行实际操作的能力，让模型从"只会说话"变成"能做事情"的智能代理。

**MCP**:  一套标准化的协议框架，用于定义、连接和管理各种工具。

**工具调用**:  工具调用本质上就是生成**结构化的JSON**；

```
# 步骤 1：定义 JSON 格式（Schema）
deploy_tool = {
    "name": "deploy",
    "parameters": {
        "version": "string",
        "environment": "string"
    }
}

# 步骤 2：AI 生成符合格式的 JSON
ai_output = {
    "name": "deploy",
    "parameters": {
        "version": "v2.0",
        "environment": "production"
    }
}

# 步骤 3：你的代码解析 JSON 并执行
if ai_output["name"] == "deploy":
    deploy_app(
        version=ai_output["parameters"]["version"],
        environment=ai_output["parameters"]["environment"]
    )

# 就这么简单！
```

**常见错误**:  让 AI 直接说话

```
user: "部署最新版本"
ai: "好的，我会部署最新版本到生产环境"

# 问题：
# 1. 这只是文字，没有实际动作
# 2. 无法验证 AI 是否理解正确
# 3. 无法自动执行
```
**合理方式**： 让 AI 输出结构化数据

```
user: "部署最新版本"
ai: {
    "action": "deploy",
    "version": "latest",
    "environment": "production"
}

# 好处：
# 1. 可以验证：版本号合法吗？
# 2. 可以执行：调用部署函数
# 3. 可以审计：记录谁做了什么
# 4. 可以测试：单元测试 JSON 解析
```
##### 实现示例
###### 示例1: 从零开始定义工具
```
# 第 1 步：定义工具的格式
def define_tools():
    return {
        "send_email": {
            "description": "发送邮件",
            "parameters": {
                "to": {
                    "type": "string",
                    "description": "收件人邮箱"
                },
                "subject": {
                    "type": "string",
                    "description": "邮件主题"
                },
                "body": {
                    "type": "string",
                    "description": "邮件正文"
                }
            },
            "required": ["to", "subject", "body"]
        },
        "查询天气": {
            "description": "查询指定城市的天气",
            "parameters": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如：北京、上海"
                }
            },
            "required": ["city"]
        }
    }

# 第 2 步：实现实际的工具函数
def send_email(to, subject, body):
    """实际发送邮件的代码"""
    print(f"发送邮件到 {to}")
    print(f"主题：{subject}")
    print(f"内容：{body}")
    # 实际的发送逻辑...
    return {"status": "success", "message": "邮件已发送"}

def query_weather(city):
    """实际查询天气的代码"""
    # 调用天气 API...
    weather_data = {
        "city": city,
        "temperature": "25°C",
        "condition": "晴天"
    }
    return weather_data

# 第 3 步：创建工具执行器
class ToolExecutor:
    def __init__(self):
        self.tools = {
            "send_email": send_email,
            "query_weather": query_weather
        }
    
    def execute(self, tool_call):
        """
        执行工具调用
        """
        tool_name = tool_call["name"]
        parameters = tool_call["parameters"]
        
        # 验证工具存在
        if tool_name not in self.tools:
            return {"error": f"工具 {tool_name} 不存在"}
        
        # 执行工具
        try:
            result = self.tools[tool_name](**parameters)
            return {"success": True, "result": result}
        except Exception as e:
            return {"success": False, "error": str(e)}

# 第 4 步：完整的流程
def agent_with_tools(user_message):
    # 1. 准备提示词和工具定义
    tools = define_tools()
    
    prompt = f"""
你是一个助手，可以使用以下工具：
{json.dumps(tools, ensure_ascii=False, indent=2)}

用户消息：{user_message}

请输出你要调用的工具，格式如下：
{{
    "name": "工具名称",
    "parameters": {{
        "参数名": "参数值"
    }}
}}
    """
    
    # 2. AI 生成工具调用
    ai_response = llm.complete(prompt)
    tool_call = json.loads(ai_response)
    
    # 3. 执行工具
    executor = ToolExecutor()
    result = executor.execute(tool_call)
    
    # 4. 返回结果
    return result

# 使用
result = agent_with_tools("给 zhang@example.com 发邮件，主题是会议提醒")
# AI 会输出：
# {
#     "name": "send_email",
#     "parameters": {
#         "to": "zhang@example.com",
#         "subject": "会议提醒",
#         "body": "这是一封会议提醒邮件"
#     }
# }
# 然后实际执行 send_email 函数
```
###### 示例2: 工具调用的日志记录
```
class ObservableToolExecutor:
    """
    可观察的工具执行器
    """
    def __init__(self):
        self.executors = {}
        self.call_history = []
    
    def execute(self, tool_call):
        """
        执行工具调用并记录
        """
        # 记录开始时间
        start_time = time.time()
        
        # 记录调用信息
        call_record = {
            "timestamp": start_time,
            "tool_name": tool_call["name"],
            "parameters": tool_call["parameters"],
            "status": "started"
        }
        
        try:
            # 执行工具
            result = self.executors[tool_call["name"]](**tool_call["parameters"])
            
            # 记录成功
            call_record.update({
                "status": "success",
                "result": result,
                "duration": time.time() - start_time
            })
            
            return {"success": True, "result": result}
            
        except Exception as e:
            # 记录失败
            call_record.update({
                "status": "failed",
                "error": str(e),
                "duration": time.time() - start_time
            })
            
            return {"success": False, "error": str(e)}
            
        finally:
            # 保存记录
            self.call_history.append(call_record)
    
    def get_statistics(self):
        """
        获取工具使用统计
        """
        stats = {}
        for record in self.call_history:
            tool_name = record["tool_name"]
            if tool_name not in stats:
                stats[tool_name] = {
                    "total_calls": 0,
                    "successful_calls": 0,
                    "failed_calls": 0,
                    "total_duration": 0
                }
            
            stats[tool_name]["total_calls"] += 1
            if record["status"] == "success":
                stats[tool_name]["successful_calls"] += 1
            else:
                stats[tool_name]["failed_calls"] += 1
            stats[tool_name]["total_duration"] += record["duration"]
        
        return stats
    
    def export_trace(self):
        """
        导出完整的执行追踪
        """
        return json.dumps(self.call_history, indent=2)

# 使用
executor = ObservableToolExecutor()

# 执行多个工具调用
executor.execute({"name": "send_email", "parameters": {...}})
executor.execute({"name": "query_weather", "parameters": {...}})

# 查看统计
stats = executor.get_statistics()
print(f"send_email: 调用 {stats['send_email']['total_calls']} 次, "
      f"成功 {stats['send_email']['successful_calls']} 次")

# 导出追踪日志
trace = executor.export_trace()
```
###### 
##### 要点总结
* 工具调用 = LLM生成Json + 代码执行；
* 定义清晰的 JSON Schema
* 实际执行是确定性代码
* 验证所有工具调用
* 记录和观察工具使用
* 工具函数应该可测试、可复用；

**工具设计的最佳实践**：

1. 单一职责：每个工具做一件事
2. 明确接口：参数清晰，返回值统一
3. 错误处理：优雅地处理异常
4. 幂等性：多次调用结果一致
5. 可测试：独立测试每个工具





### Factor 8: 拥有你的控制流（Own your control flow）
![](https://raw.githubusercontent.com/CaesarInCrypto/ImgStg/master/uPic/1765271834?token=A5SNFWEJVBM7MBC3WJ34G5DJG7UWA "")
##### **核心概念** 
**控制流**: 任务执行过程中的**逻辑流程控制和决策路径管理**，本质上是将线性的"输入-输出"扩展为具有**分支、循环、条件判断的动态执行过程**。它决定了Agent在面对复杂任务时按照什么顺序、在什么条件下执行哪些操作。

**控制流的核心流程**: 

1. **顺序执行**:  按既定步骤依次完成任务；
2. **条件分支**:  根据判断结果选择不同路径，如if-else逻辑；
3. **循环迭代**:  重复执行某操作直到满足条件，如不断搜索直到找到答案；
4. **异常处理**:  遇到错误时的备选方案；
5. **并行执行**:  同时进行多个独立任务；
6. **任务优先级管理**:  决定哪些操作更重要应先执行；

这些流程组合起来构成了Agent的"思考和行动框架"。

**拥有你的控制流**: 在普通代码中维护控制流，而不是嵌套提示

智能体 = 提示 + switch 语句 + 上下文 + 循环

```
async def sequential_flow():
    """
    顺序执行：一步步来
    """
    # 步骤 1
    result1 = await step1()
    
    # 步骤 2（使用步骤 1 的结果）
    result2 = await step2(result1)
    
    # 步骤 3
    result3 = await step3(result2)
    
    return result3
```

```
async def retry_flow(operation, max_retries=3):
    """
    循环与重试：遇到错误可以重试
    """
    for attempt in range(max_retries):
        try:
            result = await operation()
            return {"success": True, "result": result}
        except RetryableError as e:
            if attempt < max_retries - 1:
                # 让 AI 分析错误并调整策略
                strategy = await llm.analyze_error_and_suggest_fix(
                    error=str(e),
                    attempt=attempt
                )
                
                print(f"重试 {attempt + 1}/{max_retries}")
                print(f"AI 建议：{strategy.suggestion}")
                
                # 等待一段时间
                await asyncio.sleep(2 ** attempt)  # 指数退避
            else:
                return {"success": False, "error": "重试次数已用完"}
```



    async def conditional_flow(user_input):
        """
        条件分支：根据情况走不同的路
        """
        # AI 分析用户意图
        intent = await llm.analyze_intent(user_input)
        
        # ★ 根据意图走不同的流程
        if intent.type == "query":
            return await handle_query(intent)
        elif intent.type == "update":
            return await handle_update(intent)
        elif intent.type == "delete":
            # 删除操作需要确认
            confirmation = await request_confirmation()
            if confirmation.confirmed:
                return await handle_delete(intent)
            else:
                return "操作已取消"
        else:
            return "不支持的操作"
        
    async def parallel_flow():
        """
        并行执行：同时做多件事
        """
        # 同时执行多个任务
        results = await asyncio.gather(
            check_service_a(),
            check_service_b(),
            check_service_c()
        )
        
        # 汇总结果
        service_a, service_b, service_c = results
        
        # 让 AI 分析整体状态
        overall_status = await llm.analyze_system_health([
            {"service": "A", "status": service_a},
            {"service": "B", "status": service_b},
            {"service": "C", "status": service_c}
        ])
        
        return overall_status

**常见错误**:  让 AI 自己决定一切

```
# 让 AI 自己决定一切
prompt = """
你需要完成部署任务。
你可以使用的工具有：
- check_status
- run_tests  
- deploy
- rollback

请自己决定执行顺序和步骤。
"""

# 问题：
# 1. AI 可能跳过重要步骤（比如测试）
# 2. AI 可能陷入循环（一直重试）
# 3. AI 可能做出危险决定（直接部署生产环境）
# 4. 你无法在关键点插入人工审批
```
**合理方式**:  自己控制流程

```
async def deploy_workflow(version, environment):
    """
    部署工作流（人工控制流程）
    """
    # 步骤 1：检查状态（必须）
    status = check_current_status(environment)
    
    # 步骤 2：运行测试（必须）
    test_result = run_tests(version)
    if not test_result.passed:
        return "测试失败，停止部署"
    
    # 步骤 3：如果是生产环境，需要人工批准（必须）
    if environment == "production":
        # 这里控制流被"劫持"，等待人工批准
        approval = await request_human_approval(
            action="deploy",
            version=version,
            environment=environment
        )
        if not approval.approved:
            return "部署被拒绝"
    
    # 步骤 4：执行部署
    deploy_result = deploy(version, environment)
    
    # 步骤 5：验证部署
    if not verify_deployment(version, environment):
        # 自动回滚
        rollback(environment)
        return "部署失败，已回滚"
    
    return "部署成功"

# 优势：
# ✅ 流程清晰、可预测
# ✅ 在关键点可以暂停
# ✅ 可以插入人工审批
# ✅ 容易测试和调试
```
##### 实现示例
###### 示例1: 智能体循环
```
def agent_loop_explained():
    """
    智能体循环的简化版本
    """
    # 这就是所有智能体的核心！
    context = [{"role": "user", "content": "帮我部署"}]
    
    while True:
        # 1. 让 AI 决定下一步（这是 AI 的工作）
        next_step = llm.determine_next_step(context)
        
        # 2. switch 语句（这是你的控制流！）
        if next_step.tool == "deploy":
            result = handle_deploy(next_step.parameters)
        elif next_step.tool == "check_status":
            result = handle_check_status(next_step.parameters)
        elif next_step.tool == "done":
            return next_step.final_answer
        else:
            result = {"error": "未知工具"}
        
        # 3. 更新上下文
        context.append({"role": "assistant", "tool_call": next_step})
        context.append({"role": "tool", "result": result})

# 关键点：
# - AI 只负责"决定下一步"
# - 实际执行由 switch 语句控制
# - 你可以在 switch 中做任何事情
```
###### 示例2: 控制流劫持
```
class ControlledAgent:
    """
    带控制流管理的智能体
    """
    def __init__(self):
        self.paused = False
        self.pending_approval = None
    
    async def agent_loop(self, initial_message):
        """
        智能体循环，支持暂停和劫持
        """
        context = [{"role": "user", "content": initial_message}]
        
        while True:
            # AI 决定下一步
            next_step = llm.determine_next_step(context)
            
            # ★ 这里是关键：用 switch 控制流程 ★
            result = await self.execute_with_control(next_step)
            
            # 更新上下文
            context.append({"role": "assistant", "tool_call": next_step})
            context.append({"role": "tool", "result": result})
            
            # 检查是否完成
            if next_step.is_done():
                return next_step.final_answer
    
    async def execute_with_control(self, next_step):
        """
        执行工具调用，但保留控制权
        """
        tool_name = next_step.tool
        
        # Case 1: 普通工具，直接执行
        if tool_name == "check_status":
            return check_status()
        
        # Case 2: 危险操作，劫持控制流，等待批准
        elif tool_name == "deploy_production":
            # 暂停执行
            self.paused = True
            self.pending_approval = next_step
            
            # 发送批准请求
            approval_id = await request_approval(next_step)
            
            # 等待批准（这里暂停了智能体）
            approval = await wait_for_approval(approval_id)
            
            # 恢复执行
            self.paused = False
            
            if approval.approved:
                return deploy_production(next_step.parameters)
            else:
                return {"error": "部署被拒绝"}
        
        # Case 3: 长时间运行的任务，劫持控制流
        elif tool_name == "run_long_task":
            # 启动任务
            task_id = start_long_task(next_step.parameters)
            
            # 暂停智能体
            self.paused = True
            
            # 等待任务完成（可能是几分钟或几小时）
            result = await wait_for_task_completion(task_id)
            
            # 恢复智能体
            self.paused = False
            
            return result
        
        # Case 4: 错误处理，劫持控制流
        elif tool_name == "risky_operation":
            try:
                return risky_operation(next_step.parameters)
            except Exception as e:
                # 捕获错误，决定如何处理
                if is_retryable(e):
                    # 重试
                    return await self.retry_with_backoff(next_step)
                else:
                    # 升级到人工
                    return await self.escalate_to_human(next_step, e)
```




##### 要点总结
1. 关键路径用代码控制
2. 决策点让 AI 参与
3. 危险操作需要人工批准
4. 长时间任务支持暂停/恢复
5. 错误处理有明确策略



## 第三类：状态管理
这一部分解决**: 如何正确地保存和管理智能体的状态**

### Factor 5: 统一执行状态和业务状态 (Unify execution state and business state)
![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=bf4ea4532c9347bd86e84e79c3cc493a&docGuid=RxuaqJOasFaCo3 "")
##### 核心概念
**执行状态：Agent当前正在做什么、处于执行流程的哪个阶段、技术层面的运行情况**，它包括诸如"等待用户输入"、"正在调用搜索工具"、"工具执行中"、"解析工具返回结果"、"生成回复"、"任务完成"、"发生错误"等状态，这些状态**反映的是系统内部的技术执行流转**，与具体业务无关，任何Agent都会经历这些通用的执行阶段。

**业务状态：从业务逻辑角度看任务进展到了哪一步**、完成了什么业务目标、还有什么业务目标待完成，它**与具体应用场景紧密相关**，比如在客服Agent中业务状态可能是"收集用户信息"、"查询订单"、"处理退款申请"、"等待用户确认"，在旅行规划Agent中可能是"确定目的地"、"搜索航班"、"比较酒店"、"生成行程单"，这些状态描述的是业务流程的推进。





**统一执行状态和业务状态**: 不要分开跟踪"Agent在做什么"和"业务逻辑在做什么"

**分开管理两套状态的问题**: 

1. 在状态**持久化和恢复**时需要同时保存两份数据；
2. 状态之间的**映射关系复杂**且容易出错；
3. 调试时需要同时追踪两个状态流转增加**复杂度**；
4. 容易出现**状态不同步**的问题（比如执行已完成但业务状态未更新）；

在简单系统中还好，但在复杂Agent中会显著增加开发和维护成本。

**统一状态的核心思想**：每个状态**既包含执行层面的信息**（如当前正在执行什么操作、是否在等待工具响应），**也包含业务层面的信息**（如处于业务流程的哪个阶段、完成了什么业务目标）；

比如用一个状态"正在查询订单详情"就同时表达了执行动作（查询数据库）和业务含义（获取订单信息），而不是分别用"tool_calling"和"order_inquiry"两个状态。



**常见错误**:  分开存储状态

```
# 执行状态：智能体的"工作进度"
execution_state = {
    "current_step": "deploying",
    "step_number": 3,
    "total_steps": 5,
    "in_progress": True
}

# 业务状态：实际的"业务进度"
business_state = {
    "deployment_id": "deploy-123",
    "version": "v2.0",
    "environment": "production",
    "deploy_status": "running"
}
```
**正确方式**:  事件驱动（用事件流统一状态）

```
# 用“事件流”统一状态
class Thread:
    events = [
        {"type": "deployment_requested", "env": "production", "version": "v2.1.0"},
        {"type": "tests_started", "timestamp": "..."},
        {"type": "tests_passed", "duration": "2m"},
        {"type": "deployment_started", "timestamp": "..."},
        # 从事件流可以推断出执行状态
    ]
```
##### 实现示例
###### 示例1: 完整的事件驱动智能体
```
from enum import Enum
from dataclasses import dataclass
from typing import List, Dict, Any
import json

class EventType(Enum):
    """事件类型定义"""
    # 用户交互
    USER_MESSAGE = "user_message"
    ASSISTANT_RESPONSE = "assistant_response"
    
    # 工具调用
    TOOL_CALL_REQUESTED = "tool_call_requested"
    TOOL_CALL_STARTED = "tool_call_started"
    TOOL_CALL_COMPLETED = "tool_call_completed"
    TOOL_CALL_FAILED = "tool_call_failed"
    
    # 人工审批
    APPROVAL_REQUESTED = "approval_requested"
    APPROVAL_GRANTED = "approval_granted"
    APPROVAL_DENIED = "approval_denied"
    
    # 任务状态
    TASK_STARTED = "task_started"
    TASK_PAUSED = "task_paused"
    TASK_RESUMED = "task_resumed"
    TASK_COMPLETED = "task_completed"
    TASK_FAILED = "task_failed"

@dataclass
class Event:
    """事件数据结构"""
    id: str
    timestamp: str
    type: EventType
    data: Dict[str, Any]
    metadata: Dict[str, Any] = None

class EventDrivenAgent:
    """
    事件驱动的智能体
    """
    def __init__(self, agent_id: str):
        self.agent_id = agent_id
        self.events: List[Event] = []
    
    def add_event(self, event_type: EventType, data: Dict[str, Any]):
        """
        添加事件到事件流
        """
        event = Event(
            id=f"event-{len(self.events)}",
            timestamp=datetime.now().isoformat(),
            type=event_type,
            data=data,
            metadata={"agent_id": self.agent_id}
        )
        
        self.events.append(event)
        
        # 持久化到数据库
        self.persist_event(event)
        
        return event
    
    def get_conversation_history(self) -> List[Dict]:
        """
        从事件流提取对话历史
        """
        history = []
        
        for event in self.events:
            if event.type == EventType.USER_MESSAGE:
                history.append({
                    "role": "user",
                    "content": event.data["message"]
                })
            elif event.type == EventType.ASSISTANT_RESPONSE:
                history.append({
                    "role": "assistant",
                    "content": event.data["response"]
                })
        
        return history
    
    def get_tool_calls(self) -> List[Dict]:
        """
        从事件流提取所有工具调用
        """
        tool_calls = []
        
        for event in self.events:
            if event.type == EventType.TOOL_CALL_REQUESTED:
                # 找到对应的完成/失败事件
                call_id = event.data["call_id"]
                result = self.find_tool_result(call_id)
                
                tool_calls.append({
                    "tool": event.data["tool"],
                    "parameters": event.data["parameters"],
                    "result": result
                })
        
        return tool_calls
    
    def get_current_state(self) -> Dict:
        """
        从事件流计算当前状态
        """
        state = {
            "status": "idle",
            "current_task": None,
            "pending_approvals": [],
            "completed_tools": [],
            "errors": []
        }
        
        for event in self.events:
            if event.type == EventType.TASK_STARTED:
                state["status"] = "running"
                state["current_task"] = event.data["task"]
            
            elif event.type == EventType.TASK_PAUSED:
                state["status"] = "paused"
            
            elif event.type == EventType.TASK_COMPLETED:
                state["status"] = "completed"
                state["current_task"] = None
            
            elif event.type == EventType.APPROVAL_REQUESTED:
                state["pending_approvals"].append(event.data)
            
            elif event.type in [EventType.APPROVAL_GRANTED, EventType.APPROVAL_DENIED]:
                # 移除已处理的审批
                approval_id = event.data["approval_id"]
                state["pending_approvals"] = [
                    a for a in state["pending_approvals"]
                    if a["id"] != approval_id
                ]
            
            elif event.type == EventType.TOOL_CALL_COMPLETED:
                state["completed_tools"].append(event.data["tool"])
            
            elif event.type == EventType.TOOL_CALL_FAILED:
                state["errors"].append(event.data["error"])
        
        return state
    
    def can_resume(self) -> bool:
        """
        判断是否可以恢复执行
        """
        state = self.get_current_state()
        
        # 如果有待处理的审批，不能恢复
        if state["pending_approvals"]:
            return False
        
        # 如果是暂停状态，可以恢复
        if state["status"] == "paused":
            return True
        
        return False
    
    def export_events(self) -> str:
        """
        导出完整的事件流（用于审计、调试）
        """
        return json.dumps([
            {
                "id": e.id,
                "timestamp": e.timestamp,
                "type": e.type.value,
                "data": e.data
            }
            for e in self.events
        ], indent=2, ensure_ascii=False)
    
    @classmethod
    def load_from_events(cls, agent_id: str, events: List[Event]):
        """
        从事件流恢复智能体
        """
        agent = cls(agent_id)
        agent.events = events
        return agent

# 使用示例
agent = EventDrivenAgent("agent-123")

# 场景：用户请求部署
agent.add_event(EventType.USER_MESSAGE, {
    "message": "部署 v2.0 到生产环境"
})

agent.add_event(EventType.TASK_STARTED, {
    "task": "deploy",
    "version": "v2.0",
    "environment": "production"
})

# 需要测试
agent.add_event(EventType.TOOL_CALL_REQUESTED, {
    "call_id": "call-1",
    "tool": "run_tests",
    "parameters": {"version": "v2.0"}
})

agent.add_event(EventType.TOOL_CALL_COMPLETED, {
    "call_id": "call-1",
    "tool": "run_tests",
    "result": {"passed": True}
})

# 需要人工审批
agent.add_event(EventType.APPROVAL_REQUESTED, {
    "id": "approval-1",
    "action": "deploy_production",
    "details": {"version": "v2.0"}
})

# ★ 此时系统崩溃 ★

# ... 系统重启 ...

# 恢复智能体
agent = EventDrivenAgent.load_from_database("agent-123")

# 检查状态
state = agent.get_current_state()
print(f"状态：{state['status']}")  # → "running"
print(f"待审批：{state['pending_approvals']}")  # → [{"id": "approval-1", ...}]

# 可以准确知道：
# 1. 测试已完成
# 2. 正在等待审批
# 3. 还没有执行部署
# 4. 应该从审批步骤继续

if agent.can_resume():
    # 等待审批...
    pass
```
##### 要点总结
1. **事件驱动**:  不要维护两个账本，用事件流记录流程；
2. **状态派生**:  状态是事件的派生结果；
3. **可恢复&可审计**：状态可以从任何时间点恢复；还具有完整的审计日志；



### Factor 6: 简单 API 实现启动/暂停/恢复 （Launch/Pause/Resume with simple APIs）
![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=f67a61690f134bf993f1ebe7dce33451&docGuid=RxuaqJOasFaCo3 "")
##### 核心概念
**简单 API 实现启动/暂停/恢复**:  实现智能体的生命周期管理，支持中断和恢复

**使用场景**: 

```
# 1. 人类审批：智能体遇到需要批准的操作——部署需要人工批准 → 暂停 → 等待 → 恢复
if tool_call.name == "deploy_to_production":
    pause(task_id, reason="需要人类批准")
    await request_human_approval()
```
```
# 2. 长时间运行的任务： 等待外部系统完成——长时间运行的任务 → 暂停 → 异步完成 → 恢复
pause(task_id, reason="等待部署完成")
# 稍后由 webhook 恢复
```
```
# 3. 调试和测试：在特定点暂停以检查状态——需要手动调试 → 暂停 → 等待输入 → 恢复
pause(task_id, reason="调试检查点")
```


##### 实现示例
###### 示例1: 与人工审批集成
```
class ApprovalIntegratedAgent(PausableAgent):
    """
    集成人工审批的智能体
    """
    async def execute_with_approval(self, tool_call: Dict):
        """
        执行需要审批的工具调用
        """
        # 检查是否需要审批
        if self.requires_approval(tool_call):
            # 暂停并请求审批
            approval_id = await self.request_approval(tool_call)
            
            self.pause(
                reason="waiting_for_approval",
                context={
                    "approval_id": approval_id,
                    "tool_call": tool_call
                }
            )
            
            # 这里会暂停执行
            # 等待 resume() 被调用
            
            return
        
        # 不需要审批，直接执行
        return await self.execute_tool(tool_call)
    
    async def request_approval(self, tool_call: Dict) -> str:
        """
        请求人工审批
        """
        approval_id = f"approval-{uuid.uuid4()}"
        
        # 创建审批请求
        approval_request = {
            "id": approval_id,
            "agent_id": self.agent_id,
            "tool": tool_call["name"],
            "parameters": tool_call["parameters"],
            "risk_level": self.assess_risk(tool_call),
            "requested_at": datetime.now().isoformat()
        }
        
        # 保存到数据库
        db.save_approval_request(approval_request)
        
        # 发送通知（多渠道）
        await self.notify_approvers(approval_request, channels=[
            "slack",
            "email",
            "dashboard"
        ])
        
        return approval_id
    
    async def handle_approval_response(self, approval_id: str, response: Dict):
        """
        处理审批响应
        """
        # 验证审批请求存在
        approval = db.get_approval_request(approval_id)
        if not approval:
            raise ValueError(f"审批请求不存在：{approval_id}")
        
        # 验证审批人权限
        if not self.verify_approver(response["approver"], approval):
            raise ValueError("审批人无权限")
        
        # 恢复智能体，传入审批结果
        self.resume(input_data={
            "approval_id": approval_id,
            "approved": response["approved"],
            "approver": response["approver"],
            "reason": response.get("reason", ""),
            "timestamp": datetime.now().isoformat()
        })
    
    async def continue_execution(self):
        """
        恢复后继续执行
        """
        # 获取暂停时的上下文
        pause_context = self.get_last_event_of_type("task_paused")["context"]
        
        # 获取审批结果
        approval_result = self.resume_data
        
        if approval_result["approved"]:
            # 审批通过，执行工具
            tool_call = pause_context["tool_call"]
            result = await self.execute_tool(tool_call)
            
            # 记录结果
            self.add_event({
                "type": "tool_completed",
                "tool": tool_call["name"],
                "result": result
            })
        else:
            # 审批被拒绝
            self.add_event({
                "type": "approval_denied",
                "reason": approval_result.get("reason", "")
            })
            
            # 终止任务
            self.status = AgentStatus.FAILED
            return
        
        # 继续执行后续步骤
        await self.continue_workflow()

# 使用示例
agent = ApprovalIntegratedAgent("agent-123")

# 智能体执行过程中遇到部署操作
tool_call = {
    "name": "deploy_production",
    "parameters": {
        "version": "v2.0"
    }
}

# 自动暂停并请求审批
await agent.execute_with_approval(tool_call)

# ... 审批流程在后台进行 ...

# 人类审批后，调用处理函数
await agent.handle_approval_response("approval-123", {
    "approved": True,
    "approver": "manager@company.com"
})

# 智能体自动恢复并继续执行
```
###### 示例2: 长时间任务的异步处理
```
class AsyncTaskAgent(PausableAgent):
    """
    支持异步长时间任务的智能体
    """
    async def execute_long_running_task(self, task_spec: Dict):
        """
        执行长时间运行的任务
        """
        # 启动异步任务
        task_id = await self.start_background_task(task_spec)
        
        # 暂停智能体（不占用资源等待）
        self.pause(
            reason="waiting_for_background_task",
            context={
                "background_task_id": task_id,
                "task_type": task_spec["type"]
            }
        )
        
        # 注册webhook回调
        await self.register_webhook_callback(
            task_id=task_id,
            callback_url=f"/api/agent/{self.agent_id}/resume"
        )
    
    async def start_background_task(self, task_spec: Dict) -> str:
        """
        启动后台任务
        """
        task_id = f"bg-task-{uuid.uuid4()}"
        
        # 提交到任务队列
        await task_queue.submit({
            "id": task_id,
            "type": task_spec["type"],
            "parameters": task_spec["parameters"],
            "callback_url": f"/api/agent/{self.agent_id}/task-complete"
        })
        
        return task_id
    
    async def handle_task_completion(self, task_id: str, result: Dict):
        """
        处理后台任务完成
        """
        # 验证任务
        pause_context = self.get_last_event_of_type("task_paused")["context"]
        if pause_context["background_task_id"] != task_id:
            raise ValueError("任务ID不匹配")
        
        # 恢复智能体，传入任务结果
        self.resume(input_data={
            "task_id": task_id,
            "result": result
        })

# 使用示例
agent = AsyncTaskAgent("agent-123")

# 执行需要长时间的数据处理
await agent.execute_long_running_task({
    "type": "process_large_dataset",
    "parameters": {
        "dataset_id": "dataset-123",
        "operation": "analysis"
    }
})

# ... 智能体暂停，任务在后台运行（可能几小时）...

# 任务完成后，通过 webhook 通知
@app.post("/api/agent/{agent_id}/task-complete")
async def task_complete_webhook(agent_id: str, task_result: Dict):
    agent = load_agent(agent_id)
    await agent.handle_task_completion(
        task_id=task_result["task_id"],
        result=task_result["result"]
    )
    return {"status": "resumed"}
```
##### 要点总结
1. Agent任务不是一次性运行完，需要提供提供 launch/pause/resume API方法；
2. 可以在任何点暂停，保存完整的暂停上下文；
3. 支持异步恢复，恢复时可以传入新数据；



### Factor 12: 让智能体成为无状态归约器（Make your agent a stateless reducer）
![](https://raw.githubusercontent.com/CaesarInCrypto/ImgStg/master/uPic/1765271835?token=A5SNFWEARXOOJG72UHARGFLJG7UVW "")
##### 核心概念
**归约器（Reducer）**: 在Agent系统，归约器指的是一个纯函数，用于**管理状态转换**。它接收当前状态和一个事件/动作，然后计算并返回新的状态。核心思想是**通过累积处理一系列事件来推导出最终状态**。Agent系统不是直接修改状态对象，而是派发事件，归约器根据事件类型和载荷计算新状态。例如当前状态是`{phase: "idle", taskCount: 0}`，收到事件`{type: "TASK_STARTED", taskId: "123"}`，归约器返回新状态`{phase: "running", taskCount: 1, currentTask: "123"}`，这种模式使状态变化可预测、可追溯。

**事件溯源模式**: 归约器是事件溯源（Event Sourcing）架构的核心，系统不保存当前状态本身，而是**保存所有历史事件序列**，需要获取当前状态时，从初始状态开始，将所有事件依次通过归约器处理，最终"归约"出当前状态。这就像银行账户不记录余额，而是记录所有交易记录，余额通过累加所有交易计算得出。

```
# 将智能体设计为纯函数：输入状态 + 事件 → 新状态
def agent_reducer(current_state: State, event: Event) -> State:
    """
    纯函数：给定相同的输入，总是产生相同的输出
    没有副作用
    """
    # 基于当前状态和事件确定动作
    action = determine_action(current_state, event)
    
    # 计算新状态
    new_state = apply_action(current_state, action)
    
    return new_state
```
##### 实现示例
###### 示例1: 无状态智能体
```
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class AgentState:
    """
    智能体状态
    """
    task_id: str
    status: str  # "idle", "running", "paused", "completed"
    events: List[dict]
    current_step: Optional[str] = None
    variables: dict = None

@dataclass
class Event:
    """
    事件
    """
    type: str
    data: dict

class StatelessAgent:
    """
    无状态智能体归约器
    """
    
    def reduce(self, state: AgentState, event: Event) -> AgentState:
        """
        纯函数：state + event → new state
        """
        if event.type == "task_started":
            return AgentState(
                task_id=state.task_id,
                status="running",
                events=state.events + [event],
                current_step="initial"
            )
        
        elif event.type == "tool_called":
            return AgentState(
                task_id=state.task_id,
                status="running",
                events=state.events + [event],
                current_step=event.data["tool_name"]
            )
        
        elif event.type == "task_paused":
            return AgentState(
                task_id=state.task_id,
                status="paused",
                events=state.events + [event],
                current_step=state.current_step
            )
        
        elif event.type == "task_completed":
            return AgentState(
                task_id=state.task_id,
                status="completed",
                events=state.events + [event],
                current_step=None
            )
        
        else:
            # 未知事件，返回原状态
            return state
    
    async def execute(self, initial_state: AgentState, user_message: str):
        """
        执行智能体
        """
        state = initial_state
        
        # 事件 1：任务开始
        state = self.reduce(state, Event(
            type="task_started",
            data={"message": user_message}
        ))
        
        # 执行循环
        while state.status == "running":
            # LLM 决定下一步
            next_action = await self.llm.determine_next_step(
                self.build_context(state)
            )
            
            if next_action.is_done():
                # 任务完成
                state = self.reduce(state, Event(
                    type="task_completed",
                    data={"result": next_action.result}
                ))
            else:
                # 调用工具
                state = self.reduce(state, Event(
                    type="tool_called",
                    data={"tool_name": next_action.tool}
                ))
                
                # 执行工具
                result = await self.execute_tool(next_action)
                
                # 记录结果
                state = self.reduce(state, Event(
                    type="tool_completed",
                    data={"result": result}
                ))
        
        return state
    
    def build_context(self, state: AgentState) -> List[dict]:
        """
        从状态构建上下文
        """
        messages = []
        
        for event in state.events:
            if event.type == "task_started":
                messages.append({
                    "role": "user",
                    "content": event.data["message"]
                })
            elif event.type == "tool_called":
                messages.append({
                    "role": "assistant",
                    "tool_call": event.data
                })
            elif event.type == "tool_completed":
                messages.append({
                    "role": "tool",
                    "result": event.data["result"]
                })
        
        return messages

# 使用
agent = StatelessAgent()

# 初始状态
initial_state = AgentState(
    task_id="task-123",
    status="idle",
    events=[]
)

# 执行
final_state = await agent.execute(initial_state, "部署 v2.0")

# 好处：
# 1. 可以保存任何时刻的状态
# 2. 可以重放到任何时刻
# 3. 易于测试（纯函数）
# 4. 易于理解（显式状态转换）
```
##### 要点总结
1. **完整的状态可追溯性**: 每个状态变化都对应明确的事件记录。传统直接修改状态的方式只能看到当前状态，却无法知道历史演变过程。这对调试、审计、问题诊断都极其重要。
2. **时间旅行调试能力**: 可以"穿越"到任意历史时刻查看状态。如果Agent在第100个事件后出错，你可以只用前50个事件重建状态，逐步增加事件精确定位问题，甚至可以修改某个历史事件重新归约，观察不同决策的结果。
3. **天然支持撤销和重做**: 撤销只需移除最后一个事件并重新归约，重做则把事件加回来。不需要复杂的逆向操作或状态快照管理，因为状态本身就是事件序列的函数，移除事件自然回到之前的状态。
4. **简化测试和验证**: 归约器是纯函数，测试极其简单：准备输入（状态和事件）、调用归约器、断言输出。不需要复杂环境、mock依赖、模拟异步，测试运行快、结果稳定、易于覆盖各种边界情况。
5. **强大的状态恢复能力**: 只需持久化事件日志，进程崩溃后从日志读取事件，通过归约器重建状态即可精确恢复。事件日志是简单追加操作，性能高且不易损坏，甚至可以在不同机器上恢复，因为归约器是确定性的。
6. 。。。。



## 第四类：协作与可靠性
这一部分解决**: 如何与人类协作并保持系统可靠性**

### Factor 7: 用工具进行人机协同（Contact humans with tool calls）
##### 核心概念
**人机协同（Human-in-the-Loop，简称HITL）**: 通过将人类干预融入系统来提高可靠性、可控性和准确性。

**用工具进行人机协同**: 将**人机协同作为一等工具**调用，而不是边缘情况；



**使用示例：人类作为工具**

```
# 定义"联系人类"工具
human_approval_tool = {
    "name": "request_human_approval",
    "description": "请求人类批准高风险操作",
    "parameters": {
        "operation": {"type": "string"},
        "context": {"type": "object"},
        "urgency": {"type": "string", "enum": ["low", "medium", "high"]}
    }
}

# LLM 调用人类工具
tool_call = {
    "name": "request_human_approval",
    "parameters": {
        "operation": "deploy_to_production",
        "context": {"version": "v2.1.0", "environment": "production"},
        "urgency": "high"
    }
}
```
##### 实现示例
###### 示例1: 完整的人类作为工具
```
from typing import Literal, Optional
from pydantic import BaseModel

# 定义"联系人类"工具的参数
class ContactHumanParams(BaseModel):
    """
    联系人类的参数
    """
    reason: Literal["approval", "question", "notification", "help"]
    message: str
    urgency: Literal["low", "medium", "high", "critical"] = "medium"
    context: dict = {}
    channels: list[str] = ["slack", "email"]
    timeout_minutes: Optional[int] = None

# 工具定义
CONTACT_HUMAN_TOOL = {
    "name": "contact_human",
    "description": """
    联系人类获取帮助、批准或信息。
    
    使用场景：
    - 需要批准高风险操作（如生产部署、数据删除）
    - 遇到无法自己解决的问题
    - 需要人类提供额外信息
    - 重要通知需要人类知晓
    """,
    "parameters": ContactHumanParams.schema()
}

class HumanInTheLoopAgent:
    """
    支持HITL的智能体
    """
    def __init__(self):
        self.tools = {
            "deploy": self.deploy,
            "contact_human": self.contact_human,
            "delete_data": self.delete_data
        }
    
    async def contact_human(self, params: ContactHumanParams) -> dict:
        """
        联系人类的工具实现
        """
        print(f"\n{'='*50}")
        print(f"🚨 需要人类协助 - {params.urgency.upper()}")
        print(f"原因：{params.reason}")
        print(f"消息：{params.message}")
        if params.context:
            print(f"上下文：{json.dumps(params.context, indent=2, ensure_ascii=False)}")
        print(f"{'='*50}\n")
        
        # 创建人类任务
        task_id = await self.create_human_task(params)
        
        # 发送通知到各个渠道
        await self.send_notifications(task_id, params)
        
        # 暂停智能体，等待人类响应
        self.pause(
            reason=f"waiting_for_human_{params.reason}",
            context={
                "task_id": task_id,
                "params": params.dict()
            }
        )
        
        # 等待人类响应
        response = await self.wait_for_human_response(
            task_id,
            timeout=params.timeout_minutes
        )
        
        # 恢复智能体
        self.resume(input_data=response)
        
        return response
    
    async def create_human_task(self, params: ContactHumanParams) -> str:
        """
        创建人类任务
        """
        task_id = f"human-task-{uuid.uuid4()}"
        
        task = {
            "id": task_id,
            "agent_id": self.agent_id,
            "reason": params.reason,
            "message": params.message,
            "urgency": params.urgency,
            "context": params.context,
            "status": "pending",
            "created_at": datetime.now().isoformat()
        }
        
        # 保存到数据库
        await db.save_human_task(task)
        
        return task_id
    
    async def send_notifications(self, task_id: str, params: ContactHumanParams):
        """
        发送通知到多个渠道
        """
        notification = self.format_notification(task_id, params)
        
        for channel in params.channels:
            if channel == "slack":
                await self.send_slack_notification(notification)
            elif channel == "email":
                await self.send_email_notification(notification)
            elif channel == "sms":
                await self.send_sms_notification(notification)
            elif channel == "dashboard":
                await self.update_dashboard(notification)
    
    async def send_slack_notification(self, notification: dict):
        """
        发送 Slack 通知
        """
        # 构建交互式消息
        message = {
            "text": notification["message"],
            "blocks": [
                {
                    "type": "header",
                    "text": {
                        "type": "plain_text",
                        "text": f"🤖 {notification['title']}"
                    }
                },
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": notification["message"]
                    }
                },
                {
                    "type": "section",
                    "fields": [
                        {
                            "type": "mrkdwn",
                            "text": f"*紧急程度:*\n{notification['urgency']}"
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*原因:*\n{notification['reason']}"
                        }
                    ]
                },
                {
                    "type": "actions",
                    "elements": [
                        {
                            "type": "button",
                            "text": {"type": "plain_text", "text": "批准 ✅"},
                            "style": "primary",
                            "value": f"approve:{notification['task_id']}"
                        },
                        {
                            "type": "button",
                            "text": {"type": "plain_text", "text": "拒绝 ❌"},
                            "style": "danger",
                            "value": f"reject:{notification['task_id']}"
                        },
                        {
                            "type": "button",
                            "text": {"type": "plain_text", "text": "查看详情"},
                            "url": f"https://dashboard.company.com/tasks/{notification['task_id']}"
                        }
                    ]
                }
            ]
        }
        
        await slack_client.post_message(
            channel="#ai-approvals",
            message=message
        )

# 使用示例
agent = HumanInTheLoopAgent()

# AI 在执行过程中需要人类批准
async def agent_workflow():
    # 1. 运行测试
    test_result = await agent.run_tests()
    
    # 2. 测试通过，准备部署
    if test_result.passed:
        # 3. ★ 联系人类请求批准（作为工具调用）
        approval = await agent.contact_human(ContactHumanParams(
            reason="approval",
            message="测试已通过，请批准部署到生产环境",
            urgency="high",
            context={
                "version": "v2.0",
                "environment": "production",
                "test_results": test_result.summary
            },
            channels=["slack", "email"],
            timeout_minutes=30
        ))
        
        # 4. 根据批准结果决定
        if approval["approved"]:
            await agent.deploy("v2.0", "production")
            return "部署成功"
        else:
            return f"部署被拒绝：{approval['reason']}"
```
##### 要点总结
1. 人类 = 特殊的工具；人机协同 = 核心功能，不是边缘情况；



### Factor 9: 将错误压缩到上下文窗口（Compact Errors into Context Window）
![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=ef0d22072ff148ec8c1f6a5f7ce88610&docGuid=RxuaqJOasFaCo3 "")
##### 核心概念
**将错误压缩到上下文窗口**: 当错误发生时，将其压缩成有用的上下文，而不是让其破坏Agent循环

**常见错误**: 

```
# ❌ 将原始错误堆栈追加到上下文
try:
    result = execute_tool(tool_call)
except Exception as e:
    context.append(str(e))  # 庞大的堆栈跟踪
```
**正确示例**: 

```
# ✅ 压缩成有用的摘要
def compact_error(error, tool_call):
    return {
        "error_type": type(error).__name__,
        "tool": tool_call.name,
        "message": error.user_friendly_message,
        "suggested_fix": analyze_error(error),
        "retry_count": get_retry_count(tool_call),
        "can_retry": is_retryable(error)
    }

try:
    result = execute_tool(tool_call)
except Exception as e:
    error_summary = compact_error(e, tool_call)
    context.append(error_summary)
    
    # LLM 现在可以智能地决定如何处理
    next_step = llm.determine_recovery_strategy(context)
```
##### **要点总结**
1. Agent执行出现错误时， 不应该影响Agent的整体流程；
2. 需要将错误数据保存到上下文中，让Agent决策并恢复；
3. 保存错误信息时，应该进行有效的压缩；



### Factor 10: 小型、专注的智能体（Small, Focused Agents）
![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=355b3254ca6741a1922922ceed3939bf&docGuid=RxuaqJOasFaCo3 "")


##### 核心概念
**目前LLM 的局限性**: 任务越大越复杂，需要的步骤就越多，这意味着上下文窗口会更长。随着上下文的增长，LLM 更容易迷失方向或失去焦点。

**小型、专注的智能体**: 

1. **可维护性更强**：功能边界清晰，出问题容易定位和修复，不会因为修改一个功能影响其他功能。
2. **可测试性更好**：专注的Agent行为可预测，测试用例容易编写，质量更容易保证。
3. **可复用性高**：专门做邮件发送的Agent可以被多个业务流程复用，不需要重复开发。
4. **并行开发**：不同团队可以同时开发不同的专注Agent，互不干扰。



**常见错误**: 

```
# ❌ 试图做所有事情的智能体
mega_agent = Agent(
    tools=[
        "deploy_frontend", "deploy_backend",
        "manage_database", "send_emails",
        "create_invoices", "process_payments",
        "analyze_logs", "generate_reports",
        # ... 50 个工具
    ]
)
```
**合理方式**: 

```
# ✅ 专注的微智能体
deployment_agent = Agent(
    name="DeployBot",
    tools=["deploy", "rollback", "check_status"],
    max_steps=3-10  # 保持简单
)

payment_agent = Agent(
    name="PaymentBot",
    tools=["create_payment_link", "check_payment", "refund"],
    max_steps=3-10
)

reporting_agent = Agent(
    name="ReportBot",
    tools=["fetch_data", "analyze", "format_report"],
    max_steps=3-10
)
```
##### 实现示例
###### **示例1: 智能体编排**
```

class AgentOrchestrator:
    """
    智能体编排器：路由到正确的微智能体
    """
    def __init__(self):
        # 注册所有微智能体
        self.agents = {
            "order_query": OrderQueryAgent(),
            "deployment": DeploymentAgent(),
            "refund": RefundAgent(),
            "report": ReportAgent(),
            # ... 更多专门的智能体
        }
        
        # 简单的路由 LLM
        self.router_prompt = """
你是一个路由助手。根据用户请求，选择合适的专门智能体。

可用的智能体：
- order_query: 查询订单信息
- deployment: 代码部署和回滚
- refund: 处理退款申请
- report: 生成数据报表

只返回智能体名称，不要其他内容。
        """
    
    async def handle(self, user_request: str):
        """
        处理用户请求
        """
        # 1. 路由到正确的智能体
        agent_name = await self.route_to_agent(user_request)
        
        if agent_name not in self.agents:
            return f"找不到合适的智能体处理：{user_request}"
        
        # 2. 让专门的智能体处理
        agent = self.agents[agent_name]
        result = await agent.handle(user_request)
        
        return result
    
    async def route_to_agent(self, request: str) -> str:
        """
        智能路由
        """
        response = await llm.complete(
            f"{self.router_prompt}\n\n用户请求：{request}\n\n选择的智能体："
        )
        
        return response.strip().lower()

# 使用
orchestrator = AgentOrchestrator()

# 自动路由到正确的智能体
await orchestrator.handle("查询订单 12345")  # → order_query
await orchestrator.handle("部署最新版本")     # → deployment
await orchestrator.handle("处理退款")        # → refund
```


 参考

[https://12factor.net/zh_cn/](https://12factor.net/zh_cn/)[https://12factor.net/zh_cn/](https://12factor.net/zh_cn/)

[https://yanfukun.com/read/langgraph/HITL](https://yanfukun.com/read/langgraph/HITL)

###### 示例2: 智能体协作
```
class CollaborativeAgentSystem:
    """
    协作式智能体系统
    """
    
    async def handle_complex_workflow(self, user_request: str):
        """
        处理需要多个智能体协作的复杂工作流
        """
        # 场景：用户要退款并重新下单
        # "订单 12345 有问题，帮我退款并重新下单"
        
        # 1. 分解任务
        tasks = await self.decompose_task(user_request)
        # → ["退款订单 12345", "创建新订单"]
        
        results = []
        
        for task in tasks:
            # 2. 路由到相应的智能体
            agent_name = await self.route_to_agent(task)
            agent = self.agents[agent_name]
            
            # 3. 执行任务
            result = await agent.handle(task)
            results.append(result)
            
            # 4. 检查结果，决定是否继续
            if not result["success"]:
                return {
                    "success": False,
                    "failed_at": task,
                    "reason": result["error"]
                }
        
        return {
            "success": True,
            "completed_tasks": tasks,
            "results": results
        }
    
    async def decompose_task(self, complex_request: str) -> list[str]:
        """
        将复杂请求分解为简单任务
        """
        prompt = f"""
将以下复杂请求分解为简单的独立任务：
{complex_request}

格式：每行一个任务
        """
        
        response = await llm.complete(prompt)
        tasks = [line.strip() for line in response.split('\n') if line.strip()]
        
        return tasks

# 使用
system = CollaborativeAgentSystem()

result = await system.handle_complex_workflow(
    "订单 12345 有质量问题，帮我退款并重新下单同样的商品"
)

# 系统会：
# 1. 分解任务："退款订单 12345" + "创建新订单"
# 2. 用 RefundAgent 处理退款
# 3. 用 OrderAgent 创建新订单
# 4. 返回完整结果
```


### Factor 11: 灵活触发机制 （Trigger from anywhere, meet users where they are）
![](https://raw.githubusercontent.com/CaesarInCrypto/ImgStg/master/uPic/1765271836?token=A5SNFWBMV4F65FLX5QHMP33JG7UWE "")
##### 核心概念
**灵活触发机制，融入用户的工作流程**: Agent应该具备**多渠道、多场景的接入能力**，而不是**让用户必须到特定平台使用**。

用户可能在聊天软件、在网页浏览、在IDE编程、在邮件处理，Agent应该能够在这些不同场景中被触发和使用，而不是强迫用户切换到专门的AI界面。



##### 实现示例
###### 示例1: 多渠道触发
```
class AgentController:
    # 1. API 端点
    @app.post("/api/agent/run")
    async def api_trigger(request):
        return await agent.run(request.data)
    
    # 2. Slack 命令
    @slack.command("/deploy")
    async def slack_trigger(payload):
        return await agent.run(payload)
    
    # 3. Email
    @email.on_receive
    async def email_trigger(email):
        return await agent.run(parse_email(email))
    
    # 4. Webhook
    @webhook.on_event("deployment_needed")
    async def webhook_trigger(event):
        return await agent.run(event.data)
    
    # 5. Cron 任务
    @cron.schedule("0 9 * * *")
    async def scheduled_trigger():
        return await agent.run({"type": "daily_report"})
```
###### 示例2:多渠道响应
```
async def send_response(result, channels):
    # 根据渠道格式化响应
    for channel in channels:
        if channel == "slack":
            await slack.send_message(
                format_for_slack(result)
            )
        elif channel == "email":
            await email.send(
                format_for_email(result)
            )
        elif channel == "api":
            return format_for_api(result)
```
###### 示例3: 统一接口
```
class UnifiedAgentInterface:
    """
    统一的智能体接口
    """
    
    async def run(self, input_data: dict) -> dict:
        """
        统一的入口点
        """
        # 1. 标准化输入
        normalized = self.normalize_input(input_data)
        
        # 2. 执行核心逻辑
        result = await self.execute(normalized)
        
        # 3. 格式化输出（根据来源渠道）
        formatted = self.format_output(result, input_data["source"])
        
        return formatted
    
    def normalize_input(self, raw_input: dict) -> dict:
        """
        标准化输入（不同渠道 → 统一格式）
        """
        source = raw_input.get("source", "unknown")
        
        if source == "slack":
            return {
                "user": raw_input["user"],
                "message": raw_input["text"],
                "context": {"channel": "slack"}
            }
        
        elif source == "email":
            return {
                "user": raw_input["from"],
                "message": raw_input["body"],
                "context": {"channel": "email", "subject": raw_input["subject"]}
            }
        
        elif source == "api":
            return raw_input  # API 已经是标准格式
        
        elif source == "webhook":
            return {
                "event_type": raw_input["event_type"],
                "data": raw_input["data"],
                "context": {"channel": "webhook"}
            }
        
        else:
            return raw_input
    
    def format_output(self, result: dict, source: str) -> dict:
        """
        格式化输出（统一格式 → 不同渠道）
        """
        if source == "slack":
            return self.format_for_slack(result)
        elif source == "email":
            return self.format_for_email(result)
        elif source == "api":
            return result  # API 返回标准格式
        elif source == "web":
            return self.format_for_web(result)
        else:
            return result
    
   
```


##### 要点总结
* **支持多种触发方式**: Agent不应该局限于单一入口，而应该支持多种触发方式。
* **统一的核心逻辑**: 虽然触发入口多样，但背后是同一个Agent实例在服务，保持状态一致性和能力一致性。
* **渠道特定的格式化**: 对各个渠道的触发和响应数据进行格式化，保持核心逻辑不变。
* （拓展）**上下文感知**: 在不同场景触发时，Agent应该能够理解当前环境的上下文。
* **（拓展）事件驱动架构**: 支持被动触发而不仅是主动调用。当某个事件发生时（如代码提交、订单创建、数据更新、时间到达）自动触发Agent执行相应任务。

本质上是"以用户为中心"的设计哲学，让技术适应人的工作习惯，而不是让人适应技术的使用方式。



## 总结
### 3.1 通过分类梳理12 Factor
| **分类**             | **功能**                 | **具体Factor**                   | **一句话总结**                             | **关键词**         |
| -------------------- | ------------------------ | -------------------------------- | ------------------------------------------ | ------------------ |
| **输入与决策**       | 如何让 AI 理解和决策     | Factor 1: 自然语言 → 工具调用    | LLM 是翻译器，不是执行器                   | 工具、JSON、验证   |
|                      |                          | Factor 2: 拥有你的提示词         | 提示词是代码，需要版本控制                 | 可见、可测、可管理 |
|                      |                          | Factor 3: 拥有你的上下文窗口     | 主动管理，不要盲目追加                     | 成本、优化、缓存   |
| **执行与控制流**     | 如何执行和控制流程       | Factor 4: 工具就是结构化输出     | 工具调用 = JSON 生成 + 代码执行            | Schema、类型安全   |
|                      |                          | Factor 8: 拥有你的控制流         | 用代码控制流程，AI 参与决策                | Switch、循环       |
| **状态管理**         | 如何管理和保存状态       | Factor 5: 统一执行状态和业务状态 | 一切皆事件，状态从事件推导                 | Event Sourcing     |
|                      |                          | Factor 6: 启动/暂停/恢复         | 智能体是可暂停的进程                       | 生命周期、异步     |
|                      |                          | Factor 12: 无状态归约器          | 智能体 = 纯函数：state + event → new_state | Redux、可重放      |
| **人类协作与可靠性** | 如何与人类协作并保持可靠 | Factor 7: 用工具调用联系人类     | 人类是特殊的工具                           | HITL、批准         |
|                      |                          | Factor 9: 压缩错误到上下文       | 500 行堆栈 → 50 tokens 摘要                | 可恢复、可理解     |
|                      |                          | Factor 10: 小型、专注的智能体    | 3-10 步的微智能体 > 单体                   | 微服务、专注       |
|                      |                          | Factor 11: 从任何地方触发        | API、Slack、CLI、Web 都能用                | 多渠道、统一接口   |

### 3.2 待优化问题 -> Factor映射
| **希望优化的问题** | **考虑实践的Factor**                                 |
| ------------------ | ---------------------------------------------------- |
| 提示词问题         | **Factor 2**: 拥有你的提示词                         |
| 需要人工监督       | **Factor 7**: 用工具调用联系人类                     |
| 调试困难           | **Factor 5**: 统一状态**Factor 3**: 拥有上下文窗口   |
| 可靠性问题         | **Factor 6**: 启动/暂停/恢复**Factor 8**: 拥有控制流 |
| 成本问题           | **Factor 3**: 上下文管理**Factor 10**: 小型智能体    |
| 智能体行为不可预测 | **Factor 1**: 结构化输出**Factor 4**: 工具验证       |
| 难以扩展           | **Factor 5：**: 统一状态**Factor 12**: 无状态归约器  |
| 错误处理混乱       | **Factor 9**: 压缩错误                               |

### 3.3 开发实现路径
```
graph LR
    Stage1[第1阶段: 基础<br/><br/>Factor 1,2,4<br/><br/>目标: 建立可靠的<br/>LLM 代码桥梁] --> Stage2[第2阶段: 控制<br/><br/>Factor 3,5,8<br/><br/>目标: 建立可调试 <br/>可控的执行]
    
    Stage2 --> Stage3[第3阶段: 生产化<br/><br/>Factor 6,9,10<br/><br/>目标: 达到生产级<br/>可靠性]
    
    Stage3 --> Stage4[第4阶段: 优化<br/><br/>Factor 7,11,12<br/><br/>目标: 优化用户体验<br/>和可扩展性]
    
    style Stage1 fill:#e1f5ff
    style Stage2 fill:#fff4e1
    style Stage3 fill:#ffe1f5
    style Stage4 fill:#e1ffe1
```




## 参考资料
### 4.1 相关仓库
1. **基于图结构的工作流引擎**: [https://github.com/smallnest/langgraphgo](https://github.com/smallnest/langgraphgo)
    1. **特性**: 并行执行、持久化、高级状态管理、预构建代理和人机交互工作流等；
    2. **语言**: Golang
    3. **相关文档**: [https://lango.rpcx.io/repowiki/zh/_content/项目概述.html](https://lango.rpcx.io/repowiki/zh/_content/%E9%A1%B9%E7%9B%AE%E6%A6%82%E8%BF%B0.html)

2. **提示词语言**：[https://github.com/boundaryml/baml](https://github.com/boundaryml/baml)
    1. **概念**: LLM Prompts就是函数；像实现函数一样实现和管理Prompts
    2. **特性**: 全面的类型安全、流处理、重试、广泛的模型支持
    3. **支持语言**: Python/TS/Ruby/Java/C#/Rust/Golang
    4. **相关文档**: [https://docs.boundaryml.com/home](https://docs.boundaryml.com/home)

3. **HITL工作流**: [https://github.com/humanlayer/humanlayer](https://github.com/humanlayer/humanlayer)
    1. **概念：自定义编排AI coding Agents，拆解解决复杂代码库的问题**
    2. **特性**: 任务分解与多 Agent 协作；强化 HITL；复杂代码库理解与上下文管理；
    3. **支持语言**: Python/TS/Golang
    4. **支持LLM： **OpenAI（GPT-4o等）、Llama、Claude
    5. **相关文档**: [https://www.humanlayer.dev/](https://www.humanlayer.dev/)




### 4.2 相关文档
1. **提示词工程指南**: [https://www.promptingguide.ai/zh](https://www.promptingguide.ai/zh)
2. 