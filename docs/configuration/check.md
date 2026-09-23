# `check`, before you restart something people are using

```
$ fileshare check /etc/fileshare.d
ATTIC

SHARE    IMAGE               FILESYSTEM  WHO MAY CONNECT           WHO MAY WRITE   SMB  WEBDAV  NFS
photos   /srv/photos.img     fat32       alice and bob             alice           yes  yes     NO
scratch  /srv/scratch.img    ext4        anyone who authenticates  anyone …        yes  yes     yes

photos is not served over nfs: it is restricted to alice and bob, and NFSv3 has
no authentication at all: AUTH_UNIX is a claim the client makes about itself and
the wire cannot disagree with it

USER   FROM                    AUTHENTICATES WITH                 SMB  WEBDAV  SFTP
alice  the configuration file  a password from /etc/…/alice.pw    yes  yes     yes
bob    the configuration file  a password from /etc/…/bob.pw      yes  yes     -
dora   ldaps://ldap.example.org  a password check and an NT hash  yes  yes     -
eli    ldaps://ldap.example.org  a password check                 -    yes     -

this configuration can be served
```

Every image is opened **read-only** and closed again, so this is safe to run
against a live server's images.

It prints **where a credential comes from and never what is in it** — and the
per-person columns are why
[`Identity.Can`](https://github.com/go-authn/directory) exists: eli is in the
same group as dora and SMB still cannot serve him, because LDAP holds his
password and gives it to nobody.

## Reading the columns

| | |
|---|---|
| a filesystem with an asterisk | the driver was **asserted** by the share, not recognised — see [shares](shares.md) |
| `NO` under a protocol | that share is not exported there, and the reason is printed below the table |
| `-` under a protocol, per user | that person's source cannot prove what the protocol needs — see [identity](identity.md) |
