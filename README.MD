# 小鲸鱼 · README

小鲸鱼是一个运行在 QQ 上的长期陪伴型聊天助手。

它不是“接一个大模型然后自动回复”的壳，而是一套完整的本地运行时：

```text
QQ
 → SnowLuma / OneBot
 → dsh-onebot-qq 宿主
 → DSH 主模型
 → 工具 / 记忆 / 主动行为
 → QQ 出站
```

核心目标只有一句话：

> 让聊天有连续性，但不假装记住一切；让主动有分寸，但不靠硬性配额。

---

## 主要能力

- **QQ 聊天接入**：私聊、群聊、引用、@、戳一戳、图片、表情包。
- **长期记忆**：SQLite + FTS5 + 本地向量，按会话隔离，支持共享档案。
- **混合检索**：字面检索、BM25、向量召回、GPU rerank 融合。
- **写入证据门**：真实归档消息 ID + 原文片段，否则只进入候选。
- **记忆审计**：`ok / suspect / needs-evidence / legacy-no-evidence / ignored` 生命周期。
- **主动行为**：意图、四维评分、D2A 欲求缺口、私聊冒泡、群聊动态门槛。
- **媒体理解**：表情包优先情绪，截图和图片优先内容。
- **工具系统**：记忆、笔记、日记、提醒、仓库、空间、主动管理、运维。
- **可视化**：3D 记忆星图、数据面板、控制中心。
- **运维闭环**：自检、行为回归、故障演练、恢复演练、备份保留。

---

## 设计原则

```text
记忆不是越多越好
主动不是越频繁越好
工具不是越细越好
```

具体表现为：

- 记忆写入必须能回到真实原文，不靠模型自说自话。
- 未验证记忆可以被使用，但每轮限流，且必须和当前输入直接相关。
- 主动私聊不设每日硬上限，用来源质量、门槛和软冷却控制。
- 群聊按活跃度动态调门槛，群越热闹越克制，不跟着一起变活跃。
- 表情包理解情绪，截图理解内容，两条路径不混用。
- 高风险工具保持独立，不用“大而全”的入口掩盖副作用。

---

## 系统结构

```text
┌─────────────────────────────────────────────┐
│                  QQ 客户端                   │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│        SnowLuma / OneBot 协议端              │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│           dsh-onebot-qq 插件宿主             │
│                                             │
│  消息门禁 / 唤醒 / 合批 / 上下文 / 出站       │
│  记忆 / 主动 / 媒体 / 工具 / 星图             │
└───────┬──────────────────┬──────────────────┘
        │                  │
        ▼                  ▼
┌──────────────┐   ┌──────────────────────────┐
│  DSH 主模型   │   │ 本地旁路服务              │
│              │   │  rerank      :8791        │
│              │   │  记忆星图    :8765        │
│              │   │  控制中心    :8770        │
└──────────────┘   └──────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│              SQLite / 文件系统               │
│  memory.db / qq_chat / qq-stickers / logs    │
└─────────────────────────────────────────────┘
```

---

## 快速开始

### 前置条件

- Windows（当前主要运行环境）；Linux 需要自行适配启动脚本
- Node.js 22+
- Python 3.10+（rerank / 星图相关）
- QQ 协议端：SnowLuma + OneBot
- 一个可用的 DSH web 运行时
- 可用的 DeepSeek API Key（主模型与视觉）

### 1. 配置

运行时配置位于：

```text
~/.dsh/dsh-onebot-qq/config.json
```

常用环境变量：

```bash
DEEPSEEK_API_KEY=...
```

不要把真实 Key、QQ 号或聊天内容提交到仓库。

### 2. 启动

```bash
node scripts/launch-stack.mjs --no-open
```

启动后会拉起：

```text
QQ 协议端
DSH web
主模型
rerank（可选）
记忆星图
控制中心
看门狗
```

### 3. 停止

```bash
node scripts/stop-stack.mjs
```

### 4. 页面

```text
控制中心   http://127.0.0.1:8770
记忆星图   http://127.0.0.1:8765
DSH web    http://127.0.0.1:3080
```

控制中心和星图默认只监听本机。

---

## 目录结构

```text
qq-extras/plugins/          业务插件源码
  memory-v2/                长期记忆、检索、注入、rerank
  memory-audit/             记忆审计
  memory-triage/            候选整理
  memory-cleanup/           本地去重与维护
  memory-judge/             写入判定
  proactive-watch/          主动意图、D2A、私聊/群聊机会
  proactive-dm/             主动私聊回合
  vision-sidecar/           图片/截图旁路识图
  sticker-tagger/           表情包自动打标
  ...

scripts/                    启动、验证、评测、运维脚本
rerank-service/             GPU rerank sidecar
eval/                       检索与排序评测集
config/                     阈值与运行配置
小鲸鱼-完整技术档案.md       当前架构、阈值、验证矩阵
有限记忆的一生感实现方案.md   记忆设计长文
```

`qq-extras/plugins` 是源码目录；运行时实际加载的是 DSH profile 下的 `dsh-onebot-qq/plugins`。修改源码后需要部署到 profile 并重启。

---

## 记忆系统

### 分层

```text
raw        原始片段
episode    一段时间的叙事片段
fact       可复用事实
core       核心画像
```

