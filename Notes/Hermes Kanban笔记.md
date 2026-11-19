---
type: note
topic: agent
source: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
created: 2026-11-17
area: "[[HEREMS]]"
---
# Hermes Kanban笔记
Kanban 功能实际上和console, tui, desktop 差不多，它利用 <mark style="background: #BBFABBA6;">dashboard</mark> 开启一个`gateway server` 来创一个 `http://127.0.0.1:9119` 的浏览器网页入口；
它与日常对话不一样的地方是提供了一个上帝视角，去总览[[HERMES]] 的总体，它是为`Multi-Agent`配置和自动化使用的，所以不能简单地把它当终端用。

## Dashbord界面

![[Screenshot from 2026-11-17 18-12-32.png]]


## Iteration Cap
**它不是 token 预算，而是「模型调用轮数」预算。** 一次 iteration = { 一次带工具的模型调用 + 拿到工具结果}，会话循环里再跑下一轮。上限来自 `agent.max_turns`，默认 150`hermes config get agent.max_turns → 150`，所以看板 **worker** 的 150 轮就是它的 **finite budget**，90% `checkpoint notice` 在135 轮左右。

## Kanban 怎么记录?
> [!quato]
> Every task is a row in ~/.hermes/kanban.db; every handoff is a row anyone can read and write; every worker is a full OS process with its own identity.

Kanban 中的一切可读可写的内容都写进 `kanban.db` 这个KV数据库中，而且每一个worker都是独立的它没有继承上下文。

 看板工具集里（一共 16 个）：

- 读：`kanban_show`、`kanban_list`、`kanban_attachments`
- 写：`kanban_create`、`kanban_link`、`kanban_comment`、`kanban_complete`、`kanban_request_review`、`kanban_request_changes`、`kanban_block`、`kanban_unblock`、`kanban_schedule`、`kanban_attach`、`kanban_attach_url`、`kanban_heartbeat`
- `kanban_complete` / `kanban_request_review` / `kanban_block` 属于终态工具：调完这次 run 就结束，不能再继续干活

## kanban.db 的写入者有三类

| 写入者                                 | 写什么                                                                                                                                                                                |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| agent 的看板工具（上面那些）                   | 卡片的业务状态：`comment` → `task_comments` + 事件；`complete` → `tasks.status=done` + `result` + `task_runs.outcome=completed`；`request_review` → `status=review`；`block` → `status=blocked` |
| dispatcher（gateway 内每 60 s 一次）      | 生命周期层面：promote、原子 claim、spawn、心跳检查、超时杀进程、崩溃回收、`consecutive_failures` 计数、熔断 block——全都不经过任何工具                                                                                        |
| CLI / HTTP API / 桌面看板 UI / 消息通知<br> | `hermes kanban create/move/unblock/comment/log`、网页看板点按钮、`kanban_notify_subs` 回推订阅                                                                                                  |

## 库里的 8 张表

- `tasks`（卡本体：status、assignee、consecutive_failures、last_failure_error、max_retries…）
- `task_runs`（每次 attempt 一行：outcome、summary、error）、
- `task_comments`（交接评论）、
- `task_events`（append-only 事件流，看板时间线和通知都读它）、
- `task_links`（父子依赖）、
- `task_attachments`
- `kanban_notify_subs`
- `sqlite_sequence`

## Opt-in 什么意思?
| 场景                           | 90% 迭代提醒    |
| ---------------------------- | ----------- |
| 看板 worker（dispatcher 派生的）    | 默认开（0.9，自动） |
| 普通对话、`delegate_task` 子代理<br> | 默认关，要自己设才开  |
启用时它才会出现`checkpoint notice`的提示，让worker可以在iteration 工 具调用烧光前去查看是否完成，如果提示生效，没完成任务就`kanban comment` 在`.db` 中写评论，下一次`dispatcher`派新的worker来干活就先读`comment` 这样不会从0开始反工。如果，真的要完成了，就`kanban complete` 或者 `kanban request review`。

## Triage 啥意思?
简单说就是要进行搜索和分析的任务，它要进行Research给出明确的报告和计划，而不是一个明确的任务，如"去修复页面的显示不对齐问题"; 它更像是一个，研究"mcp算法并实现动态调头"

## Tenant
文档里有原话定义（kanban.md），机器翻译是租户，我觉得用任务类型标签来解释更合理。

> Tenant — optional string namespace _within_ a board. One specialist fleet can serve multiple businesses (`--tenant business-a`) with data isolation by workspace path and memory key prefix. Tenants are a soft filter; boards are the hard isolation boundary.

