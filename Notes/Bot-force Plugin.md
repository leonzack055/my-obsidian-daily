---
type: note
topic: agent
source:
created: 2026-11-19
tags:
  - fleeting
  - 笔记工具
---
# Bot-force 简单使用方法

## 最短用法

直接在对话里说：

```
make me a research analyst
```

或者中文：

```
给我建一个专门整理 Obsidian 笔记的 Bot
建一个每天盯竞品动态的分析师
```

它的行为是：先 `list_agents` 查重 → 直接 `create_agent` 建 Bot → 报告结果。设计上不问问题（zero questions），模糊的请求它会自己选一个最有用的解读，你事后调就行。

## 它会生成什么

|产物|内容|
|---|---|
|profile|独立的 `~/.hermes/profiles/<name>/`，自己的 config / memories / skills|
|Bot 名 + 头像|角色名（`Quill`、`Kairo`、`Nova`），不是 `Writer` 这种岗位词|
|`SOUL.md`|为这个岗位现写的人格 + 职责 + 边界 + 升级路径|
|起始记忆|一些基础事实 + Hermes 已知的关于你的信息|
|工具集|默认 `file web browser`，按角色加（coder 加 terminal/code_execution，社媒加 image_gen/vision/cronjob）|
|技能|按角色挂对应类目|
|routines|可选 cron，如「每天 8 点做收件箱摘要」|
|Bot Chat|会尝试自我介绍，登录态可能要你补|
|gateway|检查能否热加载|

创建失败会自动回滚清理；`sign-in` 和 gateway 就绪可能是待办，结果里会写明。

## 它提供的 14 个工具

create_agent create_team teach_agent update_agent  
copy_agent list_agents check_agents check_install  
agent_journal ask_agent share_agent import_agent  
hide_agent delete_agent

所以你可以直接命令它做后续操作：

|你说|它调|
|---|---|
|「让 Quill 更简洁一点」|`update_agent(soul_append=...)`|
|「再来一个跟 Quill 类似的」|`copy_agent` → `update_agent` 特化|
|「Quill 有哪些工具？」|`list_agents` / `check_agents`|
|「Quill 最近做了什么？」|`agent_journal(action="read")`|
|「把 Quill 备份一下」|`share_agent(mode="backup")`（含聊天历史，会提醒你）|
|「把 Quill 藏起来」|`hide_agent`（不是删除）|
|「给我配一个团队」|`create_team`（一次建 lead + 2–4 个专家）|

## 两个重要区别

1. 它建的是 Bot Mode 的 Bot，不是裸 profile

| |`hermes profile create`|bot-forge `create_agent`|
|---|---|---|
|产物|一个身份目录|profile + Bot roster 条目 + SOUL + 记忆 + 工具 + routines + Bot Chat|
|面向|手动配置|一句话上岗|
|适合|你要自己精细控制|你想要一个能直接对话/干活的「人」|

2. 可以直接拿来做 kanban 的 assignee

新 Bot 的 profile 名就是 `assignee`：

```
hermes kanban create "整理本周笔记" --assignee quill --workspace dir:/home/leon/leon/hermes笔记
```

Bot Mode 和看板是两套并行系统：Bot 有自己的 Bot Chat（聊天界面），看板把卡派给同一个 profile 时，起的是普通 worker 进程——两者共享同一份 SOUL 和记忆。

## 三个注意点

1. 当前会话可能还看不到 `create_agent`。插件工具是在会话启动时注入 schema 的；这个插件是本次会话中途出现的（技能列表的变更通知能看出来）。如果它说「没有这个工具」，开个新会话（`/new`）再试，或者重启 gateway。
2. 一句话说清角色更省事。它不提问，但「帮我建个 Bot」这种太空的话，它会按最通用解读建一个通用助手——不如直接说「每天整理 Obsidian inbox 并按 PARA 归类」。
3. 给它 terminal 就配 docker sandbox。技能里明确要求：任何拿到 `terminal`/`code_execution` 的 Bot 都该带 `sandbox: "docker"`，让 shell 跑在容器里而不是你机器上。

## 最贴合你场景的三种说法

```
给我建一个 Archivist：每天 7 点扫 Obsidian 的 inbox/，按 PARA 归类到 Notes/Resources，维护项目 next actions
建一个竞品分析师，每周一早上给我一份竞争对手动态摘要
建一个邮件管家，每天 8 点整理收件箱，只把需要我决策的挑出来
```

一句话：装好了，直接对它说「make me a …」，剩下的它自己设计、自己创建、自己报告。

（标题就是观点。用自己的话展开 3–6 句，保持原子、能独立读懂。）

关联：[[ ]]、[[ ]]
