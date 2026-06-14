---
tags:
  - agent
---

### 什么是Loop Engineering

从过去来,早期 Prompt Engineering,强调的是角色,定义操作,我们需要精确的提示词进行指导.

再到后来的Context Engineering,强调我们要提供示例,提供详尽的资料便于Agent模仿.

之后的Harness Engineering,强调我们要设计好Agent的编排,这样Agent才会有很好的能力.

现在的Loop Engineering,也算是更高一级的产物, just say less.
"你不要说那么多了,搭建好一个Loop框架,直说goal,剩下的让Agent自己loop去吧!".
总结一下,就是: 放手吧!!!


### Loop Engineering 核心概念

* *心跳*: 定义如何触发loop
* *worktree*: git的工具,避免多个Agent更改同一个文件导致错误,为每个Agent分配一个独立的工作区
* *skill*: 给Agent一些需要的能力
* *connector*: Agent与现实世界链接的接口: 数据库/日志/...
* *subAgent*: 调用子Agent进行工作
* *Memory*: 核心中的核心,是整个Agent的根基


### 实战

#### easy try in Claude

```
/loop 对于当前文件夹,你需要检查git的暂存区,找到未提交的文件,调用 connect 这个skill,建立新创建的md文件之间的联系.之后用git自动提交.
scheduled: every day
```

#### complex try in Java(搭建一个自动日志分析工具)

略.

### Loop Engineering带来了什么?

我们先泼冷水,loop Engineering不是一种新的设计范式,只是一种模式的提炼.至于我们需不需要这种范式,那就不好说了.

* 我是否有一个需要很频繁的重复性工作?
* 我是否能够设计好这个loop Agent,使之能够自动处理bad result,具备健壮性
* 我是否能抗住Token的消费
* 我的Agent是否具备了所有需要的工具:skills/MCP/tools...

Loop Engineering带来的绝对不是一场解放运动 ! ! !

对于这样的一个Agent,我们要成为一个Loop Engineering的实时维护者,不断优化保证loop的稳定性.

而不是一个 "灾难的启动者" ! ! !

---

## 相关笔记

- [[Prompt Experience]] — Agent 前沿技术与插图 prompt 模板
- [[知识库Agent搭建指南]] — 知识库 Agent 的完整搭建手册，含 7 个 Skill 定义
- [[CO-STAT实践]] — CO-STAT 架构下的 agent prompt 实践