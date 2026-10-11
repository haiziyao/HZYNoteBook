
# 理论学习

## 工具调用全过程

1. 我们每次访问告诉模型我们有哪些工具
```JSON
{
  "tools": [{
    "name": "ReadFile",
    "description": "读取指定路径的文件内容。返回带行号的文件文本。路径必须是绝对路径。",
    "input_schema": {
      "type": "object",
      "properties": {
        "path": {
          "type": "string",
          "description": "文件的绝对路径"
        }
      },
      "required": ["path"]
    }
  }]
}
```

2. 模型自己决定调用哪些工具后会返回结果
```JSON
{
  "role": "assistant",
  "content": [
    {"type": "text", "text": "让我读取这个文件的内容。"},
    {
      "type": "tool_use",
      "id": "toolu_abc123",
      "name": "ReadFile",
      "input": {"path": "/home/user/project/main.py"}
    }
  ]
}
```
3. 执行工具然后把结果告诉模型
```JSON
{
  "role": "user",
  "content": [{
    "type": "tool_result",
    "tool_use_id": "toolu_abc123",
    "content": "1\tdef main():\n2\t    print('hello')\n3\t"
  }]
}
```
4. 模型继续
模型收到工具结果后，有两种可能。一是它觉得信息够了，直接给出最终回复。二是它觉得还需要更多信息，再请求调用另一个工具。比如它先搜索找到相关文件，再读取内容，再修改代码。


>到这里,我们可以回答之前的问题了,为什么Message要这么设计
>也就是为什么Message下面有
>ToolUses, ToolResults

**本质是: 模型负责决策,我们负责执行,结果反馈给模型**


## 如何制作"工具"

1. **工具守则**

|   |   |   |
|---|---|---|
|信息|说明|示例|
|做什么|工具的核心功能|「读取指定路径的文件内容」|
|什么时候该用|典型使用场景|「需要查看文件完整内容或特定行时使用」|
|什么时候不该用|反模式提示|「二进制文件不可读取，请改用 Bash」|
|参数约束|输入的限制条件|「路径必须是绝对路径」|
|返回格式|输出长什么样|「返回带行号前缀的文本」|
|与其他工具的配合|工作流建议|「大文件建议先 Grep 定位再 ReadFile」|
所以写工具的时候，不要觉得代码逻辑最重要。花在描述上的时间，比花在代码上的时间更值得。

2. **接口设计**
``` java
工具接口 {
    name() -> string
    description() -> string
    inputSchema() -> JSON Schema
    execute(context, input) -> ToolResult
    isReadOnly() -> boolean
    isDestructive() -> boolean
    isConcurrencySafe(input) -> boolean
    category() -> string
    validateInput(input) -> error or null
}
```

3. **结果**
```Plaintext
ToolResult {
    content: string           // 返回给模型的文本内容
    isError: boolean          // 标记为错误结果
    metadata: map             // 额外信息（给 UI 用，不发给模型）
}
```


## 工具注册中心

这个很好理解,不能自己管理工具,肯定要写一个Register来管理

``` python
function setupTools(registry, config):
    // 基础读取工具始终启用
    registry.register(newReadFileTool())
    registry.register(newGlobTool())
    registry.register(newGrepTool())

    // 写操作需要显式开启
    if config.allowWrite:
        registry.register(newWriteFileTool())
        registry.register(newEditFileTool())

    // Bash 最危险，单独授权
    if config.allowBash:
        registry.register(newBashTool(config.bashTimeout))
```


## 核心工具设计
看起来,tools的设计十分简单,但是并非

### ReadFile
1. 读文件一定要用行号包裹再返回:
2. 大文件支持limit和offset:
3. 二进制文件拒绝读:

### WriteFile
1. write是覆盖写入,不是追加:
2. 如果父目录不存在怎么办:
3. 写入成功的返回值必须是: 写入N字节到path中

### EditFile
1. 相较于writefile的全覆盖,用editfile的替换更省token
2. 给出old_string和new_string
3. 如何唯一匹配old_string
	1. 找到1个old_string: 直接执行
	2. 找到0个: 报错,重新read
	3. 找到N个: 报错,需要更多上下文

### Bash
1. 先来点高血压的东西:
	1. `rm -rf /`
	2. `    ` (卡住了)
2. 命令退出码的语义解读

> 这个一看血压就大

### Glob
>Glob 就是用 `*`、`?`、`[]`、`**` 等通配符，批量匹配文件名和路径的规则。

1. 自动排除没意义的目录: `.idea` `.git` 
2. 结果按照时间修改倒序排列 `LRU` 
3. 如果结果过多怎办,不能一下子吐出来全部

### Grep

> Glob找文件name, Grep获得 content



|           |        |     |     |                    |
| --------- | ------ | --- | --- | ------------------ |
| 工具        | 分类     | 只读  | 破坏性 | 典型场景               |
| ReadFile  | file   | 是   | 否   | 查看文件内容、读取配置        |
| WriteFile | file   | 否   | 否   | 创建新文件、覆盖写入         |
| EditFile  | file   | 否   | 否   | 精确修改文件某几行，节省 token |
| Bash      | shell  | 否   | 是   | 编译、测试、安装依赖、执行命令    |
| Glob      | search | 是   | 否   | 了解项目结构、查找特定类型文件    |
| Grep      | search | 是   | 否   | 搜索代码中的函数定义、变量引用    |


# 实现: 集成到LLM