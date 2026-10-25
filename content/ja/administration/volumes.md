---
title: "ボリューム: fileshare が作成するストレージ"
weight: 10
description: "特権を持つ fileshare プロビジョナーを通じて ZFS、btrfs、XFS、ext4 のストレージを作成し、そこから共有を提供する方法。"
tags: [管理, ボリューム, zfs, btrfs, クォータ]
---

v0.21.0 以降、[管理 API]({{< relref "/administration/_index.md" >}})は、共有が提供する**ストレージを作成**し、
そこから共有を提供できます。対象は **ZFS データセット**、**btrfs サブボリューム**、
**プロジェクトクォータ下の XFS または ext4 ディレクトリ**で、サイズ指定、所有者設定、追跡が行われます。
Linux のみです。Ceph はまだサポートされていません。

そのストレージの作成には `CAP_SYS_ADMIN`(quotactl、btrfs クォータの ioctl、`/dev/zfs`、mount(2))が必要です。
`fileshare serve` は見知らぬ相手に対して 5 つのネットワークプロトコルを解析するため、特権を持ちません。
特権を必要とする部分は、同じバイナリの 2 つ目のロールである `fileshare provisioner` が担当し、
Unix ソケットと、小さく閉じたプロトコルの背後で動作します。

```
                         admin gRPC (unix socket 0600 + peer uid, or mTLS)
  operator ─────────────────────────────────────────────▶  fileshare serve
                                                             (unprivileged)
                                                                  │
                     provision gRPC (unix socket, SO_PEERCRED = fileshare's uid)
                                                                  ▼
                                                    fileshare provisioner
                                                    (CAP_SYS_ADMIN, CAP_CHOWN, CAP_FOWNER)
                                                                  │
                                    go-fsctl: zfs · btrfs · projquota (ioctls, no CLI)
```

2 つのデーモンではなく 1 つのバイナリなので、デプロイするバージョンは 1 つ、プロトコル定義も 1 つです。
ストレージ自体の操作は [go-fsctl](https://go-fsctl.github.io/) が行います(純粋な Go で、
`zfs` や `btrfs` コマンドは使いません)。設計と、構築中に変わったことは、fileshare リポジトリの
[`docs/volumes.md`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/docs/volumes.md)にあります。

## 両方のプロセスの設定 {#both-processes-configured}

`fileshare serve` を、ユーザー `fileshare`(uid 990)として実行する場合:

```hcl
# /etc/fileshare/fileshare.hcl
admin {
  listen          = "unix:///run/fileshare/admin.sock"
  state_file      = "/var/lib/fileshare/shares.json"
  source_roots    = ["/srv/fileshare/volumes"]
  provisioner     = "unix:///run/fileshare-provisioner/provisioner.sock"
  provisioner_uid = 0          # who must answer on that socket; 0 is the default
  allowed_uids    = [990]      # who may call this API
}
```

| キー | |
|---|---|
| `provisioner` | プロビジョナーのソケット、`unix:///path`。これがないと、ボリュームの呼び出しはすべて `FAILED_PRECONDITION` で応答され、ボリュームから共有は作られない |
| `provisioner_uid` | そのソケットで応答するプロセスが実行されているべき uid。接続のたびにカーネル(`SO_PEERCRED`)から読み取られる。他者がバインドしたソケットには何も渡されない。未設定なら 0 |
| `allowed_uids` | 省略可能、Unix ソケットのみ: 呼び出しに応答する唯一の uid。それ以外は `PERMISSION_DENIED` となり監査される。ソケットの 0600 モードが許すもの(その所有者と root)を**狭める**ことしかできず、たとえば root が呼び出せなくなることもある。クライアント証明書が検査となる TCP では拒否される |

ボリュームのパスは、ほかのソースと同様に `source_roots` の下になければなりません。

`fileshare provisioner` を root として実行する場合:

