# [ChatGPT接入中国Garmin](https://github.com/coutureone/gitblog/issues/53)

&ensp;我原本考虑把佳明账号换成国际区，想让训练数据更方便地进入 AI 助手。但了解之后发现，换区未必能解决我遇到的接入限制。最后，我保留了 Garmin Connect 中国区账号，借助开源项目 借助开源项目 [Garmin MCP Server](https://github.com/Taxuspt/garmin_mcp)，搭建了一条适合自己使用的链路。

&ensp;现在，我可以在 Codex 桌面端直接询问最近的跑步、睡眠和恢复情况；软路由也会定时取数，生成训练日报并通过 Bark 推送到手机。这里的“接入 ChatGPT”，具体落地在支持 MCP 的 Codex 客户端和我自己写的日报程序中。

## 整体架构

```text
Garmin Connect 中国区
        ↓
软路由 10.10.1。1 上的 garmin-mcp-cn 容器
        ↓ 软路由本机仅监听 127.0.0.1:18765
        ├─ Mac 的 SSH 隧道 → Codex 里的 garmin_cn 工具
        └─ 软路由定时脚本 → AI 训练日报 → Bark 手机推送
```

&ensp;这套方案的核心是让 MCP 服务负责读取佳明数据，让 AI 客户端和定时程序分别消费这些数据。Garmin MCP Server 使用 `python-garminconnect` 访问 Garmin Connect，并把活动、睡眠、心率、HRV、训练状态等能力封装成 MCP 工具。

## 第一步：在软路由运行 Garmin MCP

&ensp;我把服务部署在软路由的 Docker 容器中，镜像名为 `garmin-mcp-cn:local`。容器配置了 `GARMIN_IS_CN=true`，对应 Garmin Connect 中国区；传输方式是 `streamable-http`。服务在容器内监听 8000 端口，映射到软路由的 `127.0.0.1:18765`。

&ensp;佳明登录令牌保存在软路由的持久化目录，并挂载到容器中。这样服务重启后还能继续使用已有令牌，也无需把账号密码写进 Codex 的 MCP 配置。容器还通过 `GARMIN_ENABLED_TOOLS` 只开放当前需要的查询工具。

&ensp;我特意让端口只绑定在软路由的回环地址。这个 HTTP MCP 服务本身没有额外的访问认证，因此不适合直接暴露到局域网或公网；远程访问由 SSH 隧道承担。

## 第二步：用 SSH 隧道连接 Mac 和软路由

&ensp;Mac 上运行一个由 `launchd` 管理的常驻 SSH 端口转发：

```bash
ssh -N -L 127.0.0.1:18765:127.0.0.1:18765 root@10.10.1.1
```

&ensp;这样，Mac 访问本机的 `127.0.0.1:18765`，请求就会通过 SSH 转到软路由上的 Garmin MCP。隧道设置为登录后启动，并在连接中断后自动重启。

Codex 的配置很简单：

```toml
[mcp_servers.garmin_cn]
url = "http://127.0.0.1:18765/mcp"
```

&ensp;接通后，Codex 中就能看到 `garmin_cn` 提供的工具。我可以直接问“最近一周跑量比上一周增加了多少”“昨晚睡眠和 HRV 怎么样”，AI 会调用相应工具取数，再结合问题进行解释。

## 第三步：让数据每天主动来找我

&ensp;除了对话查询，我还写了一个每日训练日报程序。软路由定时启动脚本，向同一个本地 MCP 服务查询最近 30 天的活动、最近 7 天的睡眠和 HRV，以及当天的训练准备度、训练状态、心率和身体电量。

&ensp;程序先把原始结果压缩成跑量、近期训练和恢复指标摘要，再交给模型生成中文建议，最后由 Bark 推送到手机。日报会比较近 7 天与前 7 天的跑量，给出当天一项明确的训练或休息安排；如果模型接口不可用，程序会生成一份保守的降级报告。

&ensp;定时任务在早上 8:00 至 9:45，每隔 15 分钟留一次运行机会，同时用锁和当日发送标记避免重复推送。手表数据可能尚未同步，所以日报不会把“暂时查不到活动”直接理解成“今天没有运动”。



&ensp;对我来说，最大的变化是佳明数据终于能进入日常工作流：想分析时，在 Codex 里直接问；不想主动打开应用时，早上的训练日报会自己送到手机。感谢 [Garmin MCP Server 的开源作者和贡献者]借助开源项目 [Garmin MCP Server](https://github.com/Taxuspt/garmin_mcp)，让我可以在保留中国区账号的前提下，把这些数据真正用起来。