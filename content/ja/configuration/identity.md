---
title: "ユーザー、グループ、ディレクトリ"
weight: 20
description: "人がどこから来るのか(user・group ブロック、SQL、LDAP、ID プロバイダー)と、各ソースで何を利用できるか。"
tags: [設定, ID, oidc, ldap, sql]
---

`user` ブロックは、世帯程度の規模であればそれだけでディレクトリのすべてになります。
人がすでにデータベースや LDAP にいるサイトでは、時間が経つと古くなる 2 つ目の場所に
コピーすべきではないため、`users` ブロックが人を元の場所から読み取ります。

```hcl
users "sql" {
  driver   = "postgres"                 # or sqlite, or mysql
  dsn_file = "/etc/fileshare/dsn"       # a DSN holds a password: it lives in a file
  users    = "select login, null, nt_hash, ssh_keys from staff"
  groups   = "select team, member from team_members"
}

users "ldap" {
  url                = "ldaps://ldap.example.org"
  base_dn            = "ou=people,dc=example,dc=org"
  bind_dn            = "cn=reader,dc=example,dc=org"
  bind_password_file = "/etc/fileshare/bind.pw"
}
```

**クエリはお客様が書きます。** サイトの人々はすでにそのサイトなりの形で存在しており、
ここで独自のスキーマを定めれば、それを別の形にコピーさせることになるからです。
人を取得するクエリの列は、名前ではなく**位置**で読み取られます。まず名前、続いて
パスワード、NT ハッシュ、SSH 鍵(1 行に 1 つ)、TOTP シークレットのうち任意のものを、この順序で並べます。
NULL は、その人が持っていない認証情報を表します。例でパスワードに `null` を選択しているのは
そのためです。パスワードの列はパスワードそのものとして扱われます。ハッシュ化された
パスワードを指定する設定はありません。

LDAP 側は、Samba 対応のディレクトリがすでに公開しているもの、すなわち
`sambaNTPassword`、`sshPublicKey`、`memberUid` を読み取ります。人とグループの場所、
およびそれらを名指しする属性は設定可能です。`user_filter`
(既定は `(|(objectClass=posixAccount)(objectClass=account)(objectClass=uidObject))`)、
`user_attribute`(`uid`)、`group_base_dn`(`base_dn`)、`group_filter`
(`(|(objectClass=posixGroup)(objectClass=groupOfNames))`)、`group_attribute`
(`cn`)、`group_member_attribute`(既定では `memberUid` と `member` の両方)、
そして既定値のない `totp_attribute` です。`sambaNTPassword` と `sshPublicKey` は、
その名前で読み取られます。`member` の値は DN であり、人の検索が返した誰かの DN である
場合にのみ数えられます。ツリーの別の場所にある `uid=alice` は alice ではありません。
これらの既定値(v0.30.0 以降)は go-authn/authnd v0.11.0 が公開するものを読み取ります。
以前は `(objectClass=posixAccount)` と `(objectClass=posixGroup)` のみで、
組織自身のディレクトリに対しては、現在はより多くのエントリを読み取ります。

{{< callout type="error" >}}
**別のマシンへの平文バインドは不可(v0.19.0 以降)**

`users "ldap"` ブロックは、`start_tls = true` がない限り、別のマシンへの `ldap://` を
拒否します。バインドが確認するすべてのパスワードが、平文でネットワークを横切ることになる
ためです。`ldaps://` は受け付けられ、途中に誰もいないループバックアドレスへの `ldap://` と
`ldapi://` も受け付けられます。これを無効にするスイッチはありません
(go-authn/directory v0.10.0)。
{{< /callout >}}

## 順序と、唯一の例外 {#order-and-the-one-exception}

ソースは**書かれた順序で**問い合わされ、名前を最初に知っていたものがその名前を所有します。
`user` ブロックと `group` ブロックが先頭に来るため、ローカルに書かれたサービスアカウントが、
LDAP にある同名の誰かによって上書きされることはありません。

**グループは例外です。** グループのメンバーは、すべてのソースの**和集合**になります。
チームのメンバーが、ファイルにもデータベースにもいることがあるためです。

```hcl
share "photos" {
  allow   = ["@engineers", "alice"]
  writers = ["@owners"]
}
```

{{< callout type="info" >}}
**起動時に展開され、再読み込みのたびに再展開される**

v0.10.0 以降、`users` ブロック(SQL、LDAP)は `reload = "5m"` ごと、`SIGHUP` 時、
管理 API の `ReloadDirectory` 時に再読み込みされます。LDAP でメンバーシップが変わっても、
再起動を待つ必要はありません。再読み込みでその場で適用されるもの、再起動されるもの、
そしてグループが空になっても共有が全員に開放**されない**理由については、
[ディレクトリの再読み込み]({{< relref "/administration/reload.md" >}})を参照してください。
ディレクトリへの問い合わせは、依然として接続のたびには行われません。それは別の設計であり、
これはそうではありません。
{{< /callout >}}

## ソースが証明できること、各プロトコルが必要とするもの {#what-a-source-can-prove-and-what-each-protocol-needs}

ここは、マウントして初めて気づくことの多い部分なので、[`check`]({{< relref "/configuration/check.md" >}})
が最初に伝えます。

