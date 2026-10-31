---
title: jev 决策模型 - 学习与调用手册
created: 2026-10-27
tags:
  - jev
  - 决策模型
  - Atlas
  - 技术学习
  - Python评测
---

# jev 决策模型 - 学习与调用手册

> 用途：搞懂 jev 是什么、什么时候该用、怎么在 atlas 里调、怎么用 Python 做效果评测
> 代码基线：`atlas` 仓 `master`（含 `0e3abb85` jev 集成）
> 项目内文档：`docs/specs/jev决策模型集成-技术方案.md`、`docs/rules/jevUtil使用手册.md`
> 相关：[[Atlas知识生产-管理-消费全景]]、[[项目串讲]]

---

## 1. 一句话定位

**jev 是一个决策/分类模型（System One），不是对话模型。**
你给它一段文本 `state` 和一组带类型的 `questions`，它给你带概率的结构化答案；
它**一个字都不生成**。

心智模型：把「写不出一条清晰 if 的判断」外包给一个概率模型，阈值仍由你在代码里定。

---

## 2. 是什么 / 不是什么

| 是 | 不是 |
|----|------|
| 决策/分类：`state` + `questions` → 概率化答案 | 对话模型：不能让它「回一句话给用户」 |
| 端点 `POST {BASE}/v1/systemone` | 不能走 `/v1/chat/completions`（报 `no cluster found for model: jev-1.14.0`） |
| 确定性输出：同输入重复调用结果恒定 | 不能当 Claude Code 的 `/model` 用 |
| 每次固定开销约 270–300 token | 不是 embedding / 检索模型 |

> 关键区别：**jev 给概率，不给结论**。`noul` 返回 0.87，是不是「是」由你的阈值决定。

---

## 3. 三种原语（先记这三个就够）

| 原语 | 回答的问题 | 返回 | 典型用法 |
|------|-----------|------|---------|
| `noul` | 是不是？ | 命题成立概率 `[0,1]` | 是否明确要求退款、是否垃圾推广 |
| `choice` | 是哪一个？ | `choice` + `confidence` + `probabilities` 分布 | 工单该转给哪个团队 |
| `score` | 有多严重？ | 概率加权期望值（可小数）+ 分布 + `legend` | 用户情绪激烈程度 |

两个容易忽略的点：

1. **`choice` 不要只看 `choice`**，要看 `confidence` 和第二名概率。第一名 0.51、第二名 0.49，
   说明模型也分不清，这时应该转人工而不是硬路由。
2. **`score` 是期望值**，可能是 `2.09` 这种两档之间的数；`legend` 是档位说明回显。

---

## 4. 什么时候该用 jev

团队约定（来自 `docs/rules/jevUtil使用手册.md`）：

> 当业务逻辑遇到**难以准确定义判断、选择规则**时，可以优先用 jev 验证并实现。

命中任意一条就值得考虑：

| 信号 | 说明 |
|------|------|
| 规则写不出一条清晰的 `if` | 要靠语义、语气、意图分流，代码里只能堆关键词 |
| 关键词/枚举匹配总漏 | 同义表达、否定句、跨语言（「我不要求退款」也能命中「退款」） |
| 规则数量爆炸 | 为长尾 case 不断加词表，改一次回归一片 |
| 需要程度而不是二分类 | 给严重程度/置信强弱打分再分流 |
| 标注样本好拿、规则不好写 | 能举出 20 个正反例，但写不出规则 |

**不适合用 jev 的场景**（继续用 if / 规则 / 枚举）：

- 规则本身清晰：状态机、数值阈值、非空校验、格式校验、权限判断
- 强实时且不能有外部依赖：jev 是网络调用，正常几百毫秒，超时上限 10s
- 需要生成文本或解释：jev 不生成任何文本
- 判错代价极高且无法人工兜底：概率模型必须有阈值和兜底策略

---

## 5. 调用方式 A：HTTP 接口（Python 评测用）

atlas 提供了一个免登的临时验证接口：

```
POST http://pre-atlas.bilibili.co/atlas/open/jev/ask
Content-Type: application/json
```

请求体与 jev 原生结构一致：

```json
{
  "state": "我上周买的会员，今天看还在扣费，我要退款，还要投诉你们。",
  "questions": {
    "wants_refund": { "type": "noul", "instructions": "用户是否明确要求退款" },
    "team": {
      "type": "choice",
      "instructions": "该问题应转给哪个团队处理",
      "criteria": { "billing": "费用、扣费、退款、发票相关", "tech": "登录、播放、客户端故障相关" }
    },
    "anger": {
      "type": "score",
      "instructions": "用户情绪激烈程度",
      "criteria": ["平静", "略有不满", "明显愤怒", "激烈投诉"]
    }
  }
}
```

响应：

```json
{
  "code": 200,
  "data": {
    "model": "jev-1.14.0",
    "costMs": 886,
    "inputTokens": 296,
    "outputTokens": 23,
    "answers": {
      "wants_refund": { "type": "noul", "noul": 0.98 },
      "team": { "type": "choice", "choice": "billing", "confidence": 1.0, "probabilities": {...} },
      "anger": { "type": "score", "score": 2.09, "confidence": 0.55, "probabilities": {...}, "legend": {...} }
    }
  }
}
```