即：tenant 是 board 内部的可选命名空间，作用是以轻量方式分隔同一 board 上的不同业务或客户。

```
# 建卡时带上
hermes kanban create "整理客户 A 本周反馈" --assignee ops --tenant business-a
# 过滤查看
hermes kanban list --tenant business-a
hermes kanban watch --tenant business-a
```

作用： 看板 UI，每张卡带 tenant 标签，工具栏有下拉过滤，默认值取 `dashboard.kanban.default_tenant`。tenant 能防「看板很乱」，不然任务太多不好规类，你可以把它想成`gitlib` 上的 <mark style="background: #ADCCFFA6;">label</mark>。
## 核心概念
### Board
Board 可以理解为一个独立的看板，也可以理解为一个独立的任务队列。 一个 Hermes 安装里可以并存多个 board，互不干扰：每个 board 都有自己的库、自己的 workspace 目录、自己的派发循环。各 board 上的任务不会互相冲突。

#### 常用命令
查看kanban中的board
```bash
$ hermes kanban boards list
    SLUG                      NAME                          COUNTS
●   default                   Default                       (empty)

Current board: default
```
默认 board为default，`kanban list`是空的，这是最典型的新装状态。要建第二个：
```
hermes kanban boards create myproject --name "My Project"
hermes kanban boards switch myproject     # 之后的 CLI 调用都落在这个 board
hermes kanban boards list                 # 看所有 board + 任务数
```

其他还有 `boards show`（当前是哪个）、`rename`（改显示名，slug 不可变）、`export` / `import`（导出成 tar.gz 迁移）、`set-default-workdir`（给这个 board 的任务设默认工作区）。

#### 什么时候需要多开 board?
一句话：当一个工作流不该和另一个工作流抢队列、抢审计记录、抢归档策略时。
- 一个仓库一个 board（代码流水线互不干扰）
- 一个领域一个 board（“个人运维” vs “客户项目”）
- 生命周期不同的（长期 journal 的 board 永不归档，项目 board 干完就 archive）
- 隔离敏感度不同的（不同 tenant / 不同 profile 群）

> [!notice] 提示  
> 『each board has its own dispatcher loop』，实现上则是一个 dispatcher 进程每 tick 扫过所有 board（逻辑上各 board 独立推进，物理上共用一个循环）。

## Tasks
Task 的模型定义——一张卡就是数据库里的一行，不是进程、不是文件夹。进程（worker）是临时的，行才是持久的，写入数据库。
### 八个状态，以及谁推动转换

| 状态         | 含义                                             | 谁来推动                                         |
| ---------- | ---------------------------------------------- | -------------------------------------------- |
| `triage`   | 粗想法停放区，等 decomposer/specifier 展开               | dispatcher（`auto_decompose`）或人点 ⚗/✨          |
| `todo`     | 等父卡完成                                          | 父卡 `done` 后由 `recompute_ready` 自动升到 `ready`  |
| `ready`    | 可被认领                                           | dispatcher 每 tick 原子 claim → `running`       |
| `running`  | 已被某个 worker 占住（有 claim_lock + PID + heartbeat） | worker 干活中                                   |
| `blocked`  | 卡住了：缺依赖/缺输入/缺能力                                | worker `kanban_block`、人 `block`；`unblock` 放回 |
| `review`   | 实现完，等评审                                        | worker `request_review`、人 `request-review`   |
| `done`     | 完成（可逆，能 reopen）                                | `complete`（worker / 评审者 / CLI / 看板）          |
| `archived` | 终态（不可逆），默认视图隐藏，订阅自动移除                          | `archive` 或 `boards rm`                      |

### 建卡的五个入口

| 入口                         | 谁用                 | 形态           |
| -------------------------- | ------------------ | ------------ |
| `hermes kanban create ...` | 人 / 脚本 / cron      | CLI          |
| `kanban_create(...)`       | worker、编排者 profile | 工具           |
| 看板 UI 的 `+`                | 人                  | dashboard 插件 |
| `POST /tasks`              | 外部集成               | HTTP API     |
| `/kanban` 斜杠命令、消息平台        | 人在聊天里              | gateway 适配器  |

