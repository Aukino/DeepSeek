# QQ 智能体运行架构与行为策略

> 版本：2026-11-01（晚间重写）  
> 对象：QQ 私聊 / 群聊智能体（原“小鲸鱼”系统）  
> 范围：消息链路、唤醒、节奏、主动、记忆、小本本、空间、日记、表情、视觉、工具、权限、降级与验证。  
> 本文不写 QQ 号、群号、昵称、聊天原文、token 和本机绝对路径。

---

## 0. 阅读约定与当前状态

### 0.1 状态标记

本文所有结论都带标记，避免把“设计”当成“已经验证”：

```text
[已验证]
  代码在位，且自检、单测或合成流程至少通过一项。

[已部署待观察]
  代码已部署，但需要真实聊天/图片/批次数据才能判断效果。

[设计]
  方案成立，但当前没有实现或没有启用。

[已禁用/历史]
  代码可能还在，但不参与当前行为。
```

### 0.2 当前快照

```text
部署自检：200 项，失败 0
插件目录：46 个
有效记忆：129
向量：bge-small-zh-v1.5，512 维 / 2048 字节
有效向量：139（含历史/失效条目）
候选：
  pending   7
  deferred  592
  accepted  38
  rejected  200
playbook：0 条 active
控制中心：127.0.0.1:8770
```

这些数字会随聊天变化，以实时自检和数据库为准。

---

## 1. 系统边界

### 1.1 角色

```text
QQ 客户端
→ SnowLuma / OneBot v11
→ dsh-onebot-qq
→ DSH 会话模型
→ 工具调用
→ 出站装饰
→ QQ 发送
```

### 1.2 组件

```text
QQ 协议端：SnowLuma
机器人宿主：dsh-onebot-qq
模型运行时：DSH web
记忆库：本地 SQLite
控制中心：本地 HTTP 面板
记忆星图：本地可视化
```

端口约定：

```text
SnowLuma HTTP 5099
SnowLuma WS   3001
DSH web       3080
记忆星图      8765
控制中心      8770
```

### 1.3 数据目录

```text
~/.dsh/dsh-onebot-qq/
  config.json
  state/memory.db
  state/proactive-watch.json
  state/diary-daily.json
  logs/qq_chat/
  logs/wake-decisions.jsonl
  logs/memory-injections.jsonl
  logs/tool-calls.jsonl
  qq-stickers/
  workspace/日记本/
```

### 1.4 原则

```text
不抢话
不打断
不重复
不乱编
不裸奔
可解释
可回滚
表达与事实治理分离
```

---

## 2. 消息主链路

### 2.1 入站归档

所有入站消息先记录：

```text
时间 / 发送者 / 消息 id / 文本 / 媒体
```

用途：

```text
会话回放
上下文拼装
记忆窗口
引用反查
审计
```

### 2.2 门禁

```text
私聊：好友 / 主人门禁
群聊：黑名单、禁用、是否必须 @
陌生人：礼貌但保持距离
```

### 2.3 唤醒

| 类型 | 触发 |
|---|---|
| `private` | 私聊 |
| `at` | @ 机器人 |
| `quote` | 引用机器人消息 |
| `keyword` | 命中配置的唤醒词 |
| `ack` | 纯呼名，先应一声 |
| `followup` | 呼名后同一人补正文 |
| `continuation` | 她刚在群里说完，群友顺着接 |
| `proactive` | 内容/机会成立后的主动回合 |
| `welcome` | 入群欢迎 |

当前实现细节：

```text
唤醒词只认配置/人设词表
不再有代码内置的 NAME_ALIASES
群聊默认要求 @ 或唤醒词
纯呼名先 ack，不立刻长篇回复
```

### 2.4 接续窗口

```text
reply-window：最多 4 轮，窗口 180 秒
gate-wake：纯呼名后的 followup 窗口 150 秒
```

窗口内再次被唤醒词或纯呼名叫到：

```text
刷新接续
按正常接话处理
ack / followup 不再自动引用
```

### 2.5 合批

私聊：

```text
基础窗口       5000ms
活跃窗口       2500ms
最长等待      10000ms
```

群聊：

```text
基础窗口       2500ms
最长等待       7000ms
慢消息跳过     30000ms
```

目的：

```text
让对方把话说完
避免一条一条抢答
```

### 2.6 上下文拼装

```text
最近消息
群聊上文缓存
长期记忆注入
关系/情绪提示
小本本到期项
旧事复习提示
工具结果
```

预算：

```text
私聊长期事实 600 字 + 全部提示 900 字
群聊长期事实 400 字 + 全部提示 700 字
```

