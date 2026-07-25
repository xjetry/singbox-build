# singbox-build

定时构建以下项目的最新稳定版与最新预发布版，并在各自的上游构建标签基础上额外启用 `with_v2ray_api`（V2Ray gRPC 统计 API）：

- [sing-box](https://github.com/SagerNet/sing-box)
- [mbox](https://github.com/enfein/mbox)

Scheduled Linux builds of the latest stable and pre-release versions of sing-box and mbox, with `with_v2ray_api` enabled on top of each source's default build tags.

## 构建配置

sing-box 使用本仓库维护的上游默认标签及 `with_v2ray_api`：

```
with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale,with_v2ray_api
```

mbox 使用对应版本的 `release/DEFAULT_BUILD_TAGS_OTHERS` 和 `release/LDFLAGS`，并在构建标签中追加 `with_v2ray_api`。

## 产物

两个独立 workflow 每天检查一次对应上游的最新 release 与最新 pre-release。尚未发布的版本会自动构建并发布到 [Releases](../../releases)，预发布版本会带 pre-release 标记。

sing-box 保留上游 tag，例如 `v1.13.0`：

- `sing-box-<version>-linux-amd64.tar.gz`
- `sing-box-<version>-linux-arm64.tar.gz`

mbox 使用带来源前缀的 tag，例如 `mbox-v1.13.13`：

- `mbox-<version>-linux-amd64.tar.gz`
- `mbox-<version>-linux-arm64.tar.gz`

归档内的可执行文件仍命名为 `sing-box`。

每个 Release 均包含对应两个归档的 `checksums.sha256`。也可在 Actions 页面分别手动触发 workflow。
