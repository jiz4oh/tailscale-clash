# Architecture

## Goal

用官方 ShellCrash Docker 镜像提供透明代理，并支持两种入口：基础配置接收 Tailnet fake-IP 流量；Linux 旁路由配置同时接收 LAN 和 Tailnet 流量。两种配置共享 ShellCrash、Tailscale sidecar、fake-IP 网段和 TUN 透明代理路径。

## Stack

- `shellcrash`：官方 `juewuy/shellcrash` 镜像，启动 CrashCore、fake-IP DNS 和透明防火墙。
- `tailscale`：使用 `network_mode: service:shellcrash`，让 Tailnet 流量直接进入 ShellCrash 所在的网络命名空间。
- `host-shim`：仅由 Linux 旁路由完整配置启用，在宿主网络命名空间为 macvlan 容器提供宿主互访路径。
- `shellcrash/`：完整挂载到 `/etc/ShellCrash`，由 TUI 和 ShellCrash 维护订阅、规则、配置和运行时状态。

## Mode matrix

| 项目 | 基础配置 | Linux 旁路由 |
| --- | --- | --- |
| Compose | `compose.yaml` | `compose.router.yaml` |
| 平台 | 满足 Linux container/TUN/网络权限的 Docker runtime | 原生 Linux rootful Docker |
| 网络 | Docker bridge | macvlan，加 host shim |
| LAN 网关/DNS | 不需要 | ShellCrash LAN IP |
| Tailscale routes | 仅 `28.0.0.0/8` | `28.0.0.0/8` + exit-node 默认路由 |
| IPv4/IPv6 forwarding | 容器内启用 | 容器内启用 |
| fake-IP 与透明代理 | 启用 | 启用 |

## Traffic flow

基础配置：

```text
Tailnet client
  DNS -> Tailscale IPv4 of the shared namespace
  28.x -> advertised route 28.0.0.0/8
             |
             v
       tailscale0 -> ShellCrash firewall -> CrashCore -> TUN -> Internet
```

Linux 旁路由：

```text
LAN client (gateway + DNS = SHELLCRASH_IP) ----+
                                               |
Tailnet client -> 28.0.0.0/8 -----------------+-> macvlan/shared namespace
                                                   -> ShellCrash firewall
                                                   -> CrashCore / mihomo TUN
                                                   -> proxy group or DIRECT
```

ShellCrash 将 fake-IP 流量交给 CrashCore。Tailnet 源地址仍在 `100.64.0.0/10`，因此两种配置都必须在 TUI 的流量过滤自定义透明路由网段中加入该网段。

## Tailscale integration

- 基础配置中的 `TS_EXTRA_ARGS` 为空，只通告 `TS_ROUTES=28.0.0.0/8`。
- Linux 旁路由完整配置使用 `TS_ROUTER_EXTRA_ARGS`，默认包含 `--advertise-exit-node --snat-subnet-routes=true`。
- 两种配置都使用内核 TUN、IPv4/IPv6 forwarding 和 ShellCrash 防火墙；旁路由额外提供物理 LAN 路径。

## Linux host shim

macvlan 会隔离 parent interface 与 macvlan 子接口。Linux 宿主不能直接从 parent interface 访问 `SHELLCRASH_IP`，所以旁路由完整配置的 `host-shim` 在宿主网络命名空间创建同 parent 的 macvlan bridge 子接口，并添加精确路由：

```text
PARENT_IFACE
    |
    +-- macvlan mode bridge: HOST_SHIM_IFACE
          address HOST_SHIM_IP/32
          route SHELLCRASH_IP/32 via HOST_SHIM_IFACE
```

这同时提供两条可验证路径：宿主经 shim 访问 ShellCrash，容器经 `HOST_SHIM_IP` 访问宿主服务。`HOST_SHIM_IP` 必须是 LAN 中未占用且不在 `MACVLAN_IP_RANGE` 内的地址。

## Why no custom mihomo config

ShellCrash 在 `shellcrash/yamls/` 里保存订阅和自定义片段，在 `shellcrash/configs/` 里保存运行参数，并负责生成最终 `config.yaml`。这些内容全部由 TUI 或 ShellCrash 定时任务维护，仓库不提供独立的 mihomo 配置模板。

## Persistence

配置和程序文件通过 `./shellcrash:/etc/ShellCrash` 持久化。Tailscale 身份通过 `./tailscale-state:/var/lib/tailscale` 持久化；两者都不应提交到 Git。
