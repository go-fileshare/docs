---
title: "管理 API"
linkTitle: "管理"
weight: 30
description: "在服务器运行期间创建、修改、停用和授权共享的 gRPC 管理 API，它在哪里监听，以及它拒绝什么。"
tags: [管理, 管理 API, grpc]
---

自 v0.8.0 起，共享可以在服务器运行期间发生变化。`admin` 块
会启用一个 gRPC 服务，无需重启，也无需编辑文件，即可创建、修改共享并使其下线。

```hcl
admin {
  listen       = "unix:///run/fileshare/admin.sock"   # 权限设为 0600
  state_file   = "/var/lib/fileshare/shares.json"
  source_roots = ["/srv/images", "/data"]
}
```

未编写该块时，不会有任何监听。

## 它能做什么 {#what-it-does}

该服务是
[`fileshare.admin.v1.AdminService`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto)：

| | |
|---|---|
| `CreateShare`、`UpdateShare`、`DeleteShare` | 根据映像、目录或（自 v0.21.0 起）卷定义共享；修改 `read_only` 或 `protocols`（其来源不能更改：请删除后另行创建） |
| `DisableShare`、`EnableShare` | 使共享下线，以及重新上线 |
| `Grant`、`Revoke` | 授予某个主体读取或写入权限，或收回该权限 |
| `ListShares`、`GetShare` | 所有共享，包括它们提供的内容、所用的协议，以及为何不通过其他协议提供 |
| `ListUsers`、`ListGroups` | 授权可以指明哪些对象；每个用户及其凭据所能应答的协议 |
| `ReloadDirectory` | 立即重新读取人员信息，参见[重新读取目录服务]({{< relref "/administration/reload.md" >}}) |
| `GetServerInfo` | 名称、版本、启动时间、世代和监听器 |
| `ListParents`、`CreateVolume`、`ResizeVolume`、`SnapshotVolume`、`DeleteVolume`、`GetVolume`、`ListVolumes` | 通过特权配给程序创建的存储，以及由其提供的共享，参见[卷]({{< relref "/administration/volumes.md" >}})（自 v0.21.0 起） |

授权可以指明一个**用户**、一个**组**（配置文件中的 `@group`）、一个
**`oidc:groups:` 值**或一个 **`oidc:user:` 名称**，即
配置文件所用的词汇，参见
[由身份提供方指明的人员]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})。

`grpc.health.v1` 在同一个监听器上应答。块中设置 `reflection = true`
会启用 gRPC 服务器反射，供 `grpcurl` 使用。

### 每次变更都会说明服务端做了什么 {#every-change-says-what-serving-it-did}

变更以 `Applied` 消息作为应答：

```proto
message Applied {
  uint64 generation = 1;          // the generation now being served
  uint64 connections_closed = 2;  // how many the previous one had open, and closed
}
```