```hcl
# /etc/fileshare-provisioner.hcl
provisioner {
  listen     = "unix:///run/fileshare-provisioner/provisioner.sock"
  client_uid = 990
  group      = "fileshare"
  max_volume = "10T"
  state_file = "/var/lib/fileshare-provisioner/volumes.json"

  parent "tank" {
    zfs  = "tank/fileshare"
    root = "/srv/fileshare/volumes/tank"
  }
  parent "fast" { btrfs = "/srv/fileshare/volumes/fast" }
  parent "plain" {
    xfs         = "/srv/fileshare/volumes/plain"
    project_ids = "100000-199999"
  }
}
```

| キー | |
|---|---|
| `listen` | `unix://<絶対パス>` のみ。ソケットは 0660 で、プロビジョナーのユーザー(root)と `<group>` の所有として作成され、プロビジョナー以外は書き込めないディレクトリに置かなければならない。fileshare のものではなく、プロビジョナー**専用**のディレクトリ(`/run/fileshare-provisioner`)を使う |
| `client_uid` | 応答する唯一の uid、つまり fileshare のもの。**0 は拒否される** |
| `group` | すべてのボリュームのルートを所有する(`root:<group>`、モード 2770)。fileshare はこのグループに属していなければならない。ボリュームへの書き込みは、所有者としてではなくグループ経由で行う。ディレクトリの所有者は特権なしでそのプロジェクト ID を消去でき、XFS/ext4 のクォータから抜け出せてしまうため |
| `max_volume` | 1 つのボリュームに設定できる最大のクォータ(`"10T"`、`"500G"`、`"1048576"`)。これを超えると `OUT_OF_RANGE` |
| `state_file` | このプロセスが作成したもの(btrfs のサブボリューム ID、XFS/ext4 のプロジェクト ID)。再起動後に自分のものを把握するために使う |
| `parent "<id>"` | ボリュームを作成できる場所で、リクエストが名指しする ID の下に置かれる。`zfs`(ボリュームがその子になるデータセット。`root` はマウント先)、`btrfs`、`xfs`、`ext4`(ボリュームを作成するディレクトリ)のうち、ちょうど 1 つ |
| `project_ids` | XFS/ext4: この parent が割り当てられるプロジェクト ID の範囲 |
| `enable_quota` | btrfs: クォータがオフのときにプロビジョナーがオンにすることを許可する。これがないと、クォータがオフの btrfs の parent は起動を止める。スナップショットが増えると qgroup がコミットのたびに遅くなるため、それは運用者が選ぶこと |

プロビジョナーのファイルには `provisioner` ブロックしか含めません。fileshare 自身のファイルを
誤って渡した場合は、半端に読み込まれるのではなく拒否されます。

ボリュームは常に `<parent root>/<name>` です。データセット `tank/fileshare/<name>` が
`/srv/fileshare/volumes/tank/<name>` にマウントされ(`refquota` 付き)、btrfs サブボリュームは qgroup で制限され、
XFS または ext4 のディレクトリには範囲内のプロジェクト ID とハードのブロック制限が設定されます。
XFS と ext4 には `prjquota` マウントオプションが必要です(ext4 では `project,quota` 機能も必要)。

## systemd のペア {#the-systemd-pair}

プロビジョナー:

```ini
# fileshare-provisioner.service
[Service]
ExecStart=/usr/local/bin/fileshare provisioner -c /etc/fileshare-provisioner.hcl
RuntimeDirectory=fileshare-provisioner
RuntimeDirectoryMode=0755
StateDirectory=fileshare-provisioner
StateDirectoryMode=0700
CapabilityBoundingSet=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
# Run as root, or as a dedicated user with:
# AmbientCapabilities=CAP_SYS_ADMIN CAP_CHOWN CAP_FOWNER
NoNewPrivileges=yes
SystemCallFilter=@system-service @mount quotactl quotactl_fd
SystemCallArchitectures=native
LockPersonality=yes
RestrictRealtime=yes
# NOT ProtectSystem= nor ReadWritePaths=: they give the service its own
# mount namespace, and the ZFS volumes it mounts would be seen by nobody
# else. And no Landlock: a Landlock-restricted thread cannot mount(2).
```

ケーパビリティをまったく持たないサーバー:

