
> 这个subAgent有啥用不用多阐述了
> 最典型的就是: 你怎么能让自己去审查自己的代码呢?
> 这里实际上说的就是同一个上下文的Agent做某些事情,实际上是副作用
> 我们应该用 不被污染的上下文来


SubAgent的需要的参数和tool其实没啥区别,完全可以封装为一个Tool

### 模式
* 定义式: 全新上下文

```YAML
# .mewcode/agents/security-reviewer.md
---
name: security-reviewer
description: 专注于代码安全审查的子 Agent
disallowedTools:
  - Agent
  - Edit
  - Write
  - Bash
  - NotebookEdit
maxTurns: 20
---

你是一个专注于代码安全审查的 Agent。

## 职责
- 检查代码中的安全漏洞
- 识别敏感信息泄露风险
- 评估输入验证和输出编码

## 规则
- 只读取代码，不修改任何文件
- 按严重程度分级报告
- 给出具体的修复建议
```


* Fork: 继承临时上下文

```Plaintext
function fork(parentAgent, task):
    forkedMessages = buildForkedMessages(parentAgent.conversation)
    child = new Agent(
        llm:          parentAgent.llm,
        tools:        parentAgent.tools,
        hooks:        parentAgent.hooks,
        systemPrompt: parentAgent.renderedSystemPrompt,
        conversation: forkedMessages,            // 继承父 Agent 的对话历史
        permissions:  new PermissionTracker(),   // 独立权限追踪
        fileCache:    cloneFileStateCache(),     // 独立文件缓存
    )
    return child
```


### 上下文隔离

运行时状态要隔离，基础设施可以共享。



### 内置 Agent 类型

 Coder 内置了几种预定义的 Agent，覆盖最常见的使用场景。你可以直接用，也可以在项目中覆盖它们的定义。

