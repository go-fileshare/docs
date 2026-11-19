---
title: "SMB — NTLMv2"
linkTitle: "SMB: NTLMv2"
weight: 10
description: "NTLMv2 を使う SMB。ディレクトリが保持すべきもの、分離時の特権ポート、ディレクトリの再読み込みが開いている接続に与える影響。"
tags: [プロトコル, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

パスワードは**ネットワークを流れません**。クライアントは、パスワードから計算した証明を送ります。
共有は、読み取り側に読み取り専用であることを、後で書き込みに失敗させるのではなく、
試す前のアクセスマスクで伝えます。

## ディレクトリが保持すべきもの {#what-a-directory-must-hold}

NTLMv2 に必要なのは、パスワードそのものか、その MD4(`sambaNTPassword`、`nt_hash` 列)です。
**ほかのものでは代用できません。**

パスワードを*確認*するだけのディレクトリ(LDAP バインド、bcrypt の列)は、確認がどれほど優れていても
SMB には応答できません。サーバーがクライアントと同じ証明を計算しなければならないからです。
これはプロトコルの性質であり、このプログラムの制約ではありません。

[`check`]({{< relref "/configuration/check.md" >}}) は、そのような人について SMB の列に `-` を
設定時に表示し、マウントして初めて気づくことのないようにします。

## Kerberos: パスワードの代わりにチケット {#kerberos}

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"   # nfs/fs.example.org と cifs/fs.example.org
}

serve "smb" {
  addr = "0.0.0.0:445"
  ntlm = false          # Kerberos のみ: パスワードは一切検証しない
}
```

NFS が人を区別するために読む `kerberos` ブロックは、SMB にもチケットを受け付けさせます。
**keytab は 1 つ**で両方のサービスを持ち、各プロトコルは自分のサービスにだけ応答します。
SMB はブロックのレルムの `cifs/<ホスト>` プリンシパルだけを読むため、`nfs/` 向けに発行された
チケットはここでは拒否されます。`cifs/` プリンシパルのない keytab では SMB は従来どおり
NTLM のみで、サーバーは起動時にそう伝えます。

プリンシパルは NFS と同じ規則を通り(別のレルムの `alice` は誰でもありません)、そのうえで
**この設定が知っている人**でなければなりません。チケットが証明するのは KDC がその人を
知っていることであり、このサーバーが知っていることではありません。共有の `allow` と
`writers` はパスワードの場合と同じように適用されます。セッションはチケットとともに終わり、
Windows クライアントは自動的に再認証します。

fileshare v0.50.0 以降、**keytab** は通常ファイルで、root か fileshare を実行する
アカウントが所有し、他者に読み書きされず、グループにも書き込まれないものでなければ
なりません。`root:fileshare` `0640` と `fileshare:fileshare` `0600` は受理され、
`krb5:fileshare` `0640` は拒否されます。これは SMB と NFS の両方に適用されます。
クライアントの名前とレルムは**チケット**から取り、クライアントが一緒に送る
オーセンティケータからは決して取りません(jcmturner/gokrb5#577、RFC 4120 の
3.2.3 節)。NFS には go-authn/krb5 v0.3.4 以降が必要で、v0.50.0 に含まれています。

`ntlm = false` は `kerberos` ブロックがないと読み込み時に拒否され、keytab にそのレルムの
`cifs/` の鍵がなければサーバーは起動を拒否します。誰もログインできないサーバーはサーバー
ではないからです。ブロックがあれば、ディレクトリが NT ハッシュを持たない人も SMB を使え、
[`check`]({{< relref "/configuration/check.md" >}}) がそれを示します。

**ドメインに参加していない** Windows マシンは、このレルムのチケットで接続します。
`ksetup /addkdc` と `ksetup /addhosttorealmmap` を一度実行し、ログオンセッションに
TGT を入れ、`net use \\fs.example.org\photos` を実行します。アドレスでは Windows が
NTLM にフォールバックするため、名前で指定します。Windows 11 25H2(未参加)で測定した結果:
クライアントは MS-KRB5、Kerberos、NEGOEX、NTLMSSP を提示し、Kerberos の AP-REQ を送り、
署名必須のままマウント・一覧・読み取り・書き込みができ、NTLM メッセージは一切送りません。

## 特権サーバーなしの特権ポート {#a-privileged-port-without-a-privileged-server}

Unix で [`--isolate`]({{< relref "/operations/isolation.md" >}}) を使う場合、親プロセスが 445 を
バインドしてリスナーを子プロセスに渡すため、実際に SMB を話すプロセスが特権を必要とすることは
ありません。Windows には `ExtraFiles` がないため、子プロセスが自分でアドレスをバインドします。
分離は同じですが、特権ポートに関する利点は得られません。

## このプロジェクトが作っていないクライアントで検証済み {#verified-against-a-client-this-project-did-not-write}

[go-smb2](https://github.com/cloudsoda/go-smb2) は、CI の `-race` の下で、SMB 経由の 20 の並行読み取りを、
WebDAV 経由の 20 と同時に実行します。macOS は、`curl` が WebDAV 経由で同じイメージに書き込んでいる間に
SMB で共有をマウントし、WebDAV が書いた内容を読み戻します。

## 人が変わるとき {#when-the-people-change}

SMB は、共有に接続できる人を**共有が追加されたとき**に固定し、ツリー接続ごとに 1 回検査します。
そのため、[ディレクトリの再読み込み]({{< relref "/administration/reload.md" >}})で見つかった新しい人は、
SMB の稼働中のサーバーを含めてその場で追加され、どの接続にも触れません。しかし、何かが取り除かれた場合
(人、認証情報、共有の展開済みリスト)は新しい世代となり、SMB の開いている接続は閉じられるので、
失効はすでに開いているセッションにも及びます。`allow` のグループが空になった共有は、
**SMB ではまったく提供されません**。空の `AllowUsers` は全員と解釈されてしまうためです。

## 処理中のリクエスト {#requests-in-flight}

go-filesystems/smb v0.8.1 以降、1 つの接続は最大 32 個のリクエストを同時に処理し、サーバー全体では全接続で最大 128 個です。応答は `MessageId` で対応付けられます。通常のファイルの `READ` 1 つを含むフレームは他と並んで実行されます。複合チェーン(全体をまとめて順番に応答)、非同期リクエスト、`WRITE` は単独で実行されます。フレームは受信が始まってから 30 秒以内に届く必要があります。この仕組みは共有モジュール go-filesystems/inflight で、SFTP と NFS も使っています。
