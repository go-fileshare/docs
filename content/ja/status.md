---
title: "ステータス"
weight: 60
description: "このページが説明している fileshare のリリース、各リリースで追加された内容、実際のクライアントで検証済みのもの、まだ実装されていないもの。"
tags: [ステータス, リリース]
---

このページは **fileshare v0.31.2** について説明しています。

{{< callout type="info" >}}
**リリースバイナリを `govulncheck -mode binary` で確認する**

v0.31.2 から、リリースバイナリは `-s -w` ではなく `-ldflags "-w"` でビルドされます。DWARF デバッグ情報は削除され、シンボルテーブルは残ります。`govulncheck` はこのテーブルを読むため、ダウンロードしたバイナリでは、実際にリンクされているパッケージの脆弱性だけが報告されます。

```sh
govulncheck -mode binary ./fileshare
```

シンボルが取り除かれたバイナリでは、`govulncheck` はモジュール単位の判定に戻り、リンクされていないパッケージの脆弱性（たとえば GO-2026-5932、`golang.org/x/crypto/openpgp`）も報告します。バイナリは 7〜10 % 大きくなります。機能の変更はありません。
{{< /callout >}}

{{< callout type="warning" >}}
**セキュリティ: v0.31.0 ではなく v0.31.1 を使ってください**

Go 1.27.2 では、`govulncheck` が fileshare で GO-2026-6809 に到達可能と報告します。`crypto/x509` の `Certificate.Verify` が、既知の侵害された中間証明書を拒否していませんでした。Go 1.26.10 と 1.27.3 で修正されています。**より古い Go でビルドした v0.31.0 のリリースバイナリは影響を受けます。** v0.31.1 は Go 1.27.3 でビルドされ、`go` ディレクティブも 1.27.3 なので、`go install` は自動的に修正済みのツールチェーンに切り替わります。リリースバイナリは `go version -m` で `go1.27.3` と表示されます。三つの UI も同じ方法で再ビルドされています（portal v0.3.2、console v0.3.5、explorer v0.3.2）。変わったのはツールチェーンだけで、動作は変わりません。
{{< /callout >}}

## 各リリースで追加された内容 {#what-each-release-added}

