---
title: "볼륨: fileshare가 생성하는 스토리지"
weight: 10
description: "권한 있는 fileshare 프로비저너를 통해 ZFS, btrfs, XFS, ext4 스토리지를 생성하고 그로부터 공유를 제공하는 방법."
tags: [관리, 볼륨, zfs, btrfs, 쿼터]
---

v0.21.0부터 [관리 API]({{< relref "/administration/_index.md" >}})는 **공유가
제공하는 스토리지를 생성**할 수 있습니다. **ZFS 데이터셋**, **btrfs 서브볼륨**,
또는 **프로젝트 쿼터가 적용된 XFS 또는 ext4 디렉터리**를 크기와 소유자를 지정하고
추적하며 만들고, 그로부터 공유를 제공합니다. Linux 전용입니다. Ceph는 아직
지원하지 않습니다.

이 스토리지를 생성하려면 `CAP_SYS_ADMIN`(quotactl, btrfs 쿼터 ioctl,
`/dev/zfs`, mount(2))이 필요합니다. `fileshare serve`는 낯선 상대를 위해 다섯
가지 네트워크 프로토콜을 파싱하며 어떤 권한도 갖지 않으므로, 권한이 필요한
부분은 같은 바이너리의 두 번째 역할인 `fileshare provisioner`가 맡습니다. 이
역할은 유닉스 소켓 뒤에서 작고 닫힌 프로토콜로 동작합니다.

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

