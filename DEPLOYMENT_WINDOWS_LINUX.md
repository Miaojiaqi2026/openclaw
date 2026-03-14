# OpenClaw Windows 与 Linux 部署说明（按实测流程修订）

本文档基于 2026-03-13 在 Windows 环境的实测成功流程整理，重点保证“能编译、能打包、能部署、能启动、能访问”。

适用范围：
- Windows 10/11（本次已实测）
- Linux（按同逻辑部署）

---

## 1. 本次实测结果摘要

- 源码目录：`D:\OpenClaw\Develop\openclaw`
- 部署目录：`D:\OpenClaw\deploy`
- 发布包：`D:\OpenClaw\deploy\openclaw-2026.3.13.tgz`
- 运行目录：`D:\OpenClaw\deploy\openclaw-runtime-next\package`
- 访问地址：`http://127.0.0.1:18789`
- 验证结果：HTTP 200，页面可打开

---

## 2. 前置条件

### 2.1 环境要求

- Node.js >= 22（建议 22 或更高）
- pnpm（建议与项目锁文件一致版本）
- PowerShell（Windows）
- tar（Windows 11/Server 一般内置 bsdtar）

检查命令：

```powershell
node -v
pnpm -v
```

---

## 3. Windows 完整操作流程（已验证可执行）

### 3.1 从源码编译

在 `D:\OpenClaw\Develop\openclaw` 执行：

```powershell
pnpm install
pnpm ui:build
pnpm build:docker
```

说明：
- Windows 原生环境下，`pnpm build` 可能因 `bash scripts/bundle-a2ui.sh` 失败。
- 本次成功流程使用 `pnpm build:docker` 生成可运行 `dist` 产物。

### 3.2 生成发布包

在 `D:\OpenClaw\Develop\openclaw` 执行：

```powershell
pnpm pack --pack-destination D:\OpenClaw\deploy --config.ignore-scripts=true
```

说明：
- 使用 `--config.ignore-scripts=true`，避免 Windows 下执行 `prepack` 时触发 `bash` 依赖报错。

### 3.3 解压并部署到目标目录

```powershell
$deploy = "D:\OpenClaw\deploy"
$tgz = Join-Path $deploy "openclaw-2026.3.13.tgz"
$runtime = Join-Path $deploy "openclaw-runtime-next"

if (Test-Path $runtime) { Remove-Item -Recurse -Force $runtime }
New-Item -ItemType Directory -Path $runtime | Out-Null
tar -xf $tgz -C $runtime

Set-Location (Join-Path $runtime "package")
pnpm install --prod --ignore-scripts
```

### 3.4 同步 UI 资源（关键）

若未同步 `dist/control-ui`，启动后访问会报：
`Control UI assets not found`。

执行以下命令：

```powershell
$src = "D:\OpenClaw\Develop\openclaw\dist\control-ui"
$dst = "D:\OpenClaw\deploy\openclaw-runtime-next\package\dist\control-ui"

if (Test-Path $dst) { Remove-Item -Recurse -Force $dst }
New-Item -ItemType Directory -Path $dst | Out-Null
Copy-Item -Recurse -Force (Join-Path $src "*") $dst
```

### 3.5 启动服务

```powershell
Set-Location "D:\OpenClaw\deploy\openclaw-runtime-next\package"
node openclaw.mjs gateway --port 18789 --verbose
```

如果提示端口被占用或已有实例：

```powershell
node openclaw.mjs gateway stop
# 若仍未释放，按实际 PID 强制结束
Stop-Process -Id <PID> -Force
```

然后再重新启动。

### 3.6 验证访问

```powershell
Invoke-WebRequest -Uri "http://127.0.0.1:18789" -UseBasicParsing
```

成功标准：
- HTTP 状态码 200
- 返回 HTML，标题包含 `OpenClaw Control`

浏览器访问：

```text
http://127.0.0.1:18789
```
### 3.7 WEB端操作使用

你需要先在你的主机后台获取这个 Token（令牌），填入页面后才能连接并进入真正的控制/聊天界面。