另外有两个「自动建卡」的机制，也走同一个函数：
- decomposer：落进 `triage` 的卡被自动拆成一组子卡（`auto_decompose`）
- swarm 建图：`hermes kanban swarm` 一次性建一张并行 workers → verifier → synthesizer 的图
#### kanban create时发生了什么？
```
create_task()
 ├─ 生成 id：t_ + 4 字节随机 hex
 ├─ assignee 规范化（_canonical_assignee）
 ├─ completion_contract 校验（只允许 local-only / OWNER/REPO / 精确 PR URL）
 ├─ idempotency_key 查重 → 命中就直接返回已有的 id，不建新行
 ├─ tenant 解析：显式 > 父卡继承 > NULL
 ├─ 状态判定：有未完成父卡 → todo；triage → triage；initial_status=blocked → blocked；否则 ready
 ├─ workspace：dir/worktree 且没给路径 → 吃 board 的 default_workdir
 └─ 写 tasks 行 + `created` 事件（+ 可能的通知订阅）

```

建卡不等于派发。`create` 只写一行，工作区要到 spawn 时才 `mkdir`，worker 进程要等 dispatcher 下一个 tick。所以刚建完的卡通常是 `ready`（或 `todo`），不是 `running`。

create 时就有状态——不是“未初始化”，而是四种之一：==🟡`ready` / `todo` / `triage` / `blocked`（取决于有没有未完成的父卡、是否 `--triage`、是否 `--initial-status blocked`）==。

==🟠状态不是 dispatcher 单方面控制的，是多方写、单点收敛==。dispatcher 只负责“调度”那一段，不拥有状态。

#### 谁有权改哪个状态

| 转换                                                      | 谁写                             |
| ------------------------------------------------------- | ------------------------------ |
| （新建）→ `ready` / `todo` / `triage` / `blocked`           | `create_task`                  |
| `todo` → `ready`（父卡全 done）                              | dispatcher 的 `recompute_ready` |
| `ready` → `running`（原子 claim）                           | dispatcher                     |
| `running` → `ready` / `review`（崩溃、超时、重试）                | dispatcher 的回收逻辑               |
| `running` → `done` / `review` / `blocked` / `scheduled` | worker 自己（终态工具）                |
| `blocked` → `ready` / `todo`                            | 人 / CLI / 编排者（`unblock`）       |
| `ready` → `review`                                      | worker 或人（`request-review`）    |
| `review` → `done`                                       | 评审者 / 人                        |
| `review` → `ready`                                      | 评审者 `request_changes`          |
| `*` → `blocked`（熔断）                                     | dispatcher（连续失败）               |
| `done`/`*` → `archived`                                 | 人（`archive`）                   |

#### 一张卡跨三个 worker 的实际时间线

```
create            → ready
dispatcher claim  → running   [run #1]
worker 崩溃        → ready     [run #1 结束，outcome=crashed，计数+1]
dispatcher claim  → running   [run #2]
worker block      → blocked   （等人）
人 unblock        → ready
dispatcher claim  → running   [run #3]
worker complete   → done       （计数清零）
```

状态在行上一直在变，worker 换了三个，`task_runs` 里留了三行。卡的连续性来自那张行和 task run 历史表，不来自任何进程。
#### 为什么多方写不会打架？

每次状态写入都带条件（CAS），不是盲写：

```
UPDATE tasks SET status='done' ... WHERE id=? AND status IN ('running','ready','blocked','review')
UPDATE tasks SET ... WHERE id=? AND current_run_id=?    -- worker 证明自己拥有这个 run
```

## Link
任务间的依赖关系，实际上是在`task_links`这个表中写一行带有`parent`和`child`的依赖条目。`dipatcher`的执行规则是保证所有父OK后，将`task`由`todo->ready` 。对于`ready`后派给`worker`的任务都不能再使用`task_link`创建父依赖了，但只有一种情况可以；
### 唯一的例外
例外条件就三个，同时满足才放行：

| 条件                                               | 含义                                                            |
| ------------------------------------------------ | ------------------------------------------------------------- |
| `child_id` = 自己这张卡                               | 不是给别人加约束                                                      |
| `expected_child_run_id` = 自己当前的 `current_run_id` | 证明自己拥有这个 run（这个 id 由 dispatcher 通过 `HERMES_KANBAN_RUN_ID` 注入） |
| 紧接着要做 `kanban_block(kind="dependency")`          | 这条边是给“我马上要挂起”用的，不是给“我继续跑”用的                                   |
### 实际场景