接口护栏：

| 项 | 规则 |
|----|------|
| 鉴权 | 免登（`/atlas/open/jev/**` 白名单），定位是临时验证工具 |
| `state` 长度 | ≤ 20000 字符 |
| 问题数量 | ≤ 10 |
| 业务错误 | HTTP 200 + `code=400` + message，例如 `state 不能为空` |

### 5.1 用本机 Python 工作区跑

工作区：`~/Developer/Project-Data/atlas-api-eval`

```bash
cd ~/Developer/Project-Data/atlas-api-eval

# 单次调用，终端表格展示
uv run jev ask --file examples/jev-refund.json

# 只想看原始 JSON
uv run jev ask --file examples/jev-refund.json --json

# 临时拼一个是非题
uv run jev ask --state "我要退款" --noul "用户是否明确要求退款"

# 批量评测
uv run jev batch --file examples/jev-cases.jsonl --out results.csv
```

批量用例是 JSONL，每行一条，`expect` 可选：

```json
{"id":"refund-1","state":"我要退款","questions":{"wants_refund":{"type":"noul","instructions":"用户是否明确要求退款"}},"expect":{"wants_refund":0.5}}
{"id":"no-refund-1","state":"会员什么时候到期？","questions":{"wants_refund":{"type":"noul","instructions":"用户是否明确要求退款"}},"expect":{"wants_refund":{"max":0.5}}}
{"id":"team-1","state":"视频一直加载失败","questions":{"team":{"type":"choice","instructions":"该问题应转给哪个团队处理","criteria":{"billing":"费用","tech":"播放故障"}}},"expect":{"team":"tech"}}
```

`expect` 判定：

| 写法 | 含义 |
|------|------|
| 标量（noul/score） | 结果 ≥ 该值 |
| 标量（choice） | 字符串严格相等 |
| `{"min": x}` / `{"max": x}` | 区间约束；负例通常写 `{"max": 0.5}` |
| `{"eq": x}` | 精确相等 |

输出 CSV 列：`id / model / costMs / inputTokens / outputTokens / answers / expect / passed / error`。

### 5.2 在 Python 代码里直接用

```python
from atlas_api_eval import ask

data = ask({
    "state": "我要退款",
    "questions": {"q": {"type": "noul", "instructions": "用户是否明确要求退款"}},
})
print(data["answers"]["q"]["noul"])   # 0.98
```

`ask()` 会在发请求前做本地校验（state 非空、长度、问题数、type、criteria 形状），
业务错误统一抛 `JevError`。

### 5.3 curl 版

```bash
curl -s -X POST 'http://pre-atlas.bilibili.co/atlas/open/jev/ask' \
  -H 'Content-Type: application/json' \
  -d '{"state":"我要退款","questions":{"q":{"type":"noul","instructions":"用户是否明确要求退款"}}}'
```

---

## 6. 调用方式 B：Java `JevUtil`（业务代码内嵌）

静态门面，业务一行调用。核心 API：

| 方法 | 返回 |
|------|------|
| `JevUtil.noul(state, instructions)` | `double` 概率 |
| `JevUtil.noul(state, instructions, threshold)` | `boolean` |
| `JevUtil.choice(state, instructions, Map<String,String>)` | `JevAnswer.Choice` |
| `JevUtil.score(state, instructions, List<String>)` | `JevAnswer.Score` |
| `JevUtil.ask(state, Map<String,JevQuestion>)` | `JevResult`（一次多问） |
| `JevUtil.tryNoul / tryChoice / tryScore / tryAsk` | `Optional…`，失败不抛异常 |
| `JevUtil.pairs("k1","说明1","k2","说明2")` | 构造 choice 选项 |
| `JevUtil.questions("id1", q1, "id2", q2)` | 构造多问清单 |

示例：

```java
double p = JevUtil.noul(state, "用户是否明确要求退款");
if (p > 0.9) { ... }

JevAnswer.Choice team = JevUtil.choice(state, "该工单该由哪个团队处理",
        JevUtil.pairs("billing", "扣费、发票、退款",
                      "technical", "功能坏了或行为异常"));
if (team.getConfidence() < 0.6) { return HANDOFF; }   // 阈值要自己标定，别照抄

JevResult r = JevUtil.ask(state, JevUtil.questions(
        "is_spam", JevQuestion.noul("是否为垃圾推广"),
        "anger",   JevQuestion.score("情绪激烈程度",
                        Arrays.asList("平静陈述", "不满但克制", "非常愤怒"))));
```

推荐接线方式：**拆问题，不要串**。每个原语独立提问，在代码里组合：
`choice` 判类型、`score` 判程度、`noul` 判兜底。多步链式推理会掉准确率。

---

## 7. 配置与权限（最容易踩的坑）

```yaml
jev:
  model: jev-1.14.0          # 硬编码版本号，禁用 jev-latest（官方会静默切版本）
  api-key: ""                # 留空回退 llm.apiKey；各环境由配置中心覆盖
  timeout-ms: 10000          # per-call 覆盖，不受 llm 的 600s 影响
  strict-model-check: true   # 响应 model 与配置不一致时抛异常
```