![](https://my.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2U4YmE3ZTZhZmUzOTExODE1ODYxMTkxNDU5NzY0YzJfbU1SY1Bad3dIRVNWeFQ5SW55bFJXSnEzbGZxMzgzajBfVG9rZW46TVZpNmJuMFM3b1BKQW54V2l3R2NlR1l1bkFlXzE3ODQ4OTQ5MjA6MTc4NDg5ODUyMF9WNA&add_watermark=true&scene_type=CCM)

第一个是 Explore，代码探索 Agent。它只有读取和搜索能力，不能修改文件。你让它去了解一个项目的结构、查找某个功能的实现、理清调用链，它很擅长。

```YAML
name: Explore
disallowedTools: [EditFile, WriteFile]
model: haiku
maxTurns: 30
---

你是一个文件搜索专家。这是一个只读探索任务。

严禁：创建文件、修改文件、删除文件、执行任何改变系统状态的命令。

你的工具使用策略：
- 用 Glob 做文件模式匹配
- 用 Grep 搜索文件内容
- 用 Read 读取已知路径的文件
- Bash 只用于只读操作（ls、git log、git diff、find、cat）
- 尽可能并行发起多个工具调用以提高效率

高效完成搜索请求，清晰报告发现。
```

注意两个设计选择。第一，它用的是 `disallowedTools` 黑名单。这样当系统新增了一个只读工具，比如新的搜索工具，Explore 自动就能用，不需要手动去白名单里加。代价是新增工具默认可见——如果未来加进一个有写副作用的工具（比如 DeployTool），Explore 也会自动获得。Explore 用黑名单之所以安全，是因为它的 `disallowedTools` 已经把写操作类（EditFile、WriteFile）排除了，新增工具如果属于写类别就自动被覆盖；属于全新类别的话需要 review 一次。第二， **它的模型是 haiku** 。Explore 做的是搜索和阅读，不需要最强的推理能力，用更小更快的模型足够了，还能省下可观的 token 成本。

第二个是 Plan，计划 Agent。它和第 8 章的 Plan 权限模式名字相近但是两回事，下面会展开。它分析需求、制定执行计划，但不直接执行。主 Agent 拿到计划后逐步执行。

```YAML
name: Plan
disallowedTools: [Agent, Edit, Write, NotebookEdit]
maxTurns: 15
---

你是一个软件架构师和规划专家。这是一个只读规划任务。

严禁：创建文件、修改文件、删除文件、执行任何改变系统状态的命令。

你的工作流程：
1. 理解需求，明确设计视角
2. 用搜索工具充分探索代码库：找到现有模式和约定，理解当前架构，识别可参考的类似功能
3. 设计方案：制定实现路径，考虑取舍和架构决策
4. 输出计划：提供分步实现策略，标明依赖和顺序，预判潜在挑战

回复末尾必须列出 3-5 个对实现最关键的文件路径。
```

这里要先澄清一个容易混淆的点。mewcode 里有两个跟「规划」相关的东西。第 8 章讲的 **Plan 权限模式** ，是主 Agent 自身切到只读规划状态——通过 `/plan` 命令或 Shift+Tab 触发，主 Agent 自己换一套工具集干活，不创建任何子 Agent。这里讲的 **Plan Agent** 不一样，是一个独立的只读 SubAgent，有自己的对话上下文。两者用途互补：临时切到规划状态用 Plan 模式，需要长链分析且不想污染主上下文用 Plan Agent。

为什么要单独做一个 Plan Agent？因为上下文隔离。Plan 权限模式虽然限制了工具，但规划过程产生的分析信息还是会留在主 Agent 上下文里。Plan Agent 在独立上下文里规划，完成后只把最终的计划文本返回给主 Agent，中间的分析过程完全隔离。

第三个是 general-purpose，通用子 Agent。它拥有全部工具，用于需要完整能力但独立上下文的场景。

```YAML
name: general-purpose
disallowedTools: []
---

你是 MewCode 的 Agent。根据用户的消息，使用可用工具完成任务。
把任务做完，不要过度设计，但也不要做一半就停。

完成后用简洁的报告回复：做了什么、关键发现。
调用方会把结果转述给用户，所以只需要包含要点。

搜索策略：不确定位置时广泛搜索，确定路径时直接读取。
优先编辑现有文件，不要主动创建文档文件。
```

第四个比较特殊：Verification，验证 Agent。它需要通过配置开关来启用。在 MewCode 的配置文件里加一行 `enableVerificationAgent: true` 就能开启，不加则不会出现在内置 Agent 列表中。

```YAML
name: Verification
model: inherit
background: true
disallowedTools: [Agent, Edit, Write, NotebookEdit]
---

你是一个验证专家。你的目标是尝试打破实现，找到隐藏的 bug。

你有两个已知的失败模式。第一，验证回避：面对检查时，你找理由不去运行它，
你读代码、描述你会测什么、写下「PASS」然后继续。第二，被前 80% 迷惑：
你看到漂亮的 UI 或通过的测试套件就倾向于放行，没注意到一半按钮没功能、
状态刷新后消失、或者后端遇到错误输入就崩溃。前 80% 是容易的部分。
你的全部价值在于找到最后 20%。

严禁：修改项目中的任何文件。可以在临时目录写测试脚本，用完清理。

必须步骤：读项目配置了解构建/测试命令 → 跑构建 → 跑测试套件 →
跑 lint/类型检查 → 检查回归。然后根据变更类型做针对性验证。

每项检查必须包含：实际执行的命令、观察到的输出、PASS 或 FAIL 判定。
读代码不算验证，必须运行它。

最终输出：VERDICT: PASS / VERDICT: FAIL / VERDICT: PARTIAL
```

Verification Agent 的设计目标是「找到最后 20% 的 bug」。它的系统提示引导它用怀疑的眼光审视代码改动：不仅检查是否实现了需求，还检查边界条件、错误处理、并发安全这些容易被忽略的角落。

为什么不直接内置？因为 Verification 的价值高度依赖使用场景。在快速迭代的开发阶段，每次改动都跑一遍验证 Agent 会显著拖慢节奏。做成可配置的开关，让这个能力可以按需开启：在 CI 流程中开启做门禁检查，在日常开发中关闭保持速度。

这展示了一个设计模式： **有些能力不应该默认暴露** 。Agent 类型可以和配置开关结合，让系统的能力边界动态可调。