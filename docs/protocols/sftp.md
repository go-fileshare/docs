# SFTP — keys, or certificates

Every client already has it: `sftp` ships with OpenSSH, the Finder and GNOME
mount it, editors speak it. It is also the one protocol here whose shape does
not fit — a person logs in and lands in **one** filesystem, not a list of
shares.

So the shares become the top-level directories of a tree built for whoever just
authenticated:

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

A share alice may not use **is not a directory alice can see**, and a share she
may only read refuses her writes *in the tree*, before a driver that would have
allowed them.

!!! note "A rename across two shares is refused"
    It would be a copy and a delete over two images, and that is not what
    rename promises anywhere.

## Configuration

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # optional

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # nothing here: her certificate is her credential
```

With `trusted_user_ca_file`, a certificate signed by that authority and naming
the user among its principals is enough — so a person's access is **issued and
expires elsewhere**, and no file here is edited when somebody joins or leaves.

The signature, the validity window and the principals are checked by
`x/crypto/ssh`'s `CertChecker`; verified against **OpenSSH's own client**,
which also refuses the same key once its certificate is moved aside.

!!! warning "Without `host_key_file` a fresh identity is generated at every start"
    The server says so, and every client that has seen it before will warn
    about a changed key — which is the client doing its job.

## No password

A client that prompts for one is doing the thing keys exist to avoid.