| | |
|---|---|
| v0.7.0 | [`kerberos` ブロック]({{< relref "/protocols/nfs.md#the-refusal-that-used-to-be-permanent" >}})による、NFS 経由の制限付き共有 |
| v0.8.0 | gRPC による[管理 API]({{< relref "/administration/_index.md" >}})、[ディレクトリ共有]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})、[`/healthz`、`/readyz`、`/metrics`]({{< relref "/administration/health.md" >}}) |
| v0.9.0 | WebDAV、S3、NFS 向けの[TLS]({{< relref "/security/tls.md" >}})(ファイルまたは ACME)、**パスワードを使う WebDAV の平文提供を拒否** |
| v0.10.0 | [ディレクトリの再読み込み]({{< relref "/administration/reload.md" >}}): `reload = "5m"`、`SIGHUP`、`ReloadDirectory` |
| v0.11.0 | [IdP が身元を保証する人向けの SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}): その SSH CA と opkssh |
| v0.12.0 | 失効: プロバイダーの SSH 証明書向けの [KRL]({{< relref "/security/revocation-lists.md" >}})、[証明書による NFS の ID]({{< relref "/security/nfs-certificates.md" >}})向けの CRL |
| v0.13.0 | [共有シグナル]({{< relref "/security/shared-signals.md" >}}): CAEP の `session-revoked` により、プロバイダーがそれ以前に発行したすべてのものが、全プロトコルで無効になる |
| v0.14.0 | セキュリティレビューによる修正: [プロトコルが扱える共有名]({{< relref "/administration/_index.md#what-it-will-and-will-not-touch" >}})、設定ファイルやそのシークレットを含む共有の禁止、起動時の `source_roots` の再検査、SSF の `max_age` ≥ 1m と `retain` ≥ 169h |
| v0.15.0 | [`ssh_krl_url` から取得する KRL は署名と日付が必須]({{< relref "/security/revocation-lists.md#signed-and-dated-from-ssh_krl_url" >}}) |
| v0.16.0 | 失効リストは[決して巻き戻らない]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}})。`ssh_krl_state_file` / `crl_state_file` により再起動をまたいでも保たれる。トラストアンカーとリストファイルを共有の内側に置くことは不可 |
| v0.16.1 | Go 1.26.6 が必須に: 1.26.4 には到達可能な標準ライブラリのアドバイザリが 9 件あった |
| v0.16.2 | [NFS の CRL は、解析する前に署名が検証される]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}) |
| v0.16.3 | [未検証のメールアドレスで人を特定することはなくなった]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}) |
| v0.16.4 | テストのみ |
| v0.16.5 | テストのみ([#36](https://github.com/go-fileshare/fileshare/issues/36): テストがトランスポートエラーを成功と読み違えていた) |
| v0.16.6 | NFS の拒否メッセージが 2 つの解決策を示すように: `kerberos` ブロック、または [`identity = "certificate"`]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.16.7 | go-authn/oidc v0.2.2: RSA 鍵のサイズをビット単位で数えるようになった。以前は、ゼロのオクテットで埋めた 1024 ビットの鍵でも、WebDAV トークンや opkssh PK Token が検証を通っていた |
| v0.17.0 | セキュリティ監査による修正: 1 つの共有が別の共有を含むことの禁止、アクセスファイル(`authorized_keys_file`、`dsn_file` など)を共有の内側に置くことの禁止、ディレクトリの読み取りが空だったために WebDAV が匿名になることの防止、S3 が `protocols` に従うように、30 秒以内に認証しない接続のクローズ、opkssh セッションの `opkssh_max_age` での終了、失効したセッションは失効したままとなりクローズされる |
| v0.17.1 | go-authn/oidc v0.2.4(http へのリダイレクト先から鍵セットを取得しない)、servercert v0.3.0(ACME キャッシュのパスを sshd の StrictModes と同様に検査)、krl v0.5.0、revocation v0.3.0 |
| v0.17.2 | go-filesystems/s3 v0.3.0: S3 は、認証されていない呼び出し元にどのアクセスキーが存在するかを教えなくなった。署名付き URL が機能し、有効期間は最長 1 週間、自身の日付より前のものは拒否される |
| v0.18.0 | CI が `stable` ではなく Go 1.27.1 に固定された |
| v0.18.1 | `oidc` ブロックに `domains` がない場合、`fileshare check` が警告する: [プロバイダー側の素の名前が、同名のローカルアカウントに届いてしまう]({{< relref "/configuration/identity.md" >}}) |
| v0.19.0 | go-authn/directory v0.11.0: `users "ldap"` ブロックは[別のマシンへの平文バインド]({{< relref "/configuration/identity.md" >}})を拒否する。パスワードはダイジェストとして、定数時間で比較される |
| v0.20.0 | [`local_names`]({{< relref "/configuration/identity.md" >}}): プロバイダーの名前がローカル名として扱われるのは、`oidc` ブロックがそう指定したときだけ(セキュリティ監査 F4) |
| v0.21.0 | [ボリューム]({{< relref "/administration/volumes.md" >}}): 特権を持つ `fileshare provisioner` が ZFS データセット、btrfs サブボリューム、XFS/ext4 プロジェクトクォータのディレクトリを作成し、管理 API がそれらから共有を提供する。満杯のディレクトリ共有は全プロトコルで 1 つの応答になる。管理ソケットの `allowed_uids` |
| v0.21.1 | [満杯の共有]({{< relref "/administration/volumes.md#a-full-share" >}})は、SMB(`STATUS_DISK_FULL`)と NFS(`NFS3ERR_NOSPC` / `NFS3ERR_DQUOT`)でそのことを伝える |
| v0.22.0 | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}): SSH 証明書は、そのドメイン付与(`ssh-domain-grant@core.aai.geant.org`、EuroHPC SSH CA プロファイル)がこのホストを指名している場合にのみ受け入れられる(フェイルクローズ)。`ssh_accept_ungranted` で付与のない証明書も受け入れる |
| v0.22.1 | go-filesystems/sftp v0.5.1: [アドレスに固定された]({{< relref "/protocols/sftp.md#a-certificate-pinned-to-an-address" >}})証明書(`source-address`)は、`ssh_domains` の下、およびプロバイダーと OpenPubkey の証明書で、そのアドレスからの接続が受け入れられる。以前はすべてのアドレスから拒否されていた |
| v0.22.2 | テストとドキュメントのみ: `source-address` を `::1` と 2 つ目の IPv4 アドレス(`127.0.0.2`)から、`ssh_domains` の下で、また OpenPubkey 証明書でテスト。Linux では `FILESHARE_REQUIRE_JUDGE` の下で、judge がないとスキップではなく失敗する |
| v0.23.0 | [btrfs ボリューム]({{< relref "/administration/volumes.md#the-size-a-client-sees" >}})は、NFS、SMB、WebDAV でクォータのサイズとして見える(go-filesystems/nfs v0.7.0、smb v0.5.0、webdav v0.3.0 が問い合わせのたびにサイズを報告)。以前の btrfs はファイルシステム全体を表示していた。共有を管理 API から受け取るサーバーが、NFS を提供していてまだ共有がない場合でも、起動時に停止しなくなった |
| v0.24.0 | 読み取りの高速化: ディレクトリまたはボリューム共有の WebDAV GET は、TLS を使わない平文 HTTP では `sendfile(2)` で送出される。NFS は 1 MiB 単位(rtmax/wtmax)で読み書きし、SMB クライアントは 1 リクエストで 64 KiB を超えて読めるようになった(`CAP_LARGE_MTU`)。CI ランナー上の 256 MiB ファイルでは、WebDAV 6.6 GB/s、SMB 0.84、NFS 1.47(クライアント 1 台) |
| v0.24.1 | セキュリティ: アイドル状態の NFS 接続が、最大の呼び出しに必要だった 3 MiB を保持し続けなくなった(アイドル接続 24 本で 59 MiB を保持していた)。SMB のフレームヘッダーが、バイト列が届く前に通知された 8 MiB をサーバーに確保させなくなった。SMB リクエストが 1 クレジットで 4 GiB を支払えなくなった(go-filesystems/nfs v0.8.1、smb v0.6.1 と v0.6.2) |
| v0.25.0 | SFTP が `limits@openssh.com` に応答するようになり、OpenSSH の `sftp` が最大 255 KiB(+25%)単位で読めるようになった。1 セッションが開けるハンドルは最大 1024(go-filesystems/sftp v0.6.0) |
| v0.26.0 | サーバーサイドコピーに対応するすべてのプロトコルで実現: WebDAV の `COPY`、SFTP の `copy-data`(OpenSSH の `sftp cp`)、SMB の COPYCHUNK(エクスプローラー、Linux の `copy_file_range`)は、ホストのファイル間で `copy_file_range(2)` により、サーバー側で行われる。btrfs と XFS ではリフリンクとなり、CI の計測では 8 MiB のコピーで使用量 0 バイト。セキュリティ: WebDAV の `COPY` とすべての S3 `GET`/`HEAD` がファイル全体をメモリに読み込まなくなった。S3 は任意のサイズのオブジェクトをストリーミングする。訂正: v0.24.0 の WebDAV `GET` の sendfile は読み取り専用の共有でしか動作していなかった(ラッパーが書き込み可能な共有では覆い隠していた)。v0.26.0 以降は両方で動作する |
| v0.27.0 | ACME: 証明書は最初の接続時ではなく**起動時**に要求される。1 分後に再試行し、最大 1 時間まで倍々にしつつ、サーバーは提供を続ける。**SNI を送らない**クライアント(IP アドレスでの接続)には、ハンドシェイクの失敗ではなく最初のドメインの証明書が渡される(go-authn/servercert v0.5.0)。[HARICA]({{< relref "/security/tls.md#harica-gant-tcs-which-account-and-caa" >}}): チャレンジなしで発行できるのは Enterprise **Admin**(OV)アカウントだけで、CAA が `harica.gr` を許可している必要がある |
| v0.28.0 | 管理 API が[OIDC トークン向けの HTTPS]({{< relref "/administration/_index.md#over-https-for-oidc-tokens" >}})でも提供されるようになった。Connect、gRPC-Web、gRPC を 1 つのリスナーで、サーバーごとに 1 つのオーディエンス、発行者ごとにサブジェクトまたはグループで呼び出し元を指定する。Web UI とネイティブ UI への第一歩 |
| v0.28.1 | セキュリティ: Go 1.27.2 と golang.org/x/net v0.60.0 でビルド。GO-2026-6617(サーバーをクラッシュさせうる HTTP/2 HPACK のレース)、GO-2026-6613、-6612、-6611、-6610、-6603 を修正。HTTPS 上の管理 API は HTTP/2 を話す。`go install` には Go 1.27.2 が必要になり、自動的に切り替わる |
| v0.29.1 | [リロード]({{< relref "/administration/reload.md" >}})は、ディレクトリ上で空になったグループを、削除されたグループと同様に扱う。何も許可せず、リロードの注記に `group @x has no members: it grants nothing` と出力される(空のグループと存在しないグループを区別する go-authn/directory v0.12.0) |
| v0.29.2 | **NetBSD(amd64)と OpenBSD(arm64)向けバイナリ**: fileshare は NetBSD でビルドできなかった。memguard と OpenPubkey 経由で取り込む awnumar/memcall v0.1.2 が、NetBSD にないフラグ `MADV_DONTDUMP` を使っていたためである。memguard v0.23.0 と memcall v0.4.0 が上流で修正した。すべてのリリース対象プラットフォームが、プルリクエストのたびにクロスビルドされるようになった。v0.29.1 以降のすべてのリリースには、プラットフォームごとの静的バイナリが `SHA256SUMS` と出所証明とともに[リリースページ](https://github.com/go-fileshare/fileshare/releases)に添付されている |
| v0.29.0 | WebDAV: [ルートへの `PROPFIND` が共有を一覧する]({{< relref "/protocols/webdav.md#the-root-and-uploads-in-pieces" >}})(207。Depth infinity は 403、書き込みは 405)ため、ファイルマネージャーでサーバー全体をマウントできる。アップロードは分割でき、各 `Content-Range` は前のものが終わった位置に追記される(go-filesystems/webdav v0.6.0)。[ファイルエクスプローラー]({{< relref "/administration/_index.md" >}})に必要なもの |
| v0.30.0 | go-authn/directory v0.14.0: [`users "ldap"` ブロック]({{< relref "/configuration/identity.md" >}})は、go-authn/authnd v0.11.0 が公開するものを既定で読み取る。人は `(\|(objectClass=posixAccount)(objectClass=account)(objectClass=uidObject))`、グループは `(\|(objectClass=posixGroup)(objectClass=groupOfNames))`、`member` の DN は、人の検索が返した誰かの DN である場合にのみ数えられる。⛔ 組織自身のディレクトリに対しては、既定でより多くのエントリが読み取られる。明示的な `user_filter` または `group_filter` は変わらない |
| v0.31.0 | [各人として提供されるディレクトリ]({{< relref "/configuration/as-each-person.md" >}}): `identity = "system"` と `identity` ブロック。root で起動した `fileshare serve` は、人と共有ごとに、その uid、gid、グループで動作するワーカーを起動し、プロトコルは `service_user` として提供する。SFTP と WebDAV のみ。uid 0、`min_uid` 未満の uid、`deny_uids` は拒否される |
| v0.31.1 | セキュリティ: GO-2026-6809（`crypto/x509`: `Certificate.Verify` が既知の侵害された中間証明書を拒否していなかった）を修正する Go 1.27.3 でビルド。`govulncheck` が到達可能と報告した。`go install` は Go 1.27.3 を要求し、自動的に切り替わる。依存関係の更新 |
| v0.31.2 | リリースバイナリを `-s -w` ではなく `-w` でビルド。シンボルテーブルが残るため、ダウンロードしたバイナリに対する `govulncheck -mode binary` が正確になる。バイナリは 7〜10 % 大きくなる。機能の変更なし |

{{< callout type="warning" >}}
**v0.21.0 へのアップグレード**

**満杯のディレクトリ共有**の応答が変わります。`EDQUOT`(クォータ)と
`ENOSPC` は、ボリュームだけでなくすべてのディレクトリ共有で、1 つのエラー
「no space left on device (the share is full)」にまとめられました。以前は WebDAV の
`500`、NFS の `NFS3ERR_IO` だった `EDQUOT` が、`507` と `NFS3ERR_NOSPC` になり、
SFTP のテキストはパスを含まないその文になります。SMB は引き続きアクセス拒否を返します。
[満杯の共有]({{< relref "/administration/volumes.md#a-full-share" >}})を参照してください。
{{< /callout >}}

{{< callout type="warning" >}}
**v0.20.0 へのアップグレード**

プロバイダー側の `alice` は、ローカルの `alice` の共有を取得できなくなりました。
プロバイダーの名前がそのままローカル名であるサイトは、`oidc` ブロックに
`local_names = true` を追加します(あわせて `domains` も設定します)。これがないために
拒否された名前ごとに、サーバーはその旨のログ行を出力します。
{{< /callout >}}

{{< callout type="warning" >}}
**v0.19.0 へのアップグレード**

別のマシンの `ldap://` を指し、`start_tls = true` のない `users "ldap"` ブロックは
**起動しなくなりました**。`ldaps://` または StartTLS を使用してください。
{{< /callout >}}

{{< callout type="warning" >}}
**v0.17.0 へのアップグレード**

ある共有の内側に別の共有がある設定、1 つのソースに 2 つの共有がある設定、
共有の内側にアクセスファイル(`authorized_keys_file`、`dsn_file`、`bind_password_file`、
`ssf` の `ca_file`)がある設定は、**起動しなくなりました**。エラーには
共有とパスが示されます。
{{< /callout >}}

{{< callout type="warning" >}}
**v0.9.0 より前からのアップグレード**

他のマシンから到達できるアドレスで、誰かが認証する状況において、TLS なしで WebDAV を
提供する設定は**起動しなくなりました**。`tls = true` と `tls` ブロックを追加するか、
TLS を終端するプロキシの背後であれば `plaintext = true` を指定してください。
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}})を参照してください。
{{< /callout >}}