| ソースが持つもの | SMB | S3 | WebDAV | SFTP |
|---|---|---|---|---|
| パスワード(ファイル、または平文の列) | yes | yes | yes | — |
| NT ハッシュ(`sambaNTPassword`、`nt_hash`) | yes | **no** | — | — |
| バインドのみ(LDAP) | **no** | **no** | yes | — |
| 公開鍵、または信頼された CA | — | — | — | yes |

{{< callout type="error" >}}
**NTLMv2 に必要なのはパスワードかその MD4 であり、ほかのものでは代用できません**

クライアントは SMB サーバーにパスワードを送りません。そこから計算した証明を送ります。
そのため、パスワードを*確認*するだけのディレクトリは、確認がどれほど優れていても
SMB には応答できません。これはプロトコルの性質であり、このプログラムの制約ではなく、
設定でどうにかなるものではありません。WebDAV が尋ねるのは「これは正しいパスワードか」だけで、
バインドで答えられます。
{{< /callout >}}

ディレクトリが名前を挙げているのに何も証明できない人は、正当な存在です。一覧だけがあり、
シークレットは別の場所にあるというケースです。`check` は、それを自分で見つけさせるのではなく、
1 行で伝えます。

## この設定ファイルではなく、ID プロバイダーが名指しする人 {#people-the-identity-provider-names-not-this-file}

フェデレーション(たとえば [go-authn/bridge](https://github.com/go-authn/bridge) 経由の RENATER)は、
プロジェクトに誰がいるのかを把握しており、共有はそのリストをコピーするのではなく**フェデレーションに**
尋ねられます。[`oidc` ブロック]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}})を使います。

```hcl
share "photos" {
  image   = "/srv/photos.img"
  allow   = ["oidc:groups:urn:mace:univ-example.fr:photos", "oidc:user:bob@univ-example.fr", "alice"]
  writers = ["oidc:groups:urn:mace:univ-example.fr:photos"]
}
```

`oidc:groups:<value>` は、トークンの groups クレームにその値を持つ人です。
`oidc:user:<name>` は、トークンが名指しする人です。この表記は opkssh のものに合わせてあり、
シェルに到達できる人と共有に到達できる人を、同じ語彙で表せます。ルールが名指しする人は、
プロバイダーが保証する範囲でこのサーバーに知られており、開放された共有も利用できます。
どのルールにも名指しされない人は、依然として部外者です。同じルールは
[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})でも適用されますが、
そこではグループはトークンではなく証明書で運ばれます。

{{< callout type="error" >}}
**ルールが対象とするのは、プロバイダーの人だけです**

`oidc:user:bob` はプロバイダーが保証する bob を指します。パスワードを持つ `bob` という
ローカルアカウントは別人であり、どのルールにも一致しません。`oidc` ブロックのないルール、
形式の不正なルール、接続を許可されていないのに書き込みを許可するルールは、
**起動時に拒否されます**。
{{< /callout >}}

{{< callout type="error" >}}
**明示しない限り、プロバイダーの `alice` はローカルの `alice` ではありません(v0.20.0 以降)**

`allow` や `writers` の素の名前はローカルアカウントを指します。トークンやプロバイダーの証明書が
到達できるのは、共有の `oidc:` ルールだけです。プロバイダーの名前がそのままローカル名であるサイトは、
`oidc` ブロックに `local_names = true` を書きます。その際は `domains` も設定するか、プロバイダーが
管理するクレームから名前を取得してください。人が自分で `preferred_username` を選べるプロバイダーでは、
サインアップした誰にでも、書き込みを含めてローカルアカウントの共有を渡してしまうためです。
これは v0.20.0 より前の既定の動作でした(セキュリティ監査 F4)。`local_names` がないために
拒否されたフェデレーション名は、追加すべき行とともにログに記録され、`fileshare check` は
どのルールが有効かを伝えます。

NFS クライアント証明書(`identity = "certificate"`)は例外です。その CA はまさにそのために
`client_ca_file` が固定したものであり、NFS の共有が人を名指しできるのは素の名前だけなので、
`local_names` の設定にかかわらず、その名前はローカル名になります。
{{< /callout >}}

### どの機関か {#which-institutions}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
  domains  = ["univ-a.fr", "univ-b.fr"]   # nobody else from the federation gets in
}

share "projet-x" {
  image   = "/srv/projet-x.img"
  allow   = ["oidc:groups:urn:mace:univ-a.fr:projet-x", "oidc:domain:univ-b.fr"]
  writers = ["oidc:groups:urn:mace:univ-a.fr:projet-x"]
}
```

`domains` は認証時に、SFTP と WebDAV の両方で、また NFS クライアント証明書についても検査されます。
名前は `<something>@<そのいずれか>` の形でなければならず、全体が比較されます
(`evilunivb.fr` は `univb.fr` ではありません)。`oidc:domain:` は、1 つの共有に対する同じ検査です。
ドメインを信頼できるのは、プロバイダーが信頼できる範囲までです。go-authn/bridge は、IdP の
フェデレーションメタデータがそのスコープを許可していない eppn や subject-id を破棄します。

**グループ**は機関の IdP が公開するもので、go-authn/bridge の `claims { groups = [...] }` により
`groups` クレームに変換されます。既定では `eduPersonEntitlement`(研究室や VO のグループ)、
または `eduPersonScopedAffiliation`(`staff@univ-a.fr`、`student@univ-b.fr`)です。
