 对于一些**触发条件明确，执行动作固定**的操作.
 我们可以自动化.只要满足某些条件就自动触发
 这就是HooK

```YAML
hooks:
  - event: post_tool_use       # 事件：工具执行之后
    if: tool == "WriteFile"    # 条件：只在写文件时触发
    action:                     # 动作：执行什么
      type: command
      command: "lint $FILE_PATH"
```

### Hook在哪个声明周期被触发?
* 会话级事件:
	* 比如`session_start` `session_end`. 每次在有新对话的时候自动触发
* 轮次级事件:
	* turn_start 在用户发送新消息时触发，标志着一轮对话的开始；turn_end 在 Agent 完成回复时触发，标志着一轮对话的结束
* 工具级事件:
	* `pre_send` 在消息发送给 LLM 之前触发， `post_receive` 在收到 LLM 响应之后触发。这两个事件可以用来做消息的预处理和后处理。
* 系统级事件: 
* 特殊事件:
	* `permission_request`
	* `file_change`
	* `command_execute`

匹配规则

```YAML
# 工具名精确匹配
if: tool == "Bash"

# 工具参数的正则匹配
if: tool == "Bash" && args.command =~ /rm\s+-rf/

# 文件路径的 glob 匹配
if: tool == "WriteFile" && args.path ~= "*.py"

# 组合条件
if: tool == "WriteFile" && args.path ~= "*.proto"

# 反向匹配：除了 ReadFile 之外的所有工具
if: tool != "ReadFile"
```


### Hook执行器
* command

```YAML
action:
  type: command
  command: "prettier --write $FILE_PATH"
  timeout: 10s
```


* prompt

```YAML
action:
  type: prompt
  message: "请先阅读 ARCHITECTURE.md 了解项目结构，然后再开始工作。"
```

>这里可以思考一下:这个prompt会加在哪里?
>绝对不会是System Prompt, 因为缓存命中的问题
>所以应该是作为一条user信息用 `<system-reminder>`包裹


*  http

```YAML
action:
  type: http
  url: "https://hooks.slack.com/services/xxx"
  method: POST
  body: '{"text": "MewCode: Agent 修改了 $FILE_PATH"}'
```
* agent

```YAML
action:
  type: agent
  prompt: "请检查刚才写入的文件 $FILE_PATH 是否有安全漏洞。"
```


### 执行控制

> 调用Hook之后,Hook真的能被执行吗?

* once: 只执行一次
* async: 异步执行
* 错误兜底: 


![](https://my.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDAxNmQxNjY4ZGMyNDhiNDdjOWJlMWVmOWJhYWEzZjZfeUZGaTlwS1M5NklaeEN0NWV1ZVk4ZkltYUFBSFdyY3ZfVG9rZW46RlVhUWJEcEFhb3VuU2d4Tjh4OWNhSExXbm1RXzE3ODQ4OTQ0MzI6MTc4NDg5ODAzMl9WNA&add_watermark=true&scene_type=CCM)