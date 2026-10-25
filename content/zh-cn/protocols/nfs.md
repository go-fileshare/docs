---
title: "NFS：什么都没有，除非有 Kerberos 或证书"
linkTitle: "NFS：什么都没有，除非有 Kerberos 或证书"
weight: 50
description: "NFSv3 自身不对任何人进行身份验证；Kerberos 或客户端证书如何让它提供受限共享，以及 TLS 增加了什么。"
tags: [协议, nfs, kerberos]
---

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

{{< callout type="error" >}}
**NFSv3 自身不对任何人进行身份验证**

`AUTH_UNIX` 是一种**声明**：客户端说“uid 501”，而网络无法
反驳。也没有任何加密。
{{< /callout >}}

因此，**指明了谁可以使用的共享不会通过 NFS 导出**。拒绝信息
会在启动时以及 [`check`]({{< relref "/configuration/check.md" >}})中打印，并附上
原因：

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

没有 `allow` 也没有 `writers` 的共享（任何连接者都可读写）会
通过 NFS 导出，因为那里没有协议无法
遵守的规则。

## 曾经永久存在的拒绝 {#the-refusal-that-used-to-be-permanent}

`kerberos` 块让 NFS 能够区分不同的人，受限共享随后
便可像在其他地方一样通过它提供服务：

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5` 携带的是由**票据证明**的主体，而不是
客户端自行断言的 uid。

{{< callout type="info" >}}
**比较的是领域，而不只是 `@` 之前的名称**

两个领域各自都可以有一个 `alice`，而其中只有一个是你的。
{{< /callout >}}

## 通过 TLS：证明的是机器，而不是个人 {#over-tls-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

自 v0.9.0 起，配合 `tls = true` 和 [`tls` 块]({{< relref "/security/tls.md" >}})，NFS 可通过
RPC-with-TLS（RFC 9289）提供服务。它提供加密；配合 `client_ca_file` 时，它会要求
客户端出示由该颁发机构签发的证书，由此得知是哪台**主机**在挂载。
其中的 uid 仍然是 `AUTH_SYS` 所声明的那个，因此受限共享
通过它**仍然会被拒绝**。TLS 是可选提供的，而不是强制的：从不
请求 TLS 的客户端仍然可以使用开放的共享。

## 凭证书识别个人 {#a-person-from-their-certificate}

自 v0.12.0 起，证书可以指明**个人**，就像 go-authn/bridge 在
身份提供方登录之后签发的那样，此后指明了人员的共享
无需 kerberos 就可以通过 NFS 提供服务：

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # 或 crl_file：二者必须给出其一
}
```

{{< callout type="error" >}}
**Linux 客户端的证书属于一次挂载**

该挂载的每个用户都以证书所指明的那个人的身份行事。对于
一个人使用的工作站，这是正确的；对于多人登录的机器则是错误的，
请在那里使用 `sec=krb5`。
{{< /callout >}}

证书携带什么、必需的 CRL，以及用真实 Linux 客户端实测
得到的结果（明文的 `MNT`、`tlshd` 的陷阱），见
[NFS 与基于证书的身份]({{< relref "/security/nfs-certificates.md" >}})。
