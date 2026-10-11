
Git Worktree 是 Git 2.5 引入的功能。一句话概括它做的事情：允许你在同一个仓库中创建多个独立的工作目录。
```Bash
# 在当前仓库旁边创建一个新的工作目录
git worktree add ../my-project-feature-a feature-a

# 现在有两个工作目录：
# ./my-project/          -> main 分支
# ./my-project-feature-a/ -> feature-a 分支
```


### 创建WorkTree
* validateSlug
* 检查重名
* 构建路径
* git worktree add 
* 创建后初始化环境
* 记录状态

### 初始化 WorkTree

* 本地配置文件: setting.local.json
* git Hooks配置: 继承hooks_work_path
* 软连接大目录: node_modules(所以推荐用pnpm代替npm)
* 复制被忽略但需要的文件: .gitignore 里面的一些文件需要复制过来

### 进入WorkTree
最直接的想法是：把进程的工作目录 `chdir` 到 Worktree 路径，后面 Agent 的所有文件操作自然就发生在 Worktree 里了。听起来很顺，但 MewCode 实际上选了另一条路—— **不动进程级 cwd，而是把 Worktree 路径记到会话状态里，让每次工具调用自己显式取 cwd** 。


### 退出WorkTree


Agent 在 Worktree 里完成工作后，需要切回主目录。退出时有一个选择：保留这个 Worktree 还是删除它？

如果选择删除，还要考虑一个问题：Worktree 里有没有未保存的修改？

这就是退出时的 **变更保护** 。


### 自动清理WorkTree



### SubAgent+WorkTree
无敌了