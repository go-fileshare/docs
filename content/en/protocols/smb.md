---
title: "SMB — NTLMv2"
linkTitle: "SMB: NTLMv2"
weight: 10
description: "SMB with NTLMv2: what a directory must hold, the privileged port under isolation, and what a directory reload does to open connections."
tags: [protocols, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

The password **never crosses the wire**: a client sends a proof computed from
it. The share tells a reader they are one — in the access mask, before they
try, rather than by failing a write later.

## What a directory must hold

NTLMv2 needs the password itself or its MD4 (`sambaNTPassword`, an `nt_hash`
column). **Nothing else will do.**

A directory that only *checks* passwords — an LDAP bind, a bcrypt column —
cannot answer SMB, however good the check is, because the server has to compute
the same proof the client did. That is a property of the protocol, not a
limitation of this program.

[`check`]({{< relref "/configuration/check.md" >}}) prints a `-` in the SMB column for such a
person, at configuration time, rather than leaving them to discover it at a
mount.

## Kerberos: a ticket instead of a password {#kerberos}

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"   # nfs/fs.example.org and cifs/fs.example.org
}

serve "smb" {
  addr = "0.0.0.0:445"
  ntlm = false          # Kerberos only: no password is ever checked
}
```

The `kerberos` block that lets NFS tell people apart also lets SMB take a
ticket. **One keytab** holds both services, each protocol answering for its own:
SMB reads only the `cifs/<host>` principals of the block's realm, so a ticket
made for `nfs/` is refused here. A keytab with no `cifs/` principal leaves SMB
as it was, NTLM only, and the server says so at startup.

The principal goes through the same rule as over NFS (another realm's `alice` is
nobody) and must then be **somebody this configuration knows**: a ticket proves
the KDC knows the person, not that this server does. The share's `allow` and
`writers` lists apply as for a password. A session ends with its ticket, and a
Windows client authenticates again by itself.

Since fileshare v0.50.0 the **keytab** must be a regular file, owned by root or by
the account fileshare runs as, and neither readable or writable by others nor
writable by its group: `root:fileshare` `0640` and `fileshare:fileshare` `0600`
are accepted, `krb5:fileshare` `0640` is refused. This applies to SMB and NFS. The
client's name and realm are taken from the **ticket**, never from the
authenticator the client also sends (jcmturner/gokrb5#577, RFC 4120 section
3.2.3); NFS needs go-authn/krb5 v0.3.4 or later, which v0.50.0 ships.

`ntlm = false` is refused at load without a `kerberos` block, and the server
refuses to start if the keytab holds no `cifs/` key for the realm: a server
nobody can log in to is not one. With it, a person whose directory holds no NT
hash can still use SMB, and [`check`]({{< relref "/configuration/check.md" >}})
says so.

A Windows machine that is **not in a domain** reaches it with a ticket from this
realm: `ksetup /addkdc` and `ksetup /addhosttorealmmap` once, a ticket-granting
ticket in the logon session, and `net use \\fs.example.org\photos` — by name,
since Windows falls back to NTLM for an address. Measured with Windows 11 25H2
(not joined): the client offers MS-KRB5, Kerberos, NEGOEX and NTLMSSP, sends a
Kerberos AP-REQ, mounts, lists, reads and writes with signing required, and
sends no NTLM message at all.

## SMB over QUIC {#quic}

```hcl
tls {
  cert_file = "/var/lib/caddy/certificates/acme-v02.api.letsencrypt.org-directory/files.example.org/files.example.org.crt"
  key_file  = "/var/lib/caddy/certificates/acme-v02.api.letsencrypt.org-directory/files.example.org/files.example.org.key"
}

serve "smb" {
  addr      = ":443"          # UDP: it does not clash with an HTTPS proxy on TCP 443
  transport = "quic"
  tls       = true
  # client_cert    = "required"
  # client_ca_file = "/etc/fileshare/client-ca.pem"
}
```

SMB 3.1.1 over QUIC is the same protocol inside TLS 1.3 on UDP 443, with ALPN
`smb`: `net use \\files.example.org\share /TRANSPORT:QUIC` on Windows 11, with
TCP 445 closed. The certificate is the `tls` block's and its files are reloaded
when they change, so pointing them at a reverse proxy's ACME storage follows its
renewals. A Windows client checks the server name against the certificate's DNS
names: dial a name, never an address (an address also forces NTLM). A proxy on
the same host must not offer HTTP/3 on that UDP port (Caddy: `protocols h1 h2`).

By default only Kerberos tells people apart, with a `cifs/files.example.org`
principal in the keytab. `client_cert = "required"` adds a client certificate
chaining to `client_ca_file`, demanded in the TLS handshake before any SMB byte
is read: it says which machine may connect, the Kerberos principal is still who
the person is. The CA file is read when the server starts. Over QUIC the server
grants one credit per request and advertises no large MTU ([MS-SMB2] 3.3.5.4),
so a read or write is at most 64 KiB. `identity = "system"` and isolated
children do not serve QUIC yet and the server refuses to start with them. One
`serve "smb"` block carries one transport: run a second instance for TCP.

## A privileged port without a privileged server

Under [`--isolate`]({{< relref "/operations/isolation.md" >}}) on Unix, the parent binds 445
and passes the listener to the child, so the process actually speaking SMB
never needs the privilege. Windows has no `ExtraFiles`, so there the child
binds the address itself; the isolation is the same, the privileged-port half
is not.

## Verified against a client this project did not write

[go-smb2](https://github.com/cloudsoda/go-smb2) drives twenty concurrent
reads through SMB while twenty run through WebDAV, under `-race`, in CI. macOS
mounts a share over SMB while `curl` writes to the same image over WebDAV, and
reads back what WebDAV wrote.

## When the people change

SMB fixes who may connect to a share **when the share is added**, and checks it
once per tree connect. So somebody new whom a [directory reload]({{< relref "/administration/reload.md" >}})
finds is added in place, SMB's running server included, with no connection
touched; but anything taken away — a person, a credential, a share's expanded
lists — is a new generation, and SMB's open connections are closed, so a
revocation reaches the sessions already open. A share whose `allow` group has
emptied is **not offered over SMB at all**: its empty `AllowUsers` would read as
everyone.

## Requests in flight

Since go-filesystems/smb v0.8.1 a connection holds up to 32 requests in flight, and the server up to 128 across all connections. Replies are matched by `MessageId`. A frame holding one `READ` of a plain file runs beside others; compound chains (answered whole, in order), asynchronous requests and `WRITE` run alone. A frame has 30 s to arrive once it has begun. The mechanism is the shared module go-filesystems/inflight, also used by SFTP and NFS.
