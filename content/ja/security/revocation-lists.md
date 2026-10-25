---
title: "SSH 証明書の失効: KRL"
weight: 20
description: "署名と日付のある OpenSSH KRL で ID プロバイダーの SSH 証明書を失効させる方法。ログイン時と SFTP のすべての操作で検査されます。"
tags: [セキュリティ, 失効, sftp, ssh]
---

ID プロバイダーの SSH CA が署名した証明書([プロバイダーが身元を保証する人のための SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})を参照)は、
ログイン時に検査されます。go-authn/bridge で誰かを無効化すると、そのトークンは失効し、アプリケーション
パスワードは削除され、それは[ディレクトリの再読み込み]({{< relref "/administration/reload.md" >}})によってここに
反映されます。しかしリストがなければ、発行済みの証明書は期限が切れるまで SFTP を開き続け、すでに開いている
SFTP セッションも開いたままになります。

v0.12.0 以降は、リストがそれを閉じます。

```hcl
oidc {
  issuer          = "https://login.example.org"
  audience        = "fileshare"
  ssh_ca_file     = "/etc/fileshare/bridge-ca.pub"
  ssh_krl_url        = "https://bridge.example.org/ssh/krl"   # or ssh_krl_file
  ssh_krl_state_file = "/var/lib/fileshare/ssh.krl.state"    # the order, across a restart
  ssh_krl_max_age    = "1h"                                  # default; ssh_krl_refresh = "1m"
}
```