请按照以下步骤操作（参考你截图下方的“How to connect”提示）：

#### 第一步：在终端/命令行获取 Token

1. 打开你电脑上的终端（Terminal）或命令行工具（比如 cmd、PowerShell 或 macOS/Linux 的 Terminal）。
2. 确保你的 OpenClaw 网关已经在运行。如果没有，请运行：
`openclaw gateway run`
3. 接着，运行以下命令来获取带 Token 的控制台地址：
`openclaw dashboard --no-open`
4. 运行后，终端会输出一段信息，其中会包含一长串字符的 **Token**（或者是带有 `?token=...` 的网址）。请将这段 Token 复制下来。

#### 第二步：在页面中连接

1. 回到你截图的这个浏览器页面。
2. 将刚刚复制的 Token 粘贴到 **Gateway Token** 这个输入框中。
3. 点击红色的 **Connect** 按钮。

#### 第三步：开始交互

连接成功后，这个报错页面就会消失，你将进入 OpenClaw 的主控制台（Control UI）。在这个控制台界面里，你就能看到输入框，可以开始和它进行“聊天”或下达指令了。

你现在能在终端里找到那个 `openclaw dashboard --no-open` 命令输出的 Token 吗？如果找不到，可以把终端的输出内容截图发给我帮你看看。
---

## 4. Linux 完整流程（与 Windows 同逻辑）

以下为等价流程，路径示例为 `/opt/openclaw`：

```bash
cd /opt/openclaw/openclaw
pnpm install
pnpm ui:build
pnpm build:docker
pnpm pack --pack-destination /opt/openclaw/deploy --config.ignore-scripts=true

mkdir -p /opt/openclaw/deploy/openclaw-runtime-next
tar -xf /opt/openclaw/deploy/openclaw-2026.3.13.tgz -C /opt/openclaw/deploy/openclaw-runtime-next

cd /opt/openclaw/deploy/openclaw-runtime-next/package
pnpm install --prod --ignore-scripts

mkdir -p dist/control-ui
cp -r /opt/openclaw/openclaw/dist/control-ui/* dist/control-ui/

node openclaw.mjs gateway --port 18789 --verbose
```

验证：

```bash
curl -i http://127.0.0.1:18789
```

---

## 5. 常见问题与修复

### 5.1 `bash 不是内部或外部命令`

原因：Windows 原生环境执行到 bash 脚本。

处理：
- 编译使用 `pnpm build:docker`
- 打包使用 `pnpm pack --config.ignore-scripts=true`

### 5.2 `openclaw: missing dist/entry.(m)js`

原因：构建产物不完整。

处理：
- 在源码目录重新执行：`pnpm build:docker`
- 重新打包、解压部署

### 5.3 `Control UI assets not found`

原因：部署包内缺少 `dist/control-ui`。

处理：
- 执行 `pnpm ui:build`
- 将源码 `dist/control-ui` 复制到运行目录对应位置

### 5.4 `gateway already running` / `Port 18789 is already in use`

处理：
- 先执行：`node openclaw.mjs gateway stop`
- 若仍占用，结束占用端口进程后再启动

---

## 6. 日常启动/停止命令（Windows）

启动：

```powershell
Set-Location "D:\OpenClaw\deploy\openclaw-runtime-next\package"
node openclaw.mjs gateway --port 18789 --verbose
```

停止：

```powershell
Set-Location "D:\OpenClaw\deploy\openclaw-runtime-next\package"
node openclaw.mjs gateway stop
```

检查端口：

```powershell
Get-NetTCPConnection -LocalPort 18789 -State Listen
```

---

## 7. Skills 下载与更新（ClawHub）

OpenClaw 的 skills 推荐通过 ClawHub 管理。

### 7.1 安装 ClawHub CLI

任选一种：

```bash
npm i -g clawhub
```

```bash
pnpm add -g clawhub
```

### 7.2 常用命令

```bash
clawhub search "calendar"
clawhub install <skill-slug>
clawhub update --all
```

