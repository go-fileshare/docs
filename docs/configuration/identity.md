# Users, groups and directories

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

!!! note "Expansion happens once, at startup"
    A membership that changes in LDAP is picked up by a restart. Said plainly,
    because asking the directory on every connection is a different design and
    this is not it.

## What a source can prove, and what each protocol needs

This is the part a site discovers otherwise at a mount, so [`check`](check.md)
says it first:

| the source has | SMB | WebDAV | SFTP |
|---|---|---|---|
| a password (file, or a cleartext column) | yes | yes | — |
| an NT hash (`sambaNTPassword`, `nt_hash`) | yes | — | — |
| only a bind, or a bcrypt column | **no** | yes | — |
| public keys, or a trusted CA | — | — | yes |

!!! danger "NTLMv2 needs the password or its MD4, and nothing else will do"
    A client never sends a password to an SMB server — it sends a proof
    computed from it — so a directory that only *checks* passwords cannot
    answer SMB, however good the check is. That is a property of the protocol,
    not a limitation of this program, and no amount of configuration changes
    it. WebDAV asks only "is this the right password", which a bind answers.

A person a directory names but proves nothing for is legitimate — a listing
with the secrets elsewhere — and `check` says so in one line rather than
leaving them to find out.
