

### 本体论是要解决什么问题?
新时代的"知识管理".
模型化构建和管理公司的知识.


### Palantir对Ontology的三层架构
- DATA层: 解决数据源的问题
	- ERP系统
- LOGIC层: Ontology语义模型
- ACTION层: 应用层

#### 举例演示

- 定义名词: 把数据翻译为业务对象
- 绑定大脑: 教会系统怎么判断
- 配置按钮: 定义谁能做什么


>  用OAG(本体增强模型) 代替 RAG





### 四道难题 和 五种应对方式
- 不知道有多少类型 
- LLM会"幻觉"
- 粒度难控制
- 怎么验证对错

- 拆解派:
	- 标准化流水线
	- 文档输入->提取实体->提取关系->合并归一->双重验证->存入图谱
- 聚类派:
	- 用一些AP聚类进行处理
	- 文档->提取名词->向量化->AP聚类->LLM命名->Ontology
- 两步走派: 两阶段生成法
- 框架派: Schema-Guided 提取法
	- 不是自己提取实体,而是从给定列表中选择
- 直给派: 端到端Prompt法
	- 直接用Prompt约定格式提取

![[Pasted image 20261001201312.png]]

![[Pasted image 20261001201333.png]]







### Palantir一堆文件如何变成Ontology

#### Objecct
* 数据连接
	* snapshot: 全量更新
	* append: 追加
* 原始存储
	* Dataset: 存储结构化表格, json/xml
	* Media Set: 存储图片,视频,音频,pdf等
	* media reference: 
* Pipeline Transforms -- 清洗和解析
	* pipeline Builder:
	* Code Repositories:
* Ontology 映射
	* 表 -> Object Type
	* 一行数据 -> Object
	* 列 -> Property
	* 外键 -> Link Type
	* 对数据的写操作 -> Action Type

#### Logic

#### Action

#### 增量更新
- 数据同步层: append拉新数据
- Pipeline层: 只处理新行
- Ontology: 只更新有变化的对象

> 常见陷阱: 1. 源数据可以被修改,如果采用增量同步. 可能同时存在两个状态的数据,需要去重
> 2. Action是立即修改的,但是Pipeline是定时批量运行的. 但是中间会有一个时间差

> 一张超大宽表怎么处理: 


### Agent 与 Ontology 的交互分层
* Context Layer: 系统确定性触发, 让Agent看见相关上下文
* Query Layer: LLM自主决策, 让Agent主动查询对象和关系
* Logic Layer: LLM自主决策, 让Agent调用函数,模型和业务逻辑
* Action Layer: LLM决策+可选用户确认,让Agent执行受控业务动作
* governance Layer: 约束Agent的权限, 审计和安全边界



### 一些开源项目

OntoFlow
OntoEKG
nano-ontoprompt


























