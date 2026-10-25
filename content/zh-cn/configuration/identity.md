---
title: "用户、组和目录服务"
weight: 20
description: "人员从何而来（user 和 group 块、SQL、LDAP、身份提供方），以及每种来源让这些人员可以使用什么。"
tags: [配置, 身份, oidc, ldap, sql]
---

`user` 块就是一个家庭的全部目录服务。如果一个站点的人员
已经存在于数据库或 LDAP 中，就不应把这些人员复制到另一个
会过时的地方，因此 `users` 块会就地读取这些人员。

```hcl
users "sql" {
  driver   = "postgres"                 # 或 sqlite，或 mysql
  dsn_file = "/etc/fileshare/dsn"       # DSN 含有密码：所以放在文件中
  users    = "select login, null, nt_hash, ssh_keys from staff"
  groups   = "select team, member from team_members"
}

users "ldap" {
  url                = "ldaps://ldap.example.org"
  base_dn            = "ou=people,dc=example,dc=org"
  bind_dn            = "cn=reader,dc=example,dc=org"
  bind_password_file = "/etc/fileshare/bind.pw"
}
```

**查询由你自己编写**，因为一个站点的人员已经有了该站点自己的
结构；在这里另行设计一套模式，就意味着要把这些人员复制到第二套中。
人员查询的各列是**按位置**读取的，而不是按名称：先是名称，然后依次是
密码、NT 哈希、SSH 密钥（每行一个）和 TOTP 密钥中的任意几项，
顺序如此。NULL 表示此人没有该凭据，这就是
示例中密码一列选择 `null` 的原因。密码列被视为
密码本身：这里没有用于哈希密码的设置。

LDAP 一侧读取支持 Samba 的目录服务已经发布的内容，
即 `sambaNTPassword`、`sshPublicKey`、`memberUid`。人员和组
位于何处，以及用哪些属性命名它们，都是可配置的：`user_filter`
（默认 `(|(objectClass=posixAccount)(objectClass=account)(objectClass=uidObject))`）、
`user_attribute`（`uid`）、`group_base_dn`（`base_dn`）、`group_filter`
（`(|(objectClass=posixGroup)(objectClass=groupOfNames))`）、`group_attribute`
（`cn`）、`group_member_attribute`（默认同时为 `memberUid` 和 `member`），
以及没有默认值的 `totp_attribute`。`sambaNTPassword` 和
`sshPublicKey` 就按这些名称读取。`member` 的值是一个 DN，只有当它是
人员搜索所返回的某人的 DN 时才会计入：位于目录树其他位置的
`uid=alice` 并不是 alice。这些默认值（自
v0.30.0 起）读取 go-authn/authnd v0.11.0 所发布的内容；此前它们仅为
`(objectClass=posixAccount)` 和 `(objectClass=posixGroup)`，而面对
机构自己的目录服务，它们现在会读取更多条目。

{{< callout type="error" >}}
**不向另一台机器发起明文绑定（自 v0.19.0 起）**

`users "ldap"` 块会拒绝指向另一台机器的 `ldap://`，除非
设置 `start_tls = true`：绑定所检查的每个密码都会以明文
穿过网络。`ldaps://` 会被接受，指向回环
地址的 `ldap://` 和 `ldapi://` 也会被接受，因为途中没有任何人。
没有任何开关可以关闭这一点（go-authn/directory v0.10.0）。
{{< /callout >}}

## 顺序，以及唯一的例外 {#order-and-the-one-exception}

各来源**按书写的顺序**被询问，第一个认识某个名称的来源就拥有它。
`user` 和 `group` 块排在最前，因此在本地写明的服务
账户不会被 LDAP 中的同名人员覆盖。

**组是例外**：组的成员是所有来源的**并集**，
因为一个团队可能既有文件中的人员，也有数据库中的人员。

```hcl
share "photos" {
  allow   = ["@engineers", "alice"]
  writers = ["@owners"]
}
```

{{< callout type="info" >}}
**在启动时展开，并在每次重新加载时再次展开**

自 v0.10.0 起，`users` 块（SQL、LDAP）会在每个
`reload = "5m"` 间隔、收到 `SIGHUP` 时以及调用管理 API 的 `ReloadDirectory` 时重新读取；
LDAP 中变化的成员关系不再需要等待重启。重新加载
哪些内容原地应用、哪些内容需要重启，以及为什么清空的组**不会**把它的
共享开放给所有人，请参见
[重新读取目录服务]({{< relref "/administration/reload.md" >}})。目录服务
仍然不会在每次连接时被询问：那是另一种设计，而这里
不是。
{{< /callout >}}

## 来源能够证明什么，以及每个协议需要什么 {#what-a-source-can-prove-and-what-each-protocol-needs}

否则站点只能在挂载时才发现这部分内容，所以 [`check`]({{< relref "/configuration/check.md" >}})
会首先说明：