### 7.3 Skills 存储位置与持久化说明

- 本地工作目录通常为 `<当前目录>/skills`，并记录在 `.clawhub/lock.json`。
- OpenClaw 默认还会使用用户目录 `~/.openclaw/workspace/skills`（Windows 对应 `%USERPROFILE%\\.openclaw\\workspace\\skills`）。
- 本文档配套批处理在“发布更新”时会保留以下目录中的相关数据（若存在）：
	- `workspace`
	- `skills`
	- `.clawhub`
	- `extensions\\managed`
	- `extensions\\custom`
	- `data` / `storage` / `user-data`
- 用户目录 `%USERPROFILE%\\.openclaw` 不会被清理。

---

## 8. 一键批处理（菜单式）

项目根目录已提供：`deploy_menu.bat`

运行方式：

```powershell
cd D:\OpenClaw\Develop\openclaw
deploy_menu.bat
```

菜单功能：
- `1` 编译项目（`pnpm install` + `pnpm ui:build` + `pnpm build:docker`）
- `2` 生成发布包并更新发布目录（自动保留记忆/skills/MCP 插件相关数据）
- `3` 重启服务并连通性检查
- `4` 全流程（1+2+3）
- `5` Skills 搜索/安装/更新

脚本特性：
- 每一步都有进度提示，失败会中断并提示原因。
- 发布更新前自动备份持久化目录，更新后自动恢复。
- 自动同步 `dist/control-ui`，避免 `Control UI assets not found`。
- 重启服务时会先尝试优雅停止，再清理端口占用。

---

## 9. 安全建议

- 默认建议仅本机访问：优先使用 `127.0.0.1`。
- 若绑定 `0.0.0.0`（LAN 可访问），务必配置认证并控制防火墙策略。
- 不要把模型密钥和渠道 Token 明文提交到仓库。

---

## 10. Qwen 接入详细步骤（Windows 实测可用路径）

本项目已包含 Qwen 门户认证扩展：`extensions/qwen-portal-auth`。

推荐做法：先在网页中完成网关连接，再在模型配置中切到 Qwen。

### 10.1 启动网关

```powershell
Set-Location "D:\OpenClaw\deploy\openclaw-runtime-next\package"
node openclaw.mjs gateway --port 18789 --verbose
```

### 10.2 打开控制台并连接网关

浏览器访问：`http://127.0.0.1:18789`

如果页面要求 Token：

```powershell
openclaw dashboard --no-open
```

复制输出中的 token（或 `?token=...` 链接）填入页面连接。

### 10.3 在 OpenClaw 中切换 Qwen 模型

在控制台设置里将模型提供方切换到 Qwen（或 OpenAI 兼容端点接入 Qwen 服务），常见参数：

- Base URL：你的 Qwen 兼容接口地址
- API Key：Qwen 平台密钥
- Model：例如 `qwen-plus`、`qwen-turbo`（以你账号可用模型为准）

配置完成后保存，并发起一次简单测试对话（例如“你好，请回复 ok”）。

### 10.4 终端侧快速自检

```powershell
Get-NetTCPConnection -LocalPort 18789 -State Listen
Invoke-WebRequest -Uri "http://127.0.0.1:18789" -UseBasicParsing
```

成功标准：

- 18789 端口有监听
- HTTP 返回 200
- 控制台内模型回复正常

### 10.5 常见问题

- 连接后仍无法调用模型：
	- 检查 Base URL 是否为 Qwen 兼容接口地址（不要填错控制台页面地址）。
	- 检查 API Key 是否有效、是否有该模型调用权限。

- 网关日志提示模型请求失败：
	- 先用最小模型名（如 `qwen-turbo`）测试。
	- 检查公司网络代理、防火墙或 TLS 拦截。

- 页面显示 503：
	- 先确认 `dist/entry.js` 与 `dist/control-ui/index.html` 在运行目录存在。
	- 再执行发布流程第 2 步（菜单 `2`）并重启（菜单 `3`）。
