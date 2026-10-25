---
title: "プロトコルが保証できること"
linkTitle: "プロトコル"
weight: 20
description: "各プロトコルが、誰がアクセスしているかを判別できる、あるいはできない理由と、それがどの共有を提供するかにどう影響するか。"
tags: [プロトコル]
---

アクセス制御に必要な唯一のこと、つまりサーバーが**誰が**アクセスしているかを判別できるかどうかについて、
プロトコル間で一致していません。

| | |
|---|---|
| **[SMB]({{< relref "/protocols/smb.md" >}})** | NTLMv2。パスワードはネットワークを流れず、共有は読み取り側に、読み取り専用であることを(試す前に)アクセスマスクで伝える。 |
| **[WebDAV]({{< relref "/protocols/webdav.md" >}})** | [TLS]({{< relref "/security/tls.md" >}})上の HTTP Basic。到達可能なアドレスでは平文を拒否する。前段のプロキシが TLS を終端する場合は `plaintext = true` を指定する。人が利用できない共有は 403 ではなく 404 で応答し、存在を確認させない。 |
| **[SFTP]({{< relref "/protocols/sftp.md" >}})** | **公開鍵**、または信頼する認証局が発行した **SSH 証明書**。サーバーはシークレットを一切保持せず、証明書を使えば、人のアクセス権は別の場所で発行され、別の場所で失効する。パスワードはなし。パスワードを尋ねるクライアントは、鍵が避けるために存在するものそのものを行っている。 |
| **[OIDC]({{< relref "/protocols/webdav.md" >}})**(WebDAV 経由) | ID プロバイダーが署名した**ベアラートークン**。[go-authn/oidc](https://github.com/go-authn/oidc) が署名、発行者、オーディエンス、有効期限を検証する。ここにある他のプロトコルには、トークンを載せる場所がない。S3 は SigV4 で署名し、ベアラートークンのためのフィールドがない。[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})では、プロバイダーの言葉は代わりに証明書で届く。 |
| **[S3]({{< relref "/protocols/s3.md" >}})** | **SigV4**(ヘッダーまたは署名付き URL)。シークレットは HMAC を計算することで自身を証明し、ネットワークを流れない。そのため NTLMv2 と同様に、ディレクトリはパスワードを確認するだけでなく、**保持**していなければならない。共有はバケット。 |
| **[NFSv3]({{< relref "/protocols/nfs.md" >}})** | 単独では**何もない**。`AUTH_UNIX` は申告にすぎず、クライアントが「uid 501」と言えば、ネットワーク上でそれに反論する手段はない。`kerberos` ブロックでこれを解消でき、[証明書による ID]({{< relref "/security/nfs-certificates.md" >}})でも解消できる。[TLS]({{< relref "/security/tls.md" >}})は暗号化するが、証明するのはマシンであって人ではない。 |

## 結論を一度だけ述べる {#the-consequence-stated-once}

**利用できる人を指定した共有は、NFS ではエクスポートされません。** Kerberos が設定されているか、
NFS が[証明書による ID]({{< relref "/security/nfs-certificates.md" >}})を受け付ける場合を除きます。

警告でもオプションでもありません。「photos は alice のもの」という設定と、接続した誰にでも
photos を渡すプロトコルとは両立できず、黙ってアクセスを広げるのは二つの失敗のうち悪いほうだからです。
拒否は起動時と[`check`]({{< relref "/configuration/check.md" >}})で、理由とともに出力されます。

## それぞれがディレクトリに求めるもの {#what-each-one-needs-from-a-directory}

どのプロトコルである人にサービスを提供できるかは、設定だけでなく、その人のソースが何を証明できるかで
決まります。[ソースが証明できること]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}})を
参照してください。

## 暗号化されるもの {#what-is-encrypted}

WebDAV、S3、NFS は `tls = true` を受け付けます。SMB 3 は独自の鍵で暗号化し、SFTP は SSH です。
[TLS と ACME による証明書]({{< relref "/security/tls.md" >}})を参照してください。
