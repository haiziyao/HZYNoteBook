
"学习软件工程 是 为了 优雅的工程"



## LLM 调用基础知识

[A/家的  API 协议 文档](https://docs.anthropic.com/en/api/messages)

##### Message格式
- messages : arr
	- role
		- user
		- assistant
	- content : arr
		- text
		- tool_use


##### 流式输入 SSE
```Plaintext
message_start          整个响应开始，带着 input_tokens 信息
  └─ content_block_start   一个内容块开始（文本或工具调用）
       └─ content_block_delta  内容增量，文字一个词一个词地到达
  └─ content_block_stop    一个内容块结束
message_delta            消息级别的增量（output_tokens、停止原因）
message_stop             整个响应结束
```


##### example
```JSON
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 4096,

  "system": "你是 MewCode，一个终端环境中的 AI 编程助手。\n\n# Environment\n当前工作目录: /home/dev/myproject\n操作系统: Linux",

  "messages": [
    {"role": "user", "content": "帮我读一下 app.py 的内容"},
    {"role": "assistant", "content": "好的，我来读取 app.py 的内容。\ndef main():\n    print(\"hello\")\n\nif __name__ == \"__main__\":\n    main()"},
    {"role": "user", "content": "这个文件里有什么函数？"}
  ],

  "tools": [
    {
      "name": "read_file",
      "description": "读取指定路径的文件内容",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": {"type": "string", "description": "文件路径"}
        },
        "required": ["path"]
      }
    }
  ]
}
```

##### Token
Token 是 LLM 的计费单位。粗略来说，英文每个单词大约 1-2 个 token，中文每个字大约 1-2 个 token。
```JSON
"usage": {
  "input_tokens": 10,   // api 中 发送给 LLM的
  "output_tokens": 12  // LLM吐出来的
}
```

多轮对话的场景, 每一次请求都把完整的历史对话发送过去, 如果不做好上下文压缩,那么每次请求的开销 真的是"巨大"


##### Extended Thinking： 让模型先想再说
Claude 支持 Extended Thinking，让模型在正式回复之前先进行一轮内部推理。开启后响应的 content 数组里会多一个 `thinking` 类型的内容块，排在 `text` 块之前。

##### 封装
这个就没啥好说的,把不同厂家的api用策略模式封装起来就好了

##### 从单轮到多轮
在客户端维护完整的消息列表，每次用户发消息、模型回复，都要记录下来。

##### 消息模型: 两层设计

- API层: 
	- 设计简洁: role+content
- 内部层:
	- 角色扩展: user assistant system tool
	- 消息唯一ID
	- ToolUsages
	- ToolResults
	- ThinkingBlocks

上述二者可以通过JsonFormat相互转换
##### 对话管理器
处理流式接受和UI渲染的并发问题

##### 格式转换
对话管理器里最关键的方法是 `toAPIFormat()` ：把内部层的消息列表转换成 API 层的格式。
* 过滤: system或者error
* 合并: 合并相邻的同角色消息
* 保证: 确保首条消息是 user
过滤掉 system 和 error 消息，转成 API 格式，合并相邻同角色，确保首条为 user、两种角色交替出现。

##### 流式更新与多轮协作
先占位，再填充」。用户发消息后，对话管理器先创建一条空的 assistant 消息当占位符，状态标记为「正在输出」。然后一边接收流式事件，一边往这条消息里追加内容。等流式结束，把状态改成「完成」，记录 token 用量。这条消息就自然成了对话历史的一部分，下一轮请求会带上它。

##### 终端页面





 