# 办公自动化平台架构设计

> 基于 waiy-browser-use + OpenCLI + 主 Agent 的企业办公自动化平台架构，覆盖 CLI 全生命周期管理（生成→巡检→更新→推送）和任务执行两大核心链路。

---

## 1 业务背景与核心假设

### 1.1 业务场景

客户提供 **debug 账号**，我方研发团队可访问客户办公网内的各类 Web 系统（OA/ERP/CRM/钉钉/企微等）。基于此：

1. **生成网站 CLI**：通过 debug 账号登录客户系统，利用 OpenCLI 自动发现 API、生成 CLI 适配器
2. **定期巡检 CLI**：周期性验证已有 CLI 是否仍可正常执行，发现失效及时修复
3. **更新推送 CLI**：客户网站发生变化后，重新生成/修复 CLI 并推送到客户端（低频）

### 1.2 核心假设

| 假设 | 说明 |
|------|------|
| 客户提供持久 debug 账号 | Cookie/Session 可长期复用，或可通过密码重新登录 |
| 客户网络可达 | 研发侧 Agent 可通过 VPN/专线/公网访问客户 Web 系统 |
| CLI 变更低频 | 网站大改版频率低（季度/年级），巡检修复足以覆盖 |
| 客户端有 Node 运行环境 | OpenCLI Skill 在客户侧以 Node sandbox 方式运行 |

---

## 2 整体架构

```plantuml
@startuml
skinparam packageStyle rectangle
skinparam componentStyle rectangle
skinparam backgroundColor #FEFEFE

together {
  actor "客户用户" as User
  actor "研发运维" as Dev
}

package "客户侧（端侧 Agent）" {
  [主 Agent\n(任务理解 & 编排)] as MainAgent
  [OpenCLI Skill\n(Node Sandbox)] as CLISkill
  [BrowserAgent Skill\n(waiy-browser-use)] as BASkill
  [本地工具 Skill\n(Excel/OCR/文档)] as LocalSkill
  [Skill 路由器] as Router

  MainAgent --> Router
  Router --> CLISkill : 优先
  Router --> BASkill : 降级
  Router --> LocalSkill : 本地处理
}

package "云侧（CLI 生命周期管理）" {
  [CLI 工厂\n(生成 & 修复)] as Factory
  [巡检服务\n(定期验证)] as Patrol
  [Skill 仓库\n(版本管理)] as SkillRepo
  [Chrome 实例\n(debug 账号)] as Chrome

  Factory --> Chrome : CDP
  Patrol --> Chrome : CDP
  Factory --> SkillRepo : 发布
  Patrol --> Factory : 失效触发修复
}

cloud "客户 Web 系统" {
  [OA] as OA
  [ERP] as ERP
  [钉钉/企微] as IM
  [CRM] as CRM
  [其他系统] as Other
}

SkillRepo --> CLISkill : 推送更新
Chrome --> OA
Chrome --> ERP
Chrome --> IM
Chrome --> CRM
Chrome --> Other

User --> MainAgent : 自然语言指令
Dev --> Factory : 手动触发/监控
BASkill --> OA : CDP (兜底)

@enduml
```

### 架构分层

| 层级 | 组件 | 职责 |
|------|------|------|
| **客户侧** | 主 Agent | 接收用户自然语言指令，理解意图，拆解任务，调度 Skill |
| | Skill 路由器 | 根据任务类型和可用 Skill 决定执行路径（OpenCLI 优先 → BrowserAgent 降级） |
| | OpenCLI Skill | 在 Node Sandbox 中执行 CLI 命令，快速、稳定、可复现 |
| | BrowserAgent Skill | waiy-browser-use 驱动浏览器执行复杂交互，作为 OpenCLI 的兜底 |
| | 本地工具 Skill | Excel/OCR/文档处理等不需要浏览器的本地任务 |
| **云侧** | CLI 工厂 | 连接客户 Web 系统，自动发现 API、生成 CLI、测试验证 |
| | 巡检服务 | 定期执行已有 CLI，检测失效，触发修复 |
| | Skill 仓库 | CLI 适配器版本管理、分发、推送 |
| | Chrome 实例 | 承载 debug 账号登录态，为 CLI 工厂和巡检服务提供浏览器环境 |

