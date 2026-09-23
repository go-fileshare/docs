# NFS — nothing, unless Kerberos

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

!!! danger "NFSv3 on its own authenticates nobody"
    `AUTH_UNIX` is a **claim**: the client says "uid 501" and the wire cannot
    disagree with it. There is no encryption either.

So **a share that names who may use it is not exported over NFS**. The refusal
is printed at startup and in [`check`](../configuration/check.md), with the
reason:

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it
```

A share with no `allow` and no `writers` — anyone who connects, read-write — is
exported over NFS, because there is no rule there for the protocol to fail to
honour.

## The refusal that used to be permanent

A `kerberos` block makes NFS able to tell people apart, and a restricted share
is then served over it like anywhere else:

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5` carries a principal a **ticket proves**, rather than a uid the
client asserts.

!!! note "The realm is compared, not just the name before the `@`"
    Two realms can each have an `alice`, and only one of them is yours.
