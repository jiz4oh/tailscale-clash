# Agent 工作说明

## 项目定位

- Tailscale + ShellCrash Docker 透明代理栈，使用官方 `juewuy/shellcrash` 镜像。
- `tailscale` 通过 `network_mode: service:shellcrash` 共享网络命名空间；订阅、节点、规则和端口由 ShellCrash TUI 管理。

## 支持配置

- 基础配置：只加载 `compose.yaml`，普通 Docker bridge，提供 Tailnet fake-IP 入站。
- Linux 旁路由：只加载完整的 `compose.router.yaml`，使用 macvlan、host shim、IPv4/IPv6 forwarding，并通告 exit-node。
- 仓库只保留上述两种配置；不要新增第三种平台专用 Compose 文件、命名 volumes 或个人网络默认值。

## 不变量

- fake-IP range 固定为 `28.0.0.0/8`，`TS_ROUTES` 必须与其一致。
- Tailnet 流量源地址是 `100.64.0.0/10`，必须加入 TUI 流量过滤的自定义透明路由网段。
- TUI 使用 `Mix` 或 `Tun`、DNS `mix`。
- 整个 `./shellcrash/` 挂载到 `/etc/ShellCrash`，不逐项声明 ShellCrash 目录。
- Linux router 的 `host-shim` 使用 host network、`NET_ADMIN` 和官方 `tailscale/tailscale` 镜像，创建 `macvlan mode bridge`，为 `SHELLCRASH_IP/32` 加精确路由。
- 不提交 `.env`、`tailscale-state/` 或任何 auth key；提交信息使用 Conventional Commits。

## 常用命令

- 基础配置：`docker compose -f compose.yaml up -d`
- Linux 旁路由：`docker compose -f compose.router.yaml up -d`
- 进入 TUI：`docker exec -it shellcrash sh -l`
- Tailscale 状态：`docker exec shellcrash-tailscale tailscale status --json`
- 路由检查：`docker exec shellcrash-tailscale tailscale debug prefs | grep -A4 '"AdvertiseRoutes"'`
- Tailnet DNS：`docker exec shellcrash-tailscale tailscale ip -4`
- 代理验证：基础配置使用 `<TAILSCALE_IP>`；旁路由使用 `<SHELLCRASH_IP>`，端口为 `37890`。

## 修改与验证

- Compose 变更后，用测试 auth key 和文档中的显式测试网络值，对 `compose.yaml` 和 `compose.router.yaml` 分别执行 `docker compose ... config --quiet`。
- router 变更后检查完整配置包含 `proxy` macvlan 和 `host-shim`，并验证宿主到 `SHELLCRASH_IP`、容器到 `HOST_SHIM_IP` 的双向访问。
- 生产文件修改前先备份；重启后检查容器状态、Tailscale 通告、fake-IP DNS 和 Tailnet 客户端直连出站。
- 平台差异通过 Docker runtime 能力判断：基础配置要求 Linux TUN/网络权限；旁路由要求原生 Linux rootful Docker、真实 parent interface 和 macvlan。
- 部署目录、生产 `.env` 和凭据由部署者本地环境提供，仓库不绑定个人路径或网络。
