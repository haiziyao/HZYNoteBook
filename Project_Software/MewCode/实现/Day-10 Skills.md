

> 引入,一直以来,我认为skill就是专业人士总结的prompt,可能会有自己实现的一些脚本,除此之外应该没啥了
> 我希望通过这篇文章来加深自己对skill的理解

## Skill: SOP-for-Agent

`SOP` : Standard Operating Procedure

 skill其实就是: 把一段写好的 prompt 交给Agent执行

但是具有额外的能力:
* Agent能够主动发现
* 可以携带整套资源
* inline/fork 两种执行模式


一份完整的SKILL.md主要包括
* 元信息
	* name
	* description
	* allowedTools
	* mode
* Prompt Body
	* 识别目标
	* 执行任务
	* 输出结果

```Markdown
---
name: commit
description: 分析 git diff 并生成规范的 commit
mode: inline
---

# 任务

你需要帮用户创建一个 git commit。

## 步骤

1. 运行 `git status` 查看当前变更状态
2. 运行 `git diff` 和 `git diff --staged` 查看具体变更内容
3. 分析变更，确定 commit 类型和范围：
   - feat: 新功能
   - fix: 修复 bug
   - docs: 文档变更
   - refactor: 重构
   - test: 测试
   - chore: 构建/工具变更
4. 生成 commit message，格式：`type(scope): description`
5. 用 `git add` 添加相关文件（不要添加 .env、credentials 等敏感文件）
6. 执行 `git commit -m "生成的 message"`
7. 如果用户提供了额外说明，纳入 commit message

## 注意事项

- 不要用 `git add -A`，逐个文件添加
- commit message 用英文
- description 不超过 72 个字符
- 如果变更太多，建议用户拆分成多个 commit

$ARGUMENTS
```
 
 
### Skill的优先级
和上面的记忆体系相同,SKILL也有优先级之分
* 项目级
* 用户级
* 内置级



### 注册与加载
* 轻量注册: 在注册的手只会加载元信息
* 按需加载: 当Agent判断用户意图匹配某个Skill时,调用LoadSkill工具
然后将整个SOP加载到对话中,模型在下一轮迭代时就能看到完整的SOP.


### 目录型SKILL
```Plaintext
skills/
└── backend-interview/
    ├── SKILL.md               # 入口：frontmatter + SOP 流程
    ├── examples/              # 示例输出
    │   └── sample-questions.md
    └── scripts/               # 辅助脚本
        └── validate.sh
```

和单文件 Skill 的区别在于：单文件 Skill 只有 SOP 指令本身，目录型 Skill 可以携带模板、示例、脚本和参考文档。这些附属文件不会在加载 Skill 时全部塞进上下文，Agent 按需读取，只有真正需要时才消耗 token。

目录型 Skill 还带来另一个常被低估的好处： **整套能力可打包移植** 。整个 `backend-interview/` 目录是一个自包含单元，SOP、示例、脚本、参考文档全在里面。
