
> 概述,day-1其实完成的很简陋,大致有以下几点
> 封装Message
> 封装Api
> 完成最小Agent流程

## 核心代码

### Message封装
不同的api有着不同的封装方式,我们可以选择开箱即用
但是为什么还要自己体验从http来封装呢?
其实原因仅仅在于了解最基础的东西

message包含很多东西
* role
* content
* ThinkingBlock
* ToolUseBlock
* ToolResultBlock

 这部分可以说啥也没做吧,就是定义了一下格式,如果熟悉api的请求格式应该很好写出来的

[OpenAI API 协议完全指南 - 大模型应用开发的通用标准前言 这是很多刚入门 AI 应用开发者的困惑。大家都知道 - 掘金](https://juejin.cn/post/7624184992536952867)

[大模型应用开发必读：OpenAI 接口格式全方位详解与生产最佳实践-CSDN博客](https://blog.csdn.net/lishengzhen123/article/details/161475373)
我觉得看完这些文章之后应该会更通透