因此每个 RPC 都有自己的响应消息，而不是直接返回共享本身：
`.proto` 遵循 [buf](https://buf.build) 的 `STANDARD` lint 规则，
由 fileshare CI 中的 `buf lint` 检查，而不是 AIP-131 的
“返回资源”，由这层包装来携带 `Applied`。

## 它在哪里监听，谁可以调用它 {#where-it-listens-and-who-may-call-it}

{{< callout type="error" >}}
**通过 TCP 时，要么使用双向 TLS，要么不可用**

`tls_cert_file`、`tls_key_file` 和 `client_ca_file`，三者缺一不可，
**回环地址也不例外**：任何本地用户都可以访问回环地址，而这个 API
决定谁可以读取谁的文件。unix 套接字的权限设为 **0600**，其
权限即是访问控制。
{{< /callout >}}

```hcl
admin {
  listen         = "127.0.0.1:7443"
  tls_cert_file  = "/etc/fileshare/admin/server.pem"
  tls_key_file   = "/etc/fileshare/admin/server.key"
  client_ca_file = "/etc/fileshare/admin/clients.pem"
  state_file     = "/var/lib/fileshare/shares.json"
  source_roots   = ["/srv/images"]
}
```

监听器是
[grpc-transports/control](https://github.com/grpc-transports/control)。每次
变更都会记录操作者：客户端证书的 CN，或套接字
对端的 uid。

## 通过 HTTPS，用于 OIDC 令牌 {#over-https-for-oidc-tokens}

自 v0.28.0 起，当 `admin` 块包含 `web` 块时，同一套 API（相同的状态、相同的审计）
也通过 HTTPS 提供：**Connect、gRPC-Web 和
gRPC 共用一个监听器**，使用 [`tls` 块]({{< relref "/security/tls.md" >}})的
证书。Web 和原生界面与之通信的正是它。

```hcl
admin {
  listen     = "unix:///run/fileshare/admin.sock"
  state_file = "/var/lib/fileshare/shares.json"
  web {
    listen = "0.0.0.0:8443"
    issuer "https://login.example.org" {
      audience = "fileshare-a"            # 在该提供方处，本服务器的名称
      groups   = ["fileshare-admins"]
    }
    issuer "https://login.partner.example.org" {
      audience = "fileshare-a.partner"
      subjects = ["5b0c…"]                # 一个人由 (issuer, sub) 标识
    }
  }
}
```

| 字段 | |
|---|---|
| `listen` | TCP 地址；没有 `tls` 块时会被拒绝，因为持有者令牌与密码等效 |
| `issuer "<url>"` | 每个身份提供方一个块，即其 `iss`；重复给出会被拒绝 |
| `audience` | 令牌必须指明的 `aud`：**本服务器**在该提供方处的名称；必填 |
| `subjects`、`groups` | 谁可以调用：一个 `sub`，或组声明中的一个组；两者至少给出一个 |
| `jwks_url` | 跳过 OIDC 发现，用于不发布该信息的提供方 |
| `groups_claim` | 携带组信息的声明；默认为 `groups` |

**谁会得到应答。** 当某个颁发者的验证器接受其
持有者令牌（签名、`iss`、指明本服务器的 `aud`、时间）**并且**
该颁发者的块指明了令牌的主体或其所属的某个组时，调用才会得到应答。在一个
颁发者处列出的主体，在另一个颁发者处什么都不是：一个人由 (颁发者, `sub`) 标识。
检查仅依据标头进行，此时消息的任何一个字节都尚未被解码。

**每个服务器一个受众。** 发给多个服务器的令牌可能被其中任何一个
重放给其他服务器（RFC 8707、RFC 9068），因此每个服务器都有
自己的受众，同时驱动多个服务器的界面需要为每个服务器申请一个令牌。

**被拒绝的调用方能得知什么。** 只有“已拒绝”，别无其他。原因，以及
所做的每一项变更，都会以 `oidc=<issuer> <sub>` 的形式写入审计输出。

**没有 CORS。** 浏览器不会直接调用它：它们通过界面自己的
服务器访问，由该服务器持有令牌。请求大小限制为 1 MiB，连接的
每个阶段都有超时。每个颁发者都会在**启动时**被联系，以
读取其密钥；无法联系的颁发者会使启动中止，并指明该颁发者。

### 在浏览器中：控制台 {#in-a-browser-the-console}

[go-fileshare/console](https://github.com/go-fileshare/console) 是调用此 API 的
管理控制台：共享、详情、授权、创建、启用、停用、
删除、授予和撤销，以及[卷]({{< relref "/administration/volumes.md" >}})
（创建、调整大小、快照、共享、删除，绝不销毁数据），可同时覆盖所有
服务器。它在浏览器中运行，位于
[go-fileshare/portal](https://github.com/go-fileshare/portal) 之后，后者通过 OpenID Connect
让人员登录并保管其令牌：每个服务器一个，仅发给该
服务器。浏览器从不持有令牌。

```hcl
# portal.hcl
server "paris" {
  url      = "https://fs-paris.example.org:8443"   # 此 web 监听器
  issuer   = "https://login.example.org"
  resource = "https://fs-paris.example.org/"       # 上文中本服务器的受众
}
ui "console" { dir = "/usr/share/fileshare-console" }
```

同一个控制台也可以**以原生方式**运行，无需门户。它通过
系统浏览器和回环监听器登录，作为公共客户端
（[go-authn/rp](https://github.com/go-authn/rp)，RFC 8252），并使用仅限该服务器的
令牌调用每个服务器。服务器列在其自己的
`console.hcl` 中，其 `url` 和 `resource` 与此处相同。

**文件资源管理器** [go-fileshare/explorer](https://github.com/go-fileshare/explorer)
在同一个门户之后运行：浏览、上传、下载、重命名、复制和删除，
覆盖一个人有权使用的每个共享。它通过
门户（`/dav/<server>/...`）并使用该服务器的令牌访问各服务器的 WebDAV，门户对
返回的内容一律**去除可执行性**后再提供
（`Content-Security-Policy: sandbox`，作为附件）：
共享中的页面绝不会以门户的身份运行。它需要 fileshare **v0.29.0** 或
更高版本，其 [WebDAV]({{< relref "/protocols/webdav.md#the-root-and-uploads-in-pieces" >}})
在根目录列出各共享，并支持分块上传。

```hcl
# portal.hcl，同一个服务器
server "paris" {
  # ... 同上，以及它的 WebDAV：
  dav_url = "https://fs-paris.example.org"
}
ui "explorer" { dir = "/usr/share/fileshare-explorer" }
```

资源管理器同样可以**以原生方式**运行（explorer v0.2.0），无需门户，与
控制台一样：它通过系统浏览器作为公共客户端登录，并使用仅限该服务器的令牌
调用每个服务器的 WebDAV。其服务器列在 `explorer.hcl` 中，每个都带有
`dav_url`（仅限 https）和其 `oidc` 块的
`resource`。两种原生客户端在携带令牌时都不会跟随重定向：Go 的 HTTP 客户端
在重定向到同一主机时，无论使用何种协议方案都会保留 `Authorization`，而配置错误的代理若以 `307` 指向纯 `http` 地址，
就会使令牌以明文发出。

**失败的上传可以续传**（explorer v0.3.0）。部分文件以
原文件命名，即其名称、大小和修改时间，因此再次上传同一文件时，会从
部分文件的末尾继续，前提是回读该部分文件最后 64 KiB 后发现
与原文件一致。不同的文件绝不会被续传。网络或服务器故障会保留
部分文件；被拒绝（403，共享已满：507）则会将其删除。

每个服务器的 `audience` 就是门户为其设置的 `resource`。使用该 API 并不需要自行编写
控制台：任何持有该服务器令牌的 Connect、gRPC-Web 或 gRPC 客户端
都可以。

## 它会碰什么，不会碰什么 {#what-it-will-and-will-not-touch}

**它管理自己创建的共享。** 配置中写明的共享会
被列出，字段相同，而对其定义的更改会被拒绝，并返回
`FAILED_PRECONDITION`：一个共享在两处定义，是没有人愿意回答的问题，
而该共享的定义以文件为准。API 创建的共享保存在
`state_file` 中，以原子方式写入，并在下次启动时重新提供服务，这
就是没有 `state_file` 的块会被拒绝的原因。

**来源必须位于 `source_roots` 之下**，指的是解析之后的路径（跟随链接、去除 `..`），
保存的也是解析后的路径。没有 `source_roots` 时，API
**根本无法创建共享**：进程可以读取 `/dev` 和 `/etc`，
而没有人打算把它们交给调用方。每次启动都会再次检查，
`state_file` 中某个共享若不再位于任何根目录之下，启动会中止，
并指明该共享。

**每个共享至少有一项授权。** 没有任何授权的共享对任何通过身份验证的人开放。
文件可以有意这样写；API 调用不应因遗漏而这样写。
因此 `CreateShare` 需要一项授权，撤销最后一项授权也会
被拒绝，请改为删除该共享。

**协议能够承载的名称。** 共享名称最长 80 个字符，不以空格开头或结尾，
不是 `.` 或 `..`，不含不可打印字符，也不含 `:*?"<>|{}%` 中的任何字符。
主体同样不含不可打印字符，一个共享最多接受 1000 项授权，
远在此之前就该由组来承担。
以上每一种情况在 v0.14.0 之前都会被接受并写入 `state_file`，随后
在每次启动时都致命。

**任何共享都不得包含**配置、状态文件，或它们所指明的机密：
能写入其中的人就能改写谁可以做什么。自 v0.17.0 起
这包括 `authorized_keys_file`、`users` 块的 `dsn_file` 和
`bind_password_file`、`ssf` 块的 `ca_file`，以及 DSN 所指明的 sqlite 数据库：
能把自己的密钥加入他人 `authorized_keys` 的写入者
就能以该人的身份登录。**共享也不得包含另一个共享**（参见
[目录]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})）。

**变更会像配置一样被检查，先打开，再记录，然后
提供服务。** 服务器无法满足的变更，例如无法打开的映像、
目录服务中没有人使用的名称，会被拒绝，已提供的服务和已写入的内容
保持原样。

## 停用共享 {#disabling-a-share}

**停用相当于 Samba 的 `available = no`**：共享仍保持定义，所有
连接尝试都会失败；其已打开的连接被关闭，其映像或
目录被**释放**，因此文件可以在它下线期间被替换。
`EnableShare` 会让它重新提供服务，若其来源已无法
打开则会被拒绝。`CreateShare` 也可以直接创建一个已停用的共享。

与其他所有变更不同，它也适用于**配置中**的共享，
因为使共享下线是一种操作，而不是一种定义，并且它在重启后仍然有效：
它保存在状态文件中。[`fileshare check`]({{< relref "/configuration/check.md" >}})
会列出处于下线状态的共享。

## 一次变更就是一个新世代 {#a-change-is-a-new-generation}

{{< callout type="error" >}}
**一次变更会重启协议服务器，并关闭它们的连接**

SMB 在每次树连接时检查谁可以连接，SFTP 在每次登录时构建
一个人的目录树，因此在撤销之后仍然存活的会话
会保留刚被收回的访问权限。“已撤销，但已连接的人除外”
不是撤销。
{{< /callout >}}

因此变更并不是应用*到*正在运行的服务器上：它们会被根据新列表构建的
新服务器所取代，这就是一个**世代**，其方式与重启相同，
但不释放端口。这些库别无选择：
go-filesystems/smb 有 `Share` 而没有 `Unshare`，nfs 有 `Export` 而没有
`Unexport`。

端口保持绑定，因此在变更期间连接的客户端会**等待**，而不是
被拒绝。共享未发生变化的映像保留其驱动，不会被第二次
打开。客户端会重新连接；这是代价，每次变更都要付出，
这就是 API 每次调用只应用一项变更，而不是每个字段一项变更的原因。

## 限制 {#limits}

- `--isolate` 目前不能与 `admin` 块同时使用，也不能与 `metrics` 块同时使用：
  没有任何单一进程可以让变更应用于其上。这种组合
  会被拒绝。
- 使用 `-tags nogrpc` 构建的二进制文件没有管理 API，包含
  `admin` 块的配置会被**拒绝**，而不是在没有该 API 的情况下提供服务。该标签可节省
  11.7 MB；参见[只构建所需内容]({{< relref "/operations/build-tags.md#the-admin-api-and-nogrpc" >}})。

## 修改 `.proto` {#changing-the-proto}

`proto/` 下的 Go 代码是生成的并已提交，因此 `go install` 不需要
`protoc`。修改 `.proto` 之后（管理 API 的，或配给程序的
`proto/fileshare/provision/v1/provision.proto`），请使用 fileshare CI 所固定的
版本重新生成两者（protoc 34.1、protoc-gen-go v1.36.12、
protoc-gen-go-grpc v1.6.2）。CI 会重新生成它们，当已提交的代码
不同，或生成的代码未提交时，CI 会失败：

```sh
protoc -I proto --go_out=. --go_opt=module=github.com/go-fileshare/fileshare \
  --go-grpc_out=. --go-grpc_opt=module=github.com/go-fileshare/fileshare \
  proto/fileshare/admin/v1/admin.proto proto/fileshare/provision/v1/provision.proto
```