```ini
# fileshare.service
[Unit]
Requires=fileshare-provisioner.service
After=fileshare-provisioner.service

[Service]
User=fileshare
Group=fileshare
ExecStart=/usr/local/bin/fileshare serve -c /etc/fileshare/fileshare.hcl
RuntimeDirectory=fileshare
StateDirectory=fileshare
# No capability at all: CAP_SYS_RESOURCE would let it past an ext4 quota,
# and fileshare refuses to serve volumes with it.
CapabilityBoundingSet=
AmbientCapabilities=
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/srv/fileshare/volumes /var/lib/fileshare
```

サーバーでは `ProtectSystem=` を使っても問題ありません。マウントを行うのはプロビジョナーであり、
マウントはサーバーの名前空間の中へ伝播しても外へは伝播しないため、サーバーの起動後にマウントされた
ZFS ボリュームもサーバーから見えます。

{{< callout type="info" >}}
**例であり、テスト済みのユニットではありません**

どちらのユニットも CI では実行されません。エンドツーエンドのジョブは、両方のプロセスを
`sudo` の下で実行します。
{{< /callout >}}

{{< callout type="warning" >}}
**プロビジョナーを先にアップグレードしてください**

プロビジョナーは、自身のバージョンが定義していないフィールドを持つメッセージを拒否し、
接続を閉じます。そのため `fileshare serve` と `fileshare provisioner` は同じバージョンで
実行します。
{{< /callout >}}

## 管理 API のボリューム呼び出し {#the-admin-apis-volume-calls}

`fileshare serve` は、これらをプロビジョナーに中継します
([`admin.proto`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto))。

| | |
|---|---|
| `ListParents` | ボリュームを作成できる場所: 各 parent の ID、種類、ルート、空き容量と総容量、スナップショットを取れるかどうか。および `max_volume_bytes` |
| `CreateVolume` | `quota_bytes` のボリューム(0 は拒否)。名前に対して冪等: 同じクォータなら既存のボリュームを返し(`created` は false)、異なるクォータは `ALREADY_EXISTS` |
| `ResizeVolume` | クォータを増減する。使用量を下回る値は不可(`FAILED_PRECONDITION`) |
| `SnapshotVolume` | 名前付きの読み取り専用スナップショット: ZFS と btrfs のみ。XFS と ext4 では `UNIMPLEMENTED` |
| `GetVolume`、`ListVolumes` | 各ボリュームのクォータ、使用量、スナップショット、および**それを使っている共有** |
| `DeleteVolume` | 共有が使っている間は拒否される。先に共有を削除する。データやスナップショットを持つボリュームには `destroy_data` が必要 |
| `volume { parent, name }` を指定した `CreateShare` | ボリュームから、ディレクトリとして提供される共有 |

`DeleteShare` がボリュームを削除することはありません。「それを使っている共有」とは、ボリュームから作られた
共有、**および**イメージまたはディレクトリがその内側にある共有(設定ファイルのものも含み、提供中、
無効化中、利用不可のいずれも)を指します。

名前は `^[a-z0-9][a-z0-9_-]{0,62}$` に一致しなければならず、リクエストがパスを運ぶことはありません。
プロビジョナー自身の設定にある `parent` を名指しします。

`admin` ブロックで `reflection = true` を指定すれば、`grpcurl` で呼び出せます。ソケットは 0600 で、
上の `allowed_uids` は fileshare の uid だけを指定しているため、そのユーザーとして実行してください。

```sh
S=/run/fileshare/admin.sock
A=fileshare.admin.v1.AdminService

sudo -u fileshare grpcurl -plaintext -unix $S $A/ListParents

# a 32 GiB volume on the XFS parent
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "34359738368"}' \
  $A/CreateVolume

# a share served from it
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"name": "projects",
       "volume": {"parent": "plain", "name": "projects"},
       "grants": [{"subject": {"group": "staff"}, "access": "ACCESS_WRITE"}]}' \
  $A/CreateShare

# grow it to 64 GiB
sudo -u fileshare grpcurl -plaintext -unix $S \
  -d '{"parent": "plain", "name": "projects", "quota_bytes": "68719476736"}' \
  $A/ResizeVolume
```

### ステータスコード {#status-codes}