---

## 3 主 Agent 设计

### 3.1 主 Agent 职责

主 Agent 是用户的唯一交互入口，负责：

1. **意图理解**：将自然语言指令转化为结构化任务
2. **任务拆解**：复杂任务分解为多个子任务（可串行/并行）
3. **Skill 调度**：选择最优 Skill 执行每个子任务
4. **结果整合**：汇总子任务结果，返回给用户
5. **异常处理**：Skill 失败时自动降级或请求人工介入

### 3.2 主 Agent 架构

```plantuml
@startuml
skinparam componentStyle rectangle

package "主 Agent" {
  [LLM\n(Claude/GPT)] as LLM
  [意图解析器] as Intent
  [任务编排器\n(DAG)] as Orchestrator
  [Skill 路由器] as Router
  [上下文管理器] as Context
  [结果聚合器] as Aggregator

  LLM --> Intent
  Intent --> Orchestrator
  Orchestrator --> Router
  Router --> Context : 读取 Skill 能力清单
  Orchestrator --> Aggregator
}

package "Skill 注册表" {
  [OpenCLI Skill 清单\n(site + command)] as CLIRegistry
  [BrowserAgent 能力描述] as BARegistry
  [本地工具能力描述] as LocalRegistry
}

Context --> CLIRegistry
Context --> BARegistry
Context --> LocalRegistry

@enduml
```

### 3.3 Skill 路由决策

```plantuml
@startuml
skinparam activityShape roundedBox

start
:主 Agent 接收用户指令;
:LLM 解析意图 → 结构化任务;

if (纯本地处理?\n(Excel/OCR/文档)) then (是)
  :本地工具 Skill;
else (否)
  :查询 Skill 注册表;
  if (目标系统 + 操作\n有匹配的 OpenCLI 命令?) then (是)
    :OpenCLI Skill 执行;
    note right: ~1-7s，结构化输出
    if (执行成功?) then (是)
      :返回结构化结果;
      stop
    else (否)
      :记录失败原因;
      :上报巡检服务;
    endif
  else (否)
  endif

  :BrowserAgent Skill 执行;
  note right: ~15-60s，LLM 驱动
  if (执行成功?) then (是)
    :返回结果;
    :记录执行轨迹\n→ 供 CLI 工厂参考;
  else (否)
    :call-for-user\n(请求人工介入);
  endif
endif

stop

@enduml
```

### 3.4 主 Agent 与 Skill 的调用接口

```python
# 主 Agent 调用 OpenCLI Skill
class OpenCLISkill:
    async def execute(self, site: str, command: str, args: dict) -> SkillResult:
        """在 Node Sandbox 中执行 OpenCLI 命令"""
        # POST localhost:19825/command
        # 或直接 executeCommand(cmd, args)

    async def list_capabilities(self, site: str) -> list[CLICommand]:
        """查询某站点可用的 CLI 命令列表"""
        # 读取 cli-manifest.json

# 主 Agent 调用 BrowserAgent Skill
class BrowserAgentSkill:
    async def execute(self, task: str, url: str) -> SkillResult:
        """启动 waiy-browser-use Agent 执行浏览器任务"""
        agent = Agent(
            task=task,
            llm=self.llm,
            browser=self.browser_session,
        )
        history = await agent.run(max_steps=30)
        return SkillResult(
            success=history.is_done(),
            data=history.final_result(),
            trace=history,  # 执行轨迹，供 CLI 工厂分析
        )
```

---

## 4 CLI 全生命周期管理

这是区别于竞品的核心壁垒：**CLI 不是一次性生成的，而是持续维护的活资产**。

### 4.1 生命周期概览

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

state "生成" as Gen {
  state "API 发现\n(explore)" as Explore
  state "认证探测\n(cascade)" as Cascade
  state "适配器生成\n(synthesize)" as Synth
  state "注册验证\n(register + verify)" as Verify

  Explore --> Cascade
  Cascade --> Synth
  Synth --> Verify
}

