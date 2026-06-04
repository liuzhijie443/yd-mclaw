# MClaw 使用与部署教程

本文档面向最终用户，说明如何让 MClaw 切换模型、执行部署指南、配置 Cloudflare Tunnels、完成内网穿透，以及通过 SSH 连接容器。

## 0. 先理解这套部署做了什么

这套方案的核心是让 MClaw 在容器里完成以下几件事：

1. 修改 `/home/node/.openclaw/openclaw.json`，切换 OpenClaw 使用的模型。
2. 安装并启动 `cloudflared`，让容器通过 Cloudflare Tunnel 对外提供访问入口。
3. 安装并启动 `dropbear`，开放容器内 `2222` 端口给 SSH 穿透使用。
4. 写入 `/home/node/startall.sh` 和 `/home/node/watchdog.sh`，实现重启恢复和进程保活。

其中 `/home/node/startall.sh` 是整套环境的统一启动脚本。后续你让 MClaw “管理程序”或“执行初始化”时，本质上都围绕这个脚本工作。

## 1. 先让 MClaw 切换模型

把下面这段提示语发给 MClaw：

```md
使用node.js修改模型，配置文件在/home/node/.openclaw/openclaw.json，完成后启用并重启openclaw。
baseUrl：<修改字段>
apiKey：<修改字段>
model：<修改字段>
```

这一步的目标是让 MClaw 自动修改 OpenClaw 的模型配置，并启用后重启生效，特别注意Token消耗，如果还在消耗就让他切换。

如果你只是单独换模型，不涉及隧道或 SSH，这一步就足够了。

## 2. 登录 Cloudflared 并创建隧道

用户需要先在本机登录 Cloudflare，并新建一个 Tunnel，拿到 Cloudflare Tunnels Token。

操作要求：

1. 登录 Cloudflare Zero Trust 或 Tunnel 管理页面。
2. 创建一个新的 Tunnel。
3. 获取该 Tunnel 的 `token`。
4. 保存好这个 `token`，后续要写入部署配置，ey开头到结尾的值。

建议同时规划好两个域名用途：

1. 一个用于 OpenClaw WebUI。
2. 一个用于 SSH Access。
<img width="1515" height="594" alt="59948e9a774217b8b9d79334a66cb3f4" src="https://github.com/user-attachments/assets/31f29363-19b4-4fe5-ab3e-724b40695173" />

<img width="1251" height="678" alt="604b5f02248bb6180609089c851415a0" src="https://github.com/user-attachments/assets/24153945-5a4f-40d2-82ff-9c37a79f188e" />

## 3. 下载并修改 `MClaw部署指南.txt`

将项目中的 [MClaw部署指南.txt] 下载到本地，按文档中的变量配置修改以下变量：

```bash
GATEWAY_AUTH_TOKEN=""
BASE_URL="模型BASE_URL"
API_KEY="模型密钥"
MODEL="模型名称"
CF_TOKEN="你的 Cloudflare Tunnel Token"
```

部署前建议明确这几个值的用途：

1. `BASE_URL`、`API_KEY`、`MODEL` 用于 OpenClaw 的模型接入。
2. `CF_TOKEN` 用于启动 Cloudflare Tunnel。
3. `GATEWAY_AUTH_TOKEN` 用于 Gateway Token 登录和热更新，不需要时可以留空。

必须特别注意：

`GATEWAY_AUTH_TOKEN` 一旦设置，会导致官方客户端失联。设置后必须通过内网穿透访问 WebUI，不能再依赖官方客户端界面。这个配置会改变访问方式，请谨慎操作。

如果你不清楚是否需要改这个值，建议先保持为空，确认内网穿透链路可用后再决定是否启用。后续也可以让 MClaw 修改该字段，再执行 Gateway 热更新。

## 4. 让 MClaw 执行部署指南
<img width="1024" height="455" alt="9b42bbdcb9efe4bbb02b4a4380446743" src="https://github.com/user-attachments/assets/510c124f-443f-4e25-8414-bb77a8921f2c" />

把你修改好的 `MClaw部署指南.txt` 交给 MClaw，让它严格按文档执行。执行过程中，它通常会完成这些内容：

1. 写入 `MEMORY.md`，让智能体记住容器环境限制、安装策略和程序管理方式。
2. 安装 `cloudflared`。
3. 安装并启动 `dropbear SSH`，端口为 `2222`。
4. 写入 `/home/node/startall.sh` 和 `/home/node/watchdog.sh`。
5. 首次执行 `bash /home/node/startall.sh`。
6. 在设置了 `GATEWAY_AUTH_TOKEN` 的情况下，通过 `gateway config.patch` 做热更新。
7. 启用 `boot-md` Hook，确保容器重启后可以自动拉起底层服务。