プロビジョナーのコードは、返されたとおりに戻ります。`INVALID_ARGUMENT`(文法に合わない名前、不明な parent、
0 のクォータ)、`OUT_OF_RANGE`(`max_volume` 超過)、`RESOURCE_EXHAUSTED`(クォータまたはリサイズによる
増分が parent の現在の空きを超える、またはプロジェクト ID の範囲を使い切った)、
`ALREADY_EXISTS`、`NOT_FOUND`、`FAILED_PRECONDITION`、`UNIMPLEMENTED` です。

次の 3 つは fileshare 独自のものです。

- `provisioner` が設定されていない場合: `FAILED_PRECONDITION`。
- プロビジョナーがこのサーバーの uid を拒否する場合も `FAILED_PRECONDITION`。これは管理呼び出し元が
  してはいけないことではなく、デプロイの誤り(`client_uid` が間違っている)であるため。
- プロビジョナーが応答しない場合: `UNAVAILABLE`。

## ボリュームを提供する前に検査されること {#what-is-checked-before-a-volume-is-served}

fileshare はプロビジョナーにボリュームの場所を尋ね、そのうえで自分で次を検査します。

- パスがクリーンで絶対パスであり、解決後に `source_roots` の下にあること。
- ディレクトリであること。
- `statfs` がその種類のファイルシステムを報告すること(`ZFS_SUPER_MAGIC`、
  `BTRFS_SUPER_MAGIC`、`XFS_SUPER_MAGIC`、`EXT4_SUPER_MAGIC`)。
- **ZFS**: マウントポイントであること。そのデバイスが親のデバイスと異なること。マウントされていない
  データセットのディレクトリへの書き込みは、クォータなしで行われてしまいます。(そこにどのデータセットが
  マウントされているかは検証されません。プロビジョナーの応答にはそれが含まれないためです。)
- **btrfs**: サブボリュームのルートであること。parent の下の通常のディレクトリには qgroup がありません。
- **XFS と ext4**: 新しいファイルが継承する、0 でないプロジェクト ID を持つこと。どのプロジェクトにも
  属さないディレクトリは、制限なしで提供されてしまいます。

状態ファイルが保持するのはボリュームの**名前**であり、パスではありません。起動のたびに問い合わせと
検査が再度行われます。ボリュームが消えている、検査に失敗する、またはプロビジョナーが 10 秒以内に
応答しない場合、その共有は**定義されたまま提供されず**、サーバーの残りは起動します。このことは、
起動時、[`fileshare check`]({{< relref "/configuration/check.md" >}})、および共有の
`unavailable` フィールドで伝えられます。`EnableShare` で再試行できます。

{{< callout type="error" >}}
**`fileshare serve` は、root として、または `CAP_SYS_RESOURCE` を持つ状態でのボリュームの提供を拒否します**

ext4 では、そのどちらもプロジェクトクォータを超えて書き込めます(fs/quota/dquot.c、
`ignore_hardlimit`)。go-fsctl/projquota の CI では、root が 8 MiB のプロジェクトに 16 MiB を
書き込みました。XFS にはそのような例外はありませんが、ルールはすべての種類に共通する、
プロセスに対する 1 つのルールです。fileshare は実効 uid と `/proc/self/status` の
`CapEff`/`CapPrm` を読み取り(permitted のケーパビリティは、`capset(2)` 1 回で effective になる)、
読み取れないステータスは保持しているものとして扱います。拒否するのは**提供**であって、ボリュームの
呼び出しではありません。root でストレージを作成するのは無害ですが、root でそこに書き込むのは
無害ではないからです。`fileshare check` はそのどちらであるかを伝えます。プロビジョナーが
`client_uid = 0` を拒否するのも、上のサーバーのユニットに空の `CapabilityBoundingSet=` があるのも
そのためです。
{{< /callout >}}

## 満杯の共有 {#a-full-share}

XFS は満杯のプロジェクトを `ENOSPC` で伝え、ext4、btrfs、ZFS は `EDQUOT` で伝えます。
v0.21.1 以降は、すべてのプロトコルが「満杯」と応答します。

