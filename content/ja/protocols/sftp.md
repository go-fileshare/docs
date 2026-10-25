---
title: "SFTP — 鍵、または証明書"
linkTitle: "SFTP: 鍵、または証明書"
weight: 30
description: "公開鍵または SSH 証明書を使う SFTP。ローカルとプロバイダーの認証局、OpenPubkey、ドメイン付与、source-address、失効。"
tags: [プロトコル, sftp, ssh]
---

どのクライアントにも最初から備わっています。`sftp` は OpenSSH に同梱され、Finder や GNOME はこれを
マウントでき、エディターもこれを話します。同時に、ここにあるプロトコルのうち唯一、形が合わないものでもあります。
人はログインすると、共有の一覧ではなく**1 つの**ファイルシステムに着地するからです。

そこで、共有は、認証したばかりの人のために構築されたツリーのトップレベルのディレクトリになります。

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

alice が利用できない共有は、**alice に見えるディレクトリではありません**。また、読み取りしか
許されていない共有への書き込みは、それを許してしまうはずのドライバーより手前の、*ツリーの中で*拒否されます。

{{< callout type="info" >}}
**2 つの共有にまたがる名前変更は拒否されます**

それは 2 つのイメージにまたがるコピーと削除になりますが、名前変更が約束するのはどこでもそのようなことでは
ありません。
{{< /callout >}}

