# singbox-build

定时构建最新稳定版 [sing-box](https://github.com/SagerNet/sing-box),在上游默认构建标签的基础上额外启用 `with_v2ray_api`(V2Ray gRPC 统计 API)。

Scheduled builds of the latest stable sing-box with the `with_v2ray_api` build tag enabled on top of the upstream default tags.

## 构建标签

```
with_gvisor,with_quic,with_dhcp,with_wireguard,with_utls,with_acme,with_clash_api,with_tailscale,with_v2ray_api
```

## 产物

每天检查一次上游最新 release,若本仓库尚未发布对应版本则自动构建并发布到 [Releases](../../releases):

- `sing-box-<version>-linux-amd64.tar.gz`
- `sing-box-<version>-linux-arm64.tar.gz`
- `sing-box-<version>-darwin-arm64.tar.gz`
- `checksums.sha256`

也可在 Actions 页面手动触发 workflow。
