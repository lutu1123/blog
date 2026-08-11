---
title: "Hermes Agent Kanban Swarm 功能深度解析(面向 AI 开发者)"
date: 2026-08-11
tags: ["hermes-agent", "kanban", "multi-agent", "ai-agents", "orchestration"]
author: "路途"
---

你有没有遇到过这样的场景:让 AI 跑一条多步骤流水线——调研、写作、审核、发布——结果子代理崩了,上下文全丢;任务做到一半,没人知道它卡在哪;想插入一个人类审核环节,却要改代码、重启进程。如果你被这类问题折磨过,Hermes Agent 的 Kanban Swarm 值得一看。

它本质上是一个**基于 SQLite 看板的多智能体协作工作队列**:每个任务就是数据库里的一行,每个 worker 就是带独立身份(自己的 config/skills/memory)的完整 OS 进程。设计目标很直白:替代"脆弱的进程内子代理群"。本文基于官方文档与本机实测,拆解它的核心机制。

## 核心概念:五个词看懂架构

- **Board**:独立的 SQLite 数据库(`~/.hermes/kanban/boards/<slug>/kanban.db`),带独立的 workspaces/、logs/ 和 dispatcher 循环。board 是**硬隔离边界**:worker 只能看到自己 board 的任务,跨板 link 被禁止。
- **Task**:一行记录,含 title、body、单一 assignee(profile 名)、status(`triage|todo|ready|running|blocked|review|done|archived`),以及 workspace_kind、parents、current_run_id、model_override 等字段。
- **Profile**:worker 的身份。assignee 必须是已存在的 profile,worker 以 `hermes -p <assignee>` 拉起,带独立 `HERMES_HOME`。
- **Dispatcher**:常驻调度循环(默认在 gateway 进程内),每 tick 做三件事:回收过期 claim → promote ready 任务 → 原子认领并 spawn 对应 profile。
- **Workspace**:三类。`scratch`(默认)是临时目录,**任务完成即删**;`dir:<绝对路径>` 指向既有共享目录(相对路径在 dispatch 时被拒,防 confused-deputy);`worktree` 是 git worktree,可指定分支。要保留产出,就用后两者。

为什么重要:board 是隔离、task 是持久化、profile 是身份、dispatcher 是心脏、workspace 是战场——五个概念对应五个独立的工程决策。

## 快速上手:三条命令跑起来

```bash
# 1. 初始化看板(幂等创建 kanban.db,可重复执行)
hermes kanban init

# 2. 启动 gateway——dispatcher 默认驻留在 gateway 进程内,负责调度
hermes gateway start

# 3. 建任务、看事件流、看统计
hermes kanban create "调研 Kanban 文档" --assignee researcher --workspace worktree
hermes kanban watch   # 全板事件流(completed/blocked/stale 等)
hermes kanban stats   # 每状态 + 每 assignee 计数,含 oldest-ready 年龄
```

`create` 的高频 flag 值得记住:`--parent`(可重复,建依赖边)、`--workspace`(scratch|worktree|`dir:<绝对路径>`)、`--priority`、`--max-runtime`(如 30m / 2h / 1d)、`--skill`(强制加载技能)、`--model/--provider`(单卡钉模型)、`--idempotency-key`(自动化去重)。调试期还有 `dispatch --dry-run` 手动跑一次调度、`diagnostics` 看板健康快照、`gc` 清理归档任务的工作区与旧日志。

另有 `hermes kanban swarm "主题" --workers researcher,architect --verifier reviewer --synthesizer writer` 一键生成 swarm 拓扑:原子提交"根卡(黑板,共享上下文以结构化 JSON 评论存根卡)+ N 个并行 worker + verifier 门控 + synthesizer 汇聚"的完整结构。

## 任务生命周期:崩溃恢复下沉到基础设施

状态机:`todo → ready → running → done | blocked`(另有 triage / review / scheduled / archived)。

1. **claim(原子认领)**:dispatcher 用事务认领 ready 任务,在 `task_runs` 建行并把 `tasks.current_run_id` 指向它。
2. **spawn**:注入 `HERMES_KANBAN_TASK` / `HERMES_KANBAN_WORKSPACE` / `HERMES_KANBAN_BOARD` 三个环境变量,拉起完整进程。
3. **heartbeat 保活**:长任务须定时调用 `kanban_heartbeat(note=...)`——可能超过 1 小时的任务必须每小时至少一次。运行超 `kanban.dispatch_stale_timeout_seconds`(默认 4h)且 1h 无心跳的任务会被 stale reclaim 回 ready 重派,**不计失败计数,但当前 run 进度丢失**。
4. **超时与失败熔断**:`--max-runtime` 超时触发 `timed_out`,SIGTERM(5s 宽限后 SIGKILL)并重新排队;连续 spawn 失败达 `failure_limit`(默认 2)后自动 block(gave_up),防止无限 thrash。此外还有 respawn guard:上次 run 遇 quota/auth/429、1 小时内刚成功、或近期评论含 GitHub PR 链接时,dispatcher 会拒绝重派,防 worker 风暴。
5. **protocol_violation 防护**:worker 进程在任务仍 running 时以 0 退出(典型:只写了文字没调 complete/block),会触发违规事件,连续违规自动 block。

另外,`kanban_complete` 里声明的 `created_cards` 若含不存在的 phantom id,会被内核拒绝导致完成失败——结构化交接是有校验的。

这套机制的妙处在于:**任何一个 worker 挂了,任务不会死,只会被回收重派**——"崩溃恢复"从代码层面下沉到了基础设施层面。

## Worker 与 Orchestrator 的工具集