**权限是分开授予的**：同一个 key 能正常调 `deepseek-v4-pro`，却可能对 jev 全部版本返回
`403 model_not_found`。2026-10-23 实测就遇到过「chat 能调、jev 403」，申请授权后恢复。
所以生产环境要在 Paladin 显式配 `jev.api-key`，不要依赖回退。

---

## 8. 阈值标定与评测方法

jev 没有默认阈值，**阈值必须自己标**。落地顺序：

1. 攒 20–50 个已标注样本（正例/反例都要）。
2. 用 `noul` 跑一遍，看两组概率分布是否分得开。
3. 在分布空隙处取阈值；分不开就拆成多个问题，或改用 `score`。
4. 阈值写进业务常量并注释来源，不要散落。

评测时要看的指标：

| 指标 | 用途 |
|------|------|
| 正例概率分布 / 负例概率分布 | 判断问题是「可分」还是「模型没懂」 |
| 选定阈值下的准确率 / 召回率 / 误杀率 | 决定阈值取哪 |
| `choice` 的 `confidence` 分布 | 定「转人工」阈值 |
| `costMs` / `inputTokens` / `outputTokens` | 评估成本，固定开销约 300 token |
| 同输入重复调用差异 | 验证确定性（应为 0） |

因为结果是确定性的，**同一批样本可以只跑一次**；要复跑是因为改了 `instructions` 或换了模型版本。

---

## 9. 常见坑

| 坑 | 表现 | 处理 |
|----|------|------|
| 空 state | 服务端返回 HTTP 200 + 默认低概率 0.05，被误判成「否」 | 本地先校验，空文本直接拒绝 |
| 非法 type | 服务端只回含糊的 `Invalid request.` | 本地校验并指出问题 ID |
| 走 chat 端点 | `no cluster found for model: jev-1.14.0` | 固定 `/v1/systemone` |
| 用 `jev-latest` | 官方静默切版本，口径漂移 | 硬编码版本号，日志记录响应真实 model |
| 固定 token 开销 | 短文本也要约 290 token | 一次多问合并，别一问一请求 |
| 照抄示例阈值 | 0.6 / 0.9 只是示例 | 按自己的样本分布标定 |
| 以为有缓存 | 每次都是网络调用 | 高频重复判断需要上层自己缓存 |
| key 复用想当然 | chat 能用、jev 403 | 单独申请 jev 权限并显式配置 key |

---

## 10. 实测记录

### 2026-10-23（Java 封装联调，真实网关）

- 6 例全过，总耗时 3.1s
- 肯定句 > 0.5、否定句 < 0.2，非关键词匹配
- 同输入两次调用结果完全相等（maxDiff=0）
- 空 state 被本地拦截，未发出请求
- per-call 超时 10000ms 生效

### 2026-10-27（pre 测试接口，Python 工作区）

样本：「我上周买的会员，今天看还在扣费，我要退款，还要投诉你们。」

| 问题 | 结果 |
|------|------|
| `wants_refund`（noul） | **0.980** |
| `team`（choice，billing/tech/content） | **billing**，confidence=1.000 |
| `anger`（score，4 档） | **2.090**（0.55 概率落在「明显愤怒」） |
| 耗时 / token | 886ms，296 in / 23 out |

批量 3 例（正例/负例/choice）全部 `passed=true`；负例 `no-refund-1` 得 0.03。
护栏：空 `state` → HTTP 200 + `code=400`「state 不能为空」；非法 type → `code=400`。

---

## 11. 参考资料

| 资料 | 位置 |
|------|------|
| 技术方案（设计取舍、错误处理、实测） | `atlas/docs/specs/jev决策模型集成-技术方案.md` |
| 使用手册（API 速查、阈值标定、坑） | `atlas/docs/rules/jevUtil使用手册.md` |
| Java 封装 | `atlas/.../utils/jev/JevUtil.java`（提交 `d5b4667e`） |
| 测试接口 | `atlas/.../api/http/JevTestController.java`（提交 `006f91ae`） |
| Python 评测工作区 | `~/Developer/Project-Data/atlas-api-eval` |
| fastjson 响应坑 | `atlas/docs/lessons/028-框架JSON库与字段类型不匹配致响应体恒null.md` |

---

## 12. 自测题（能答出来就算懂了）

1. 为什么 `JevUtil` 不允许把 `api-key` 缓存在静态字段里？
2. `choice` 返回 `confidence=0.51`，业务上应该怎么处理？为什么不能直接信 `choice`？
3. 为什么空 `state` 必须在本地拦截，而不是交给服务端？
4. 一次要判「是否垃圾 / 情绪多高 / 该转哪个团队」，应该发几次请求？为什么？
5. 正例概率集中在 0.45–0.55、负例集中在 0.40–0.60，这个判断还能用单条 `instructions` 吗？
6. jev 返回的 `score=2.09` 是什么含义？和「第 2 档」有什么区别？
