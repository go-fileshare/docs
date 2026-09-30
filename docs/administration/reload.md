# Reading the directory again

```hcl
reload = "5m"   # and on SIGHUP, and on the admin API's ReloadDirectory
```

The people come from directories other things change — go-authn/bridge writes
an application password into a table, and deletes the row when the person is
disabled. Without a reload the server serves whoever was there when it started.
Since v0.10.0 it reads them again:

- every `reload` interval, a Go duration of **at least a second** — a directory
  read many times a second is a load on somebody's database, not a fresher
  server;
- on **`SIGHUP`** (Unix);
- on the [admin API](index.md)'s **`ReloadDirectory`**, which answers with who
  was added, removed and changed, whether a new generation started, and notes a
  person should read — a group that no longer exists, somebody a share names
  who is no longer in the directory.

Without `reload`, the directory is read at the start, on `SIGHUP` and on
`ReloadDirectory` only.

It is the `users` blocks — SQL, LDAP — that are read again. The `user` and
`group` blocks of the configuration file **are** the configuration, read at the
start.

## What a reload does depends on what changed

The line is drawn at **revocation**:

| what changed | what happens |
|---|---|
| **only additions** — somebody new whose arrival changes no share's expanded lists (a new application password, a person the shares reach through `oidc:` rules or none at all) | added **in place**, SMB's running server included, and **no connection is touched**. A bridge creating application passwords all day disturbs nobody. |
| **anything taken away** — somebody gone, a credential changed, a share whose expanded lists changed | a **new generation**, and the old one's connections **closed**, exactly as an [admin change](index.md#a-change-is-a-new-generation) is. A session that outlived its person's removal is the thing a reload exists to end. |
| **a directory that cannot be read** — the database unreachable, a query failing | **nothing changes**. An outage must not empty a file server, and "nobody" is what a failed read looks like. |

Somebody new who joins a group a share names *does* change that share, and goes
through a new generation like any other change to it: SMB fixes a share's lists
when it starts.

A share naming somebody who is gone, or a group that no longer exists, is served
**without them** — the safe direction — and said, rather than refused the way a
startup refuses it: refusing would keep the old lists, and the old lists are the
access that was just taken away.

## An empty group is not everyone

!!! danger "A share written for `@engineers` whose last engineer has left is served to nobody"
    An empty `allow` means "anyone who authenticates". So what decides whether
    a share is open is what was **written**, not what it expands to now: a share
    written for a group stays closed when that group empties. Such a share is
    not offered over SMB at all, whose empty `AllowUsers` would read as
    everyone.

## Watching it

`fileshare_directory_reloads_total{result}` counts reloads that found nothing
(`unchanged`), added in place (`added`), started a generation (`swapped`) or
could not read the directory (`failed`); `fileshare_directory_people` is how
many people the last read held. See [health and metrics](health.md).

!!! note "What a reload does not reach"
    People the identity provider vouches for through a token or a certificate
    are not in the directory, so no reload concerns them. Taking **them** back
    is the job of [revocation lists and shared signals](../security/index.md).