| 来源具有 | SMB | S3 | WebDAV | SFTP |
|---|---|---|---|---|
| 密码（文件，或明文列） | 是 | 是 | 是 | — |
| NT 哈希（`sambaNTPassword`、`nt_hash`） | 是 | **否** | — | — |
| 仅有绑定（LDAP） | **否** | **否** | 是 | — |
| 公钥，或受信任的 CA | — | — | — | 是 |

{{< callout type="error" >}}
**NTLMv2 需要密码或其 MD4，其他任何东西都不行**

客户端从不向 SMB 服务器发送密码，它发送的是由密码
计算出的证明，因此只能*检查*密码的目录服务无法
应答 SMB，无论这种检查有多好。这是协议的属性，
而不是本程序的限制，再多的配置也无法
改变它。WebDAV 只问“这个密码对不对”，绑定可以回答这个问题。
{{< /callout >}}

目录服务指明了某人却无法为其证明任何内容，这是合理的，
就像机密存放在别处的列表一样，而 `check` 会用一行说明这一点，
而不是让人自己去发现。

## 由身份提供方而非本文件指明的人员 {#people-the-identity-provider-names-not-this-file}

联盟（比如通过
[go-authn/bridge](https://github.com/go-authn/bridge) 接入的 RENATER）知道谁属于某个
项目，共享可以向**它**询问，而不必复制名单。使用
[`oidc` 块]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}})：

```hcl
share "photos" {
  image   = "/srv/photos.img"
  allow   = ["oidc:groups:urn:mace:univ-example.fr:photos", "oidc:user:bob@univ-example.fr", "alice"]
  writers = ["oidc:groups:urn:mace:univ-example.fr:photos"]
}
```

`oidc:groups:<value>` 指其令牌的组声明中包含该值的人；
`oidc:user:<name>` 指令牌所指明的人。这种写法沿用 opkssh 的，
这样同一套词汇既说明谁能进入 shell，也说明谁能访问共享。
规则所指明的人，就提供方而言是本服务器所认识的（开放的
共享也是如此），而没有任何规则指明的人仍然是陌生人。同样的规则
适用于 [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})，
只是组随证书而不是令牌传递。

{{< callout type="error" >}}
**规则只针对提供方的人员**

`oidc:user:bob` 是提供方所担保的那位 bob；一个名为
bob、带有密码的本地账户是另一个人，不匹配任何规则。没有
`oidc` 块的规则、格式错误的规则，或者可以写入却不被允许
连接的规则，会在**启动时被拒绝**。
{{< /callout >}}

{{< callout type="error" >}}
**除非明确说明，提供方的 `alice` 不是本地的 `alice`（自 v0.20.0 起）**

`allow` 或 `writers` 中的普通名称是本地账户；令牌或
提供方证书只能通过共享的 `oidc:` 规则访问。提供方的名称
就是其本地名称的站点，需要在 `oidc` 块中写 `local_names = true`，
并且应同时设置 `domains`，或者从提供方所控制的
声明中取得名称：否则，在允许人员自行选择
`preferred_username` 的提供方，任何注册的人都会获得
某个本地账户的共享，包括写入权限。这在 v0.20.0 之前是默认行为
（安全审计 F4）。因缺少 `local_names` 而被拒绝的联盟名称
会连同应添加的那一行一起记入日志，而 `fileshare check` 会说明
生效的是哪条规则。

NFS 客户端证书（`identity = "certificate"`）是例外：
它们的 CA 正是 `client_ca_file` 为此而固定的那个，而普通
名称是 NFS 共享指明某人的唯一方式，因此无论 `local_names` 如何设置，
它们的名称都是本地名称。
{{< /callout >}}

### 哪些机构 {#which-institutions}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
  domains  = ["univ-a.fr", "univ-b.fr"]   # 联盟中的其他人一律不得进入
}

share "projet-x" {
  image   = "/srv/projet-x.img"
  allow   = ["oidc:groups:urn:mace:univ-a.fr:projet-x", "oidc:domain:univ-b.fr"]
  writers = ["oidc:groups:urn:mace:univ-a.fr:projet-x"]
}
```

`domains` 在身份验证时检查，SFTP 和 WebDAV 都一样，也适用于
NFS 客户端证书：名称必须
是 `<something>@<其中之一>`，并且整体比较（`evilunivb.fr` 不是
`univb.fr`）。`oidc:domain:` 是针对单个共享的同一项测试。该域名
的可信程度取决于提供方：go-authn/bridge 会丢弃其 IdP 的联盟
元数据未授予相应作用域的 eppn 或 subject-id。

**组**是机构的 IdP 所释放的内容，由 go-authn/bridge 的
`claims { groups = [...] }` 转换为 `groups`
声明：默认是 `eduPersonEntitlement`
（实验室或虚拟组织的组），或者是 `eduPersonScopedAffiliation`
（`staff@univ-a.fr`、`student@univ-b.fr`）。