### 2.7 出站

```text
send_message / send_image
→ 出站装饰
→ 引用 / @ 规则
→ 脱敏
→ 留档
→ QQ 发送
```

不变量：

```text
聊天消息不允许绕过 deliver/sendMedia
内部错误不原文发到群里
风控错误不露原始错误码
```

---

## 3. 发送校验与工具校验

### 3.1 发送校验

```text
[已验证]
```

模型回合结束后，如果正文有内容但 `turnCtx.delivered` 仍为 false：

```text
普通回合 → 提醒一次补调 send_message
主动回合 → 允许安静，不追问
校验提示只进模型会话
不把校验提示发到 QQ
```

### 3.2 工具意图校验

```text
[已验证：代码与自检]
[已部署待观察：真实触发频率]
```

检测最终 assistant 正文里的明确动作：

```text
发空间 / 写日记 / 记小本本 / 收藏
```

时态规则：

```text
现在时：校验 qzone / diary / note / repo
未来时：只校验 note(due_at) 或 proactive_intent(notBefore)
```

边界：

```text
同一轮最多提醒一次
提醒后仍不调用 → 放过
发送校验未通过时跳过工具校验
主动回合跳过
校验提示绝不 deliver 到 QQ
```

---

## 4. 主动行为系统

### 4.1 总结构

```text
意图门
→ 硬条件
→ 四维评分 / 综合阈值
→ 主模型
```

旁边还有：

```text
D2A 欲求缺口 → 生成候选意图
群聊 ambient → 生成候选意图
idle-browse → 生成候选意图
notebook → 生成候选意图
```

### 4.2 意图生命周期

```text
pending
→ offered
→ consumed
→ expired
→ cancelled
```

历史限制：

```text
同一会话 10 分钟最小间隔：已拆
同一意图最多提供 2 次：已拆
attempts 只作统计，不再淘汰
```

意图保留到：

```text
consumed
expired（TTL）
cancelled（小本本划掉）
```

### 4.3 硬条件

这些不参与“综合评分”，不满足就不能执行：

```text
gate-wake 夜间静默（23:30–09:00）
正在回话 / turnQueue 非空
私聊同时只跑一个主动回合
群聊是否允许主动
目标是否允许
意图存在且 notBefore 到
未过期
未命中主人明确负面偏好
```

群聊权限独立：

```text
groupsAll
groups
```

不再借用旧 pacing 的群列表。

### 4.4 四维评分

```text
欲求：有没有想说的
信念：有没有理由/证据
机会：现在适不适合
因果：为什么是现在、对这个人、说这件事
```

基础权重：

```text
欲求 0.30
信念 0.30
机会 0.20
因果 0.20
```

上下文相似度：

```text
0.45 × 字面相似
+ 0.55 × 向量相似
```

向量复用记忆模型：

```text
bge-small-zh-v1.5
512 维
```

向量不可用：

```text
回落纯字面相似
```

动态权重来自：

```text
情绪六维：social / curiosity / play / achievement / recognition / rest
关系三维：closeness / trust / warmth
事实/片段：证据、未完成约定、相关 episode
```

规则：

```text
每个维度最多小幅修正
修正后归一化
综合阈值默认 0.58
情绪不会让所有维度一起上涨
关系只对具体对象生效
```

### 4.5 失败阻尼

```text
[已验证：代码]
[已部署待观察：真实频率]
```

主动发出后一段时间无人回应：

```text
proactiveMiss +1
欲求/因果轻微下调
综合阈值轻微上调
最多累计到 3 次
```

有人回应：

```text
proactiveMiss 清零
```

目的：

```text
会失落，也会收
不是无限降门槛
```

### 4.6 D2A 欲求缺口

```text
[已验证：合成流程]
[已部署待观察：真实私聊]
```

参考 D2A（Desire-driven Autonomy）的“欲求缺口 → 候选活动”，但不照搬“缺口一高就行动”。

六个欲求：

```text
社交联结 / 自我实现 / 自我照料
安全感 / 愉悦感 / 成就感
```

满足度由现有数据推算：

```text
social        ← social + closeness
actualization ← curiosity + achievement
selfCare      ← rest + warmth
security      ← trust + closeness
pleasure      ← play + social
achievement   ← achievement + recognition
```

生成条件：

```text
私聊
没有待处理意图
至少 2 小时没有互动
同一对象两次提案至少间隔 180 分钟
```

只允许：

```text
社交缺口 → 轻量 checkin「想看看主人最近怎么样」
安全缺口 → 轻量 checkin「想确认主人这边还好吗」
```

