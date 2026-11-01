# Windows 适配与内嵌构建补丁

这两个补丁是相对上游 [`deepseek-ai/dsh-superagent-prototype`](https://github.com/deepseek-ai/dsh-superagent-prototype)
起始提交 `6337fc8`（`feat: add the dsh-superagent research prototype`）的纯 `git diff`，
**按 1 → 2 的顺序**应用。已在干净 worktree 上用 `git apply --check` 验证可干净应用。

| 补丁 | 内容 | 是否必需 |
|---|---|---|
| `1-windows-adapt.patch` | Windows 服务管理器（Node 替代 Python+tmux）、路径/可执行文件/工具白名单适配、控制台闪窗、Git PATH 补齐、`WINDOWS-NOTES.md` | 在 Windows 上运行必需 |
| `2-standalone-embed.patch` | 管理面板的独立构建入口 `src/client/standalone.tsx` + tsdown 构建目标，用于在 DSH NEXT 里内嵌面板 | 只在需要内嵌到 DSH NEXT 时需要 |

## 应用

```bash
git clone https://github.com/deepseek-ai/dsh-superagent-prototype.git
cd dsh-superagent-prototype
git checkout -b windows-adapt 6337fc8

git apply ../patches/1-windows-adapt.patch
git apply ../patches/2-standalone-embed.patch

npm install
npx tsdown    # 应用过 2 号补丁后才会多出 lib/standalone.js（约 546 KB）
```

不想用 `git apply` 时，`patch -p1 < 1-windows-adapt.patch` 亦可；两个补丁都是纯文本 diff，无二进制块。

## 1 号补丁改了哪些文件

| 文件 | 改动 |
|---|---|
| `scripts/dsh-superagent.mjs`（新增 321 行） | Windows 服务管理器：start/status/stop/restart、退出码 0/1/2/3、`--respect-stop`、`.artifacts` 日志、锁清理、`ensureGitBashPath()` |
| `scripts/dsh-superagent.cmd`（新增） | `node …mjs` 的 cmd 包装 |
| `src/deployment.ts` | `stateDir` 接受 `C:\` / `C:/`；`executable()` 按 PATHEXT 补 `.exe/.cmd/.bat` |
| `src/app/sessions.ts` | 工具白名单/`noWait` 按平台选 `bash` / `pwsh` |
| `src/infra/commands.ts` | `execFile` 加 `windowsHide: true`（消除 gh 控制台闪窗） |
| `src/sources/feishu-loops.ts` | `spawn` 加 `windowsHide: true` |
| `WINDOWS-NOTES.md`（新增） | 现场笔记：本机安装位置、启动器用法、与原型不同的 4 处、已知限制、修复记录 |

## 2 号补丁改了哪些文件

| 文件 | 改动 |
|---|---|
| `src/client/standalone.tsx`（新增） | 独立入口：内置 zh/en 字典代替 DSH locale 服务；`openSession` 改为 postMessage；把 `/orchestrator/api/*` 重写成 `../orchestrator/api/*`，经 DSH NEXT 的同源代理访问服务 |
| `tsdown.config.ts` | 新增 `format=esm, platform=browser` 构建目标：React 打进包，`process.env.NODE_ENV` 固化为 `production` |

产物 `lib/standalone.js` 供服务侧插件
[`dsh-superagent-bridge`](https://github.com/deepseekv41flash/dsh-superagent-bridge) 的
`dist/standalone.js` 使用；完整背景与踩坑记录见
[`docs/windows-setup-and-dsh-next-bridge.md`](../docs/windows-setup-and-dsh-next-bridge.md)。

## 注意

- 补丁基于上游 `6337fc8`；上游后续改动可能导致上下文不一致，应用前先看 `git apply --check` 的输出。
- `.artifacts/service-env.json`（含 `GH_CONFIG_DIR` 等本机路径）与运行日志不在补丁里，也不应提交到仓库。
