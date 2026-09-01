# OpenClaw 8.1 升级后网关启动失败：不用重装，两条命令救回来

*[English version](./README.en.md)*

> 一次真实的救援记录 —— 存量多 Agent 环境从 `2026.7.x` 升级到 `2026.8.1` 后网关拒绝启动。
> 环境：Windows 10/11 · Node.js 24.x · OpenClaw `2026.7.1-2` → `2026.8.1`
> 结果：**零重装、零 Agent 重配、零数据丢失**

---

## TL;DR

升级到 8.1 后网关起不来，**大概率不是你的配置写错了**，而是 8.1 要求你先跑一次状态迁移。

```bash
openclaw doctor --fix     # 迁移遗留状态（这才是关键）
openclaw gateway run      # 再启动
```

→ **完整修复流程（含备份与验证）：[第四节](#repair)** · 一键清单：[附录](#checklist)

如果你正打算「删掉重装、重新配置所有 Agent」——**先停手，读完这篇**。

另外有一个**不报错的隐蔽后果**：8.1 改了 Agent 工作目录的解析规则，部分 Agent 会被静默换到一个空目录，看上去像「记忆全丢了」（实际没丢）。修好启动后请务必看一眼：[坑 6](#pitfall-6)。

---

## 一、症状

之前一直跑得很稳的配置，升级到 8.1 后网关直接启动失败。控制台看不出所以然，仪表盘打不开，所有 Bot 全部离线。`openclaw.json` 语法完全正确，配置一个字没改。

这时很容易走上一条弯路 —— 冒出一个听起来很合理的假设：

> 「是不是 Agent 配得太多、配置太复杂，所以起不来了？」

然后开始反复改 `openclaw.json`、删 Agent、删通道 —— 越改越乱，最后往「删掉重装、重新配一遍」的方向跑。

**这个假设是错的。** Agent 数量和这次故障**毫无关系**，下面会看到真因具体到某一个文件。

为什么特别提这一条：「配置太复杂所以坏了」是启动故障里最具诱惑力的错误假设 —— 它无法被直接反驳，且它给出的“解法”（推翻重来）恰好能掩盖真因。代价是你白白弄丢全部配置和历史会话。

---

## 二、转折点：报错原文一直躺在硬盘上

排查的第一步不该是改配置，而是**找到真正的报错**。OpenClaw 启动失败时会写一份结构化崩溃日志：

```
~/.openclaw/logs/stability/openclaw-stability-<时间戳>-<pid>-gateway.startup_failed.json
```

Windows PowerShell 下按时间倒序找最新几份：

```powershell
Get-ChildItem "$env:USERPROFILE\.openclaw\logs\stability" -Recurse |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 5 LastWriteTime, FullName
```

打开它，答案清清楚楚：

```json
{
  "reason": "gateway.startup_failed",
  "error": {
    "message": "Legacy session store requires migration: C:\\Users\\<你>\\.openclaw\\agents\\main\\sessions\\sessions.json. Run \"openclaw doctor --fix\" against the same state/config before starting OpenClaw."
  }
}
```

**程序把解决方案直接写在报错里了。**

> **经验教训一**：`openclaw.json` 语法没错却起不来时，不要凭猜测改配置。
> 先读 `logs/stability/*.json`，再跑 `openclaw doctor`。改配置是最后一步，不是第一步。

---

## 三、根因：8.1 把状态从 JSON 迁到了 SQLite

8.1 是一次**状态存储层的架构升级**。会话记录、provider catalog（模型目录）、workspace 状态等，从散落各处的 JSON 文件统一迁进了 SQLite：

```
~/.openclaw/state/openclaw.sqlite
```

为了防止数据损坏，8.1 的策略是**发现遗留格式就拒绝启动**，并要求你显式执行迁移。这是保护机制，不是 Bug。

所以：

- 症状看起来像「配置坏了」
- 实际上是「状态没迁移」
- 于是**改配置怎么改都好不了**

### 为什么这是存量用户的必然遭遇

这一点值得说清楚，因为它决定了「你会不会中招」：

- **全新安装**的人不会遇到 —— 一上来就是 SQLite 格式，没有遗留物。
- **存量用户**只要磁盘上还留着旧格式的 `sessions.json` / `catalog.json`，就**一定**撞上这道门。

也就是说，和「配得多不多、Agent 几个」无关，只和「你装得早不早」有关。装得越早、用得越久，遗留文件越多。

另外，从故障日志看，这台机器在 8.1 之前的一次大版本升级中**也出现过 `gateway.startup_failed`**。这说明「大版本升级卡在启动」是 OpenClaw 的**迁移设计模式**，不是 8.1 的一次性事故 —— 以后遇到同类症状，第一反应仍然应该是 `doctor --fix`。

### 关键细节：这里有两道门，是串联的

修好第一个错之后，网关能走到 `ready`，然后**崩在第二个错**：

```
PreparedModelRuntimeOwnerNotPublishedError:
  prepared model catalog owner was not published for the requested config
  (C:\Users\<你>\.openclaw\agents\<agent>\agent)
```

这是 provider catalog（模型目录）也没迁移，位置在：

```
~/.openclaw/agents/<agent>/agent/plugins/<provider>/catalog.json
```

配了几个模型 provider 就有几份。**如果只看到「起不来」就认定是配置问题，很容易在第一道门前反复折腾。**

好消息是：同一条 `doctor --fix` 把两道门一起解决。

---

<a id="repair"></a>

## 四、完整修复流程

### 第 0 步：备份（这一步不能跳）

**动手之前先备份。** 迁移会改写状态，出问题时你需要退路。

```powershell
$ts = Get-Date -Format "yyyyMMdd-HHmmss"
$bk = "D:\openclaw-backup-$ts"
New-Item -ItemType Directory -Force -Path $bk | Out-Null

# 配置文件
Copy-Item "$env:USERPROFILE\.openclaw\openclaw.json" "$bk\openclaw.json" -Force

# 全部 Agent 状态（含会话历史，体积可能较大）
robocopy "$env:USERPROFILE\.openclaw\agents" "$bk\agents" /E /NFL /NDL /NJH /NJS /R:1 /W:1

# 看一下备份体积
"{0:N1} MB" -f ((Get-ChildItem $bk -Recurse -File | Measure-Object -Sum Length).Sum / 1MB)
```

会话历史在 `agents/` 里，所以**不能只备份 `openclaw.json`**。实测这个目录可以到几百 MB 量级。

> ⚠️ **`~/.openclaw` 内含明文凭据，备份绝对不要推到公开仓库。** 详见第六节。

### 第 1 步：先诊断，看清要改什么

```bash
openclaw doctor
```

不带 `--fix` 的 `doctor` 是**只读预览**，会列出所有待迁移项。先看一遍再决定。

### 第 2 步：执行迁移

```bash
openclaw doctor --fix
```

实测这一步会做的事情（单次记录，你的环境不一定完全相同）：

| 动作 | 说明 |
| --- | --- |
| 迁移遗留会话存储 | **拦住启动的第一道门** |
| 迁移全部 provider catalog 进 SQLite | **第二道门** |
| 规范化重复会话键 | 保留历史 |
| 清理失效的模型路由固定 | 防止会话被钉在旧 provider 上 |
| 迁移 workspace 状态与配置审计日志 | — |

### 第 3 步：启动验证

```bash
openclaw gateway run
```

看到 `[gateway] ready` 且**不再崩溃**才算成功：

```
[gateway] ready
[telegram] [<账号>] starting provider (@YourBot)
[discord] Discord bot probe resolved @YourBot
```

### 第 4 步：确认业务真的活了

`ready` 只代表进程起来了，要验证端到端：

```powershell
# 仪表盘（注意：如果配了代理，本地请求要绕过）
$env:HTTP_PROXY=""; $env:HTTPS_PROXY=""
(Invoke-WebRequest "http://127.0.0.1:18789/" -UseBasicParsing).StatusCode   # 期望 200

openclaw status         # Agents / 各通道账号是否 OK
openclaw models list    # 模型是否 resolvable
```

日志里出现真实模型调用返回 200，才是真的通了：

```
[model-fetch] response provider=<provider> model=<model> status=200
```

### 第 5 步：确认遗留物真的清空了（可选但推荐）

迁移完之后可以直接数一遍，比看日志更硬：

```powershell
# 期望：两个都是 0
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "sessions.json" -EA SilentlyContinue).Count
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "catalog.json"  -EA SilentlyContinue).Count

# SQLite 状态库应该存在且有正常体积
Get-ChildItem "$env:USERPROFILE\.openclaw\state" -File | Select-Object Name, Length
```

两个计数归零 + `openclaw.sqlite` 有几 MB 体积 = 迁移确实完成了，不是「暂时躲过去了」。

---

## 五、配套踩坑

### 坑 1：插件版本漂移（必修）

主程序升到 8.1，但插件还留在 7.x：

```
Plugin version drift: N active official plugins not on gateway 2026.8.1
```

这也是那一堆 `requires capability consent` 警告的根源。逐个更新：

```bash
openclaw plugins update <插件名>
```

改完之后 `openclaw plugins list` 里所有插件的版本列都应该是 `2026.8.1`，没有残留旧版本号。

**国内网络需要代理。** OpenClaw 走 npm registry 拉插件：

```powershell
$env:HTTP_PROXY  = "http://127.0.0.1:7890"
$env:HTTPS_PROXY = "http://127.0.0.1:7890"
$env:NO_PROXY    = "localhost,127.0.0.1,.local"    # 本地网关必须绕过代理
openclaw plugins update <插件名>
```

动手前先确认代理真的通：

```powershell
(Invoke-WebRequest "https://registry.npmjs.org/-/ping" -Proxy "http://127.0.0.1:7890" -UseBasicParsing).StatusCode
```

`NO_PROXY` 那一行别漏 —— 否则后面验证仪表盘时，本地 `127.0.0.1:18789` 的请求会被塞进代理，白白报错半天。

### 坑 2：`GatewayLockError` —— 网关运行时抢占状态锁

更新插件时可能看到：

```
Skipped plugin doctor state migrations because exclusive state ownership is unavailable:
GatewayLockError: failed to acquire gateway state ownership
```

意思是**网关正在运行、独占着状态锁**，插件的状态迁移被跳过了。插件本身更新成功，但迁移没做完。

> **正确姿势：先停网关 → 更新插件 → 再启动。**

### 坑 3：某些插件要显式同意权限

```bash
openclaw plugins enable <插件名> --accept-capabilities
openclaw plugins update <插件名> --accept-capabilities
```

签字前先看它到底要什么权限：

```bash
openclaw plugins inspect <插件名>
```

有些插件的权限列表其实是空的（`Capability mode: plain`），纯属形式手续。**但请养成先 `inspect` 再 `--accept-capabilities` 的习惯，别盲签。**

### 坑 4：删 Agent 目录会让网关拒绝启动

配置里已经删掉的 Agent，磁盘目录可能还在。直接删目录后，网关**拒绝启动**：

```
OpenClaw startup migrations did not complete cleanly; refusing to report the gateway ready.
- Skipped missing registered agent database ...\<agent>\agent\openclaw-agent.sqlite
```

因为 SQLite 注册表里还有这条记录。删完目录**必须**补一条：

```bash
openclaw doctor --fix     # 清理失效注册项
openclaw gateway run
```

> 顺序记牢：**确认已备份 → 删目录 → `doctor --fix` → 启动**。

### 坑 5（Windows 专属）：`gateway restart` 直接报错

修复过程中如果你想热重启，在 Windows 上会撞到这个：

```
Gateway restart failed: TypeError [ERR_UNKNOWN_SIGNAL]: Unknown signal: SIGUSR1
```

`openclaw gateway restart` 的热重载走 POSIX 信号（`SIGUSR1`），**Windows 没有这个信号**，所以这条命令在 Windows 上不可用。

替代方案：

```powershell
# 手动停掉网关进程，然后重新启动
openclaw gateway run
```

好消息是：**大部分配置改动不需要重启**。改完 `openclaw.json` 后直接用 `openclaw config get <路径>` 回读，如果新值已经能读出来，说明网关是实时读配置的，不用重启。

<a id="pitfall-6"></a>

### 坑 6：Agent 工作目录被静默重定向（最隐蔽的一个）

**这个坑不报错、不崩溃，`doctor` 也不提醒**，但它会让你以为数据全丢了。

症状：网关修好、一切正常，但某个 Agent 打开一看 —— 工作目录里空空如也，只有一套全新的 `BOOTSTRAP.md` / `AGENTS.md` / `SOUL.md` / `USER.md`。你辛苦积累的记忆文件、项目目录、历史笔记，全都不见了。

**根因**：8.1 改变了 `agents.defaults.workspace` 的语义。

| 版本 | `agents.defaults.workspace` 的含义 |
| --- | --- |
| 8.1 之前 | 没有显式 `workspace` 的 Agent **直接使用**这个目录 |
| 8.1 之后 | 这个目录被当作**父目录**，Agent 落到 `<父目录>\<agentId>` |

于是如果你的配置是：

```json
{
  "agents": {
    "defaults": { "workspace": "C:\\Users\\<你>\\.openclaw\\workspace" },
    "entries": {
      "main": { "name": "Main_Manager" }
    }
  }
}
```

升级后 `main` 就不再看 `...\workspace\`，而是被扔进新建的 `...\workspace\main\` —— 一个空目录，OpenClaw 还会贴心地给它铺一套全新初始化文件，看上去就像「记忆被重置了」。

**关键点**：写了显式 `workspace` 的 Agent **完全不受影响**。所以典型现场是「6 个 Agent 好的，就 1 个瞎了」，非常容易误判成那一个 Agent 配坏了。

**诊断**：

```powershell
# 老目录还在不在？内容还全不全？
Get-ChildItem "$env:USERPROFILE\.openclaw\workspace" -Force | Select-Object Name, LastWriteTime

# 新建的空目录是不是刚刚才出现（时间戳 ≈ 你升级那一刻）？
Get-ChildItem "$env:USERPROFILE\.openclaw\workspace\<agentId>" -Force
```

如果老目录内容完好、新目录时间戳是升级时刻 —— **数据一个都没丢，只是 Agent 站错了房间。**

**修法**：给受影响的 Agent 补一条显式 `workspace`，指回原目录。

```bash
openclaw config set agents.entries.main.workspace "C:\Users\<你>\.openclaw\workspace"
openclaw config get agents.entries.main.workspace    # 回读确认
```

然后重启网关（Windows 见坑 5），进去确认 Agent 能读到原来的 `MEMORY.md` 等文件。

> **经验**：升级后不要只验证「网关是否 ready」，还要抽查**每个 Agent 的工作目录解析结果**。路径语义的变更不会报错，只会安静地把你换到另一个房间。

---

## 六、安全项：明文凭据（请一定看完）

`doctor` 会报这么一条，值得单独说：

```
WARNING: openclaw.json contains plaintext secret-bearing config fields.
  Paths: gateway.auth.token, channels.<channel>.accounts.*.botToken, ...
```

**问题在哪**：Bot Token、API Key、网关 Token 是**明文**存在 `openclaw.json` 里的。任何能读到这个文件的东西都能拿走它们 —— 包括：

- 你自己的 Agent（它们通常有文件读取工具）
- 你随手打包的备份（第 0 步刚做的那个）
- 你贴到聊天窗口求助的配置片段

对多 Agent 玩家风险更高：拿到 Bot Token 等于能完全冒充你的 Bot 收发消息。

**怎么办**：8.1 支持把它们迁成 SecretRef（配置里只留引用，真值另存）：

```bash
openclaw secrets configure
openclaw secrets audit --check
```

三条最低要求，现在就该做：

1. `~/.openclaw` **永远不要**进 Git（或任何公开仓库）
2. 分享配置求助时，Token / Key **必须打码**
3. 一旦怀疑泄露，立刻去对应服务商后台（BotFather 等）**重置**

---

## 七、最终结果

| 项目 | 状态 |
| --- | --- |
| 网关 | `ready`，稳定运行 |
| 仪表盘 | HTTP 200 |
| 各通道账号 | 全部 OK |
| Agent 与会话历史 | 全部保留，无丢失 |
| 模型 | 全部 resolvable |
| 插件漂移 | 已清零（全部 `2026.8.1`）|
| 遗留 `sessions.json` / `catalog.json` | 均为 0 |
| 实际模型调用 | 返回 200 |

**重装次数：0。重配 Agent 数：0。丢失数据：0。**

---

## 八、给同样踩坑的人：五条经验

1. **先读日志，再改配置。** `~/.openclaw/logs/stability/*.json` 里的 `error.message` 通常**直接写着解决方案**。配置语法没错却起不来，八成不是配置问题。

2. **大版本升级 = 先跑迁移。** 遇到启动失败，第一反应应该是 `openclaw doctor --fix`，而不是重装。它是官方迁移工具，不是玄学修复。

3. **备份是动手的前提，不是可选项。** 而且要备 `agents/` 整个目录（会话历史在里面），不只是 `openclaw.json`。

4. **警惕「配置太复杂所以坏了」这类结论。** 它听起来合理、难以反驳，但常常是没找到真因时的托词，而且它的“解法”代价极大。真因一般具体到某个文件、某行报错。**任何拿不出报错原文支撑的诊断——包括你自己的直觉——就还只是猜测。**

5. **`ready` 不等于可用。** 一定要验证到端到端：仪表盘 200、`status` 全绿、日志里有真实模型调用返回 200、遗留文件计数归零。

6. **不报错的变更最危险。** 迁移类故障会大声崩给你看，但**路径/语义类变更是静默的**（坑 6 就是）。升级后除了看网关状态，还要抽查每个 Agent 实际落在哪个工作目录。

---

<a id="checklist"></a>

## 附：一键排查清单

```powershell
# 1. 看真实报错（最重要的一步）
Get-ChildItem "$env:USERPROFILE\.openclaw\logs\stability" -Recurse |
  Sort-Object LastWriteTime -Descending | Select-Object -First 3 FullName

# 2. 备份（含会话历史）
$bk = "D:\openclaw-backup-$(Get-Date -f yyyyMMdd-HHmmss)"
New-Item -ItemType Directory -Force $bk | Out-Null
Copy-Item "$env:USERPROFILE\.openclaw\openclaw.json" $bk -Force
robocopy "$env:USERPROFILE\.openclaw\agents" "$bk\agents" /E /NFL /NDL /NJH /NJS

# 3. 只读诊断
openclaw doctor

# 4. 执行迁移
openclaw doctor --fix

# 5. 启动
openclaw gateway run

# 6. 验证（本地请求绕过代理）
$env:HTTP_PROXY=""; $env:HTTPS_PROXY=""
(Invoke-WebRequest "http://127.0.0.1:18789/" -UseBasicParsing).StatusCode
openclaw status

# 7. 确认遗留物清空（两个都该是 0）
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "sessions.json" -EA SilentlyContinue).Count
(Get-ChildItem "$env:USERPROFILE\.openclaw\agents" -Recurse -Filter "catalog.json"  -EA SilentlyContinue).Count

# 8. 抽查每个 Agent 的工作目录是否还指向原处（坑 6，静默故障）
Get-ChildItem "$env:USERPROFILE\.openclaw" -Directory -Force |
  Where-Object Name -like "workspace*" | Select-Object Name, LastWriteTime
```

---

## 版本信息

| 项 | 值 |
| --- | --- |
| OpenClaw | `2026.7.1-2` → `2026.8.1` |
| Node.js | 24.x |
| 系统 | Windows (10.0.26200) |
| 日期 | 2026-09-01 |

---

## 说明

本文命令与报错均来自一次真实修复过程，路径、账号、Token 等已替换为占位符。
第四、五节中标注为「实测」的项目已在同一环境复核；具体迁移条目数量因环境而异，不应视为普适数值。

如果这篇帮你省下了一次重装，欢迎转发给下一个要升级的人。
