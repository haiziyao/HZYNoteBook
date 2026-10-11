
引入: 有了工具系统之后的对话,AI不会自己loop怎么办


## ReAct范式

```Plaintext
Think: 用户想写 HTTP 服务器，先看看项目里有哪些文件。
Act:   Glob(pattern="**/*")
Observe: main.py, handler.py, requirements.txt

Think: 已经有 handler.py 了，看看现有路由怎么组织的。
Act:   ReadFile(path="/project/handler.py")
Observe: from flask import Flask ... def handle_health(): ...

Think: 用 Flask，加新路由就行。改完编译看看。
Act:   EditFile(path="/project/handler.py", ...)
Observe: 文件修改成功
Act:   Bash(command="python -m py_compile handler.py")
Observe: exit code: 0
```

|   |   |   |   |
|---|---|---|---|
|范式|核心思路|优点|局限|
|Chain-of-Thought|只推理，不行动|推理质量高|无法与环境交互|
|Act-only|只行动，不推理|执行快|盲目调工具，容易出错|
|ReAct|推理与行动交替|两全其美：想清楚再做|每轮都要一次 LLM 调用，成本较高|
|Plan-then-Execute|先出完整计划，再逐步执行|全局规划好|计划可能过时，不如边走边看灵活|


### Agent Loop 
就是一个一个while循环
``` python
function agentLoop(userMessage):
    messages = [...历史消息, userMessage]
    while true:
        response = callLLM(systemPrompt, messages, tools)
        if response 没有 tool_use:
            return response
        messages.append({role: "assistant", content: response.content})
        results = []
        for each toolUse in response.toolUses:
            result = executeTool(toolUse.name, toolUse.input)
            results.append(tool_result(toolUse.id, result))
        messages.append({role: "user", content: results})
```

### 如何停下来
如果模型进入了某种循环状态，比如它反复调用 ReadFile 读同一个文件，每次都觉得还需要再看一遍，那这个循环就永远不会结束。或者更糟糕的场景，工具执行总是返回错误，模型总是重试，API 费用蹭蹭往上涨，而你还在傻等。
1. 模型主动回答是否做完: 如果api返回结果有结束标记符号,并且没有tool_use, 那么表明确实已经结束了
2. 迭代上限: 达到了一个最大循环次数
3. 用户取消: 用户主动终止了当前循环


### AgentEvent设计
在loop阶段,我们肯定更倾向于去看一些执行过程

|   |   |   |
|---|---|---|
|事件类型|含义|携带的数据|
|stream_text|模型正在输出的文字增量|一小段文本|
|tool_ use|模型请求调用工具|工具名、工具输入、请求 ID|
|tool_ result|工具执行完成|执行结果、是否出错、耗时|
|turn_ complete|一轮 LLM 调用完成|当前轮次序号|
|loop_ complete|整个循环结束|总轮次|
|usage|Token 用量更新|累计输入/输出 token 数|
|error|发生错误|错误信息|


### 状态机 与 工具分批
对于工具执行的状态机这里指的是: Loop中的状态
* 是否继续执行: 看 tool_use是否有新的内容
* 工具执行是否需要鉴权: 给不给bash权限
* 限流等其他状态


分批逻辑这里指的是: 工具可以串行执行，但模型一次可能返回多个工具调用
所以我们做分批处理:
* 安全工具 并发执行
* 不安全的工具 串行职系那个
``` python
function partitionToolCalls(toolUses, registry) -> Batch[]:
    batches = []
    for tc in toolUses:
        tool = registry.get(tc.name)
        safe = tool is not null and tool.isConcurrencySafe(tc.input)

        if safe and batches is not empty and batches.last().isConcurrencySafe:
            batches.last().calls.append(tc)
        else:
            batches.append(Batch(isConcurrencySafe=safe, calls=[tc]))
    return batches
```

### System prompt与 LoopAgent
对于一个Agent Loop来说, sys prompt是尤为重要的
 我们需要额外记录比较多的附加信息
 * 角色设定
 * 环境信息
 * 模式指令

``` python
function buildSystemPrompt(config):
    parts = []

    // 角色设定
    parts.append("你是 MewCode，一个终端环境中的 AI 编程助手。
    你擅长阅读代码、编写代码和调试问题。
    你会先思考再行动，每一步都解释你的推理过程。")

    // 环境信息
    parts.append("# Environment")
    parts.append("当前工作目录：" + config.workDir)
    parts.append("操作系统：" + getOS())

    // 模式特定指令
    if config.planMode:
        parts.append(PLAN_MODE_INSTRUCTIONS)

    return parts.join("\n\n")
```

环境信息很容易被忽略，但它非常重要。如果模型不知道当前工作目录在哪里，它执行命令的时候就不知道该用绝对路径还是相对路径。如果不知道操作系统是什么，它可能在 Linux 上给你写 Windows 的命令。 这些信息对人类来说是不言自明的，但模型需要你明确告诉它。工作目录和 OS 在一次会话内不会变，放在 system 里正好可以利用 Prompt Cache。


### Plan Mode
我们刚才了解到,只要调用api返回的结果里面有tool_use信息,模型就会一直运行下去,就会变成Agent Loop.
那么我们如何实现Plan Mode呢?

> 直接把tools信息不注入request不就行了.

上面虽然是一个正确的想法,但是不太可能,比如,把工具ban了之后,模型连最简单的`Read`功能都没有了.

> Plan 不能只是 "空想", 也需要一些 tool_use

而实际上,Plan Mode的实现核心是通过Prompt指令约束模型行为的
```Plaintext
Plan mode is active. 你不能执行任何修改操作，不能编辑文件、不能提交代码、不能修改配置。
唯一可以写入的文件是下面指定的 plan file。

你的工作流程：
1. 用 ReadFile、Grep、Glob、Bash（只读命令）探索代码
2. 分析用户需求，设计实现方案
3. 把计划写入 plan file
4. 等待用户确认后再执行
```

Plan Mode 下的权限矩阵和 Default 模式 **完全一致** （read=allow, write=ask, command=ask）。权限系统唯一的特殊处理是 plan 文件自动放行，不需要用户确认。如果 LLM 不听 prompt 的话尝试写非 plan 文件，用户会看到和正常模式一样的权限确认框，由用户决定是否批准。

>你可能会问：既然权限没变，Plan Mode 的安全保障是不是太弱了？实际上 prompt 约束对目前主流的 LLM 已经非常有效。大部分模型在收到「不要用写工具」的指令后，几乎不会主动尝试调用写工具。权限系统保持 ask 而不是 deny 的好处是灵活性：万一你在规划过程中确实需要让 Agent 做一些小修改（比如写 plan 文件的同时顺手创建一个测试数据文件），你可以手动批准，不会被硬性拦截卡住。