执行完成后，你需要确认至少有以下结果：

1. 连接器里有机器上线。
2. OpenClaw 模型已经切换到你指定的模型。
3. `cloudflared` 和 `dropbear` 已经启动。

## 5. 等待部署执行完成并配置应用路由

等待 `MClaw部署指南.txt` 中的执行步骤全部完成。执行完成后，连接器中会有机器上线。

机器上线后，继续回到 Cloudflare Tunnel 管理界面，在对应隧道中添加发布应用程序路由，完成内网穿透配置。

建议至少配置两个路由：

1. 一个用于访问 OpenClaw WebUI，指向容器内端口 `18789`。
2. 一个用于 SSH Access，指向容器内端口 `2222`。

完成后，就可以通过 Web 域名访问 OpenClaw，并通过 Cloudflare Access 做 SSH 穿透。


<img width="820" height="725" alt="8c95851599b3d4c9386c923403f402b9" src="https://github.com/user-attachments/assets/8c09da7d-88e0-4d5b-b821-f9dea2235702" />



## 6. SSH 穿透准备

SSH 登录信息如下：

```text
用户名：node
密码：无密码
容器内网端口：2222
```

本机需要先下载安装 `cloudflared`，然后执行下面的命令，把 Cloudflare Access SSH 映射到本地端口：

```bash
cloudflared.exe access ssh --hostname <设置的host> --url localhost:2222
```

这个命令需要在本机保持运行。

这里的 `<设置的host>` 就是你在 Cloudflare Tunnel 或 Access 中为 SSH 配置的域名。

## 7. 进行 SSH 连接

在保持上一步命令运行的前提下，打开新的终端窗口，执行：

```bash
ssh node@localhost -p 2222
```

## 8. 部署后的日常使用

完成部署后，建议按下面的原则使用：

1. 想切换模型，直接让 MClaw 修改模型配置。
2. 想调整 `GATEWAY_AUTH_TOKEN`、`BASE_URL`、`API_KEY`、`MODEL`、`CF_TOKEN`，直接让 MClaw 改，它会同步到脚本中。
3. 想恢复底层服务，优先执行 `bash /home/node/startall.sh`。
4. 想查看和管理现有程序，直接对 MClaw 说“管理程序”。

这套方案部署完成后，也可以继续让 MClaw 在当前环境中安装和接管常见附加程序，例如：

1. 青龙面板
2. OpenList
3. CliProxyApi
4. tinyproxy

这类程序通常也会被纳入 `startall.sh` 的统一启动和管理范围，方便后续重启恢复与集中维护。

## 常见问题

### 1. 失联怎么办

如果 MClaw 失联，重置 MClaw 后直接发送以下内容：

```md
执行/home/node/startall.sh，直接执行，不需要查看不需要询问。
```

对应执行命令是：

```bash
bash /home/node/startall.sh
```

这个操作会重新设置模型和 Cloudflare Tunnels 相关内容。

如果此前设置过 `GATEWAY_AUTH_TOKEN`，那么官方客户端失联通常是正常现象，不代表部署失败。此时应该优先通过你配置好的 Cloudflare 域名访问 OpenClaw WebUI。

### 2. 如何进行程序管理

直接对 MClaw 说：

```text
管理程序
```

它会读取 `startall.sh` 中登记的程序，并按现有流程进行启动、停止、重启、查看状态、卸载或关闭自启等管理操作。

### 3. 怎么修改配置

直接让 MClaw 修改即可。

模型、`GATEWAY_AUTH_TOKEN`、`API_KEY`、`BASE_URL`、`CF_TOKEN` 这类配置修改后，应同步到 `/home/node/startall.sh` 和 `/home/node/watchdog.sh`。按当前方案，直接让 MClaw 修改即可。

### 4. 怎么安装软件

直接让 MClaw 安装即可。

系统信息和运行方式已经提前告知给它，正常情况下会按当前容器环境选择可行安装方式。

这套方案部署后，除了基础的 Tunnel、SSH 和模型切换能力，还可以继续安装常见程序，例如青龙面板、OpenList、`nano`、CliProxyApi、`tinyproxy` 等。

# 来自Linux Do

[LINUX DO社区](https://linux.do/)
