# UCON：开始允许，使用过程中也要一直检查

UCON 全称是 **Usage Control**，中文叫使用控制。

先记一句话：

> 权限检查不是只做一次；资源正在被使用时，系统仍会持续检查，条件失效就暂停或终止使用。

## 一、先看一个真实例子

某视频平台规定：

```text
有效会员可以播放付费电影；
同一账号最多播放 30 分钟；
会员到期或超过时长后，正在播放的视频也必须停止。
```

张三上午 10:00 开始播放：

```text
会员有效
+ 已使用 0 分钟，小于 30 分钟
= 允许开始播放
```

到了 10:31：

```text
本次已使用 31 分钟
>= 允许时长 30 分钟
= 系统主动停止播放
```

这里的重点是：10:00 允许开始，不代表以后永远允许。

## 二、普通鉴权和 UCON 的区别

普通接口鉴权经常是：

```text
请求进来时检查一次
→ 允许
→ 操作完成
```

UCON 是：

```text
使用前检查
→ 允许开始
→ 使用中反复检查
→ 条件不再满足
→ 暂停或终止
```

它特别适合一次使用会持续较长时间的资源，例如视频、远程连接、设备租用和临时凭证。

## 三、最简单需要保存哪几类信息

UCON 是一种思想，不强制规定必须有固定数量的表。

这个例子需要知道：

```text
1. 用户的会员是否有效
2. 资源允许使用多久
3. 当前这次使用什么时候开始、用了多久、是否还在进行
```

可以使用三张表：

```text
user_subscription
protected_resource
usage_session
```

## 四、先看三张表中的实际数据

### 1. 用户订阅表

| user_id | status | expires_at |
| ---: | --- | --- |
| 1 | ACTIVE | 2026-12-31 23:59:59 |

这一行表示：

```text
用户 1 的会员当前有效，到 2026 年 12 月 31 日过期。
```

### 2. 受保护资源表

| id | resource_name | max_minutes |
| ---: | --- | ---: |
| 1001 | 付费电影 A | 30 |

这一行表示：

```text
电影 1001 单次最多允许使用 30 分钟。
```

### 3. 使用会话表

| id | user_id | resource_id | started_at | used_minutes | status |
| ---: | ---: | ---: | --- | ---: | --- |
| 5001 | 1 | 1001 | 2026-09-22 10:00:00 | 12 | USING |

这一行表示：

```text
用户 1 正在使用电影 1001，
本次会话从 10:00 开始，目前已经使用 12 分钟。
```

## 五、三张表怎么连起来

```text
user_subscription
用户 1 的会员有效
        │
        │ usage_session.user_id
        ▼
usage_session
用户 1 正在使用资源 1001，已使用 12 分钟
        │
        │ usage_session.resource_id
        ▼
protected_resource
资源 1001 最多允许使用 30 分钟
```

系统不断比较：

```text
会员仍然有效吗？
会话仍然是 USING 吗？
used_minutes 仍然小于 max_minutes 吗？
```

## 六、一次完整使用怎么判断

### 使用前

张三点击播放：

```text
1. 会员状态是否是 ACTIVE？是。
2. 会员是否尚未过期？是。
3. 是否满足资源的其他限制？是。
4. 创建一条 status=USING 的 usage_session。
5. 允许开始播放。
```

### 使用中

播放器每隔一段时间发送心跳，系统重新检查：

```text
1. 会员现在是否仍然有效？
2. 本次会话是否仍然有效？
3. 已使用时长是否仍然小于 30 分钟？
```

全部满足就继续播放。

### 条件失效

当已使用 31 分钟时：

```text
31 < 30 不成立
→ 把会话改为 TERMINATED
→ 告诉播放器停止播放
```

## 七、最简单的建表 SQL

```sql
CREATE TABLE user_subscription (
    user_id BIGINT PRIMARY KEY,
    status VARCHAR(20) NOT NULL,
    expires_at DATETIME NOT NULL
);

CREATE TABLE protected_resource (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    resource_name VARCHAR(100) NOT NULL,
    max_minutes INT NOT NULL
);

CREATE TABLE usage_session (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    resource_id BIGINT NOT NULL,
    started_at DATETIME NOT NULL,
    used_minutes INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL
);
```

## 八、插入示例数据

```sql
INSERT INTO user_subscription
    (user_id, status, expires_at)
VALUES
    (1, 'ACTIVE', '2026-12-31 23:59:59');

INSERT INTO protected_resource
    (id, resource_name, max_minutes)
VALUES
    (1001, '付费电影 A', 30);

INSERT INTO usage_session
    (id, user_id, resource_id, started_at, used_minutes, status)
VALUES
    (5001, 1, 1001, '2026-09-22 10:00:00', 12, 'USING');
```

## 九、最核心的持续检查 SQL

判断会话 5001 现在能否继续：

```sql
SELECT 1
FROM usage_session us
JOIN user_subscription sub
  ON us.user_id = sub.user_id
JOIN protected_resource pr
  ON us.resource_id = pr.id
WHERE us.id = 5001
  AND us.status = 'USING'
  AND sub.status = 'ACTIVE'
  AND sub.expires_at > CURRENT_TIMESTAMP
  AND us.used_minutes < pr.max_minutes
LIMIT 1;
```

把 SQL 翻译成人话：

```text
当前会话还在使用中，
会员仍有效且未过期，
已使用时长还没有达到资源上限，
才返回允许。
```

每次心跳更新使用量：

```sql
UPDATE usage_session
SET used_minutes = used_minutes + 1
WHERE id = 5001
  AND status = 'USING';
```

条件不满足时终止会话：

```sql
UPDATE usage_session
SET status = 'TERMINATED'
WHERE id = 5001;
```

## 十、UCON 适用场景

* 视频和数字内容播放。
* 云主机、软件许可证和 API 配额。
* 共享设备、车辆和会议室租用。
* VPN、远程桌面和数据库临时访问。
* 临时下载链接和限时凭证。

## 十一、高级扩展：三个关键词

UCON 经常继续讨论：

```text
Authorization（授权）
是否满足权限规则。

Obligation（义务）
使用者必须完成什么，例如播放前同意协议、使用后提交日志。

Condition（条件）
与用户意愿无关的环境条件，例如系统风险等级、设备位置。
```

另一个关键点是属性可变：

```text
余额会减少
已用时长会增加
会员会过期
设备风险会变化
```

因此系统不但读取属性，还可能在使用过程中修改属性，并重新做决定。

真正的“持续检查”可以通过心跳、定时任务、网关会话或消息事件实现，不一定是数据库不断轮询。

## 十二、面试时怎么说

> UCON 的特点是权限决策贯穿资源使用的整个过程，而不是只在开始时检查一次。系统会建立使用会话，持续检查会员状态、时长、余额或环境条件；一旦条件不再满足，就暂停或终止使用。典型场景包括视频播放、许可证、设备租用和远程连接。

