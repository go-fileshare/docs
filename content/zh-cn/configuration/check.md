---
title: "check：在重启人们正在使用的服务之前"
linkTitle: "check，重启之前"
weight: 30
description: "fileshare check 在配置投入服务之前会输出什么，以及如何解读。"
tags: [配置, check]
---

```
$ fileshare check /etc/fileshare.d
ATTIC

SHARE    IMAGE             FILESYSTEM  WHO MAY CONNECT           WHO MAY WRITE             SMB  WEBDAV  NFS  SFTP
photos   /srv/photos.img   fat32       alice and bob             alice                     yes  yes     NO   yes
scratch  /srv/scratch.img  ext4        anyone who authenticates  anyone who authenticates  yes  yes     yes  yes

photos is not served over nfs: it is restricted to alice and bob, and NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client makes about itself and the wire cannot disagree with it. A kerberos block lifts this: sec=krb5 carries a principal a ticket proves; so does identity = "certificate" on the nfs serve block: RPC-over-TLS with a client certificate naming the person

USER   FROM                      AUTHENTICATES WITH                         SMB  WEBDAV  SFTP
alice  the configuration file    a password and 1 key from /etc/…/alice.pw  yes  yes     yes
bob    the configuration file    a password from /etc/…/bob.pw              yes  yes     -
dora   ldaps://ldap.example.org  a password check and an NT hash            yes  yes     -
eli    ldaps://ldap.example.org  a password check                           -    yes     -

this configuration can be served
```

每个映像都会按服务器打开它的方式打开，然后再关闭；不会提供
任何服务。对于没有声明 `read_only = true` 的共享，这是**读写**方式
打开，只有设备始终以只读且独占方式打开，因此正在运行的服务器所持有的设备
在这里可能会因繁忙而被拒绝。

它输出的是**凭据来自哪里，而绝不输出其内容**，而按人员划分的列
正是 [`Identity.Can`](https://github.com/go-authn/directory) 存在的原因：eli 与
dora 在同一个组中，但 SMB 仍然无法为 eli 提供服务，因为 LDAP 持有 eli 的
密码，并且不会把它交给任何人。

## 解读各列 {#reading-the-columns}

| | |
|---|---|
| 文件系统带星号 | 驱动是由共享**断言**的，而不是识别出来的，参见[共享]({{< relref "/configuration/shares.md" >}}) |
| 协议下方为 `NO` | 该共享未在该协议上导出，原因打印在表格下方 |
| 协议下方为 `-`（按共享） | 该共享的 `protocols` 将该协议排除在外 |
| 协议下方为 `-`（按用户） | 此人的来源无法证明该协议所需的内容，参见[身份]({{< relref "/configuration/identity.md" >}}) |

## 它还会说明什么 {#what-else-it-says}

在表格之后，`check` 会在适用时说明，人们在依赖该配置之前
需要知道什么：

- 对于带有 `oidc` 块的配置，说明其令牌仅在 WebDAV 上被接受，以及
  提供方的名称是否为本地名称（`local_names`），参见
  [身份]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})；
- 按协议说明哪些是**加密的**、哪些是**有意明文的**，
  参见 [TLS]({{< relref "/security/tls.md#what-check-says" >}})；
- **撤销列表**来自哪里，以及无法获取时
  会发生什么，参见[什么撤销什么]({{< relref "/security/_index.md" >}})；
- 哪些共享通过管理 API 的 `DisableShare` 被**下线**，
  因而不提供服务，参见[管理 API]({{< relref "/administration/_index.md#disabling-a-share" >}})。

对于带有 KRL、基于证书的 NFS 身份和共享信号的服务器，这一部分如下：

```
sftp: the provider's SSH certificates are checked against the KRL at https://bridge.example.org/ssh/krl, at login and on every operation; while it is unknown or older than 1h0m0s they are refused

federated people: revocations come from the shared signals transmitter https://bridge.example.org (CAEP session-revoked, polled); whatever the provider issued them before a revocation is refused -- tokens, SSH and OpenPubkey certificates, NFS certificates -- and open sessions stop. Refused while it is not heard from within 10m0s; revocations kept 192h0m0s in /var/lib/fileshare/revocations.json

nfs: identities come from client certificates signed by /etc/fileshare/bridge-x509-ca.pem, revoked by the CRL at https://bridge.example.org/x509/crl (refused while it is unknown or older than 1h0m0s).
     ⛔ a Linux client's certificate belongs to a MOUNT: every user of that mount is the person it names
```

`check` 不会获取这些列表：它说明它们来自哪里，以及缺失时
会造成什么后果。而身份提供方的发现文档和密钥则**会**被读取：
颁发者无法访问的 `oidc` 块会被 `check` 拒绝。