## 設定 {#configuration}

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # optional

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # nothing here: her certificate is her credential
```

`trusted_user_ca_file` を指定すると、その認証局が署名し、プリンシパルにそのユーザーを含む証明書だけで
十分になります。そのため、人のアクセス権は**別の場所で発行され、別の場所で失効**し、人が加わったり
去ったりしても、ここのファイルを編集する必要はありません。

署名、有効期間、プリンシパルは `x/crypto/ssh` の `CertChecker` が検査します。**OpenSSH 自身のクライアント**
で検証されており、証明書を脇に移すと、同じ鍵も拒否されることも確認されています。

{{< callout type="warning" >}}
**`host_key_file` がないと、起動のたびに新しい ID が生成されます**

サーバーはそのことを伝え、以前にそのサーバーを見たことのあるクライアントはすべて、鍵が変わったと警告します。
それはクライアントがその役割を果たしているということです。
{{< /callout >}}

## このホスト向けの証明書: ドメイン付与 {#certificates-meant-for-this-host-the-domain-grant}

v0.22.0 以降。[EuroHPC Federation Platform の SSH CA](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-overview/)
(GÉANT / MyAccessID)は、フェデレーションのすべてのサイトが信頼する証明書に署名します。各証明書は、
どのホスト向けかを `ssh-domain-grant@core.aai.geant.org` 拡張で示します。sshd はこの拡張を無視し、
fileshare も v0.22.0 までは無視していたため、他人のマシン向けに付与された証明書もここで通用していました。
`ssh_domains` を指定すると、付与は仕様どおりに [go-authn/sshcert](https://github.com/go-authn/sshcert) が
読み取り、解析し、照合します。付与がこのホストの名前を 1 つも指していない証明書は拒否されます。

```hcl
trusted_user_ca_file = "/etc/fileshare/efp-ssh-ca.pub"
ssh_domains          = ["files.example.org", "sftp.example.org"]
# ssh_accept_ungranted = true   # only if a trusted CA never writes a grant
```

| キー | |
|---|---|
| `ssh_domains` | 付与が名指しする形式での、このホストの名前。証明書の付与のパターンの 1 つがこれらのいずれかに一致すれば、証明書は受け入れられる。`*` は**1 つのラベル内の 1 文字以上**で、`*.example.org` は `files.example.org` を許可するが `a.files.example.org` は許可せず、`files-*.example.org` は `files-.example.org` を許可しない。比較では大文字小文字を区別しない。これらはパターンではなくホスト名であり、それぞれ 1 回だけ記述する。少なくとも 1 つの認証局(`trusted_user_ca_file`、または `oidc` ブロックの `ssh_ca_file`)が信頼されている必要がある。 |
| `ssh_accept_ungranted` | 付与が**ない**証明書を受け入れる。既定ではオフ。`ssh_domains` がない場合は無意味として拒否される。 |

| 証明書 | `ssh_domains` | `+ ssh_accept_ungranted` | `ssh_domains` なし |
|---|---|---|---|
| `ssh_domains` のいずれかを付与されている | 許可 | 許可 | 許可 |
| 他のホストのみ、または `[]` を付与されている | 拒否 | 拒否 | 許可 |
| 付与なし | 拒否 | 許可(ログ記録) | 許可 |
| 解析できない付与(`null`、`[1]`、コンパクトでないものなど) | 拒否 | **拒否** | 許可 |

`ssh_domains` がなければ、何も読み取られず、何も変わりません。付与が読み取られるのは、**認証局**が
署名した証明書、つまり `trusted_user_ca_file` のものとプロバイダー(`ssh_ca_file`)のものです。
OpenPubkey 証明書はユーザー自身の鍵で署名されているため、その中の付与はユーザー自身についての
本人の言葉にすぎません。そのオーディエンスは代わりに `opkssh_client_id` です。通常の鍵は付与を
持たないので、対象外です。

拒否のたびに、証明書のシリアル番号、鍵 ID、理由がログに記録されます。

```
sftp: alice: certificate 42 ("alice@efp") refused: its domain grant [login.example.eu] names none of this host's names [files.example.org sftp.example.org]
sftp: alice: certificate 43 ("staff") refused: it has no domain grant (ssh-domain-grant@core.aai.geant.org), and ssh_domains requires one
```

各判定は `fileshare_sftp_domain_grant_total{result}` にも数えられ、結果は `granted`、
`accepted_ungranted`、`refused_not_granted`、`refused_absent`、`refused_malformed` のいずれかです。
クライアントは、鍵を持っていることを証明する前に証明書を提示するため、これが数えるのは人ではなく
試行です。

{{< callout type="error" >}}
**フェイルクローズである理由**

2 つ目の認証局を信頼するホストを考えてみてください。スタッフ用の自前の CA、テスト用の CA、同じファイルに
追加された EFP のステージング CA、付与のない go-authn/bridge クライアントなどです。これらの認証局の証明書には
付与がありません。付与がないことが「制限なし」を意味するなら、そのホストではフィルターが何もフィルターしません。

ここでは、付与のない証明書は、`ssh_accept_ungranted` が別の指定をしない限り拒否されます。付与が存在しても
解析できない場合は、その内容にかかわらず拒否されます。付与を書いた認証局は、証明書を制限するつもりだった
からです。go-authn/sshcert は
[複数 CA のシナリオ](https://github.com/go-authn/sshcert#why-fail-closed-the-multi-ca-scenario)を
詳しく説明しています。
{{< /callout >}}

### EFP の CA を使う {#with-efps-ca}

`trusted_user_ca_file` は OpenSSH の `TrustedUserCAKeys` です。EFP の
[信頼のページ](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-trust/)にあるとおり、
<https://sshca.my-eurohpc.eu/config> で EFP が公開している鍵を入れます。
その URL は `{"PublicKey": "ssh-ed25519 ..."}` を返し、ファイルには `PublicKey` の値を 1 行に書きます。

```sh
curl -s https://sshca.my-eurohpc.eu/config | jq -r '.PublicKey' > /etc/fileshare/efp-ssh-ca.pub
```

そのうえで、`ssh_domains` に、ホスティング機関の付与が使う名前を設定します。EFP 証明書の唯一のプリンシパルは
MyAccessID の識別子(`<id>@myaccessid.org`)であり、それがログインに使うユーザー名になります。
その名前の `user` ブロックが、自前の認証情報を持たなくても、これを受け付けます。

```hcl
user "u1234@myaccessid.org" {}
```

### go-authn/bridge を使う {#with-go-authnbridge}

`ssh_certificates = true` と `ssh_domain_grants = ["files.example.org"]` を指定した
[go-authn/bridge](https://github.com/go-authn/bridge) クライアント(v0.19.0 以降)は、
このホスト向けに付与された証明書を発行します。`ssh_principal_claim = "voperson_id"` を指定すると、
唯一のプリンシパルは EFP プロファイルと同様に人の `voperson_id` になり、それがここに書く `user` ブロックです。
bridge の CA は、このサーバーが他の CA を信頼するのと同じ方法で信頼します。

- ローカルアカウント向けには `trusted_user_ca_file` として。bridge は `/ssh/config` で EFP 形式の鍵を提供するので、
  上の `curl | jq` の行は bridge の URL でも使えます。
- または、[プロバイダーが身元を保証する人](#people-the-identity-provider-vouches-for)向けには、
  `oidc` ブロックの `ssh_ca_file` として。

bridge の他のクライアントは付与を書きません。それらの証明書をここで使い続ける必要があるなら、
そのための設定が `ssh_accept_ungranted` です。ただしこれは、信頼されたすべての認証局の付与のない証明書を
**すべて**受け入れてしまうため、それらのクライアントに付与を持たせるほうが望ましい方法です。

### アドレスに固定された証明書 {#a-certificate-pinned-to-an-address}

bridge の `ssh_source_address` は、`ssh-keygen -O source-address=...` と同様に `source-address` クリティカル
オプションを書き込みます。これを持つ証明書は、`ssh_domains` の有無にかかわらず、**それらのアドレスからのみ**
受け入れられます。証明書が許可するアドレスからのログインは、`ssh_domains` が設定されている場合はその付与が
このホストも名指ししている限り受け入れられます。それ以外のアドレスからのログインは拒否されます。

| fileshare | `trusted_user_ca_file` | `oidc` の `ssh_ca_file`、および OpenPubkey |
|---|---|---|
| v0.22.0 より前 | 適用される | **すべてのアドレスから拒否** |
| v0.22.0、`ssh_domains` なし | 適用される | **すべてのアドレスから拒否** |
| v0.22.0、`ssh_domains` あり | **すべてのアドレスから拒否** | **すべてのアドレスから拒否** |
| v0.22.1、`ssh_domains` の有無を問わず | 適用される | 適用される |

「すべてのアドレスから拒否」はフェイルクローズです。証明書の制限が緩められたことはなく、失われたのは
ログインだけです。v0.22.1 より前、go-filesystems/sftp の sshd は、fileshare に判断を委ねる証明書に
クリティカルオプションがあるとすべて拒否していました。これはプロバイダーの証明書、OpenPubkey の証明書、
そして `ssh_domains` の下ではすべての証明書に該当します。v0.22.1 は go-filesystems/sftp v0.5.1 を土台に
しており、その sshd はそこで `source-address` を受け入れ、すべての証明書の経路で、接続元のアドレス
(IPv4 または IPv6)に対して適用します。

その他のクリティカルオプション(`force-command`、`verify-required` など)は、このサーバーが
対応しないため拒否されます。

## ID プロバイダーが身元を保証する人 {#people-the-identity-provider-vouches-for}

v0.11.0 以降、`oidc` ブロックは SFTP にも届きます。SSH にはトークンを載せる場所がないため、
トークンではなく**証明書**によってです。

```hcl
oidc {
  issuer           = "https://login.example.org"
  audience         = "fileshare"
  ssh_ca_file      = "/etc/fileshare/bridge-ca.pub"   # go-authn/bridge's ssh_ca
  opkssh_client_id = "opkssh"                         # OpenPubkey logins
  opkssh_max_age   = "24h"                            # 12h, 24h, 48h, 1week
}
```

- **プロバイダーの SSH CA が署名した証明書**。`authn-bridge ssh-cert` が、フェデレーションでの
  ログイン後に書き込みます。プリンシパルがその人で、`groups@go-authn.org` 拡張がそのグループなので、
  [`oidc:groups:` ルール]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})が適用されます。
  これはローカルアカウントのための証明書を扱う `trusted_user_ca_file` では**ありません**。ローカルの認証局の
  証明書がプロバイダーのグループを名乗っても、名指ししているローカルアカウントとして読まれます。
- **OpenPubkey 証明書**。`opkssh login` が書き込むもので、ユーザー自身の鍵で署名され、その鍵にコミットする
  ID トークンを伴います。`opkssh verify` と同じ方法で検査されます。つまり
  [openpubkey](https://github.com/openpubkey/openpubkey) の検証器(プロバイダーの署名を公開鍵に対して、
  nonce のコミットメント、クライアント ID、経過時間)、続いて証明書の鍵をトークンに対して検証します。
  SSH のユーザー名はトークンのユーザー名でなければなりません。ここには両者を対応付ける `auth_id` ファイルが
  ないためです。`sftp alice@univ-example.fr@files.example.org` のようになります。
  `opkssh_client_id` は専用のクライアント ID にし、他のアプリケーションのものを使ってはいけません。
  ID トークンは、その人がログインするすべてのサーバーに渡されるためです。

証明書が運ぶグループを確認するには、次のようにします。

```sh
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub     # the Extensions section
```

また、サーバーはフェデレーションされたログインのたびに
`sftp: alice@univ-a.fr, vouched for by the provider, in groups [...]` と出力します。

{{< callout type="error" >}}
**身元が保証されることは、受け入れられることではありません**

プロバイダーが身元を保証する人であっても、ルールが名指ししているか、`trust_all` でプロバイダーがそのまま
ディレクトリだと明示しているか、`local_names = true` でプロバイダーの名前がこのサーバーの名前であり
ローカルアカウントが同じ名前を持つと明示していない限り、ここでは依然として部外者です。これは WebDAV で
トークンが通過するのと同じ検査です。**プリンシパルがない**プロバイダーの証明書は、形式の定義上
*誰にでも*有効であるため、拒否されます。
{{< /callout >}}

`-tags noopenpubkey` は OpenPubkey の検証器を除外します(約 2 MB)。そのようなビルドで
`opkssh_client_id` を指定した設定は拒否されます。

## 証明書を取り消す {#taking-a-certificate-back}

{{< callout type="error" >}}
**証明書はログイン時に検査されます**

プロバイダーでその人を失効させても、それだけでは発行済みの証明書は失効しません。証明書は期限が切れるまで
SFTP を開き、すでに開いている SFTP セッションもそれまで開いたままです。これらの人はディレクトリにいないため、
[再読み込み]({{< relref "/administration/reload.md" >}})の対象にはなりません。この期間は証明書の有効期間で、
bridge の `ssh_ca { validity }`(既定では 12h、最大 168h、IdP セッションの終了を超えない)と、
ここでの `opkssh_max_age` です。クライアントの再ログインが許す範囲で、できるだけ短くしてください。
v0.17.0 以降は、すでに開いている opkssh セッションも、証明書(本人が自分で署名するもの)が何と言おうと、
`opkssh_max_age` で終了します。
{{< /callout >}}

この期間を閉じる仕組みが 2 つあります。

- プロバイダーの **[KRL]({{< relref "/security/revocation-lists.md" >}})**(`ssh_krl_url`):
  失効した証明書はログイン時に拒否され、その証明書が開いたセッションのすべての操作が、開いているファイルを
  含めてリストを再度参照します。
- **[共有シグナル]({{< relref "/security/shared-signals.md" >}})**(`ssf`): CAEP の
  `session-revoked` より前にプロバイダーがその人に発行したすべてのものが拒否されます。
  どのリストにも載らない OpenPubkey 証明書に到達できる唯一の方法です。

どちらもフェイルクローズです。

## パスワードなし {#no-password}

パスワードを尋ねるクライアントは、鍵が避けるために存在するものそのものを行っています。

## 転送サイズと開いているファイル {#transfer-size-and-open-files}

fileshare は OpenSSH の `limits@openssh.com` に応答する(v0.25.0 以降)ため、OpenSSH の `sftp` は
1 リクエストあたり 32 KiB ではなく最大 255 KiB を読み書きします。OpenSSH 10.3 での計測で約 +25% です。
1 つのセッションが同時に保持できる開いたファイルとディレクトリは最大 1024 で、ディレクトリ共有では
それぞれがホストのディスクリプターであり、この上限によって 1 人のユーザーがそのすべてを使い切るのを防ぎます。