不用于：

```text
自我照料缺口 → 休息不是打扰
群聊 → 不凭空 broadcast
```

生成后仍走：

```text
notBefore = 5 分钟后
四维评分
静默 / 忙碌 / 串行
主模型最终决定
```

合成验证：

```text
D2A 生成 checkin：通过
单独 checkin 四维评分：0.63 > 0.58
```

### 4.7 群聊 ambient

```text
[已验证：代码与自检]
[已部署待观察：真实群聊]
```

条件：

```text
群里连续 3 条以上
最后一条不是纯媒体
本地话题分 ≥ 0.3
```

动作：

```text
登记轻量 followup
主题固定为「群聊正在继续」
摘要取最近两条
20 分钟内只保留一条
```

不是“必须接话”，只是给主模型一个可拒绝的候选。

### 4.8 其他意图来源

#### idle-browse

```text
每 180 分钟一次
每天最多 3 次
无人回应足够久才触发
浏览后写收藏
值得分享则登记 proactive_intent
不再直接 send_message
```

#### 小本本

```text
proactive=true + due_at
→ 生成带 notBefore 的主动意图
→ 到点走机会门
→ 主模型决定
```

#### 群聊问题

```text
最后一条是没人接的问题
至少 6 个字
不是纯图片/表情/视频/语音
→ 登记 followup
```

### 4.9 空间

```text
[已验证：配置/代码]
[已部署待观察：真实发帖]
```

```text
qzone-nudge：默认关闭
不再每天固定时间自动起“要不要发空间”回合
```

发不发、什么时候发，由她在正常回合自己判断并调用 `qzone`。

保留：

```text
只有主人私聊能发
allowGroup = false
每天自主发帖上限 2 条
夜间静默 23:30–09:00
默认好友可见
```

空间评论：

```text
每 20 分钟检查一次
静默 23:30–09:00
```

### 4.10 日记

```text
每天 22:30
每天只问一次
写入 workspace/日记本/YYYY-MM-DD.txt
当天写没写成都不再重试
```

漏写检查：

```text
只覆盖开始写日记之后的日期
问一次
不自动补
材料不足不编
```

### 4.11 连续性账本

```text
[已验证：代码与自检]
```

```text
state/proactive-watch.json → ledger
```

记录：

```text
queued / offered / sent / cancelled
target / source / kind / intentId / 摘要
最多保留 200 条
```

来源：

```text
proactive-watch
idle-browse
notebook
group-watch
group-ambient
follow-up
```

`follow-up` 是延迟发送，不重新起模型；它只写入同一本账，不强行并入模型准入。

---

## 5. 记忆系统

### 5.1 作用域

```text
shared          人的长期资料
chat:private    私聊
group:<id>      群聊
```

`shared` 只记人的称呼、忌讳、偏好、约定、身份；不记聊天原文和事件经过。

默认：

```text
privacy_level = subject_only
```

### 5.2 类型

```text
fact     稳定事实
episode  带时间/地点/情绪/感受的经历
raw      原话，默认不注入
```

### 5.3 向量

```text
本地模型：bge-small-zh-v1.5
维度：512 / 2048 字节
```

旧 768 维向量在启动时清理并按 512 维回填。

历史日志里的 768 维告警是修复前记录，不代表当前数据库。

### 5.4 写入

```text
归一化
→ 敏感/长度校验
→ scope/身份
→ 分类/标签/类型/显著性
→ 精确/近似去重
→ 冲突/换值
→ 向量化
→ 归簇
→ INSERT
→ 容量淘汰
```

### 5.5 候选

状态：

```text
pending
accepted
rejected
deferred
expired
archived
```

固定批处理：

```text
每天 04:00
开机错过 04:00 → 启动后 10 分钟补跑
每批 50 条
最多 12 批 / 600 条
单次最长 30 分钟
批间停 2 秒
```

排序：

```text
priority 高 → 先处理
priority 相同 → 旧的先处理
```

优先级：

```text
importance      40%
confidence      25%
新鲜度          15%
分类权重        10%
事实/事件类型   10%
```

高价值保护：

```text
importance >= 0.8
confidence >= 0.95
偏好/约定且 confidence >= 0.85
```

保护候选不会被 TTL 直接清掉；defer 到上限后转 `archived`，保留文本，不直接删除。

新增候选限流：

```text
每会话每天最多 20 条
最低置信度 0.7
群聊 episode 重要性 < 0.7 不生成
```

