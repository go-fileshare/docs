---
title: "Users, groups and directories"
weight: 20
description: "TODO"
tags: [configuration, identity, oidc, ldap, sql]
---

A `user` block is the whole directory for a household. A site whose people are
already in a database or in LDAP should not copy them into a second place that
goes stale, so a `users` block reads them where they are.

```hcl
users "sql" {
  driver   = "postgres"                 # or sqlite, or mysql
  dsn_file = "/etc/fileshare/dsn"       # a DSN holds a password: it lives in a file
  users    = "select login, nt_hash, ssh_keys from staff"
  groups   = "select team, member from team_members"
}

users "ldap" {
  url                = "ldaps://ldap.example.org"
  base_dn            = "ou=people,dc=example,dc=org"
  bind_dn            = "cn=reader,dc=example,dc=org"
  bind_password_file = "/etc/fileshare/bind.pw"
}
```

The **queries are yours**, because a site's people are already in that site's
shape; a schema invented here would mean copying them into a second one. The
LDAP side reads what a Samba-aware directory already publishes —
`sambaNTPassword`, `sshPublicKey`, `memberUid` — and every name is
configurable.

{{< callout type="error" >}}
**No cleartext bind to another machine (since v0.19.0)**

A `users "ldap"` block refuses `ldap://` to another machine unless
`start_tls = true`: every password a bind checks would cross the network
in the clear. `ldaps://` is accepted, and so are `ldap://` to a loopback
address and `ldapi://`, where nobody is on the way. There is no switch to
turn this off (go-authn/directory v0.10.0).
{{< /callout >}}

## Order, and the one exception

Sources are asked **in the order they are written**, and the first one that
knows a name owns it. The `user` and `group` blocks come first, so a service
account written down locally is not overridden by somebody with the same name
in LDAP.

**Groups are the exception**: a group's members are the **union** of every
source, because a team can have people in a file and in a database.

```hcl
share "photos" {
  allow   = ["@engineers", "alice"]
  writers = ["@owners"]
}
```

{{< callout type="info" >}}
**Expanded at startup, and again at every reload**

Since v0.10.0 the `users` blocks — SQL, LDAP — are read again every
`reload = "5m"`, on `SIGHUP`, and on the admin API's `ReloadDirectory`; a
membership that changes in LDAP no longer waits for a restart. What a reload
applies in place and what it restarts, and why a group that empties does
**not** open its share to everybody, is in
[reading the directory again]({{< relref "/administration/reload.md" >}}). The directory
is still not asked on every connection: that is a different design, and this
is not it.
{{< /callout >}}

## What a source can prove, and what each protocol needs

This is the part a site discovers otherwise at a mount, so [`check`]({{< relref "/configuration/check.md" >}})
says it first:

| the source has | SMB | S3 | WebDAV | SFTP |
|---|---|---|---|---|
| a password (file, or a cleartext column) | yes | yes | yes | — |
| an NT hash (`sambaNTPassword`, `nt_hash`) | yes | **no** | — | — |
| only a bind, or a bcrypt column | **no** | **no** | yes | — |
| public keys, or a trusted CA | — | — | — | yes |

{{< callout type="error" >}}
**NTLMv2 needs the password or its MD4, and nothing else will do**

A client never sends a password to an SMB server — it sends a proof
computed from it — so a directory that only *checks* passwords cannot
answer SMB, however good the check is. That is a property of the protocol,
not a limitation of this program, and no amount of configuration changes
it. WebDAV asks only "is this the right password", which a bind answers.
{{< /callout >}}

A person a directory names but proves nothing for is legitimate — a listing
with the secrets elsewhere — and `check` says so in one line rather than
leaving them to find out.

## People the identity provider names, not this file

A federation — RENATER through
[go-authn/bridge](https://github.com/go-authn/bridge), say — knows who is in a
project, and a share can ask **it** rather than copy the list. With an
[`oidc` block]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}):

```hcl
share "photos" {
  image   = "/srv/photos.img"
  allow   = ["oidc:groups:urn:mace:univ-example.fr:photos", "oidc:user:bob@univ-example.fr", "alice"]
  writers = ["oidc:groups:urn:mace:univ-example.fr:photos"]
}
```

`oidc:groups:<value>` is somebody whose token's groups claim holds the value;
`oidc:user:<name>` is somebody the token names. The spelling is opkssh's, so
that one vocabulary says who reaches a shell and who reaches a share. Somebody
a rule names is known to this server as far as the provider goes — the open
shares too — and somebody no rule names is still a stranger. The same rules
apply over [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}),
where the groups travel in the certificate instead of the token.

{{< callout type="error" >}}
**A rule is about the provider's people only**

`oidc:user:bob` is the bob the provider vouches for; a local account called
bob, with a password, is somebody else and matches no rule. A rule with no
`oidc` block, a malformed one, or one that may write without being allowed
to connect is **refused at startup**.
{{< /callout >}}

{{< callout type="error" >}}
**The provider's `alice` is not the local `alice` unless you say so (since v0.20.0)**

A plain name in `allow` or `writers` is a local account; a token or a
provider certificate is reached only by a share's `oidc:` rules. A site
whose provider's names ARE its local names writes `local_names = true` in
the `oidc` block — and should set `domains` with it, or take the name from
a claim the provider controls: a provider where people choose their own
`preferred_username` would otherwise hand anybody who signs up a local
account's shares, write included. That was the default before v0.20.0
(security audit F4). A federated name refused for want of `local_names`
is logged with the line to add, and `fileshare check` says which rule is
in force.

NFS client certificates (`identity = "certificate"`) are the exception:
their CA is the one `client_ca_file` pins for exactly that, and plain
names are the only way an NFS share can name somebody, so their names are
local names whatever `local_names` says.
{{< /callout >}}

### Which institutions

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
  domains  = ["univ-a.fr", "univ-b.fr"]   # nobody else from the federation gets in
}

share "projet-x" {
  image   = "/srv/projet-x.img"
  allow   = ["oidc:groups:urn:mace:univ-a.fr:projet-x", "oidc:domain:univ-b.fr"]
  writers = ["oidc:groups:urn:mace:univ-a.fr:projet-x"]
}
```

`domains` is checked at authentication, over SFTP and WebDAV alike: a name must
be `<something>@<one of them>`, compared whole (`evilunivb.fr` is not
`univb.fr`). `oidc:domain:` is the same test for one share. The domain can be
trusted as far as the provider: go-authn/bridge drops an eppn or subject-id
whose scope the IdP's federation metadata does not grant it.

**Groups** are what the institution's IdP releases, turned into the `groups`
claim by go-authn/bridge's `claims { groups = [...] }`: `eduPersonEntitlement`
by default (a lab's or a VO's groups), or `eduPersonScopedAffiliation`
(`staff@univ-a.fr`, `student@univ-b.fr`).