데몬 두 개가 아니라 바이너리 하나입니다. 배포할 버전도 하나, 프로토콜 정의도
하나입니다. 스토리지 작업 자체는 [go-fsctl](https://go-fsctl.github.io/)이
수행합니다(순수 Go이며 `zfs`나 `btrfs` 명령을 쓰지 않음). 설계와 구축 과정에서
바뀐 점은 fileshare 저장소의
[`docs/volumes.md`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/docs/volumes.md)에
있습니다.

## 두 프로세스의 구성 {#both-processes-configured}

`fileshare serve`는 `fileshare` 사용자(uid 990)로 실행합니다.

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

| 키 | |
|---|---|
| `provisioner` | 프로비저너의 소켓, `unix:///path`. 없으면 모든 볼륨 호출이 `FAILED_PRECONDITION`으로 응답하며 볼륨으로 만든 공유도 없습니다 |
| `provisioner_uid` | 해당 소켓에서 응답하는 프로세스가 실행되어야 하는 uid. 연결할 때마다 커널(`SO_PEERCRED`)에서 읽으며, 다른 누군가가 바인딩한 소켓은 아무것도 얻지 못합니다. 지정하지 않으면 0입니다 |
| `allowed_uids` | 선택 사항이며 유닉스 소켓 전용입니다. 호출에 응답하는 유일한 uid이며, 그 밖의 uid는 `PERMISSION_DENIED`가 되고 감사 기록에 남습니다. 소켓의 0600 모드가 허용하는 대상(소유자와 root)을 **좁히기만** 할 수 있으므로, 예를 들어 root가 호출하지 못하게 될 수 있습니다. 클라이언트 인증서가 검사 수단인 TCP에서는 거부됩니다 |

볼륨의 경로는 다른 소스와 마찬가지로 `source_roots` 아래에 있어야 합니다.

`fileshare provisioner`는 root로 실행합니다.

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

| 키 | |
|---|---|
| `listen` | `unix://<절대 경로>`만 허용됩니다. 소켓은 0660으로 만들어지고 소유자는 프로비저너의 사용자(root)와 `<group>`이며, 프로비저너 외에는 아무도 쓸 수 없는 디렉터리에 있어야 합니다. fileshare의 디렉터리가 아니라 프로비저너 **자신의** 디렉터리(`/run/fileshare-provisioner`)여야 합니다 |
| `client_uid` | 응답하는 유일한 uid, 즉 fileshare의 uid. **0은 거부됩니다** |
| `group` | 모든 볼륨 루트를 소유합니다(`root:<group>`, 모드 2770). fileshare가 이 그룹에 속해야 합니다. fileshare는 소유자로서가 아니라 그룹을 통해 볼륨에 씁니다. 디렉터리의 소유자는 권한 없이 프로젝트 id를 지울 수 있고, 그러면 XFS/ext4 쿼터를 벗어날 수 있기 때문입니다 |
| `max_volume` | 볼륨 하나가 가질 수 있는 최대 쿼터(`"10T"`, `"500G"`, `"1048576"`). 이를 넘으면 `OUT_OF_RANGE`입니다 |
| `state_file` | 이 프로세스가 생성한 것(btrfs 서브볼륨 id, XFS/ext4 프로젝트 id)을 기록하여, 재시작 후에도 무엇이 자신의 것인지 알 수 있게 합니다 |
| `parent "<id>"` | 볼륨을 만들 수 있는 위치이며, 요청이 지정하는 id로 식별됩니다. `zfs`(볼륨이 그 자식이 되는 데이터셋이며, 마운트되는 위치는 `root`), `btrfs`, `xfs`, `ext4`(볼륨이 생성되는 디렉터리) 중 정확히 하나 |
| `project_ids` | XFS/ext4: 이 부모가 부여할 수 있는 프로젝트 id의 범위 |
| `enable_quota` | btrfs: 쿼터가 꺼져 있을 때 프로비저너가 켜도록 허용합니다. 이 옵션이 없으면 쿼터가 꺼진 btrfs 부모는 시작을 중단시킵니다. 스냅샷이 늘어날수록 qgroup이 모든 커밋을 느리게 하므로 이는 운영자가 선택할 일입니다 |

프로비저너의 파일에는 `provisioner` 블록만 있습니다. fileshare 자체의 파일을
실수로 넘기면 일부만 읽지 않고 거부됩니다.

볼륨은 항상 `<부모 루트>/<이름>`입니다. 데이터셋 `tank/fileshare/<name>`은
`/srv/fileshare/volumes/tank/<name>`에 (`refquota`와 함께) 마운트되고, btrfs
서브볼륨은 qgroup으로 제한되며, XFS 또는 ext4 디렉터리는 범위에서 받은 프로젝트
id와 하드 블록 제한을 가집니다. XFS와 ext4에는 `prjquota` 마운트 옵션이
필요합니다(ext4는 `project,quota` 기능도 필요).

## systemd 쌍 {#the-systemd-pair}

프로비저너:

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

권한을 전혀 갖지 않는 서버:

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

서버에는 `ProtectSystem=`을 써도 됩니다. 마운트를 수행하는 것은 프로비저너이며,
마운트는 서버의 네임스페이스 안으로 전파될 뿐 밖으로는 전파되지 않으므로, 서버가
시작된 뒤에 마운트된 ZFS 볼륨도 서버에서 보입니다.

{{< callout type="info" >}}
**예시이며, 테스트된 유닛이 아닙니다**

두 유닛 모두 CI에서 실행되지 않습니다. 엔드투엔드 작업은 두 프로세스를
`sudo`로 실행합니다.
{{< /callout >}}

{{< callout type="warning" >}}
**프로비저너를 먼저 업그레이드하십시오**

프로비저너는 자신의 버전이 정의하지 않은 필드를 담은 메시지를 거부하고
연결을 닫습니다. 그러므로 `fileshare serve`와 `fileshare provisioner`는 같은
버전으로 실행해야 합니다.
{{< /callout >}}

## 관리 API의 볼륨 호출 {#the-admin-apis-volume-calls}

`fileshare serve`는 다음 호출을 프로비저너에 중계합니다
([`admin.proto`](https://github.com/go-fileshare/fileshare/blob/v0.28.0/proto/fileshare/admin/v1/admin.proto)).

| | |
|---|---|
| `ListParents` | 볼륨을 만들 수 있는 위치: 각 부모의 id, 종류, 루트, 여유 공간과 전체 공간, 스냅샷 지원 여부, 그리고 `max_volume_bytes` |
| `CreateVolume` | `quota_bytes` 크기의 볼륨(0은 거부됨). 이름 기준으로 멱등합니다. 같은 쿼터면 이미 있는 볼륨으로 응답하고(`created`가 false), 다른 쿼터면 `ALREADY_EXISTS`입니다 |
| `ResizeVolume` | 쿼터를 늘리거나 줄입니다. 사용 중인 양보다 작게는 안 됩니다(`FAILED_PRECONDITION`) |
| `SnapshotVolume` | 이름이 있는 읽기 전용 스냅샷. ZFS와 btrfs에서 지원하며 XFS와 ext4에서는 `UNIMPLEMENTED`입니다 |
| `GetVolume`, `ListVolumes` | 각 볼륨과 그 쿼터, 사용량, 스냅샷, 그리고 **이를 사용하는 공유** |
| `DeleteVolume` | 공유가 사용하는 동안에는 거부됩니다. 먼저 공유를 삭제하십시오. 데이터나 스냅샷이 있는 볼륨에는 `destroy_data`가 필요합니다 |
| `volume { parent, name }`을 지정한 `CreateShare` | 볼륨을 디렉터리로 제공하는 공유 |

`DeleteShare`는 볼륨을 삭제하지 않습니다. "이를 사용하는 공유"는 볼륨으로 만든
공유 **그리고** 이미지나 디렉터리가 볼륨 안에 있는 모든 공유(구성 파일의 공유,
제공 중이든 비활성화되었든 사용할 수 없든 포함)입니다.

이름은 `^[a-z0-9][a-z0-9_-]{0,62}$`에 일치해야 하며, 요청은 경로를 담지
않습니다. 프로비저너 자체 구성에 있는 `parent`를 이름으로 지정합니다.

`admin` 블록에 `reflection = true`가 있으면 `grpcurl`로 호출할 수 있습니다.
소켓은 0600이고 위의 `allowed_uids`는 fileshare의 uid만 지정하므로, 그
사용자로 실행하십시오.

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

### 상태 코드 {#status-codes}

프로비저너의 코드는 받은 그대로 돌아옵니다. `INVALID_ARGUMENT`(문법에 맞지 않는
이름, 알 수 없는 부모, 0인 쿼터), `OUT_OF_RANGE`(`max_volume` 초과),
`RESOURCE_EXHAUSTED`(쿼터 또는 크기 조정으로 늘어나는 양이 부모의 현재 여유
공간보다 많거나 프로젝트 id 범위를 모두 사용함), `ALREADY_EXISTS`, `NOT_FOUND`,
`FAILED_PRECONDITION`, `UNIMPLEMENTED`입니다.

다음 세 가지는 fileshare 자체의 코드입니다.

- `provisioner`가 구성되지 않음: `FAILED_PRECONDITION`
- 프로비저너가 이 서버의 uid를 거부함: 역시 `FAILED_PRECONDITION`. 관리 호출자가
  할 수 없는 일이 아니라 배포 실수(`client_uid`가 잘못됨)이기 때문입니다.
- 프로비저너가 응답하지 않음: `UNAVAILABLE`

## 볼륨을 제공하기 전에 확인하는 것 {#what-is-checked-before-a-volume-is-served}

fileshare는 프로비저너에게 볼륨의 위치를 묻고, 그런 다음 직접 확인합니다.

- 경로가 정리되어 있고 절대 경로이며, 해석한 결과가 `source_roots` 아래에 있음
- 디렉터리임
- `statfs`가 해당 종류의 파일 시스템을 보고함(`ZFS_SUPER_MAGIC`,
  `BTRFS_SUPER_MAGIC`, `XFS_SUPER_MAGIC`, `EXT4_SUPER_MAGIC`)
- **ZFS**: 마운트 지점임 — 그 장치가 부모의 장치와 다릅니다. 마운트되지 않은
  데이터셋의 디렉터리는 쿼터 없이 쓰기를 받게 됩니다. (그 위치에 어떤 데이터셋이
  마운트되어 있는지는 확인하지 않습니다. 프로비저너의 응답에 그 정보가 없습니다.)
- **btrfs**: 서브볼륨의 루트임 — 부모 아래의 일반 디렉터리에는 qgroup이 없습니다.
- **XFS와 ext4**: 새 파일이 상속하는 0이 아닌 프로젝트 id를 가짐 — 어떤
  프로젝트에도 속하지 않은 디렉터리는 제한 없이 제공됩니다.

상태 파일에는 볼륨의 경로가 아니라 **이름**이 보관되며, 시작할 때마다 다시 묻고
확인합니다. 볼륨이 사라졌거나 확인에 실패했거나 프로비저너가 10초 안에 응답하지
않으면 그 공유는 **정의된 채 제공되지 않으며**, 이는 시작할 때,
[`fileshare check`]({{< relref "/configuration/check.md" >}})에서, 그리고 공유의
`unavailable` 필드에서 알려집니다. 나머지 서버는 정상적으로 시작합니다.
`EnableShare`가 다시 시도합니다.

{{< callout type="error" >}}
**`fileshare serve`는 root 또는 `CAP_SYS_RESOURCE`로는 볼륨 제공을 거부합니다**

ext4는 둘 중 어느 쪽이든 프로젝트 쿼터를 넘어 쓰는 것을 허용합니다
(fs/quota/dquot.c, `ignore_hardlimit`). go-fsctl/projquota의 CI에서 root가 8 MiB
프로젝트에 16 MiB를 썼습니다. XFS에는 이런 예외가 없지만, 이 규칙은 모든 종류에
적용되는 프로세스 단위의 규칙 하나입니다. fileshare는 유효 uid와
`/proc/self/status`의 `CapEff`/`CapPrm`을 읽으며(허용된 권한은 `capset(2)` 한 번이면
유효 권한이 됩니다), 읽을 수 없는 status는 해당 권한을 가진 것으로 간주합니다.
거부하는 것은 볼륨 호출이 아니라 **제공**입니다. root로 스토리지를 생성하는 것은
해롭지 않지만, root로 그 안에 쓰는 것은 그렇지 않습니다. 어느 쪽인지는
`fileshare check`가 알려 줍니다. 프로비저너가 `client_uid = 0`을 거부하고, 위의
서버 유닛이 비어 있는 `CapabilityBoundingSet=`을 갖는 이유가 이것입니다.
{{< /callout >}}

## 가득 찬 공유 {#a-full-share}

XFS는 가득 찬 프로젝트를 `ENOSPC`로, ext4, btrfs, ZFS는 `EDQUOT`로 알립니다.
v0.21.1부터 모든 프로토콜이 "가득 참"으로 응답합니다.

| 프로토콜 | 공간 없음(`ENOSPC`) | 쿼터(`EDQUOT`) |
|---|---|---|
| WebDAV | `507 Insufficient Storage` | `507 Insufficient Storage` |
| NFS | `NFS3ERR_NOSPC` (28) | `NFS3ERR_DQUOT` (69), RFC 1813과 Linux의 nfsd를 따름 |
| SFTP | `SSH_FX_FAILURE`, "no space left on device (the share is full)" — 버전 3에는 이를 위한 코드가 없습니다 | 동일 |
| SMB | `STATUS_DISK_FULL` (0xC000007F) | `STATUS_DISK_FULL`, Samba와 동일("Windows apps need this, not NT_STATUS_QUOTA_EXCEEDED") |
| S3 | 읽기 전용으로 제공 | 읽기 전용으로 제공 |

v0.21.0에서는 SMB가 `STATUS_ACCESS_DENIED`로, NFS가 쿼터를 `NFS3ERR_NOSPC`로
응답했습니다.

이는 볼륨뿐 아니라 모든 [디렉터리 공유]({{< relref "/configuration/shares.md" >}})에
적용됩니다. [업그레이드 안내]({{< relref "/status.md" >}})를 참고하십시오.

## 클라이언트에 보이는 크기 {#the-size-a-client-sees}

모든 프로토콜은 질의할 때마다 서버에 공유의 크기를 묻습니다. NFS의 `FSSTAT`,
SMB의 `FileFsFullSizeInformation`, WebDAV의 RFC 4331 쿼터 속성입니다.

- **ZFS, XFS, ext4**: 크기는 볼륨 안에서 `statfs`가 알려 주는 값이며, 커널이
  쿼터(ZFS는 `refquota`, XFS와 ext4는 프로젝트 쿼터)로 응답합니다.
- **btrfs** (fileshare v0.23.0부터): btrfs의 `statfs`는 qgroup을 무시하고
  파일 시스템 전체를 보고하므로, 서버는 대신 프로비저너의 값을 사용합니다. 크기는
  볼륨의 쿼터이고, 여유 공간은 쿼터에서 qgroup이 참조하는 바이트를 뺀 값이거나,
  그보다 작다면 파일 시스템 자체의 여유 공간입니다. 서버는 클라이언트가 요청하는
  동안 최대 5초에 한 번 백그라운드에서 프로비저너에 묻고, 그사이에는 마지막
  값으로 응답합니다. btrfs는 트랜잭션을 커밋할 때(기본 30초마다) qgroup의 계수를
  갱신하므로, 클라이언트에 보이는 여유 공간은 쓰기보다 그만큼 늦을 수 있습니다.
  v0.23.0 이전에는 btrfs 볼륨이 파일 시스템 전체를 보여 주었지만, 쿼터는 그대로
  적용되었습니다.

사용량은 프로비저너(`used_bytes`)에서 가져옵니다. XFS나 ext4 프로젝트의 사용량을
읽으려면 fileshare에 없는 `CAP_SYS_ADMIN`이 필요하기 때문입니다.

## 프로비저너가 보장하는 것 {#what-the-provisioner-promises}

- **커널에서 읽은 단 하나의 uid.** 상대는 연결할 때 `SO_PEERCRED`로 식별되며,
  `client_uid` 외에는 root를 포함해 누구든 요청의 한 바이트도 디코딩되기 전에
  `PERMISSION_DENIED`가 됩니다.
- **닫힌 동사 집합.** 서비스에 없는 메서드, 디코딩되지 않거나 정의되지 않은
  필드를 담은 메시지에는 응답이 없고 연결이 닫힙니다.
- **자신이 만들지 않은 것은 절대 건드리지 않습니다.** ZFS: 데이터셋 자체의
  `fileshare:volume` 속성 — 사용자 속성은 상속되고 상속된 표시는 자신의 것이
  아니므로 데이터셋에 직접 설정합니다. btrfs: `state_file`에 기록된 서브볼륨 id와
  uuid. XFS/ext4: 부모의 범위 안에 있고 *또한* 기록된 프로젝트 id. 크기 조정,
  스냅샷, 삭제 전에 매번 확인합니다.
- **삭제는 데이터와 스냅샷이 있으면 거부합니다.** `destroy_data`가 설정된
  경우는 예외이며, 실행되기 전에 기록됩니다.

## 측정 결과 {#measured}

fileshare의 CI에서 실제 풀과 루프백 기반 파일 시스템 위에서 root로, `fileshare serve`를 `nobody`로 실행하고 관리 API로 구동하여 측정했습니다.
WebDAV와 SFTP를 통한 쓰기는 네 종류 모두에서 32 MiB 볼륨의 쿼터에서
멈춥니다.

## 아직 지원하지 않는 것 {#not-yet}

- **Ceph.** CephFS 쿼터와 RBD는 이후 단계입니다.