| プロトコル | 空きなし(`ENOSPC`) | クォータ(`EDQUOT`) |
|---|---|---|
| WebDAV | `507 Insufficient Storage` | `507 Insufficient Storage` |
| NFS | `NFS3ERR_NOSPC` (28) | `NFS3ERR_DQUOT` (69)。RFC 1813 および Linux の nfsd と同じ |
| SFTP | `SSH_FX_FAILURE`、"no space left on device (the share is full)"。バージョン 3 にはそのためのコードがない | 同じ |
| SMB | `STATUS_DISK_FULL` (0xC000007F) | `STATUS_DISK_FULL`。Samba と同じ("Windows apps need this, not NT_STATUS_QUOTA_EXCEEDED") |
| S3 | 読み取り専用で提供 | 読み取り専用で提供 |

v0.21.0 では、SMB は `STATUS_ACCESS_DENIED` で応答し、NFS はクォータに `NFS3ERR_NOSPC` で応答していました。

これはボリュームだけでなく、すべての[ディレクトリ共有]({{< relref "/configuration/shares.md" >}})に適用されます。
[アップグレードの注意]({{< relref "/status.md" >}})を参照してください。

## クライアントから見えるサイズ {#the-size-a-client-sees}

すべてのプロトコルは、問い合わせのたびにサーバーに共有のサイズを尋ねます。NFS の `FSSTAT`、
SMB の `FileFsFullSizeInformation`、WebDAV の RFC 4331 クォータプロパティです。

- **ZFS、XFS、ext4**: サイズはボリューム内で `statfs` が返すものであり、カーネルはクォータ
  (ZFS では `refquota`、XFS と ext4 ではプロジェクトクォータ)で応答します。
- **btrfs**(fileshare v0.23.0 以降): btrfs の `statfs` は qgroup を無視してファイルシステム全体を
  報告するため、サーバーは代わりにプロビジョナーの数値を使います。サイズはボリュームのクォータで、
  空き容量はクォータから qgroup の参照バイト数を引いたもの、またはそれより少ない場合は
  ファイルシステム自身の空き容量です。サーバーはバックグラウンドでプロビジョナーに問い合わせ、
  クライアントが尋ねている間は最大 5 秒に 1 回とし、その間は直近の数値で応答します。btrfs は
  トランザクションのコミット時(既定では 30 秒ごと)に qgroup のカウントを更新するため、
  クライアントに見える空き容量は、書き込みに対してその時間だけ遅れることがあります。
  v0.23.0 より前は、btrfs ボリュームはファイルシステム全体を表示していましたが、クォータは
  変わらず有効でした。

使用量はプロビジョナーから取得します(`used_bytes`)。XFS や ext4 のプロジェクトの使用量の読み取りには
`CAP_SYS_ADMIN` が必要で、fileshare はそれを持たないためです。

## プロビジョナーが保証すること {#what-the-provisioner-promises}

- **1 つの uid、カーネルから読み取る。** 相手は接続時に `SO_PEERCRED` で識別され、`client_uid` 以外は
  root を含めて、リクエストが 1 バイトもデコードされる前に `PERMISSION_DENIED` となります。
- **閉じた動詞の集合。** サービスの外にあるメソッド、デコードできないメッセージ、定義にない
  フィールドを持つメッセージには、応答がありません。接続が閉じられます。
- **自分が作成していないものには決して触れない。** ZFS: データセット自身の `fileshare:volume`
  プロパティ。ユーザープロパティは継承されるため、継承されたマークは自分のものではないので、
  データセットに直接設定されたもの。btrfs: `state_file` に記録されたサブボリューム ID と uuid。
  XFS/ext4: parent の範囲内**かつ**記録されたプロジェクト ID。リサイズ、スナップショット、削除の
  たびに検査されます。
- **削除は、`destroy_data` が設定されていない限り、データとスナップショットを拒否する。**
  また、実行前に記録されます。

## 計測 {#measured}

fileshare の CI で、実際のプールとループバックのファイルシステム上で root として、`fileshare serve` を
`nobody` として実行し、管理 API 経由で操作しました。WebDAV と SFTP での書き込みは、4 種類すべてで
32 MiB のボリュームのクォータで止まります。

## まだ実装されていないもの {#not-yet}

- **Ceph。** CephFS のクォータと RBD は後のフェーズです。