## 検証済み {#verified}

以下は、このプロジェクトが作っていないクライアントに実際に行わせたことであり、
コードに関する主張ではありません。

- **macOS** は、`curl` が WebDAV 経由で同じイメージに書き込んでいる間に SMB で共有を
  マウントし、WebDAV が書いた内容を読み戻します。読み取りしか許されていない共有への
  Bob の WebDAV 書き込みは `403`、誤ったパスワードは `401` です。
- **go-smb2**(このプロジェクトが作っていないクライアント)は、CI の `-race` の下で、
  SMB 経由の 20 の並行読み取りを、WebDAV 経由の 20 と同時に実行します。
- アクセスルールは、設定だけでなく**すべてのプロトコルを通して**検査されます。
  alice は書き込め、bob は書き込めません。これは SMB でも WebDAV でも同じで、
  NFS にはそもそも共有が提供されません。
- SFTP 証明書は **OpenSSH 自身のクライアント**で検証されます。証明書を脇に移すと、
  同じ鍵も拒否されることも確認されています。
- 証明書による NFS の ID は、go-filesystems/nfs の CI で**実際の Linux クライアント**
  (カーネル 6.17、ktls-utils 0.9)を使って計測されました。
  [平文の `MNT` と `tlshd` の落とし穴]({{< relref "/security/nfs-certificates.md#measured-with-a-real-linux-client" >}})
  はこれで見つかりました。