worker 干到一半发现：「我要写的这个迁移脚本，依赖另一张卡先把 schema 定义出来。」
正确做法不是硬着头皮写、也不是 `block(needs_input)` 等人：
```
# 1. 把缺失的前置工作建成一张卡（或指向已有的卡）
pid = kanban_create(title="定义 users 表 schema", assignee="architect")

# 2. 把它挂到自己头上——我就是 child，并出示我的 run id
kanban_link(parent_id=pid, child_id=我自己的task_id, expected_child_run_id=<我的 run id>)

# 3. 立刻挂起，理由写 dependency
kanban_block(reason="等 users 表 schema 定稿（t_xxxx）", kind="dependency")

```

之后 dispatcher 会等 `t_xxxx` 变 `done`，自动把这张卡从 `todo`（dependency block 会停在 `todo`）升回 `ready` 并重新派发——不需要人来 unblock。
举个例子:
```
worker 发现自己缺前置工作
   ↓
kanban_create(...)  ← 建父卡
   ↓
kanban_link(parent_id=新父卡, child_id=自己, expected_child_run_id=我的run)
   ↓
kanban_block(reason="等 t_xxx", kind="dependency")   ← 必须紧跟
   ↓
卡 → todo（不是 blocked 列！dependency 类停在 todo）
   ↓
父卡 done → recompute_ready 自动升回 ready → 重新派发新 worker

```

<mark style="background: #FF5582A6;">如果 link 完不 block、继续干到底并 `complete`，就变成一笔假账</mark>：**子卡 done，父卡还没做，依赖边只留在历史里，`_parents_satisfied` 已经放行过一次，之后再 reopen 就会读到不一致的状态**。所以文档用 “immediately before” 把这两个动作绑成一个原子动作。




## Dispatcher
它是一个「看板的操作系统调度器」——不管 worker 内部怎么干活，只负责让队列始终保持正确的状态。
### 一个 tick 内的执行顺序（默认 60 秒）

|步骤|动作|失败/边界处理|
|---|---|---|
|1a|回收 stale claim：claim 过期（默认 15 min）仍在 `running` 的卡|回 `ready`/`review`，不计失败|
|1b|回收 crashed worker：PID 已死但 TTL 还没到期|立刻 `crashed`，计入失败计数；只处理本机 claim|
|1c|reap 超期存活的 worker：见下|记 `terminal_worker_reaped`|
|2|计算本 tick 能 spawn 几个|受 `max_in_progress`（全局）、`max_spawn`（单 board）、`max_in_progress_per_profile` 三层约束|
|3|promote：父卡全 done 的 `todo` → `ready`|顺带处理 review 泳道|
|4|原子 claim + spawn：每条 `ready`（和 `review`）行，claim 成功后调 spawn|失败走 `_record_task_failure`|
|5|WAL checkpoint（仍在锁内）|避免 WAL 无限增长|

整个 tick 在单写者锁里跑（`_dispatch_tick_lock`，按 db 路径加锁）。两个 dispatcher 同时 tick 时，输的那个直接返回 `skipped_locked=True`、一个字节都不写。


## Comment
comment 是 agent 之间唯一的通用沟通渠道——一块共享留言板，不是消息队列。

### 它的协议属性

|属性|含义|
|---|---|
|append-only|只能追加，不能编辑或删除；写入时同时落一条 `commented` 事件|
|没有收件人|不指定发给谁，任何能看到这张卡的人/agent 都能读|
|持久|存在 `task_comments` 表里，跨 run、跨进程、跨重启|
|人机同权|worker 用 `kanban_comment`，人用 `hermes kanban comment` 或看板输入框，写进同一张表|
|不改状态|纯旁路信息，不推动生命周期（唯一的例外是影响 `active_pr` 重派保护）|
### 使用场景
因为「人机协作」需要一个不需要改状态的沟通口。 大多数需要人介入的时刻，人只想说一句「这个字段保留，别删」，不想去动卡的状态——评论就是这个载体。

### 三个入口，同一张表

```
# 1. CLI
hermes kanban comment t_1a2b3c4d "users.email 保留，别删；重复数据保留最新一条"
# author 默认取 $HERMES_PROFILE，没有就记 "user"；可显式 --author leon

# 2. HTTP API
POST /tasks/:id/comments

# 3. 看板 UI
卡片详情里的评论输入框
```

人写的和 worker 写的进同一个 `task_comments` 表，按时间排序，渲染格式完全一样（都包成 `comment from worker \`author``）。

#### 典型的闭环长这样

```
worker:   kanban_block(reason="users.email 唯一索引与现有重复数据冲突，需要决策")
          ↓ 卡停在 blocked，人收到通知