state "运行" as Run {
  state "Skill 仓库\n(版本管理)" as Repo
  state "客户端执行\n(Node Sandbox)" as Exec
  state "执行反馈\n(成功/失败)" as Feedback

  Repo --> Exec : 推送
  Exec --> Feedback
}

state "巡检" as Patrol {
  state "定时触发\n(cron)" as Cron
  state "执行验证\n(dry-run)" as DryRun
  state "健康报告" as Report

  Cron --> DryRun
  DryRun --> Report
}

state "修复" as Fix {
  state "诊断失效原因" as Diagnose
  state "重新 explore" as ReExplore
  state "差异对比\n& 更新适配器" as Diff
  state "回归测试" as Regression

  Diagnose --> ReExplore
  ReExplore --> Diff
  Diff --> Regression
}

[*] --> Gen : debug 账号接入
Gen --> Run : 发布到 Skill 仓库
Run --> Patrol : 持续监控
Patrol --> Fix : 发现异常
Fix --> Run : 修复后重新发布
Run --> [*] : 客户下线

@enduml
```

### 4.2 阶段一：CLI 生成（CLI 工厂）

利用客户提供的 debug 账号，在云侧 Chrome 实例中完成。

```plantuml
@startuml
skinparam sequenceMessageAlign center

actor "研发/AI Agent" as Dev
participant "CLI 工厂" as Factory
participant "Chrome\n(debug 账号)" as Chrome
participant "客户 Web 系统" as WebSys
database "Skill 仓库" as Repo

Dev -> Factory: 触发生成\n(url, site_name, goals)

== Step 1: Explore — API 发现 ==
Factory -> Chrome: page.goto(url)
Chrome -> WebSys: 加载页面
Factory -> Chrome: 开启 Network 监控 (CDP)
Chrome -> WebSys: 用户操作 / auto-click
Chrome --> Factory: 收集 Fetch/XHR 请求\n(JSON 响应)
Factory -> Factory: 端点评分 & 能力推理\n(hot/search/feed...)

== Step 2: Cascade — 认证探测 ==
Factory -> Chrome: 逐级尝试\nPUBLIC → COOKIE → HEADER
Chrome -> WebSys: fetch(url) / fetch+cookie / fetch+CSRF
Chrome --> Factory: 确定最低认证策略

== Step 3: Synthesize — 适配器生成 ==
Factory -> Factory: 生成 YAML/TS 适配器\n(pipeline: fetch→map→filter)

== Step 4: Verify — 验证 ==
Factory -> Chrome: executeCommand(cli)
Chrome -> WebSys: 执行 CLI 对应的请求
Chrome --> Factory: 返回结果
Factory -> Factory: 对比页面数据\n验证准确性

== 发布 ==
Factory -> Repo: 提交适配器\n(版本号 + 测试快照)
Repo --> Dev: 通知：新增 N 个 CLI

@enduml
```

**关键实现细节**：

| 步骤 | OpenCLI 现有能力 | 需要扩展 |
|------|-----------------|---------|
| Explore | `opencli explore <url> --site <name>` 自动发现 API | 支持批量 URL 输入；自动登录（debug 账号） |
| Cascade | `opencli cascade <api-url>` 5 级认证探测 | 无需改动 |
| Synthesize | `opencli synthesize <site>` 生成适配器 | 增加 waiy-browser-use 执行轨迹作为输入参考 |
| Verify | `opencli verify` 验证适配器 | 增加结果快照对比（golden test） |

### 4.3 阶段二：CLI 巡检

```plantuml
@startuml
skinparam activityShape roundedBox

start
:定时触发（每日/每周）;

:读取 Skill 仓库中所有 CLI;