- KRL の署名なし形式は計測結果です。OpenSSH 9.6 と 10.3 は署名付き KRL を読み、
  署名をスキップします(go-authn/krl、両方に対して)。
- **go-authn/bridge の相互運用レーン**([#12](https://github.com/go-authn/bridge/pull/12)、
  [#17](https://github.com/go-authn/bridge/pull/17))は、fileshare v0.13.0 を本物の
  bridge に対して判定します。KRL、NFS 上の CRL、共有シグナルで失効した WebDAV トークン、
  スコープで無効化された IdP、共有シグナル上の opkssh です。

## まだ実装されていないもの {#not-yet}

**S3 での書き込み。** `PUT` と `DELETE` は 403 を返します。ライブラリは書き込めますが、
読み取りしか許されていない人の共有は、SFTP と同じ場所で拒否しなければならず、
それがまだ組み込まれていません。中途半端に書かれたオブジェクトよりも、拒否するほうが安全です。

**S3 での OIDC トークン。** トークンが使えるのは [WebDAV]({{< relref "/protocols/webdav.md" >}})
だけです。S3 の通常の方法は STS の `AssumeRoleWithWebIdentity` で、
トークンを一時的な認証情報と交換し、クライアントがそれで署名します。
これはベアラートークンを受け入れるのとは別の仕組みであり、実装されていません。

**`admin` ブロックまたは `metrics` ブロックがある場合の `--isolate`。** 拒否されます。
API の変更を適用できる単一のプロセスが存在しないためです。
[プロトコルごとに 1 プロセス]({{< relref "/operations/isolation.md#not-with-an-admin-or-a-metrics-block-yet" >}})を参照してください。

**共有 Linux NFS クライアントでの、証明書によるユーザーごとの人の特定。** Linux
クライアントは証明書をマウントに紐づけるため、そのマウントのすべてのユーザーが、
証明書が指名する人として扱われます。これはクライアントの性質であり、
その場合の答えは `sec=krb5` です。
[保証できないこと]({{< relref "/security/nfs-certificates.md#what-it-cannot-promise" >}})を参照してください。

[S3 自体]({{< relref "/protocols/s3.md" >}})は提供済みです。共有はバケットであり、
SFTP が使うのと同じユーザーごとのツリー上で提供されます。

## 計測済みであり、計測結果として記載しているもの {#measured-and-stated-as-measured}

このページの主張のうち、不具合の再現ではなく保険や計測にあたるもので、
ソースにもそのように書かれているものです。

- ラップされていない fat32 ドライバーに対して 8 つの goroutine が `-race` の下で書き込んでも、
  現時点ではレースは発生しません。[共有ロック]({{< relref "/operations/locking.md" >}})は
  契約に対する保険であり(`go-filesystems/interface` は、パスベースの並行呼び出しについて
  何も保証していません)、観測された障害の修正ではありません。
- [ビルドタグごとのサイズ]({{< relref "/operations/build-tags.md" >}})は計測されました。
  `hashicorp/go-plugin` だけで 13.2 MB かかり、これがプラグインフレームワークを採用しない
  決め手になりました。また、管理 API が持ち込む gRPC は 11.7 MB で、これが `nogrpc` が
  存在する理由です。
- 独自のグループタグ証明書拡張を設計して Go の X.509 パーサーで**計測**したところ、
  証明書全体が拒否されました。そのため、グループは代わりに
  [タグ URI]({{< relref "/security/nfs-certificates.md#what-the-certificate-carries" >}})
  として運ばれます。
