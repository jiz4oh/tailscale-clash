# Linux 旁路由说明

## 启动

旁路由配置只适用于原生 Linux 和 rootful Docker：

```sh
docker compose -f compose.router.yaml up -d
```

`.env` 需要填写：

- `PARENT_IFACE`：承载 LAN 的物理网卡或 Linux bridge parent。
- `LAN_SUBNET`、`GATEWAY_IP`、`SHELLCRASH_IP`、`MACVLAN_IP_RANGE`：macvlan 网络参数。
- `HOST_SHIM_IP`：LAN 中未占用的宿主 shim 地址；填写纯 IP，服务会配置为 `/32`。
- `HOST_SHIM_IFACE`：shim 接口名；未填写时使用 `shellcrash-host-shim`。

`HOST_SHIM_IP` 应避开 `MACVLAN_IP_RANGE`，并在启动前确认没有被局域网设备使用。

## 为什么需要 shim

macvlan 会隔离 parent interface 与 macvlan 子接口。旁路由容器虽然拥有 `SHELLCRASH_IP`，Linux 宿主却不能直接从 parent interface 访问该地址。`host-shim` 在宿主网络命名空间创建一个同 parent 的 macvlan bridge 子接口，并添加精确路由：

```text
PARENT_IFACE
    |
    +-- macvlan mode bridge: HOST_SHIM_IFACE
          address HOST_SHIM_IP/32
          route SHELLCRASH_IP/32 via HOST_SHIM_IFACE
```

因此：

- 宿主机访问 `SHELLCRASH_IP` 走 shim 的 `/32` 路由。
- 容器访问宿主机使用 `HOST_SHIM_IP`，不使用 parent interface 的原始地址。

## 检查

```sh
ip -o link show <HOST_SHIM_IFACE>
ip -4 addr show <HOST_SHIM_IFACE>
ip route get <SHELLCRASH_IP>
curl -x http://<SHELLCRASH_IP>:37890 -o /dev/null -w '%{http_code}\n' https://www.gstatic.com/generate_204
docker exec shellcrash-host-shim ip -d link show <HOST_SHIM_IFACE>
docker exec shellcrash nc -z -w 3 <HOST_SHIM_IP> <HOST_PORT>
```

`ip -d link show` 必须显示 `macvlan mode bridge`。host shim 使用官方 `tailscale/tailscale` 镜像提供完整的 `iproute2`；在受限的 BusyBox `ip` 环境中，同一命令曾创建出 `vepa` 模式，导致宿主与容器互访失败。shim 容器重启时会清理并重新创建接口和路由。

## 平台边界

普通 Docker bridge 不存在这条 macvlan parent 隔离，因此 Tailnet-only 配置不需要 shim。WSL2、Docker Desktop for macOS 和其他非原生 Linux Docker runtime 使用基础配置；只有能够操作真实 parent interface、创建 macvlan 并授予 `NET_ADMIN` 的原生 Linux runtime 使用旁路由配置。
