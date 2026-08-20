# tailscale-clash

Tailscale + ShellCrash 的 Docker 透明代理栈，只支持两种配置：

1. 基础配置：在具备 Linux 容器、TUN 和所需网络权限的 Docker 系统上，提供 Tailnet fake-IP 入站。
2. Linux 旁路由配置：在原生 Linux rootful Docker 上，同时提供 LAN 旁路由和 Tailnet fake-IP 入站。

两种配置共用 ShellCrash、Tailscale sidecar、fake-IP 网段和 TUN 透明代理路径。基础配置使用 `compose.yaml`；Linux 旁路由使用包含完整服务定义的 `compose.router.yaml`。

## 配置边界

| 配置 | 用途 | 网络入口 | Tailscale 通告 |
| --- | --- | --- | --- |
| 基础配置 | Tailnet 客户端使用 fake-IP 访问外网 | 普通 Docker bridge | 仅 `28.0.0.0/8` |
| Linux 旁路由 | LAN 客户端和 Tailnet 客户端共用透明代理 | macvlan，加 host shim | `28.0.0.0/8`、exit-node 默认路由 |

基础配置不要求宿主机物理网卡、macvlan、LAN 网关或固定 LAN IP；仍要求 Docker runtime 提供 Linux 容器、`/dev/net/tun`、`NET_ADMIN`、`NET_RAW`、容器内转发和防火墙能力。Linux 旁路由还要求 rootful Docker 能操作真实 LAN parent interface、创建 macvlan，并在宿主网络命名空间创建 host shim。

## 前置条件

- Docker Engine 和 Docker Compose v2.24.4+
- `TS_AUTHKEY`，用于首次加入 Tailnet
- ShellCrash 透明模式所需的 `/dev/net/tun`、`NET_ADMIN` 和 `NET_RAW`
- 基础配置可运行在 Linux、WSL2 或 Docker Desktop for macOS，前提是其 Linux container runtime 提供上述设备和权限
- Linux 旁路由仅适用于原生 Linux rootful Docker；Docker Desktop、WSL2 的 Docker runtime 和 rootless Docker 不提供本项目所需的 LAN parent 接管能力

## 首次准备

复制环境变量示例，并按所选配置填写变量：

```sh
cp .env.example .env
```

首次部署时初始化完整的 ShellCrash 目录；已有 `shellcrash/` 配置时跳过：

```sh
docker create --name shellcrash-init juewuy/shellcrash
docker cp shellcrash-init:/etc/ShellCrash/. ./shellcrash/
docker rm shellcrash-init
```

## 启动

基础配置：

```sh
docker compose -f compose.yaml up -d
```

Linux 旁路由配置：

```sh
docker compose -f compose.router.yaml up -d
```

切换配置时使用目标配置文件执行 `up -d --force-recreate`。

## 环境变量

公共变量：

- `TS_AUTHKEY`、`TS_HOSTNAME`、`TS_IMAGE_TAG`
- `TS_ROUTES` 必须保持为 `28.0.0.0/8`

Linux 旁路由还需要 `TS_ROUTER_EXTRA_ARGS`、`PARENT_IFACE`、`LAN_SUBNET`、`GATEWAY_IP`、`SHELLCRASH_IP`、`MACVLAN_IP_RANGE`、`HOST_SHIM_IP` 和 `HOST_SHIM_IFACE`。这些网络参数没有项目默认值，必须填写部署环境的实际值；LAN 客户端的网关和 DNS 都指向 `SHELLCRASH_IP`。

`HOST_SHIM_IP` 必须是 LAN 中未占用、且不在 `MACVLAN_IP_RANGE` 内的地址。它由 host shim 以 `/32` 配置，用于宿主访问 `SHELLCRASH_IP`，也作为容器访问宿主服务的地址。

## ShellCrash TUI

进入 TUI：

```sh
docker exec -it shellcrash sh -l
```

两种配置都按 [docs/shellcrash-tui.md](docs/shellcrash-tui.md) 配置：

1. 导入订阅或配置文件。
2. 路由模式选择 `Mix` 或 `Tun`。
3. DNS 模式选择 `mix`。
4. fake-IP range 固定使用 `28.0.0.0/8`。
5. 在流量过滤的自定义透明路由网段中加入 `100.64.0.0/10`。

## 验证

检查基础配置渲染：

```sh
env TS_AUTHKEY=tskey-auth-test TS_HOSTNAME=test TS_ROUTES=28.0.0.0/8 \
  docker compose -f compose.yaml config --quiet
```

检查 Linux 旁路由配置渲染：

```sh
env TS_AUTHKEY=tskey-auth-test TS_HOSTNAME=test \
  TS_ROUTES=28.0.0.0/8 \
  TS_ROUTER_EXTRA_ARGS='--advertise-exit-node --snat-subnet-routes=true' \
  PARENT_IFACE=test0 \
  LAN_SUBNET=192.0.2.0/24 \
  GATEWAY_IP=192.0.2.1 \
  SHELLCRASH_IP=192.0.2.2 \
  MACVLAN_IP_RANGE=192.0.2.0/29 \
  HOST_SHIM_IP=192.0.2.12 \
  HOST_SHIM_IFACE=test-shim \
  docker compose -f compose.router.yaml config --quiet
```

检查 Tailscale 通告：

```sh
docker exec shellcrash-tailscale tailscale debug prefs | grep -A4 '"AdvertiseRoutes"'
```

基础配置只应通告 `28.0.0.0/8`；旁路由配置还应通告 `0.0.0.0/0` 和 `::/0`。

从 Tailnet 客户端验证 fake-IP：

```sh
dig @<TAILSCALE_IP> www.google.com +short
curl -o /dev/null -w '%{http_code}\n' https://www.gstatic.com/generate_204
```

DNS 应返回 `28.x.x.x`，直连请求应能通过 ShellCrash 出站。旁路由配置还应从 LAN 客户端执行 DNS 检查，并验证宿主与容器的双向访问：

```sh
# 宿主机经 shim 路由访问 ShellCrash
ip -d link show <HOST_SHIM_IFACE>
ip route get <SHELLCRASH_IP>
curl -x http://<SHELLCRASH_IP>:37890 -o /dev/null -w '%{http_code}\n' https://www.gstatic.com/generate_204

# 容器经 shim 地址访问宿主服务
docker exec shellcrash nc -z -w 3 <HOST_SHIM_IP> <HOST_PORT>
```

`ip -d link show` 应显示 `macvlan mode bridge`。host shim 使用官方 `tailscale/tailscale` 镜像提供完整的 `iproute2`，并在重启时重新创建接口和精确的 `SHELLCRASH_IP/32` 路由。

## 持久化与安全

- 整个 `./shellcrash` 目录挂载到 `/etc/ShellCrash`，配置、程序文件和运行时状态一起持久化。
- `tailscale-state/` 持久化 Tailscale 身份，不要提交到 Git。
- 不要提交 `.env` 或 Tailnet auth key。
- 不使用 ShellCrash 命名 volumes；部署目录由 Compose 文件旁的 `shellcrash/` 目录承载。

## 目录

- `compose.yaml`：基础配置，提供 ShellCrash、Tailscale 和 Tailnet fake-IP 入站
- `compose.router.yaml`：Linux 旁路由完整配置，提供 macvlan、host shim 和 exit-node 通告
- `shellcrash/`：ShellCrash 持久化配置，Git 已忽略
- `docs/architecture.md`：两种配置的流量路径和能力边界
- `docs/router.md`：Linux 旁路由和 host shim 说明
- `docs/shellcrash-tui.md`：TUI 配置步骤
