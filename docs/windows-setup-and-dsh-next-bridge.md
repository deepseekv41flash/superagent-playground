# 在 Windows 上自建 DSH Superagent，并把它桥接进 DSH NEXT 看板

本文记录一条已经实测跑通的路径：从 `deepseek-ai/dsh-superagent-prototype`（下称"原型"）起步，
在 Windows 11 上把 dsh-superagent 服务跑起来、完成一次端到端派活验证，最后把 Superagent 自己的
管理面板嵌进 **DSH NEXT** 的同一个窗口里。

面向的读者：在 Windows 上搭这套东西、或者想把一个"跑在独立 DSH profile 里的服务"接进 DSH NEXT 的开发者。
重点写在**踩过的坑**和对应解决办法上；所有本地路径、账号名和令牌都已脱敏，补丁本身见
[`patches/`](../patches)。

- 环境：Windows 11，Node.js 24.x，Git for Windows，GitHub CLI（`gh`），DSH `0.2.1-rc.4` / desktop-next `2.0.21-next`
- 成果：服务可用、端到端验证通过（Issue → Ticket → 执行 Session → PR）；DSH NEXT 侧栏出现 Superagent 入口，
  内嵌完整面板（概览/来源/触发/评估/执行/设置），并可从面板直接打开 orchestrator 主 Session

---

## 1. 最终形态

```
DSH NEXT (桌面端, 页面源 dsh-app://app)
├── 插件 dsh-superagent-panel
│   ├── 侧栏入口 sidebar.panellist(id=superagent)
│   ├── 主面板：工具条 + iframe
│   │     ├── 同源反向代理 /@dsh-superagent-panel/proxy/*  →  http://127.0.0.1:3082/*
│   │     ├── iframe 加载 <proxy>/superagent-bridge/（独立面板页）
│   │     └── postMessage：主题变量 ↓ / 打开主 Session ↑
│   └── 服务管理接口 /@dsh-superagent-panel/api/{status,action}
│          → 运行 checkout 里的 scripts/dsh-superagent.mjs
│
└── 独立进程：dsh-superagent（profile = superagent）
    ├── DSH host 监听 127.0.0.1:3082，注册 /orchestrator/api/* 管理接口
    ├── 插件 dsh-superagent-bridge
    │     ├── GET /superagent-bridge/       页面外壳（深色兜底主题 + 主题/会话消息桥）
    │     └── GET /superagent-bridge/app.js 面板 ESM（原面板 React 应用，独立构建）
    ├── GitHub Source 每 60s 轮询 → 生成 Trigger/Thread
    ├── Policy/Eval → Ticket → 执行器为每个工单开一个 DSH Session（worktree）
    └── stateDir 下的 SQLite + work/ + worker-bin/
```

要点：**服务是一个独立 DSH host**（自己的 profile、自己的端口），不是 DSH NEXT 的一个插件；
两边通过 DSH NEXT 侧的"同源代理 + iframe"和"日志 token / 服务管理接口"连接。

