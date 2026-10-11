
MCP 类似于 USB 接口. 不在赘述他是啥,反正就是一个协议而已.


## MCP组成

* 运行时角色
	* Host: 
	* Client:
	* Server:

`Coder` 作为一个`Host`,在内部建立多个`Client`和多个`Server`链接

* 协议层面
	* Data Layer(数据层):  JSON-RPC 2.0
		* 定义消息体
		* 如何连接
		* 能力
		* 工具
	* Transport  Layer(传输层): 
		*  定义消息传输的协议


### Server 

* Tools
	* JSON Schema

```JSON
{
  "name": "search_issues",
  "description": "搜索 GitHub Issue",
  "inputSchema": {
    "type": "object",
    "properties": {
      "repo": { "type": "string", "description": "仓库名，格式 owner/repo" },
      "query": { "type": "string", "description": "搜索关键词" },
      "state": { "type": "string", "enum": ["open", "closed", "all"] }
    },
    "required": ["repo"]
  }
}
```

* Resource

```JSON
{
  "uri": "db://myapp/schema",
  "name": "数据库表结构",
  "mimeType": "application/json"
}
```

* Prompts

```JSON
{
  "name": "generate_query",
  "description": "根据自然语言生成 SQL 查询",
  "arguments": [
    { "name": "table", "description": "目标表名", "required": true },
    { "name": "intent", "description": "查询意图的自然语言描述" }
  ]
}
```



### Client
* Roots: 工作目录 或 工作区边界
* Sampling: 允许 Server 反过来请求 Host 调用 LLM
* Elicitation: 允许Server 请求 Host 向用户追问 额外信息



### 补充

### Stdio 和 Streamable HTTP和 Json PRC
上面两个是标准的 MCP 传输格式定义

* stdio: Host把MCP Server 作为子进程启动,通过 stdin/stdout管道读写消息
* Streamable HTTP: MCP Server 是一个独立运行的HTTP服务,当然可以正常访问



## 一次完整的 MCP 会话
* 初始化握手: JSON RPC
* 工具发现
* 工具调用
* 请求-响应异步匹配机制



## 工具延迟加载问题
假设用户配置了 4 个 MCP Server，每个提供 15-20 个工具，加上 MewCode 自己的 6 个内置工具，工具列表一下子膨胀到 80 个。每个工具的完整 schema 包含名称、描述、参数定义和类型约束，平均占 100 到 300 个 token。80 个工具就是 8,000 到 24,000 个 token 的工具定义，每一轮对话都要带着。还没开始干正事，上下文窗口就被工具定义吃掉了一大块。

token 浪费只是表面问题。更深层的影响是模型的选择质量。当工具列表里挤满了几十个名字相似的工具（比如 Grafana 下面有 query_prometheus、query_prometheus_histogram、query_loki_logs、query_loki_stats），模型需要在这些选项里做决策。Anthropic 在工程博客 Advanced Tool Use 中公布了量化数据：50 个以上工具的场景下，启用延迟加载后 token 开销降低约 85%，Opus 4 的工具选择准确率从 49% 提升到 74%。工具太多确实会干扰模型判断。