人:       hermes kanban comment t_x "清数据：保留最新一条，其余标 deleted"
人:       hermes kanban unblock t_x
          ↓ dispatcher 重派
新 worker: kanban_show() → 在评论线程里看到人的指令 → 照做
```

**注意一个关键点**：评论本身不会唤醒任何人。卡在 `blocked` 时，你写完评论它还是 `blocked`——必须再 `unblock`（或 `promote`），dispatcher 才会派发；worker 是在被 spawn 时读到这条评论的。



## 摘录与评注

> [!quote] 原文摘录
> Dispatcher-owned workers get one checkpoint notice near 90% of their finite iteration budget, attached to a fresh tool result while another tool-capable call remains.

这里的 <mark style="background: #BBFABBA6;">checkpoint</mark> 是「存档点／进度检查点」，不是训练模型那种权重快照。

在这个机制里的具体含义: 
Kanban 的 Dispatcher拥有的worker 是独立的 OS 进程，跑在有限迭代预算里（例如 50 轮）。一旦预算耗尽、崩溃、或被 dispatcher 的 reaper 杀掉，进程里的一切（对话上下文、内存里的中间结果）就全没了——任务会回到看板重新派给下一个 worker。下一个 worker 唯一能看到的东西，只有 ~/.hermes/kanban.db 里那条任务行上的持久化内容。

所以「checkpoint notice」就是提醒 worker：在还没被硬上限截断之前，把当前状态写入磁盘，三选一：

- `kanban_complete` — 任务已按验收要求验证完毕，标记完成
- `kanban_request_review` — 成果已可评审，转给人或另一角色
- `kanban_comment` — 还没做完，把「做到哪一步、关键结论、下一步该做什么」写成交接备注，然后继续
也就是断点续传式的「存档」, 不存，**下一个接手的 agent 就得从零重做**。

容易混淆的点:
- 不算 `git commit/diff`。文档特意写「A diff or commit alone is not completion evidence」——代码改了不等于任务做完了，`checkpoint` 要的是对任务要求的验证结论或可读的进度交接。
- 改了一算，如果一个worker结束，它没有进行上面三个Kanban工具标记收尾，那么这个就白工作了。

>[!quato]
>The hard cap, toolless final summary, and consecutive-failure circuit breaker are unchanged: workers that still exhaust their budget remain subject to bounded retries.

那段看板文档想说的是——我们在 90% 处好心提醒一次，让 worker 有机会自己收尾；但没有放宽任何限制：该断还是断，断了就只能写总结，失败多了照样熔断，而且提醒本身不保证生效。它是一层「礼貌的提醒」，不是「宽限期」。 它实际上是上一句`checkpoint notice`的一种免责说明，`checkpoint notice` 可能会被模型忽略掉。

- `hard cap` : `IterationBudget.consume()` 在 `used >= max_total` 时返回 False，循环立刻 break，`turn_exit_reason = "budget_exhausted"`。源码注释写得很直白：<mark style="background: #CACFD9A6;">『The warning never extends the hard budget.』</mark>——`checkpoint notice`不会多给一轮。
- `toolless final summary`:  预算耗尽且本轮还没有成品回复时，`_handle_max_iterations()` 会额外发一次不带工具的请求，让模型写个总结交差（CLI 上会显示`「Iteration budget exhausted — requesting summary...」`）。这一次请求里模型调不了任何工具，所以 无法再 `kanban_complete`——这正是要提前提醒的原因.
-  `consecutive-failure circuit breaker`:  dispatcher 对同一张卡连续失败计数：**spawn** 连续失败超过 `kanban.failure_limit（默认 2）`、协议违规连续超过 `_PROTOCOL_VIOLATION_FAILURE_LIMIT（默认 3）`，就自动 **block** 掉这张卡，不再重派，避免同一个坑反复撞。预算耗尽走的是`_record_task_failure(outcome="timed_out")`,计入这条熔断线

它的总体流程:
```
预算用掉 ~90%
  └─ 注入 checkpoint 提醒（附在最新 tool 结果上，工具仍可用）
        ├─ 模型照做 → kanban_complete / request_review / comment ✅ 优雅收尾
        └─ 模型忽略 → 继续烧
              └─ 预算真正耗尽（hard cap）
                    ├─ 一次无工具请求，逼模型写总结（文字交差，不算完成）
                    ├─ 看板记 outcome=timed_out，失败计数 +1
                    └─ dispatcher 有限次重派；连续失败到阈值 → 自动 block

```




关联：[[ HERMES]]、[[ ]] 