`HERMES_KANBAN_TASK` 置位即注入专用工具集:`kanban_show`(读任务 + 父任务 handoff + 评论线程)、`kanban_complete`(summary + metadata 结构化交接)、`kanban_block`(按 kind 路由)、`kanban_heartbeat`、`kanban_comment` / `kanban_attach`;orchestrator 额外获得 `kanban_create` / `kanban_link` / `kanban_unblock` 等路由工具。约定:**worker 不 fan-out,orchestrator 不执行实现**。为什么用工具而非 shell out 到 CLI?远程终端后端(容器里没有 hermes 和 kanban.db)、无 shell 引号脆弱性、结构化 JSON 错误——三条理由都成立。

## 关键机制:fan-in 与 fan-out

在讲编排之前,先补一个容易被忽略的架构点:**双入口**。模型侧通过 `kanban_*` 工具集直接驱动看板;人、脚本、cron 则走 `hermes kanban …` CLI 或 `/kanban …` 斜杠命令,再或者用 dashboard 插件。三个入口共用同一 `kanban_db` 层,读写不漂移——这意味着人工介入(comment、unblock、改状态)和 agent 自动化可以在任意时刻交错发生,这正是"对等协作"的基础。

- **fan-in(parents 依赖)**:`kanban_create(..., parents=[...])`。子卡在所有 parent 达到 done 前停在 todo,最后一个 parent 完成时自动 promote;若创建时 parents 已全 done,子卡直接以 ready 创建。
- **fan-out(orchestrator 拆解)**:orchestrator 建 N 张并行子卡 + 1 张聚合卡(parents 指向全部子卡),然后 complete 收尾自身。**决策必须在 fan-out 前定**:并行子卡互相看不到,每个子卡 body 必须自包含其依赖的所有决策。
- **动态建边**:`kanban_link(parent_id, child_id)` 事后补依赖,无需重建任务。
- **上下文交接**:parent link 就是交接通道——子 worker 的 worker_context 含"Parent task results"段落,每个已完成的 parent 的 summary + metadata 原样带入。已完成卡是不可变历史;重试走 prior attempts,后续工作则建新子卡。

值得一提的还有 task 与 run 的语义分离:**task 是逻辑单元,run 是一次尝试**。`task_runs` 表每次尝试一行,summary/metadata 挂在 run 上——所以"谁在什么时候做过什么、结果如何"全程可审计,这也是它与一次性子代理调用最本质的区别。

## 横向对比:和 delegate_task / cron 怎么选

| 维度 | delegate_task | cronjob | 独立进程 | Kanban |
|---|---|---|---|---|
| 形态 | RPC 调用(fork→join) | 定时调度器 | 完全独立进程 | 持久队列 + 状态机 |
| 持久性 | 父进程退出即丢 | DB 存 job | 靠外部管理 | SQLite,重启可恢复 |
| 身份 | 匿名子代理 | 无 | 独立会话 | 具名 profile + memory |
| 时长 | 分钟级 | 3 分钟硬中断 | 小时/天 | 不限,靠心跳机制 |
| 协调 | 层级式 | 单向触发 | 手工 relay | **对等**:任何 profile 读写任何任务 |
| 人工介入 | 不支持 | 不适用 | 可 PTY | comment / unblock 任意时刻 |
| 审计 | 压缩即丢 | 有投递记录 | 会话日志 | events + runs 永久留存 |

一句话区分:**delegate_task 是函数调用,Kanban 是工作队列**。二者可共存——kanban worker 内部照样可以调 delegate_task。

## 实战示例:本文就是这么写出来的

这条博客流水线就是 Kanban Swarm 的真实运行实例。收到写作请求后,orchestrator 拆解出五张卡,靠 parents 依赖链串成 pipeline:

```text
✓ t_d1cbcdcd  done      orchestrator  拆解
● t_936719c4  running   researcher    调研素材
○ t_6c70e5da  todo      writer        写作(约 2000 字,parent: researcher)
○ t_da3616b2  todo      reviewer      审核(parent: writer)
○ t_62c5d719  todo      publisher     发布到 GitHub Pages(parent: reviewer)
```

每个下游卡以上游为 parent,前序完成才自动 ready——全程零人工干预,发布凭证这类敏感信息随卡传递,文章只讲机制不复述内容。

## 对 AI 开发者的启发与最佳实践

什么时候该用 Kanban Swarm:跨 agent 边界、需要存活重启、需要人工门、需要事后可发现的流程——研究分流、定时运维、工程流水线、舰队作业都合适。

常见坑:①**未知 assignee 静默滞留**——卡永远停在 ready,dispatcher 静默失败,先 `hermes profile list` 校准;②scratch 工作区完成即删,要保留产出就用 worktree/dir;③并行子卡互相看不到,决策提前钉进 body;④它是**单机设计**(本地 SQLite + 本机 PID 判崩溃),多机请每机独立 board 再桥接。

成本策略上,官方建议 frontier 模型跑 orchestrator(拆解需要判断力)、廉价模型跑 worker(大头 token 花在 worker 上)——每 profile 独立 config.yaml,dispatcher 按 profile 注入 `HERMES_HOME`;偶发质量敏感卡用 `hermes kanban set-model` 单卡钉回强模型。观察变更也有现成钩子:`kanban_task_claimed/completed/blocked` 生命周期事件可注册在 dispatcher profile,集中观测全板变迁。

从"进程内子代理群"到"持久化工作队列",Kanban Swarm 补上的其实是工程化最后一块拼图:可观测、可恢复、可人工介入。对正在构建多智能体系统的开发者,这五个词——board、task、profile、dispatcher、workspace——值得写进你的架构词汇表。