while (遍历每个 CLI) is (还有)
  :在 Chrome 实例中执行 CLI\n(dry-run 模式);

  if (HTTP 状态正常\n且返回数据非空?) then (是)
    if (返回数据结构\n与 golden snapshot 一致?) then (是)
      :标记 ✅ 健康;
    else (否)
      :标记 ⚠️ 结构变化\n(可能需要更新 columns/map);
    endif
  else (否)
    if (认证失败 401/403?) then (是)
      :标记 🔑 登录态过期\n→ 重新登录 debug 账号;
    else (否)
      :标记 ❌ 接口失效\n→ 触发修复流程;
    endif
  endif
endwhile (完毕)

:生成巡检报告;
:推送通知（钉钉/邮件）;

stop

@enduml
```

**巡检策略**：

| 巡检项 | 检测方式 | 频率 | 失败动作 |
|--------|---------|------|---------|
| 接口可达性 | HTTP status code | 每日 | 触发修复 |
| 数据结构一致性 | JSON schema diff vs golden snapshot | 每周 | 告警 + 自动适配 |
| 认证有效性 | 401/403 检测 | 每日 | 自动重新登录 |
| 响应时间 | latency > 10s | 每日 | 告警 |
| 数据正确性 | 关键字段非空 + 采样比对 | 每周 | 告警 |

### 4.4 阶段三：CLI 修复与更新推送

```plantuml
@startuml
skinparam sequenceMessageAlign center

participant "巡检服务" as Patrol
participant "CLI 工厂" as Factory
participant "Chrome" as Chrome
participant "客户系统" as Web
participant "waiy-browser-use\nAgent" as BUA
database "Skill 仓库" as Repo
participant "客户端" as Client

== 触发修复 ==
Patrol -> Factory: CLI 失效通知\n(site, command, error)

== 诊断 ==
Factory -> Factory: 分析失效原因\n(URL 变化? 参数变化? 认证变化?)

alt URL/参数变化
  Factory -> Chrome: 重新 explore\n(同一页面)
  Chrome -> Web: 抓取新 API
  Chrome --> Factory: 新端点列表
  Factory -> Factory: diff 新旧端点\n→ 更新适配器
else 页面重构（无 API）
  Factory -> BUA: 启动 browser-use\n任务: "完成原 CLI 同等操作"
  BUA -> Chrome: DOM 操作
  Chrome -> Web: 页面交互
  BUA --> Factory: 执行轨迹 + 结果
  Factory -> Factory: 从轨迹生成新适配器\n(strategy: UI)
end

== 验证 ==
Factory -> Chrome: 执行新 CLI
Chrome -> Web: 请求
Chrome --> Factory: 返回结果
Factory -> Factory: 对比 golden snapshot

== 发布 ==
Factory -> Repo: 提交更新\n(版本号递增)

== 推送 ==
Repo -> Client: 增量推送\n(仅变更的适配器)
Client -> Client: 热加载新 CLI\n(无需重启)

@enduml
```

---

## 5 关键模块详细设计

### 5.1 Skill 注册表

Skill 注册表是主 Agent 进行路由决策的依据。结构如下：

```yaml
# skill-registry.yaml — 每个客户一份
sites:
  - site: dingtalk
    domain: oa.dingtalk.com
    auth_strategy: COOKIE
    commands:
      - name: attendance
        description: 导出考勤数据
        args: [date_from, date_to, department]
        columns: [name, date, clock_in, clock_out, status]
        health: ok           # 最近巡检状态
        last_check: 2026-06-13
        version: 1.2.0
      - name: send_message
        description: 发送群消息
        args: [group_id, content]
        health: ok
        last_check: 2026-06-13
        version: 1.0.0

  - site: kingdee
    domain: erp.customer.com
    auth_strategy: HEADER
    commands:
      - name: voucher_list
        description: 查询凭证列表
        args: [period, type]
        health: degraded     # 结构有变化但仍可用
        last_check: 2026-06-13
        version: 2.1.0
```

主 Agent 的 LLM 收到用户指令时，Skill 注册表以摘要形式注入 system prompt：

```
你可以使用以下 OpenCLI 命令：
- dingtalk attendance: 导出考勤数据 (参数: date_from, date_to, department)
- dingtalk send_message: 发送群消息 (参数: group_id, content)
- kingdee voucher_list: 查询凭证列表 (参数: period, type) [⚠️ 结构有变化]
如果以上命令不能满足需求，你可以使用 BrowserAgent 直接操作网页。
```

### 5.2 Chrome 实例管理（debug 账号）

```plantuml
@startuml
skinparam componentStyle rectangle