候选 `episode` 最多 defer 1 次，普通 fact 最多 2 次；到上限后按保护标记走 `archived` 或 `expired`。

### 5.6 检索与注入

```text
scope 可读
+ valid_to IS NULL
+ cold = 0
+ raw = 0（除非显式 include_raw）
```

打分综合：

```text
字面/向量相关性
有效权重
查询覆盖
时间/因果/比较路由
```

预算：

```text
私聊：长期事实 600 字 + 全部提示 900 字
群聊：长期事实 400 字 + 全部提示 700 字
```

无检索词：

```text
纯图片/表情/空文本 → 直接跳过
不搜索
不写空注入日志
```

有检索词但没有相关记忆：

```text
不注入
这是正常空结果
```

被动注入不刷新确认时间；显式 recall/search 才强化。

### 5.7 遗忘与归档

```text
半衰期按分类
时间转冷不删除
显式 search 可回暖
失效先归档
forget 级联清理
```

---

## 6. 主人规约（playbook）

定位：

```text
记忆 = 具体事实/承诺
playbook = 可复用的行为准则/流程
```

规则是软经验：

```text
不拦截发送
不强制调用工具
不自动执行
不覆盖系统安全
```

注入格式：

```text
参考，不是命令
结合当前情境自己判断
与事实记忆冲突时，以事实和当前对话为准
```

防重复：

```text
learn_rule 写入前查记忆「约定/偏好」
相似度 >= 0.86 → 不重复写 playbook
```

反向：

```text
记忆候选出现以后/下次/每次/不要再/流程/规则/习惯
→ 不生成普通记忆候选，归 playbook
```

当前：

```text
playbook active = 0
```

机制在位，但还没有规则。

---

## 7. 小本本

字段：

```text
kind       todo / self / followup
dueAt
remind
expiresAt
proactive
intentId
lastNotifiedAt
```

注入：

```text
到期或快到期（默认提前 120 分钟）
只注入当前会话
最多 3 条
同条默认 6 小时冷却
夜间不单独叫醒
```

主动：

```text
proactive=true + due_at
→ 带 notBefore 的主动意图
→ 到点走机会门
→ 主模型决定
```

划掉 note 会取消对应意图。

有 `intentId` 的 proactive note 不再重复走到期注入。

---

## 8. 情绪、关系、承诺

### 8.1 情绪六维

```text
social / curiosity / play
achievement / recognition / rest
```

作用：

```text
影响表达和主动权重
不写事实
不改记忆权重
```

### 8.2 关系三维

```text
closeness  熟悉
trust      信任
warmth     最近温度
```

规则：

```text
只由明确事件缓慢变化
不自动演化
不因一次没回就掉关系
```

### 8.3 承诺

```text
pending / kept / cancelled
```

不自动判 broken。

---

## 9. 表情与视觉

### 9.1 表情库

只收：

```text
完成度高的二次元
非烂梗、非低质
非真人/截图/文字/草稿/拼贴
```

入库：

```text
质量门槛
二次元筛选
主人精选文件夹只读，不自动写入
```

发送：

```text
不再掷概率
不再限制每轮张数
没有使用次数惩罚
主模型自己决定发不发、发几张
```

主人精选权重更高，但其他表情仍有机会。

### 9.2 视觉

```text
[已验证：配置/代码/自检]
[已部署待观察：真实图片]
```

```text
旁路识图：超时 30 秒
max_tokens 4000 + 2000 重试

表情打标/质量判断：
默认 4000
失败重试上限 8000
```

只接受正文，不拿 `reasoning_content` 当标签。

失败处理：

```text
只记日志
不猜图
不把失败当描述写进记忆
```

---

## 10. 工具与权限

常用：

```text
send_message / send_image
memory / recall
note
diary
repo
sticker
qzone
alarm
contacts
follow_up
proactive_intent
proactive_stats
```

工具中间件：

```text
before 权限/参数
执行
after 计数
审计 logs/tool-calls.jsonl
```

工具意图校验：

```text
现在时校验 qzone/diary/note/repo
未来时只校验 note/proactive_intent
一次、软提醒、不发给 QQ
```

运维工具只允许主人私聊，写操作需要 nonce 确认。

控制中心：

```text
127.0.0.1:8770
状态 / 启动 / 停止 / 重启 / 日志 / 设置
```

---

## 11. 失败处理

```text
向量失败 → 字面检索
DB 失败 → 跳过注入，聊天继续
摘要失败 → 保留旧摘要
图片失败 → 说没看清，不猜
候选整理失败 → 保留状态，不空转
工具校验失败 → 只记日志
```