| 组件 | 位置 | 作用 |
|---|---|---|
| 服务本体 | 原型 checkout（本仓库的 `patches/1` 适配 Windows） | Source、Policy、执行器、管理 API、面板页面 |
| 服务启动器 | `scripts/dsh-superagent.mjs`（`patches/1` 新增） | 原型的 Python+tmux 管理器在 Windows 不可用，改为 Node 等价实现 |
| 服务侧桥接插件 | [dsh-superagent-bridge](https://github.com/deepseekv41flash/dsh-superagent-bridge) | 把面板暴露成不依赖 DSH 长连接的独立页 |
| DSH NEXT 侧面板插件 | [dsh-superagent-panel](https://github.com/deepseekv41flash/dsh-superagent-panel) | 侧栏入口、同源代理、启停/重启、打开主 Session |

---

## 2. 环境与前置

| 项 | 说明 |
|---|---|
| 操作系统 | Windows 11（本文实测环境；Linux/macOS 不需要 `patches/1`） |
| Node.js | ≥ 22（实测 24.x） |
| Git for Windows | 原型用 `gh`/`git` 完成 GitHub 操作；`Git\usr\bin` 里的 `sh`/`env` 也会被服务用到 |
| GitHub CLI | 至少两个账号：**操作者自己的账号** + **agent 账号**（agent 用它开 PR / 回评论） |
| DSH CLI | 能与服务版本匹配的一份（我们用了隔离安装的 `@deepseek-ai/dsh@0.2.1-rc.3`，避免升级影响日常 `dsh`） |
| DSH NEXT | 桌面端（用于内嵌；本文用 `2.0.21-next`） |

准备两个 `gh` 配置目录：操作者日常的（默认 `~/.config/gh`）和 agent 专用的（例如
`%USERPROFILE%\.config\gh-agent`）。服务只通过 `GH_CONFIG_DIR` 指向后者，**不碰操作者自己的凭据**。

```powershell
# 给 agent 账号单独登录一次（Git 凭据问题选 No，避免写入全局 git 配置）
$env:GH_CONFIG_DIR = "$env:USERPROFILE\.config\gh-agent"
gh auth login
gh auth status
```

---

## 3. 从原型到可运行服务

### 3.1 获取原型、应用 Windows 补丁

```powershell
git clone https://github.com/deepseek-ai/dsh-superagent-prototype.git
cd dsh-superagent-prototype
git checkout -b windows-adapt <upstream-commit>   # 我们基于 6337fc8 起步

# 两个补丁按顺序应用（见 patches/README.md）
git apply ..\patches\1-windows-adapt.patch
git apply ..\patches\2-standalone-embed.patch
npm install
```

`2-standalone-embed.patch` 只影响"内嵌到 DSH NEXT"这一步；只想把服务跑起来可以只应用 `1`。

### 3.2 服务环境变量：`.artifacts/service-env.json`

启动器会读 `<repo>/.artifacts/service-env.json`，把里面的键值注入服务进程环境：

```json
{
  "GH_CONFIG_DIR": "%USERPROFILE%\\.config\\gh-agent",
  "DSH_SUPERAGENT_DSH": "<隔离安装的 dsh 入口，例如 %USERPROFILE%\\src\\dsh-cli\\node_modules\\@deepseek-ai\\dsh\\lib\\bin.js>"
}
```

- `GH_CONFIG_DIR`：agent 账号的凭据目录（服务里所有 `gh` 调用都用它）。
- `DSH_SUPERAGENT_DSH`：启动器用哪个 DSH 入口起服务（也可以放 `dsh` 到 PATH 上）。
- 这个文件不要提交到仓库（`.artifacts/` 默认被忽略）。

然后在 DSH 的某个 profile（示例叫 `superagent`）里把服务挂起来：

```bash
dsh plugin --profile superagent add "link:<原型 checkout 路径>"
# profile 的 dsh.profile.bundles 里也要有 dsh-superagent（插件管理器会自动写）
```

### 3.3 启动器

原型的 `scripts/dsh-superagent` 是 Python + `tmux` + `fcntl` 写的，Windows 上没有等价物，
`patches/1` 增加了一个 Node 实现，参数与退出码语义对齐：

```powershell
node scripts\dsh-superagent.mjs status  --profile superagent --url http://127.0.0.1:3082
node scripts\dsh-superagent.mjs start   --profile superagent
node scripts\dsh-superagent.mjs stop    --profile superagent
node scripts\dsh-superagent.mjs restart --profile superagent
```

退出码：`0` 正常 / `1` 未运行或出错 / `2` 部件报错 / `3` 等待 setup。

启动后面板 URL（含一次性 token）打在 `.artifacts\dsh-superagent-superagent-*.log`；服务自己的
DSH Web 面板就监听在 `http://127.0.0.1:3082`。

---

## 4. 端到端验证

### 4.1 我们实际跑通的链路

1. 在测试仓库（`<owner>/superagent-playground`）开 Issue：`@<agent-account> 请在 README 末尾加一行 …`；
2. **约 1 分钟**内面板出现 Ticket（`open` / 阶段 `person`），GitHub Source 记录 received=1；
3. 执行器在 `stateDir\work\...` 建 git worktree，为工单开一个 worker Session；
4. agent 账号推送分支并开 PR（PR 描述以"自动化 agent"披露行开头），Issue 收到一条确认与一条完成回复；
5. PR 合并后，Ticket 走向结束状态；管理面板的"执行 / 工单详情"能看到触发原文、决策理由、订阅线程与 GitHub 链接。

观察点（都用只读接口，避免靠猜测）：

```bash
# 服务状态
node scripts/dsh-superagent.mjs status --profile superagent --url http://127.0.0.1:3082
# 管理 API：工单 / 来源 / 概览（本机 Host + 标记头）
curl -H "x-orchestrator-request: 1" http://127.0.0.1:3082/orchestrator/api/tickets
curl -H "x-orchestrator-request: 1" http://127.0.0.1:3082/orchestrator/api/sources
curl -H "x-orchestrator-request: 1" http://127.0.0.1:3082/orchestrator/api/state
```

### 4.2 复现要点

- Issue 里必须 **@ 到 agent 账号**；否则默认策略下没有候选（也可以在设置里打开"接收新 Issue/PR/评论/CI"）。
- 第一次成功前 setup 会做一次账号/仓库检查；网络抖动会让它卡在 `agentCliSignedOut`（见坑 #17）。
- 执行器用 `prepare` 命令建 worktree；命令要能被 Git for Windows 的 `sh` 执行（见坑 #8）。

---

## 5. 内嵌到 DSH NEXT

### 5.1 两堵墙

1. **跨协议 iframe**：DSH NEXT 的页面源是自定义协议 `dsh-app://app`。Chromium 对
   `dsh-app://app` → `http://127.0.0.1:3082` 的 iframe **静默拒绝**——既没有 load 也没有 error 事件，
   只能看到空白（我们用一个同源 iframe 做对照探针才确认）。
2. **DSH 客户端长连接**：在宿主里做一条同源反向代理（`dsh-app://app/<prefix>/proxy/*` → 3082），
   页面确实能加载了，但 **DSH 客户端的事件长连接在该代理源下建立不起来**，界面一直显示
   「dsh-superagent 没有响应 / 重新连接中」，而所有 `/proxy/orchestrator/api/*` 请求都是 200。
   这不是代理写错了，而是客户端连接层绑定在 `dsh-app://app` 环境上。

**结论（也是本方案的核心）**：不要试图把"DSH 应用"整页搬进来，而是让服务侧**另出一个不依赖 DSH 连接层的页面**。

### 5.2 最终架构

- **服务侧插件 `dsh-superagent-bridge`**：把原面板的 React 应用打包成**独立 ESM**（React 打进包里），
  用 `GET /superagent-bridge/` + `/app.js` 提供；页面所有数据走 `../orchestrator/api/*` 普通 HTTP 轮询。
- **DSH NEXT 侧插件 `dsh-superagent-panel`**：
  - 宿主里注册同源反向代理，把 3082 搬到 `dsh-app://app/@dsh-superagent-panel/proxy/*`；
  - 代理剥掉 `Origin`/`Referer`/`Sec-Fetch-*`/`x-dsh-desktop-renderer`（3082 会 403 跨站请求），
    并用最新日志里的 token URL 换一次上游 cookie，401 时重换；
  - iframe 指向 `<proxy>/superagent-bridge/?lang=zh`；
  - 外层读 `--dsw-alias-*` 主题变量，postMessage 注入 iframe（独立文档拿不到宿主 CSS 变量）；
  - iframe 里的"打开主 Session"postMessage 回外层，由 `uiWorkspace.openSession` 跳转。

### 5.3 独立面板的构建（`patches/2`）

`2-standalone-embed.patch` 给原型增加了：

- `src/client/standalone.tsx`：独立入口。用内置 zh/en 字典代替 DSH locale 服务；`openSession` 改为
  postMessage；安装一个 `fetch` 重写，把 `/orchestrator/api/*` 换成 `../orchestrator/api/*`（相对代理路径）。
- `tsdown.config.ts`：新增一个 `format=esm / platform=browser` 的构建目标，React 打进包，
  `process.env.NODE_ENV` 固化为 `production`。

```bash
npx tsdown                 # 产出 lib/standalone.js（约 546 KB，gzip ~137 KB）
```

把产物放进桥接插件的 `dist/standalone.js` 即可（该插件默认优先用它）。

### 5.4 安装

```bash
# A. 服务侧（装进跑服务的那个 profile，然后重启服务）
dsh plugin --profile superagent add dsh-superagent-bridge
node scripts/dsh-superagent.mjs restart --profile superagent

# B. DSH NEXT 侧（装进 DSH NEXT 用的 profile，然后重启 DSH NEXT）
dsh plugin --profile desktop add dsh-superagent-panel
```

配置（可选）：两个插件都支持在 profile patch 的条目里写 `config`，也支持环境变量
`DSH_SUPERAGENT_REPO`（原型 checkout）、`DSH_SUPERAGENT_STANDALONE`（面板 ESM 路径）。

### 5.5 宿主机重启后怎么启动

服务本体是独立进程，**重启后不会自动启动**。三种方式：

1. 打开 DSH NEXT 的 Superagent 面板，点「启动」；
2. 命令行：`node scripts/dsh-superagent.mjs start --profile superagent`；
3. 让面板插件随 DSH NEXT 一起拉起（面板插件 0.1.3+）：在 DSH NEXT 所用 profile 的
   `cordis.patch.yml` 末尾加

   ```yaml
   - id: dsh-superagent-panel
     name: dsh-superagent-panel
     config:
       autoStart: true
   ```

   `autoStart` 只在服务未运行时生效，手动 `stop` 留下的 `manually-stopped` 标记会被尊重；
   实测 DSH NEXT 启动后 **14 秒**内服务就绪（插件延迟 8 秒再拉起，启动本身约 6 秒）。
   注意前提是 DSH NEXT 自己先启动——是否需要开机自启由桌面端/系统登录项决定，与插件无关。


---

## 6. 踩过的坑与解决办法

### 6.1 服务侧（Windows 适配）

| # | 现象 | 根因 | 解决 |
|---|---|---|---|
| 1 | 原型启动不了 | 服务管理器是 Python + `tmux` + `fcntl`，Windows 无此组合 | 用 Node 重写 `scripts/dsh-superagent.mjs`（+ `.cmd` 包装），参数/退出码/日志/`--respect-stop` 语义对齐 |
| 2 | `stateDir` 配置校验失败 | 原型只接受 POSIX 绝对路径 | `src/deployment.ts` 接受 `C:\` / `C:/` |
| 3 | `gh` 在 PATH 上"找不到" | Windows 用 PATHEXT，命令名要补 `.exe/.cmd/.bat` | `src/deployment.ts` 的 `executable()` 按 PATHEXT 探测 |
| 4 | 恢复 orchestrator Session 时报 `tools.restrict() names unknown global tool "bash"` | Windows 上 DSH 只注册 `pwsh`；白名单写死 `bash` 会让插件挂载失败 | `src/app/sessions.ts` 按平台选 `bash`/`pwsh` |
| 5 | 强制停止后马上启动：`Lock file is already being held`；**或重启后管理 API 全部 404、面板显示服务无响应** | lockfile 用 `<stateDir>.lock`，Windows `taskkill` 不会释放；`proper-lockfile` 的 stale 窗口是 30s。更隐蔽的一种：上一次失败的重启删掉了 `service.pid` 却留下新鲜的锁目录，启动器 `stop` 因“没有存活 pid”跳过清锁，紧接着的 `start` 被锁挡住，`dsh-superagent` 插件整体挂载失败 → `/orchestrator/api/*` 全 404 | 启动器 `stop`/`restart` **无条件**强制清锁；`start` 在“无存活 pid 且 URL 无 orchestrator 应答”时也强制清锁，不再等 30s；外部崩溃的 31s 兜底保留 |
| 6 | 服务运行时每分钟闪一批控制台窗口（6–8 个） | GitHub Source 每 60s 起 6–8 个 `gh`，`execFile` 默认 `windowsHide:false`，而服务本身无控制台 | `src/infra/commands.ts`、`src/sources/feishu-loops.ts`、启动器的 `where/taskkill` 全部 `windowsHide:true`；95s 观察窗 15 → 0 |
| 7 | 从 DSH NEXT 面板拉起服务后卡在 `setup: agentCliSignedOut`，`values.output = "spawn env ENOENT"` | 插件的 setup 探针要跑 `env`、prepare 要跑 `sh`；Git for Windows 把它们放在 `Git\usr\bin`，安装器只把 `Git\cmd` 加进 PATH。从 Git Bash 启动时天然有，从桌面进程启动时没有 | 启动器 start 时 `ensureGitBashPath()`：把存在的 `Git\cmd`、`Git\usr\bin`、`Git\mingw64\bin`、`Git\bin` 前置进子进程 PATH |
| 8 | agent 账号隔离 | 原型用 `~/.local/bin/gh-agent` 这种 `#!/bin/sh` 包装脚本；Windows 不执行 sh shim，`blockedCommands` 失效 | 改用 `GH_CONFIG_DIR` 指向 agent 专用配置目录；注意 `blockedCommands` 在 Windows 不生效（已知限制） |

### 6.2 内嵌到 DSH NEXT

| # | 现象 | 根因 | 解决 |
|---|---|---|---|
| 9 | 页面里 `fetch('/api/...')` 一律 403 | 桌面端 `desktop-preferences.json` 有 `browserAccess:false`，非 Electron 请求被拒 | 走宿主内的 webServer 路由（渲染器请求带 `x-dsh-desktop-renderer` 头/会话 cookie）；代理内部转发时保留该链路 |
| 10 | iframe 一片空白，无 load/error | Chromium **静默拒绝**跨协议 iframe（`dsh-app://app` 嵌 `http://`） | 改为同源代理路径；用同源 iframe 探针证实"同源可用、跨协议静默失败" |
| 11 | 试 Electron `<webview>` 内嵌 → DSH 渲染进程崩溃 | `dshDesktop.browser.acquire()` 租约 + 原生 webview 在 React StrictMode 双调用下有 attach/detach 竞态 | 放弃原生 webview，回滚；只走 iframe |
| 12 | 代理转发 403 | 3082 管理 API 拒绝跨站 `Origin`（`dsh-app://app` 被当成跨站） | 代理剥掉 `Origin`/`Referer`/`Sec-Fetch-*` 等；A/B 实测：带 origin 403 / 去掉 200 |
| 13 | 页面能加载但一直「没有响应 / 重新连接中」 | DSH 客户端事件长连接在代理源下建立不起来（所有 HTTP API 却是 200） | **关键转折**：不再内嵌 DSH 应用，服务侧另出独立页面（本项目 `dsh-superagent-bridge`） |
| 14 | 独立页面的 API 全 404 | 页面在 `<proxy>/superagent-bridge/` 下，绝对路径 `/orchestrator/api/*` 解析到 `dsh-app://app/` 根，不走代理 | 页面里把 `/orchestrator/api/*` 重写成 `../orchestrator/api/*`（相对代理路径） |
| 15 | 桥接页 `?lang=zh` 返回 `unknown route` | 前缀路由拿到的 `req.url` 带 query，子路由比对用了完整串 | 比对前 `split('?')[0]` |
| 16 | 桥接页主题发灰/样式错乱 | 同源 iframe 是独立文档，拿不到宿主 `--dsw-alias-*` 变量 | 外层读 computed style → postMessage → 页面逐项 `setProperty`；桥接页内置深色兜底 |
| 17 | 页面里「打开主 Session」跳不动 | iframe 是独立文档，拿不到 DSH 的 `uiWorkspace` | postMessage 回外层，由面板插件调 `uiWorkspace.openSession(masterId)` |
| 18 | 服务重启后 iframe 仍显示旧 token 页面/401 | 面板 URL 的 token 每次重启都变 | 从最新日志重新取 token URL 换 cookie；401 时强制重换一次 |
| 19 | **本地端口耗尽**：所有本机连接报 `EADDRINUSE`，连 `Test-NetConnection` 都异常 | 曾让桥接插件把 API 转发给"自己"（服务进程内 `http.request` 到本机 3082），叠加面板轮询产生约 1.5 万个 `TIME_WAIT` | 取消自转发：API 改走 DSH NEXT 侧的同源代理（`../orchestrator/api/*`）；`TIME_WAIT` 约 2 分钟自然回收 |
| 20 | 面板里出现两个标题栏/两份状态 | 外层工具条与内嵌面板都在展示同一批信息 | 外层只保留状态徽标与操作按钮，信息交给内嵌面板；删掉重复的「状态」tab |
| 21 | 重启服务后内嵌面板“没有响应”/接口 404 | 与 6.1 #5 同源：锁未清导致 `dsh-superagent` 插件挂载失败，管理 API 404。排查：`curl -H "x-orchestrator-request: 1" http://127.0.0.1:3082/orchestrator/api/state` 返回 404，且服务日志里有 `Lock file is already being held` | 已修复（见 6.1 #5）；临时处置：再 `stop` + `start` 一次即可恢复 |

### 6.3 环境类（不是代码问题，但会消耗大量排查时间）

- **GitHub API 间歇性失败**：`gh api` 报 `EOF` / `net/http: TLS handshake timeout`，表现为 Source 健康度
  `pollFailures`、setup 卡在 `agentCliSignedOut`。本机到 `api.github.com` 的链路（代理/VPN）抖动，
  shell 里重试几次常常恢复；服务每 60s 重试，恢复后健康度自动变回 ready。
  排查时先用**同一台机器上的 `gh api notifications`** 对照，区分"服务配置问题"和"链路问题"。
- **`npm publish` 首次报 `E409 Cannot publish over previously staged version "0.1.0"`**：npm 新包的
  staged publish 语义导致 0.1.0 冲突；换 `0.1.1` 首发即可。
- **注入器（dsh-super-injector）相关**：早期版本的 slot 白名单没有 `sidebar.panellist` / `main`，
  以及 junction 默认写到 `web` profile 而不是当前 `desktop` profile——升级注入器或手工建 junction 解决。

---

## 7. 补丁说明

两个补丁都是相对上游 `6337fc8` 的纯 `git diff`，应用顺序 1 → 2。

### `patches/1-windows-adapt.patch`（435 行新增/5 行修改）

| 文件 | 改动 |
|---|---|
| `scripts/dsh-superagent.mjs`（新增 321 行） | Windows 服务管理器：start/status/stop/restart、退出码 0/1/2/3、`--respect-stop`、`.artifacts` 日志、锁清理、`ensureGitBashPath()` |
| `scripts/dsh-superagent.cmd`（新增） | `node …mjs` 的 cmd 包装 |
| `src/deployment.ts` | `stateDir` 接受 `C:\`/`C:/`；`executable()` 按 PATHEXT 补后缀 |
| `src/app/sessions.ts` | 工具白名单/`noWait` 按平台选 `bash`/`pwsh` |
| `src/infra/commands.ts` | `execFile` 加 `windowsHide:true` |
| `src/sources/feishu-loops.ts` | `spawn` 加 `windowsHide:true` |
| `WINDOWS-NOTES.md`（新增） | 上面这些坑的现场笔记（本文是它的整理版） |

### `patches/2-standalone-embed.patch`（新增 76 行）

| 文件 | 改动 |
|---|---|
| `src/client/standalone.tsx`（新增） | 独立入口：内置字典、postMessage 打开 Session、API 路径重写 |
| `tsdown.config.ts` | 新增 browser ESM 构建目标（React 打包、production define） |

---

## 8. 复现清单

```powershell
# 0) 前置：Git for Windows、Node 22+、gh、DSH NEXT
# 1) 取原型 + 打补丁
git clone https://github.com/deepseek-ai/dsh-superagent-prototype.git
cd dsh-superagent-prototype
git checkout -b windows-adapt 6337fc8
git apply <本仓库>/patches/1-windows-adapt.patch
git apply <本仓库>/patches/2-standalone-embed.patch
npm install

# 2) 服务环境
#    写 .artifacts/service-env.json（GH_CONFIG_DIR / DSH_SUPERAGENT_DSH）
#    建 superagent profile：dsh plugin --profile superagent add "link:<此目录>"，并在 bundles 中登记

# 3) 起服务
node scripts/dsh-superagent.mjs start --profile superagent
node scripts/dsh-superagent.mjs status --profile superagent      # 0 = 正常

# 4) 端到端：在测试仓库开 Issue 并 @agent 账号，约 1 分钟出 Ticket
curl -H "x-orchestrator-request: 1" http://127.0.0.1:3082/orchestrator/api/tickets

# 5) 桥接 + 面板
npx tsdown                                                        # 产出 lib/standalone.js
npm install -g dsh-superagent-bridge dsh-superagent-panel         # 或从仓库安装
dsh plugin --profile superagent add dsh-superagent-bridge
node scripts/dsh-superagent.mjs restart --profile superagent
dsh plugin --profile desktop add dsh-superagent-panel
# 重启 DSH NEXT → 侧栏 Superagent → 内嵌完整面板
```

---

## 9. 安全与脱敏

- agent 账号的凭据只放在 `GH_CONFIG_DIR` 指向的独立目录里，服务进程通过环境变量使用；
  操作者自己的 `gh`/git 凭据不受影响，也**没有**写进任何全局 git 配置。
- 面板 URL 带一次性 token，只落在 `.artifacts` 日志里；桥接插件不转发、不缓存它，
  DSH NEXT 侧的代理只在换上游 cookie 时读取最新日志。
- 本仓库的文档、补丁与两个插件仓库都不含 token、账号密码、本机用户名路径；
  示例统一用 `<owner>`、`<agent-account>`、`%USERPROFILE%`、`<repo>` 等占位符。
- `.artifacts/service-env.json`、运行日志、SQLite 状态目录都不要提交到仓库。

---

## 10. 已知限制与后续

- Windows 上 `blockedCommands` 不生效（sh shim 不可执行），agent 账号的限制依赖 `GH_CONFIG_DIR` 隔离。
- 停止是"先 pause、等执行器跑完，再 `taskkill`"，不等待 Session 清理回调（SQLite 崩溃安全，可接受）。
- 内嵌面板的「部署」设置组只读：它依赖 DSH Plugin Manager 的 config form 服务，独立页面不可用
  （在 DSH 内直接打开该面板时同样如此）。
- DSH NEXT 的页面源是 `dsh-app://app`，任何"整页内嵌另一个 DSH 应用"的尝试都会撞上坑 #13；
  更稳的方向仍是"服务侧提供一个不依赖 DSH 客户端连接层的独立页"，这也是两个插件采用的结构。

## 参考

- 原型：<https://github.com/deepseek-ai/dsh-superagent-prototype>
- 服务侧桥接插件：<https://github.com/deepseekv41flash/dsh-superagent-bridge>
- DSH NEXT 侧面板插件：<https://github.com/deepseekv41flash/dsh-superagent-panel>
- npm：`dsh-superagent-bridge`、`dsh-superagent-panel`
