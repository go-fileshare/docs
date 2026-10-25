---
title: "check: 人が使っているものを再起動する前に"
linkTitle: "check、再起動する前に"
weight: 30
description: "fileshare check が提供前の設定について出力する内容と、その読み方。"
tags: [設定, check]
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

すべてのイメージは、サーバーが開くのと同じ方法で開かれ、また閉じられます。何も
提供されません。`read_only = true` を指定していない共有では、これは**読み書き**で開くことを意味します。
常に読み取り専用かつ排他的に開かれるのはデバイスだけで、そのため実行中のサーバーが
保持しているデバイスは、ここでビジー状態として拒否されることがあります。

出力されるのは**認証情報の出所であり、その中身ではありません**。人ごとの列があるのは、
[`Identity.Can`](https://github.com/go-authn/directory) が存在する理由そのものです。
eli は dora と同じグループにいますが、LDAP が彼のパスワードを保持していて誰にも渡さないため、
SMB では依然として彼にサービスを提供できません。

## 列の読み方 {#reading-the-columns}

| | |
|---|---|
| アスタリスク付きのファイルシステム | ドライバーは共有によって**主張された**ものであり、認識されたものではない。[共有]({{< relref "/configuration/shares.md" >}})を参照 |
| プロトコルの下の `NO` | その共有はそこではエクスポートされず、理由が表の下に出力される |
| 共有ごとの、プロトコルの下の `-` | 共有の `protocols` がそのプロトコルを除外している |
| ユーザーごとの、プロトコルの下の `-` | その人のソースでは、そのプロトコルが必要とするものを証明できない。[ID]({{< relref "/configuration/identity.md" >}})を参照 |

## ほかに表示される内容 {#what-else-it-says}

表の後に、該当する場合、設定を信頼する前に知っておくべきことを `check` が伝えます。

- `oidc` ブロックがある場合は、そのトークンが WebDAV でのみ受け入れられること、および
  プロバイダーの名前がローカル名として扱われるかどうか(`local_names`)。
  [ID]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})を参照。
- プロトコルごとに、何が**暗号化**されていて、何が**意図的に平文**であるか。
  [TLS]({{< relref "/security/tls.md#what-check-says" >}})を参照。
- **失効リスト**の取得元と、取得できない間に何が起こるか。
  [何が何を失効させるか]({{< relref "/security/_index.md" >}})を参照。
- 管理 API の `DisableShare` で**オフライン化**され、提供されていない共有。
  [管理 API]({{< relref "/administration/_index.md#disabling-a-share" >}})を参照。

KRL、証明書による NFS の ID、共有シグナルを使うサーバーでは、この部分は次のようになります。

```
sftp: the provider's SSH certificates are checked against the KRL at https://bridge.example.org/ssh/krl, at login and on every operation; while it is unknown or older than 1h0m0s they are refused

federated people: revocations come from the shared signals transmitter https://bridge.example.org (CAEP session-revoked, polled); whatever the provider issued them before a revocation is refused -- tokens, SSH and OpenPubkey certificates, NFS certificates -- and open sessions stop. Refused while it is not heard from within 10m0s; revocations kept 192h0m0s in /var/lib/fileshare/revocations.json

nfs: identities come from client certificates signed by /etc/fileshare/bridge-x509-ca.pem, revoked by the CRL at https://bridge.example.org/x509/crl (refused while it is unknown or older than 1h0m0s).
     ⛔ a Linux client's certificate belongs to a MOUNT: every user of that mount is the person it names
```

`check` はリストを取得しません。取得元と、それが得られない場合に何が起こるかを伝えるだけです。
一方、ID プロバイダーのディスカバリードキュメントと鍵は**読み取られます**。
発行者に到達できない `oidc` ブロックは、`check` によって拒否されます。
