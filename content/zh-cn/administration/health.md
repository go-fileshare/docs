---
title: "健康检查与指标"
weight: 20
description: "metrics 块提供的 /healthz、/readyz 和 /metrics 端点，它们暴露的指标，以及应当针对什么设置告警。"
tags: [管理, 指标]
---

```hcl
metrics { listen = "127.0.0.1:9100" }   # 或 unix:///run/fileshare/metrics.sock
```

三个端点，由
[go-net-health/endpoint](https://github.com/go-net-health/endpoint) 在
专属监听器上提供，**绝不使用任何协议所服务的端口**，因此能够访问
WebDAV 的外部网络无法访问它们：

| | |
|---|---|
| `/healthz` | 进程有响应 |
| `/readyz` | 每个协议都已绑定，**并且**有一个世代正在提供服务。在启动、[应用变更]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}})或停止期间返回 `503`，原因写在响应正文中 |
| `/metrics` | Prometheus 文本格式 |

unix 套接字用绝对路径指定，即 `unix:///like/this`；其他形式
必须是 `host:port`，否则该块会被拒绝。

## 度量内容 {#what-is-measured}

| 指标 | 统计内容 |
|---|---|
| `fileshare_shares{origin}` | 正在提供服务的共享，`origin` = `config` 或 `api` |
| `fileshare_shares_disabled` | 通过管理 API 下线的共享 |
| `fileshare_generation` | 自启动以来已提供服务的共享列表数量 |
| `fileshare_connections_accepted_total{protocol}` | 已接受的连接 |
| `fileshare_connections_open{protocol}` | 当前打开的连接 |
| `fileshare_admin_changes_total{result}` | 管理变更，`applied` 或 `refused` |
| `fileshare_admin_requests_total{method,code}` | 管理 API 调用，按方法和 gRPC 状态统计 |
| `fileshare_directory_people` | 目录服务在最近一次读取时包含的人员数 |
| `fileshare_directory_reloads_total{result}` | [重新加载]({{< relref "/administration/reload.md" >}})：`unchanged`、`added`、`swapped`、`failed` |
| `fileshare_revocation_list_age_seconds{list}` | [撤销列表]({{< relref "/security/revocation-lists.md" >}})最近一份有效副本的存在时长；从未获取过时为 `-1` |
| `fileshare_revocation_list_fetch_failures_total{list}` | 该列表获取失败的次数 |
| `fileshare_ssf_last_heard_seconds` | 自[共享信号]({{< relref "/security/shared-signals.md" >}})发送方上次应答以来的秒数；从未应答时为 `-1` |
| `fileshare_ssf_revoked_subjects` | 身份提供方已撤销其先前凭据的人员，按当前保存的状态统计 |
| `fileshare_sftp_domain_grant_total{result}` | 被 [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}) 查验的 SFTP 证书：`granted`、`accepted_ungranted`、`refused_not_granted`、`refused_absent`、`refused_malformed` |
| `fileshare_start_time_seconds` | 此进程的启动时间 |

此外还有 Go 运行时和构建信息。撤销与共享信号相关的指标族
只有在配置中包含列表或 `ssf` 块时才会出现，域授权指标族
只有在设置了 `ssh_domains` 时才会出现，管理请求指标族
只有在存在 `admin` 块时才会出现。

{{< callout type="error" >}}
**没有任何指标会指明某个共享或某个人**

对于某人无权使用的共享，WebDAV 返回 **404**，以免确认其存在，
而一次指标抓取也不得确认这一点。标签只有
协议、来源、结果和方法名，绝不包含共享或人员。
{{< /callout >}}

## 应当针对什么设置告警 {#what-to-alert-on}

撤销检查[以拒绝为默认]({{< relref "/security/_index.md#every-one-of-them-fails-closed" >}})：
列表过旧，或长时间没有收到发送方的消息，服务器就会
**拒绝**它所管辖的凭据。请在此之前告警，而不是之后：

- `fileshare_revocation_list_age_seconds` 接近 `ssh_krl_max_age` 或
  `crl_max_age`（默认 1h）；
- `fileshare_ssf_last_heard_seconds` 接近 `ssf` 块的 `max_age`
  （默认 10m）；
- `fileshare_directory_reloads_total{result="failed"}` 持续上升：目录服务
  无法读取，因此[不会有任何变化]({{< relref "/administration/reload.md" >}})，包括移除操作。

{{< callout type="info" >}}
**`--isolate`**

`metrics` 块目前不能与 `--isolate` 同时使用；这种组合
会被拒绝。
{{< /callout >}}
