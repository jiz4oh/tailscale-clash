# ShellCrash TUI 配置

本文适用于基础配置 `compose.yaml` 和 Linux 旁路由完整配置 `compose.router.yaml`。两种配置都启用透明代理；差异只影响网络入口和 Tailscale 通告范围。

进入 TUI：

```sh
docker exec -it shellcrash sh -l
```

首次使用建议按以下顺序配置：

## 1. 导入订阅或配置

主菜单选择配置文件管理（菜单 `6`），按 TUI 提示导入订阅链接或本地 `config.yaml`。ShellCrash 会把它保存到 `yamls/`。

## 2. 路由模式

主菜单选择功能设置（菜单 `2`），进入路由模式：

- 推荐 `Mix` 或 `Tun`，能同时代理 TCP 和 UDP。
- 有 TUN 设备时这两个模式才可选；Compose 已映射 `/dev/net/tun`。

## 3. DNS 模式

功能设置里的 DNS 菜单选择 `mix`。本项目 fake-IP range 固定为 `28.0.0.0/8`，不要改成其他网段，否则 Tailscale 通告会对不上。

## 4. 添加 Tailscale 网段

功能设置 -> 流量过滤 -> 自定义透明路由网段，输入：

```text
100.64.0.0/10
```

这会让来自 Tailnet 的 DNS 和 fake-IP 流量被 ShellCrash 劫持。基础配置的 DNS 服务地址使用共享网络命名空间的 Tailscale IPv4；Linux 旁路由的 LAN 客户端使用 `SHELLCRASH_IP`。

## 5. 防火墙后端

在支持的 Linux container runtime 中优先使用 `nftables`，并确认运行时提供对应的防火墙能力。仓库不提供平台专用的防火墙镜像或第三种 Compose 文件；如果 runtime 无法提供 TUN、转发或重定向能力，该平台不满足基础配置的前置条件。

## 6. 启动服务

回到主菜单启动服务。后续调整订阅、节点、规则、端口都直接在这个 TUI 里完成，配置持久化在 Compose 同级的 `shellcrash/` 目录。

## 7. 按配置检查 Tailscale

- 基础配置：批准 `28.0.0.0/8`，不启用 exit-node 通告。
- Linux 旁路由：批准 `28.0.0.0/8` 和 exit-node 默认路由。
