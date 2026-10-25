---
title: "設定"
linkTitle: "設定"
weight: 10
description: "HCL 設定の構成、user ブロックが持てるもの、起動時に拒否されるもの。"
tags: [設定]
---

1 つのファイル、または小さなファイルを集めたディレクトリで設定します。`--config /etc/fileshare.d` を
指定するとそれらがマージされるため、ある一つのファイルのユーザーと別のファイルの共有を
組み合わせて使えます。

```hcl
name = "ATTIC"

user "alice" { password_file = "/etc/fileshare/alice.pw" }
user "bob"   { password_file = "/etc/fileshare/bob.pw" }

group "family" { members = ["alice", "bob"] }

share "photos" {
  image   = "/srv/photos.img"
  allow   = ["@family"]        # a group, or a person, in either list
  writers = ["alice"]          # bob gets it read-only
}

share "scratch" {
  image = "/srv/scratch.img"   # anyone who authenticates, read-write
}

tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true                  # HTTP Basic is the password: never in the clear
}
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

{{< callout type="error" >}}
**v0.9.0 以降、パスワードを使う WebDAV は平文では提供されません**

`serve "webdav" { addr = "0.0.0.0:8080" }` は、それまでこのページに載せていた例ですが、
誰かが認証する場合は**起動時に拒否される**ようになりました。HTTP Basic では、
すべてのリクエストがパスワードそのものだからです。上の例のように `tls` ブロックとあわせて
`tls = true` を指定するか、前段のプロキシが TLS を終端する場合は `plaintext = true` を
指定してください。[TLS]({{< relref "/security/tls.md" >}})を参照してください。
{{< /callout >}}

共有は、イメージの代わりに[デバイス]({{< relref "/configuration/shares.md#a-device-not-only-an-image" >}})や
[ホストのディレクトリ]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})にもできます。

## `user` ブロックが持てるもの {#what-a-user-block-can-carry}

上の例はすべて `password_file` を使っています。これは最も一般的なケースで、しばらくの間は
このプログラムが読み取る唯一の方法でもありました。しかしブロックの内容はこれだけではなく、
その違いによって**どのプロトコルで人にサービスを提供できるか**が決まります。
マウントしてから気づくよりも、一か所にまとめておくほうがよいでしょう。

| フィールド | 内容 | 得られるもの |
|---|---|---|
| `password` | パスワード(ファイル内に直接記述) | パスワードで応答できるすべてのプロトコル |
| `password_file` | 同じものを専用のファイルに置く | 同じだが、シークレットが設定ファイルに入らない |
| `nt_hash` | `MD4(UTF16LE(password))`、16 進 32 文字 | **SMB**(このサイトがパスワードを保持していない人向け) |
| `authorized_keys` | `authorized_keys` の行を直接記述 | **SFTP** |
| `authorized_keys_file` | 同じものをファイルから読み込む | **SFTP** |
| `totp_secret` | base32 のワンタイムコード用シークレット | 今のところなし。読み込まれて検査はされるが、ここには二要素目を求めるプロトコルがない |

⛔ `password` と `password_file` の同時指定は、起動時に人の名前を示して拒否されます。
「その人のパスワードは何か」という問いに答えが 2 つあると、どちらが優先されるのかという
問題になり、それを知るために設定を 2 度読まされるべきではないからです。
`authorized_keys` と `authorized_keys_file` の同時指定も同様です。

⛔ `password`、`password_file`、`authorized_keys`、`authorized_keys_file` のいずれも
持たない `user` ブロックも、`trusted_user_ca_file` が設定されていない限り、
*has no way to authenticate* として拒否されます。`nt_hash` と `totp_secret` はここでは
数えません。

{{< callout type="default" >}}
**インラインのユーザーも SMB で提供できます**

最近まで、このブロックが読み取るのはパスワードのフィールドだけでした。そのため、ここに書かれた
人に SMB を提供するには、このファイルにパスワードを平文で渡すか、データベースに移す必要が
ありました。`nt_hash` を使えばその必要はありません。Samba が保存するものを保存しているサイトは、
代わりにそれを書けます。ただしブロックには、上記のフィールドのいずれか(たとえば鍵)か
`trusted_user_ca_file` が別に必要です。`nt_hash` だけでは起動時に拒否されます。
同じ表を一段上から見たものが
[ソースが証明できること]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}})です。
{{< /callout >}}

グループは、人を書ける場所ならどこでも `@name` と書きます。

## 後からではなく、起動時に拒否されるもの {#what-is-refused-at-startup-rather-than-later}

{{< callout type="warning" >}}
**誰のものでもない名前**

`allow = ["alise"]` と書くと、Alice が自分の共有から締め出されたまま、何事もなく
起動してしまいます。どの `user` ブロックにも [`users` ディレクトリ]({{< relref "/configuration/identity.md" >}})にも
存在しない名前は拒否されます。どれにも存在しない `@group` も同様です。
{{< /callout >}}

{{< callout type="warning" >}}
**NFS で、利用できる人を指定した共有**

「photos は alice のもの」という設定と、接続した誰にでも photos を渡すプロトコルとは、
両立できません。そのような共有は NFS ではエクスポートされません。
`kerberos` ブロックまたは `identity = "certificate"` によって NFS が人を区別できる場合は
別です。また、提供するものが何もなくなった `nfs` の serve ブロックは拒否されます。
[NFS]({{< relref "/protocols/nfs.md" >}})を参照してください。
{{< /callout >}}

{{< callout type="warning" >}}
**言葉を載せる場所のないプロバイダー**

[`oidc`]({{< relref "/protocols/webdav.md" >}}) ブロックは、`Authorization` ヘッダーを運べる唯一の
プロトコルである WebDAV が提供されていない限り、または SFTP がプロバイダーの証明書を有効にして
(`ssh_ca_file` または `opkssh_client_id`。[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})を参照)
提供されていない限り、拒否されます。SMB と NFS は、どちらも載せる場所を持っていません。
{{< /callout >}}

{{< callout type="warning" >}}
**平文のパスワード**

他のマシンから到達できるアドレスで、誰かが認証する状況において、`tls = true` も
`plaintext = true` も指定せずに WebDAV を提供する場合です。
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})を参照してください。
{{< /callout >}}

{{< callout type="warning" >}}
**安全に提供できないブロック**

`state_file` のない `admin` ブロック、相互 TLS なしで TCP で待ち受ける `admin` ブロック、
どの `serve` ブロックからも使われない `tls` ブロック、1 秒より短い `reload`、
平文 HTTP で取得される失効リストです。それぞれ該当するページで説明しています。
[管理]({{< relref "/administration/_index.md" >}})、
[TLS]({{< relref "/security/tls.md" >}})、[reload]({{< relref "/administration/reload.md" >}})、
[失効]({{< relref "/security/_index.md" >}})。
{{< /callout >}}

{{< callout type="warning" >}}
**このバイナリがビルドされていないプロトコル**

「そのようなプロトコルはありません」ではなく、*そうである*ことが伝えられます。
これは、タイプミスと[ビルドタグ]({{< relref "/operations/build-tags.md" >}})の違いです。
{{< /callout >}}

`addr` のない `serve` ブロックは、ループバックの `127.0.0.1` で、特権が不要なポートで待ち受けます。
SMB は 4445、WebDAV は 8080、SFTP は 2222、NFS は 2049、S3 は 9000 です。

## 共有の `protocols` {#protocols-on-a-share}

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

どのプロトコルで提供するかを指定します。単独でも有用で、SMB 専用と宣言した共有は
WebDAV プロセスには一切通知されません。また、[`--isolate`]({{< relref "/operations/isolation.md" >}}) の下では
設定によっては必須です。結果として何も提供しなくなる `serve` ブロックも、イメージを開く前に
拒否され、そこから除外された共有とその理由が示されます。ただし、後から共有を与える
`admin` ブロックがある場合は除きます。