### 作用域

```text
shared              共享档案
chat:private:<id>   私聊
chat:group:<id>     群聊
```

跨会话读取只允许主人私聊；群聊只读当前群。

### 写入

```text
真实 messageId
+ 原文逐字片段
→ 才允许进入 memory
否则 → candidate
```

### 审计状态

```text
ok                  证据支持
suspect             有证据但模型不确定，保留热存
needs-evidence      缺证据，优先修复来源
legacy-no-evidence  无证据历史，终态
ignored             临时状态/一次性内容
```

未验证记忆每轮最多注入 1 条，且必须和当前输入直接相关。

---

## 检索与 rerank

```text
FTS5 trigram
  3 字以上：MATCH + bm25
  2 字以下：LIKE / bigram

向量召回
  bge-small-zh-v1.5，512 维

GPU rerank
  本地 cross-encoder
  与字面/向量分数融合
```

当前默认：

```text
rerankWeight = 0.65
自动注入门槛 = 0.10
检索候选 floor = 0.08
```

评测命令：

```bash
node scripts/retrieval-eval.mjs
node scripts/hybrid-ab.mjs
node scripts/rank-check.mjs
```

---

## 主动行为

主动系统只负责“发现机会”，不直接替模型说话。

```text
意图登记
  ↓
机会调度
  ↓
硬条件检查
  ↓
四维评分：desire / belief / opportunity / causal
  ↓
低于阈值 → veto
过线 → 交给主模型判断
```

当前策略：

```text
基础阈值            0.58
私聊质量门槛        0.62
私聊轻冒泡门槛      0.56
群聊动态门槛        0.52 ~ 0.74
```

私聊入口：

```text
private-followup   上次主动后没回
private-whim       具体念头 / 在吗 / 表情
explicit-share     明确分享意图
```

群聊按最近消息密度动态提高门槛：

```text
安静群  → 门槛降低
活跃群  → 门槛升高
不设每日硬上限
```

---

## 表情包与图片

两条路径：

```text
表情包
  → 情绪 / 语气 / 用法
  → 默认不按画面内容解释

截图 / 图片
  → 文字 / 界面 / 代码 / 内容
```

表情包只有在用户明确问“图里是什么 / 这个表情什么意思 / 写了什么”时，才附画面参考。

---

## 工具入口

模型可见工具经过合并，目前 18 个：

```text
alarm
ask_master
contacts
diary
follow_up
idle_browse_skip
memory
note
ops
ops_apply
proactive
proactive_dm_skip
qzone
repo
rules
self_log
usage_report
watch
```

聚合入口：

```text
proactive  action=intent|targets|stats
rules      action=learn|search|list|feedback
follow_up  action=schedule|later
```

高风险入口保持独立：

```text
contacts / repo / qzone / ops / ops_apply
```

---

## 运维与验证

### 自检

```bash
node scripts/verify-install.mjs
```

### 质量门禁

```bash
node scripts/quality-gate.mjs
```

包含：

```text
部署自检
阈值审计
行为回归
主动策略模拟
媒体提示
表情情绪
工具入口
数据健康
证据门
相似度底层
BM25
故障演练
接续窗口
戳一戳
恢复演练
```

### 数据健康

```bash
node scripts/memory-health.mjs
```

### 备份

```bash
node scripts/prune-backups.mjs --keep 7 --keep-repair 7 --keep-transient 1
node scripts/prune-backups.mjs --keep 7 --keep-repair 7 --keep-transient 1 --apply
```

先 dry-run，确认后再加 `--apply`。  
`purge-before-*` / `retention-before-*` 只保留最近 1 份。

### 恢复演练

```bash
node scripts/restore-drill.mjs
```

---

## 数据与隐私

- 运行时数据只保存在本机 `~/.dsh/dsh-onebot-qq/`。
- 仓库不提交 QQ 号、群号、聊天原文、个人称呼、API Key 或 token。
- 表情包和图片不直接暴露文件路径给模型。
- 控制中心与星图默认只监听 `127.0.0.1`。
- 删除记忆与清理备份都需要显式操作，不自动执行破坏性删除。
- 备份中的 `purge-before-*` / `retention-before-*` 可能仍含已删除数据，只保留最近一份。

---

## 已知边界

```text
不是全量记忆
不是可回放的人生
不是人格/关系状态机
不是自动叙事系统
不是多 Agent 集群
不是开箱即用的产品
```

它更适合作为一个长期运行、持续调优的个人项目：

```text
本地优先
可观测
可回滚
可评测
```

---

## 更多文档

- [小鲸鱼-完整技术档案.md](小鲸鱼-完整技术档案.md)
  - 架构、消息链路、记忆、检索、主动、审计、星图、控制台、阈值、验证矩阵
- [有限记忆的一生感实现方案.md](有限记忆的一生感实现方案.md)
  - 记忆分层、遗忘、叙事与长期陪伴的设计思路
- [记忆星图-3D元素清单.md](记忆星图-3D元素清单.md)
  - 3D 星图的视觉与交互说明

---

## 一句话总结

> 小鲸鱼不是“记住一切的聊天机器人”，  
> 而是一个会筛选、会遗忘、会重排、会克制的有限记忆智能体。