发送风控：

```text
result=241 视为风控
私聊冷却 60 秒
群聊冷却 30 秒
不立即重试
聊天里不露原始错误码
```

---

## 12. 已知边界

```text
[已部署待观察]
群聊 silent 的旧 48h 样本是 95.8%；
ambient 上线后还没有新的 48h 样本。
不能直接放宽 @ 门禁来压指标。

[已部署待观察]
候选仍有约 592 条 deferred；
review_after 未到，04:00 批处理还没消化。

[已部署待观察]
D2A 目前只生成私聊 social/security 的 checkin；
selfCare 等缺口只影响评分，不直接发。

[已部署待观察]
D2A 满足度由情绪/关系/最近互动推算，
没有单独写“欲求满足历史”。

[已验证：行为]
无检索词的记忆注入直接跳过；
有检索词但无命中的空结果仍属正常。

[设计边界]
follow-up 是延迟发送，不进模型准入；
它只写入统一 ledger。

[已部署待观察]
视觉模型已提高超时和 max_tokens，
真实效果要等下一批图片/表情。

[已禁用/历史]
旧 pacing 群主动、proactive-dm 不参与当前主动。
```

---

## 13. 验证矩阵

| 项目 | 状态 | 证据 |
|---|---|---|
| 部署自检 | 已验证 | 200 项，失败 0 |
| 记忆向量维度 | 已验证 | 139 条，512 维 / 2048 字节 |
| 候选状态机 | 已验证 | 状态与批处理代码在位 |
| 四维主动 | 已验证 | 代码 + 合成评分 |
| 字面/向量混合 | 已验证 | 合成流程 |
| D2A checkin | 已验证 | 合成生成 + 0.63 评分 |
| 群聊 ambient | 已部署待观察 | 代码/自检通过，缺新样本 |
| 统一 ledger | 已验证 | 代码/自检 |
| 工具意图校验 | 已验证代码 | 缺真实触发样本 |
| 记忆空 query 跳过 | 已验证代码 | 缺新日志样本 |
| 视觉 token 修复 | 已部署待观察 | 等真实图片 |
| 群聊 silent 下降 | 待观察 | 旧样本 95.8%，新样本未出 |
| 候选积压下降 | 待观察 | 592 deferred，等批处理 |
| follow-up 统一准入 | 设计边界 | 只统一记录，不统一模型 |

说明：

```text
“已验证”不等于“线上效果已经好”，
只表示对应代码路径和最低验证成立。
```

---

## 14. 代码地图

```text
qq-extras/plugins/memory-v2
  长期记忆 / 候选 / 检索 / 向量 / 注入

qq-extras/plugins/memory-window
  窗口抽取与 raw 暂存

qq-extras/plugins/memory-triage
  固定批处理 / priority / archive

qq-extras/plugins/memory-cleanup
  重复归档

qq-extras/plugins/memory-judge
  逐条判定

qq-extras/plugins/recent-summary
  短期摘要

qq-extras/plugins/playbook
  主人规约

qq-extras/plugins/proactive-watch
  主动意图 / 四维评分 / D2A / 群聊 ambient / ledger

qq-extras/plugins/idle-browse
  空闲浏览，只产意图

qq-extras/plugins/notebook
  短期待办 / 自我提醒

qq-extras/plugins/follow-up
  延迟追话，写入 ledger

qq-extras/plugins/qzone
  空间发帖 / 评论

qq-extras/plugins/diary
  日记读写 / 每日日记

qq-extras/plugins/stickers
  表情库

qq-extras/plugins/sticker-tagger
  表情打标 / 视觉质量判断

qq-extras/plugins/vision-sidecar
  旁路识图

scripts/launch-stack.mjs
scripts/stop-stack.mjs
scripts/verify-install.mjs
scripts/control-center.mjs
```

---

## 15. 本版变更摘要

```text
主动从硬闸门改为四维评分
加入字面 + 向量混合相似度
加入情绪/关系动态权重
加入 D2A 欲求缺口 checkin
加入群聊 ambient
加入统一连续性 ledger
关闭空间每日 nudge
空间 allowGroup 改为 false
视觉 timeout / max_tokens 修复
记忆无检索词跳过
工具意图校验加入现在时/未来时
文档改为按“已验证 / 待观察 / 设计 / 已禁用”标注
```

---

## 16. 一句话总结

```text
这套系统不是“概率决定她说不说话”，
而是：
  欲求产生候选
  四维判断该不该
  硬条件决定能不能
  主模型决定说不说
  真实结果再校正下一次。
```