package "Chrome 实例池" {
  [Chrome #1\n钉钉 debug 账号] as C1
  [Chrome #2\nERP debug 账号] as C2
  [Chrome #3\n税务系统 debug 账号] as C3
}

[Session Manager] as SM
[Cookie Store\n(加密存储)] as CS

SM --> C1 : CDP ws://
SM --> C2 : CDP ws://
SM --> C3 : CDP ws://

SM --> CS : 持久化登录态
CS --> SM : 恢复登录态

note right of SM
  职责：
  1. 启动/销毁 Chrome 实例
  2. 登录态保活（定期刷新 Cookie）
  3. 登录态过期时自动重登
  4. 并发隔离（CLI 工厂/巡检/BrowserAgent 共享但不冲突）
end note

@enduml
```

**登录态管理策略**：

| 策略 | 实现 | 适用场景 |
|------|------|---------|
| Cookie 持久化 | 导出/导入 Chrome Cookie 到加密存储 | 大多数 Web 系统 |
| 定时刷新 | 每 4 小时访问一次目标页面，防止 Session 过期 | Session 有效期短的系统 |
| 自动重登 | 检测到 401/403 时，用 debug 账号密码自动登录 | 所有系统 |
| 人工介入 | 短信验证码/U 盾等安全验证，通过通知转交研发处理 | 银行/税务等高安全系统 |

### 5.3 Skill 仓库与推送机制

```plantuml
@startuml
skinparam componentStyle rectangle

package "Skill 仓库（云侧）" {
  [Git 仓库\n(适配器源码)] as Git
  [版本管理\n(语义化版本)] as Ver
  [构建服务\n(build-manifest)] as Build
  [分发服务] as Dist
}

package "客户端" {
  [Skill 更新检查器\n(定时轮询/WebSocket)] as Checker
  [本地 Skill 缓存\n(~/.opencli/clis/)] as Local
  [Node Sandbox] as Sandbox
}

Git --> Build : push 触发
Build --> Ver : 打版本标签
Ver --> Dist : 生成增量包

Dist --> Checker : 推送通知\n(版本号 + changelog)
Checker --> Dist : 拉取增量包
Checker --> Local : 热更新适配器
Local --> Sandbox : 加载执行

note bottom of Dist
  推送策略：
  - 灰度发布（先推送到测试环境）
  - 增量推送（仅变更文件）
  - 回滚能力（保留最近 3 个版本）
end note

@enduml
```

---

## 6 安全设计

| 安全领域 | 措施 |
|---------|------|
| **debug 账号保护** | 账号密码加密存储（AES-256）；仅云侧 CLI 工厂和巡检服务可访问；审计日志记录所有操作 |
| **客户数据隔离** | 每个客户独立 Chrome 实例 + 独立 Cookie Store；CLI 执行结果不跨客户共享 |
| **CLI 执行沙箱** | OpenCLI 在 Node Sandbox 中运行（受限文件系统 + 网络白名单）；执行超时 30s 强制终止 |
| **传输安全** | Skill 推送走 HTTPS + 签名校验；Chrome CDP 连接走 WSS |
| **权限最小化** | debug 账号仅赋予只读 + 必要写入权限；CLI 适配器声明 `access: read/write` |
| **操作审计** | 所有 CLI 执行记录上报（who/when/what/result）；异常操作实时告警 |

---

## 7 客户接入流程

```plantuml
@startuml
skinparam activityShape roundedBox

|客户|
start
:提供 debug 账号\n(账号/密码 + 系统 URL 清单);

|研发|
:创建客户 Chrome 实例;
:使用 debug 账号登录各系统;

|CLI 工厂|
:批量 explore 各系统 URL;
:生成 CLI 适配器;
:执行验证 & 生成 golden snapshot;

|研发|
:人工 review CLI 质量;
:确认后发布到 Skill 仓库;

|客户侧|
:部署主 Agent + Skill 运行环境;
:拉取 CLI 到本地缓存;
:用户开始使用;

|巡检服务|
:启动定期巡检;
:异常自动修复 & 推送;

stop

@enduml
```

**典型接入周期**：

| 阶段 | 耗时 | 产出 |
|------|------|------|
| 账号对接 & 环境准备 | 1 天 | Chrome 实例 + 登录态 |
| CLI 批量生成（AI 自动） | 1-2 天 | 每个系统 10-30 个 CLI |
| 人工 review & 调优 | 1-2 天 | 验证通过的 CLI 集合 |
| 客户端部署 & 测试 | 1 天 | 主 Agent 上线 |
| **总计** | **4-6 天** | 完整的数字员工能力 |

---

## 8 与现有代码的对应关系

| 架构组件 | 现有实现 | 需要新增/改造 |
|---------|---------|-------------|
| **主 Agent** | 无（需新建） | 新建主 Agent 服务，集成 LLM + Skill 路由 + 任务编排 |
| **OpenCLI Skill 执行** | `src/execution.ts` — `executeCommand()` | 封装为 Python 可调用的 Skill 接口（通过 HTTP daemon 19825 端口） |
| **BrowserAgent Skill 执行** | `browser_use/agent/service.py` — `Agent.run()` | 封装为 Skill 接口，接受 (task, url) 返回 SkillResult |
| **CLI 生成** | `opencli explore / cascade / synthesize` | 增加批量模式 + debug 账号自动登录 |
| **CLI 巡检** | `opencli verify`（基础验证） | 扩展为定时巡检服务 + golden snapshot 对比 |
| **CLI 推送** | 无 | 新建增量分发服务 |
| **Chrome 实例管理** | bux `browser_keeper.py`（Browser Use Cloud） | 改造为多账号、多实例管理 |
| **Skill 注册表** | `cli-manifest.json`（静态清单） | 扩展为动态注册表（含健康状态、版本） |
| **登录态管理** | OpenCLI COOKIE/HEADER 策略 | 增加自动重登 + Cookie 持久化 |
| **Node Sandbox** | OpenCLI 本地执行 | 增加沙箱隔离（文件系统 + 网络限制） |

---

## 9 技术选型建议

| 组件 | 建议方案 | 理由 |
|------|---------|------|
| 主 Agent 框架 | Python async + LLM（Claude） | 与 waiy-browser-use 同技术栈，便于集成 |
| 任务编排 | 内置 DAG 引擎（轻量级） | 初期任务链简单，不需要 Temporal/Airflow 的复杂度 |
| Skill 注册表存储 | SQLite + JSON 文件 | 单客户数据量小，无需重型数据库 |
| Skill 仓库 | Git 仓库 + 语义化版本 | 天然支持版本管理、diff、回滚 |
| 加密存储 | Python keyring + AES-256 | 账号密码和 Cookie 的安全存储 |
| 巡检调度 | APScheduler / cron | 轻量定时任务 |
| 客户端推送 | HTTP 长轮询 / WebSocket | 低延迟推送 CLI 更新 |

---

## 10 MVP 范围（建议）

第一版聚焦 **"能跑通一个客户的钉钉考勤导出"** 这一条端到端链路：

| 模块 | MVP 范围 |
|------|---------|
| 主 Agent | 支持单轮对话 + 单 Skill 调用（不需要 DAG 编排） |
| OpenCLI Skill | 通过 HTTP daemon 调用已有 CLI |
| BrowserAgent Skill | 直接调用 `Agent.run()` 作为降级路径 |
| CLI 生成 | 手动触发 `opencli explore` + `synthesize` |
| 巡检 | cron + shell 脚本，每日执行一次 |
| 推送 | 手动 scp / rsync 到客户端 |
| Chrome 管理 | 单实例，手动维护登录态 |

**MVP → 完整版的迭代路径**：手动 → 自动、单客户 → 多客户、单系统 → 多系统。
