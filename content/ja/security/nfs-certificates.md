---
title: "NFS、証明書による ID"
weight: 30
description: "X.509 クライアント証明書で名指しされた人に、CRL で失効させつつ NFS で制限付き共有を提供する方法と、共有クライアントで保証できないこと。"
tags: [セキュリティ, nfs, tls, 失効]
---

NFSv3 は単独では誰も認証しません。そのため、利用できる人を指定した共有は、何かが人を区別できない限り
[NFS ではエクスポートされません]({{< relref "/protocols/nfs.md" >}})。`kerberos` ブロックがその答えの 1 つです。
v0.12.0 以降は、**人を名指しする**クライアント証明書がもう 1 つの答えです。

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file; required
  crl_state_file = "/var/lib/fileshare/nfs.crl.state"     # the order, across a restart
}
```

## 証明書が運ぶもの {#what-the-certificate-carries}

ID プロバイダーでのログイン後、go-authn/bridge は、FreeBSD の `rpc.tlsservd -u` が読み取る形式で人を
名指しする、**有効期間の短い X.509 クライアント証明書**を発行します。

- 人: SubjectAltName の **otherName `1.3.6.1.4.1.2238.1.1.1`**。UTF8String の `user@domain`
  ([draft-cel-nfsv4-rpc-tls-othername](https://www.ietf.org/archive/id/draft-cel-nfsv4-rpc-tls-othername-04.html)
  が説明している。その OID はまだ割り当てられていない)。
- そのグループ: それぞれ 1 つの SubjectAltName URI、
  `tag:go-authn.github.io,2026:group:<group>`。**タグ URI**(RFC 4151: 登録不要で、まさにこの用途のためのもの)。
  FreeBSD の `rpc.tlsservd` は最初の ID otherName を読み取り、他のすべての SAN エントリをスキップする。

{{< callout type="info" >}}
**独自の拡張ではなくタグ URI である理由**

UUID から導出したアーク(`2.25.<128 ビットの数>`)の下で独自の拡張を設計し、**計測**しました。
Go の `x509.ParseCertificate` は証明書全体を拒否します(*malformed extension OID field*)。
ASN.1 オブジェクト識別子のアークがそこでは `int` だからです。Go の TLS サーバーなら、どれもこの証明書を
突き返したはずです。URI SAN には OID が不要で、どのパーサーも読み取れます。`urn:x-…` も使いません。
RFC 8141 が実験用の URN 名前空間を廃止しており、厳格なパーサーはそれを拒否します。
{{< /callout >}}

## すべての呼び出しで問われること {#what-every-call-is-asked}

`identity = "certificate"` を指定すると、人を指定した共有は **kerberos なし**で NFS で提供されます。
その上のすべての呼び出しは TLS 経由で届き、次の条件を満たす証明書を伴っていなければなりません。

1. **CRL にない**こと。
2. **ちょうど 1 人**の人を名指ししていること。
3. このサーバーが**トークンを受け入れるのと同じ方法で**受け入れる人を名指ししていること。つまりルール、
   `trust_all`、`domains` です。
   [ID プロバイダーが名指しする人]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})を参照。
4. **共有が許可している**人を名指ししていること。

さらに、[`ssf` ブロック]({{< relref "/security/shared-signals.md" >}})がある場合、証明書は、その人に対する
どの失効よりも**後に**発行(その `NotBefore`)されていなければなりません。

## 失効: CRL {#revocation-the-crl}

**CRL は必須です**。`crl_url` または `crl_file` です。何も失効させられない人の証明書は、
その人が削除されてからも、有効期間の全体にわたって生き残ります。

| フィールド | 既定値 | |
|---|---|---|
| `crl_url` | | HTTPS のみ |
| `crl_file` | | または絶対パス。どちらか一方 |
| `crl_ca_file` | システムのもの | CRL の HTTPS サーバーの認証局を固定する |
| `crl_refresh` | `1m` | 取得の頻度 |
| `crl_max_age` | `1h` | 最後の正常なコピーが古くなってよい期間 |
| `crl_state_file` | | 検証に成功した最後の CRL を保持する場所。再起動後もどちらが新しいかを記憶する |

[KRL]({{< relref "/security/revocation-lists.md" >}})と同様に取得・保持され、同じようにフェイルクローズです。
自身の **`NextUpdate`** を過ぎた CRL は、`crl_max_age` が許していても不明として扱われます。CRL はクライアント CA が
署名していなければなりません。誰でも書ける CRL は好きなものを失効させ、さらに悪いことに、失効を取り消せてしまいます。
CRL に到達できれば、**変更後の次の呼び出しが拒否されます**。証明書の有効期間が失効を制限するのは、CRL に到達できない
場合だけです。

v0.16.2 以降、署名は CRL が解析される**前に**、生のバイト列に対して
[go-authn/revocation](https://github.com/go-authn/revocation) 経由で検査されます。先に解析すると、
誰でも署名できる大きな CRL が、拒否される前にメモリと秒数を消費していました。このサーバーが完全なリストとして
読み取れないものも拒否されます。差分 CRL、不明なクリティカル拡張やエントリ拡張、CRL 番号なし、`NextUpdate` なしです。
保持しているものより番号が低い CRL、または同じ番号でより早く発行された CRL は、古い
[KRL]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}})と同様に拒否されます。

`identity = "certificate"` のない CRL のフィールドは拒否され、`tls = true` と `client_ca_file` のない
`identity = "certificate"` も拒否されます。ID は、クライアントが提示する証明書の中にあるためです。

## 保証できないこと {#what-it-cannot-promise}

{{< callout type="error" >}}
**Linux クライアントの証明書は人ではなく、マウントに属します**

サーバーが見るのは**接続の**証明書であり、Linux クライアントは証明書をマウントに紐づけます
(`tlshd`、`mount -o xprtsec=mtls,...` のキーリングのシリアル)。**そのマウントのすべてのユーザーが、
証明書が名指しする人として動作します。** 一人が使うワークステーションではまさにその人ですが、
複数人がログインするマシンでは、マウントした人が誰であっても、その人になります。そこでは
`sec=krb5` を使ってください。

RFC 9289 だけではユーザー認証を保証しないのはこのためで、素の
[TLS 上の NFS]({{< relref "/security/tls.md#nfs-over-tls-proves-the-machine-not-the-person" >}})が証明するのが
依然としてマシンだけであるのもこのためです。
{{< /callout >}}

## 実際の Linux クライアントで計測 {#measured-with-a-real-linux-client}

カーネル 6.17、ktls-utils 0.9、go-filesystems/nfs の CI にて。

- **MOUNT の `MNT` は、`xprtsec=` の指定にかかわらず平文で届きます。** カーネルのマウントクライアントは
  TLS を持たないためです。これには応答し、TLS なしで行われた NFS 呼び出しはすべて拒否されます。そのため、
  共有が許可していない人には、`mount` 時ではなく最初のアクセスで *access denied* が見えます。
  **共有のものは何も平文で流れません。**
- `tlshd` は、このサーバーの証明書を**システムの**トラストストアだけに対して検査し(`x509.truststore` は無視)、
  クライアントの証明書と鍵が**root 所有で、鍵のモードが 600** であることを求めます。そうでないと
  *gnutls: Error in the certificate (-43)* で失敗し、どちらが原因かは示されません。