go-authn/bridge は、失効させた SSH 証明書(無効化された人、無効化された IdP)を OpenSSH の **KRL**
(`PROTOCOL.krl`、`sshd` の `RevokedKeys` が読むリスト)として公開し、fileshare はそのコピーを保持して、
[go-authn/krl](https://github.com/go-authn/krl) で読み取ります。

## ログイン時と、セッションの途中で拒否される {#refused-at-login-and-in-the-middle-of-a-session}

失効した証明書はログイン時に拒否され、**それがすでに開いたセッションへの提供も停止されます**。
SSH は証明書を、ハンドシェイクで一度だけ検査します。そのため、フェデレーションされた SFTP セッションの
すべての操作は、**開いているファイルを含めて**、KRL を再度参照します。このサーバーのリストのコピーが
変わった後の最初の操作は拒否されます。

## フィールド {#fields}

| フィールド | 既定値 | |
|---|---|---|
| `ssh_krl_url` | | リストの取得元。**HTTPS のみ** |
| `ssh_krl_file` | | またはファイル(絶対パス)。どちらか一方で、両方は不可 |
| `ssh_krl_ca_file` | システムのもの | リストの HTTPS サーバーの検証に使う認証局を固定する |
| `ssh_krl_refresh` | `1m` | 取得の頻度。1 秒以上 |
| `ssh_krl_max_age` | `1h` | それが管理する証明書が拒否されるまでに、最後の正常なコピーが古くなってよい期間 |
| `ssh_krl_state_file` | | 検証に成功した最後のリストを保持する場所。再起動後も順序を記憶する |

`refresh` より短い `max_age` は拒否されます。次の取得の前にすべてのコピーが古くなってしまうためです。
`ssh_ca_file` のない KRL も拒否されます。KRL が失効させるのはプロバイダーの証明書であり、その CA がなければ
証明書が存在しないためです。

## フェイルクローズ {#it-fails-closed}

{{< callout type="error" >}}
**KRL を取得できない間、プロバイダーの証明書は拒否されます**

リストを取得できない間、または最後の正常なコピーが `ssh_krl_max_age` より古い間は、プロバイダーの CA が署名した
**すべての**証明書が拒否されます。リストに到達できないときにすべてを通す失効チェックは、リストを
ブロックできる攻撃者に破られるものだからです。したがって、リストは、それが管理するログインと同じ信頼性で
提供されなければなりません。`max_age` に達する前にアラートを出す対象のメトリクスは
`fileshare_revocation_list_age_seconds{list="ssh_krl"}` です。
[ヘルスチェックとメトリクス]({{< relref "/administration/health.md#what-to-alert-on" >}})を参照してください。
{{< /callout >}}

## 署名と日付が必要、`ssh_krl_url` から {#signed-and-dated-from-ssh_krl_url}

v0.15.0 以降、**`ssh_krl_url` から取得する KRL は、署名されていて、有効期限を示していなければなりません**。
これは [go-authn/revocation](https://github.com/go-authn/revocation) が説明し、go-authn/bridge が
v0.10.0 から提供しているものです。

- **署名:** `<url>.sig` は、プロバイダーの SSH CA(`ssh_ca_file`。そのため `ssh_krl_url` にはこれが必須)による、
  名前空間 `krl@go-authn.github.io` の分離 SSHSIG 署名です。リストの ETag に対する `If-Match` で取得され、
  その間にリストが再発行されていれば、直ちに再取得されます。
- **有効期限:** `expires@go-authn.github.io` 拡張。これを過ぎると、`304` で直前に確認されていたとしても、
  リストは最新ではありません。

HTTPS(`ssh_krl_ca_file` で認証局を固定)が認証するのは、応答したサーバーであって、リストではありません。
ミラー、キャッシュ、または侵害された Web サーバーが空のリストを返す可能性があり、KRL だけでは
それが古いことを示すものは何もありません。検証できないリストは拒否され、最後の正常なコピーが保持されます。
`ssh_krl_file` で手動で置いたリストは、それを保持するファイルシステムと同じだけ信頼され、署名も有効期限も
不要です。リストが持っているものは尊重されます。

{{< callout type="warning" >}}
**v0.15.0 へのアップグレード**

署名なしの KRL、または有効期限のない KRL を提供する `ssh_krl_url` は、もう受け付けられません。
入手できないリストと同様に、プロバイダーの証明書は拒否されます。先に go-authn/bridge v0.10.0 以降を
実行してください。
{{< /callout >}}

## 決して巻き戻らない、再起動をまたいでも {#never-backwards-across-a-restart-too}

v0.16.0 以降、保持しているものより古いリスト、つまりバージョン(KRL のバージョン、CRL の番号)が低いもの、
または同じバージョンでより早く発行されたものは拒否されます。`ssh_krl_state_file`
([NFS]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})では `crl_state_file`)を指定すると、
検証に成功した最後のリストがディスクに保持され、起動時に再検証されて、次のリストの順序を決めます。
これだけでは、リストが取得されるまで最新とは見なされません。これがなければ、再起動で順序が忘れられ、
fileshare は起動時にそのことを伝えます。署名があり期限切れでない古いリストが受け入れられてしまうからです。

これらのファイルは、`ssh_ca_file`、CA ファイル、リストファイル、そして(v0.17.0 以降)`authorized_keys_file`、
`dsn_file`、`bind_password_file`、`ssf` の `ca_file` と同様に、**共有の内側に置いてはいけません**。そこに書き込める人が、
誰が入れるかを決めることになるからです。

## fileshare と並ぶ通常の `sshd` {#a-plain-sshd-next-to-fileshare}

同じ検証は fileshare なしでも利用できます。go-authn/revocation の `authn-revokd` がリストを取得し、検証に成功した
ものだけを保持し、その順序をディスクに保存し、`sshd` 用の `RevokedKeys` を書き出します。リストが失効すると、
CA 自体を失効させるものを書き出すため、`sshd` もフェイルクローズになります。

## 対象外のもの {#what-it-does-not-cover}

**OpenPubkey(opkssh)証明書は、どの KRL にも載りません。** 発行したのは本人の鍵だけだからです。
それらは `opkssh_max_age` で制限され、[共有シグナル]({{< relref "/security/shared-signals.md" >}})で取り消されます。
WebDAV のアクセストークンも同様です。

同じ仕組み(取得し、保持し、フェイルクローズ)が、[証明書による NFS の ID]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})の
**X.509 CRL** も運びます。
