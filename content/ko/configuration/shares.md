---
title: "공유: 이미지, 장치, 디렉터리"
weight: 10
description: "공유가 제공할 수 있는 것(디스크 이미지, 장치, 호스트 디렉터리)과 그 파일 시스템 및 파티션을 선택하는 방법."
tags: [구성, 공유]
---

공유는 세 가지 중 하나를 제공합니다. **디스크 이미지**, **장치** 노드, 또는
**호스트의 디렉터리**입니다. 이미지와 장치는 이 프로그램이 직접 읽는 파일
시스템을 담고 있으며, 디렉터리는 호스트 자체의 것입니다.

## 이미지와 그 안의 파일 시스템 {#an-image-and-the-filesystem-inside-it}

이미지 안의 파일 시스템은 **선언하는 것이 아니라 알아냅니다**.
[`go-filesystems/detect`](https://github.com/go-filesystems/detect)가 매직
바이트를 읽고 그것을 소유한 드라이버를 돌려줍니다. **fat32, exfat, ext4, ntfs,
ufs, iso9660, squashfs, hfsplus** 중 하나이며, 모든 드라이버는 같은 형태인
`OpenReader(io.ReaderAt, int64)`, 즉 오프셋 0의 파일 시스템입니다.

## 탐지할 수 없는 네 가지 {#the-four-that-cannot-be-sniffed}

**apfs, btrfs, xfs, zfs**는 각각 *디스크* 이미지를 열고 **파티션**을 선택하므로,
오프셋 0에서 찾을 매직 바이트가 없습니다. 공유가 어느 것인지 지정합니다.

```hcl
share "photos" {
  image      = "/srv/disk.img"
  filesystem = "xfs"
  partition  = 2       # or leave it out: the driver takes the first data partition
}
```

## HFS+ 이미지는 공유가 동의하지 않으면 읽기 전용 {#hfsplus-images-are-writable}

v0.44.0(hfsplus v0.9.0)부터 HFS+가 들어 있는 이미지의 공유는 FAT, exFAT, ext4처럼
쓰기용으로 열 수 있습니다. 클라이언트는 파일을 만들고, 바꾸고, 지울 수 있으며, 설정한 시간,
모드, 소유자도 저장됩니다. 이전에는 HFS+ 이미지가 읽기 전용으로 열렸습니다.

**v0.45.1부터 기본으로 꺼져 있고, 지금도 꺼져 있습니다.** hfsplus v0.9.0은 충돌에 안전하지
않았습니다(이미지 전체를 메모리에 두고, 작업마다 전체를 다시 쓰고, 볼륨을 사용 중으로 표시하지
않았습니다). 그래서 v0.45.1이 쓰기를 껐습니다(v0.44.0과 v0.45.0은 기본으로 썼습니다. 최대
256 MiB). **v0.46.0(hfsplus v0.10.0)부터** 쓰기 가능한 HFS+ 볼륨은 **제자리에서** 기록됩니다.

- **메모리 비용이 없습니다.** 이미지를 메모리에 복사하지 않으며(8 MiB 읽기 캐시만 있음), 커밋이
  바꾼 섹터만 기록합니다. 64 GiB 상한도 없어졌습니다.
- **충돌이 보입니다.** 커밋마다 볼륨을 사용 중으로 표시합니다. 마운트 해제 비트와 저널 비트는 첫
  헤더 쓰기에서 지워지고 마지막에 복원되므로 `fsck_hfs -q`가 충돌을 알아볼 수 있습니다. 저널이 있는
  볼륨도 마찬가지입니다.
- **쓰기 실패는 볼륨을 멈춥니다.** 플러시가 실패하면 공유를 다시 열 때까지 볼륨은 쓰기를
  거부합니다(`ErrVolumeFailed`).

그래도 **옵트인**으로 남아 있습니다. hfsplus에 아직 끝나지 않은 일이 있기 때문입니다. 내구성 있는
작은 쓰기는 커밋당 fsync 3번이 필요해 macOS보다 약 3배 느리고(go-filesystems/hfsplus#35),
충돌 테스트는 아직 강화되지 않았으며(go-filesystems/hfsplus#37), 충돌 뒤에는 저널이 있는 볼륨이
`diskutil enableJournal`을 실행할 때까지 저널 없이 마운트됩니다. 이것들이 고쳐지면 기본값을 되돌릴
수 있습니다. 먼저 이미지를 백업하세요. 공유는 `hfsplus_writable_max`로 동의하며, 기본값은
`"0"`(쓰지 않음; 바이트 수, 또는 K, M, G, T가 붙은 수)입니다.

```hcl
share "mac" {
  image                = "/srv/mac.img"
  hfsplus_writable_max = "256M"   # 동의
}
```

v0.46.0부터 이 크기는 더 이상 메모리를 제한하지 않습니다. v0.44.0~v0.45.1용으로 쓴 설정이
예전과 같은 뜻을 유지하도록 크기로 남겨 두었습니다(`"256M"`이라고 쓴 공유가 2 TiB 이미지를 쓰기
시작하지는 않습니다). `"1T"` 같은 큰 값은 단순한 동의가 됩니다.

기본값일 때, 크기를 넘을 때, 크기를 알 수 없을 때 공유는 **읽기 전용**으로 열리며(범위 읽기와 시간은
그대로 동작합니다), 서버는 이미지와 이유를 밝히는 한 줄을 씁니다. 예를 들어 설정이 없을 때:

```text
/srv/mac.img is an HFS+ volume and hfsplus_writable_max is 0, the default: HFS+ images are written only when a share sets a size, because a durable small write is about 3x slower than on macOS (go-filesystems/hfsplus#35), the crash tests are not finished (go-filesystems/hfsplus#37), and after a crash a journaled volume mounts without its journal until diskutil enableJournal, so mac is read-only
```

크기를 넘을 때:

```text
/srv/mac.img is an HFS+ volume and it is 1073741824 bytes, over hfsplus_writable_max (268435456 bytes), so mac is read-only
```

**다시 불러오기는 설정을 따릅니다(v0.46.0부터).** 다시 불러오기로 HFS+ 이미지를 쓰는지 여부가
바뀌면(설정을 지우거나, 추가하거나, 이미지 크기를 사이에 두고 옮긴 경우) 이미지를 다시 엽니다.
동의를 지우면 공유가 즉시 읽기 전용이 되고, 추가하면 쓰기 가능해집니다. v0.46.0 이전에는 열려 있는
드라이버를 그대로 써서, 동의를 지워도 서버를 다시 시작할 때까지 공유가 쓰기 가능한 채로 남았습니다.
쓰기 가능한 크기에서 다른 쓰기 가능한 크기로 바꾸는 것은 아무것도 바꾸지 않습니다.

**저널이 있는** 볼륨(`hdiutil create -fs "Journaled HFS+"`나 macOS가 만든 것)은 저널이 비어 있을
때만 기록됩니다. 정상적으로 마운트 해제하면 비어 있게 됩니다. 트랜잭션이 있는 저널이나 검증할 수
없는 저널은 서버가 쓴 내용 위에 macOS가 재생해 버리므로 드라이버가 거부하고, **공유는 대신 읽기
전용으로 열립니다**. 서버는 계속 시작하며, 공유를 열 때 이미지, 공유, 이유를 밝히는 한 줄을 씁니다.

```text
/srv/mac.img is an HFS+ volume and its journal holds transactions (hfsplus: journaled volume with a journal that is not empty: transactions pending (start 17408, end 34816)), so mac is read-only
```

이런 공유를 쓰기 가능하게 하려면 먼저 저널을 재생시켜야 합니다. Mac에서 이미지를 마운트한 뒤
정상적으로 꺼내거나, 이미지에 `fsck_hfs`를 실행하세요. 저널이 있는 볼륨을 쓰는 동안에는 저널 비트가
지워져 있고, 충돌 뒤에도 지워진 채로 남으므로 볼륨은 `diskutil enableJournal`을 실행할 때까지
저널 없이 마운트됩니다. `read_only = true`인 공유, 장치, 파티션은 쓰기용 열기를 시도하지 않습니다.

볼륨은 공유 자신의 파일 디스크립터로 기록됩니다(hfsplus v0.11.0). 이미 격리되고 잠긴 바로 그
파일이며, 이미지 이름을 다시 여는 일은 없습니다. 볼륨과 맞지 않는 크기는 공유를 읽기 전용으로
열고 한 줄을 남깁니다.

**다른 쓰기 주체.** 같은 이미지에 두 번째 쓰기 주체가 있으면 이미지가 손상됩니다. 쓰기용으로 열 때
이미지에 배타적인 권고 잠금 `flock`을 잡습니다(Linux, macOS, BSD). 다른 프로세스가 잡고 있으면(예:
같은 이미지의 두 번째 fileshare) 공유는 읽기 전용으로 열리고 그 사실을 알립니다. 잠금은 권고용이라
macOS 마운트는 잡지 않으며, 다른 플랫폼에는 잠금이 없습니다.

⛔ **마운트된 이미지에는 쓰지 마세요.** 이 서버로든 다른 것으로든 마찬가지입니다. 어느 쪽도 상대의
변경을 알지 못합니다. 다른 것이 열어 두지 않은 이미지를 제공하세요.

## OS가 마운트한 쓰기 가능 이미지는 읽기 전용으로 제공됩니다 {#an-image-the-os-has-mounted}

v0.45.0부터, 쓰기 가능한 이미지 공유(FAT32, exFAT, ext4, UFS, btrfs, XFS, ZFS, HFS+ 등 드라이버와 무관)를 쓰기용으로 열기 전에 서버는 OS가 해당 이미지를 연결했거나 마운트했는지 OS에 묻습니다. 그렇다면 공유는 **읽기 전용**으로 열리고 그렇게 표시되며, 서버는 이미지, OS의 장치와 마운트, 공유 이름을 밝히는 한 줄을 기록합니다.

```text
/srv/disk.img is in use by the OS (/dev/loop3, mounted at /mnt/disk), and a second writer would corrupt it, so disk is read-only
```

위에서 설명한 권고형 `flock`으로는 이를 막을 수 없습니다. OS는 이미지를 마운트할 때 잠금을 잡지 않기 때문입니다. 이미지는 장치와 inode로 대조하며 경로 문자열을 비교하지 않으므로, 마운트된 파일에 대한 심볼릭 링크나 하드 링크도 감지됩니다.

| OS | 확인 방법 |
| --- | --- |
| Linux | 각 루프 장치의 `/sys/block/loop*/loop/backing_file`(`mount -o loop`, `losetup`, udisks가 만드는 것). 마운트 지점은 `/proc/self/mountinfo`에서 읽습니다. 연결된 루프 장치는 아무것도 마운트되지 않았더라도 해당됩니다. |
| macOS | `hdiutil info -plist`, 연결된 각 이미지의 `image-path`. 로드당 한 번 호출하며 15초 제한이 있습니다. `-nomount`로 연결한 이미지도 해당됩니다. |
| FreeBSD, 기타 BSD, Windows | **감지할 수 없습니다.** 공유는 이전처럼 쓰기 가능합니다. |

**확인 자체가 실패하면**(컨테이너에 `/sys/block`이 없음, `hdiutil`이 없거나 시간 초과, 출력이 속성 목록이 아님) 이미지가 마운트되었는지 아무도 알 수 없습니다. `image_mount_unknown`이 결정합니다.

```hcl
share "disk" {
  image               = "/srv/disk.img"
  image_mount_unknown = "read_only"   # 기본값. "write"는 그래도 쓰기 가능으로 제공합니다
}
```

기본값은 `read_only`입니다. 읽기 전용 공유는 원인을 밝히는 로그 줄이 있는 눈에 보이는 상태이며 사람이 고칠 수 있습니다. 마운트 아래에서 쓰기 가능한 공유는 양쪽 데이터를 모두 손상시키고 나중에야 알게 됩니다. `write`는 확인이 동작할 수 없는 호스트(sysfs가 없는 컨테이너)에서 아무도 이미지를 마운트하지 않을 때를 위한 것입니다. OS가 마운트한 것으로 확인된 이미지는 이 설정과 상관없이 읽기 전용입니다. 확인이 읽을 수 없는 백업 경로(권한 거부, 심볼릭 링크 순환)는 "다른 이미지"가 아니라 "판단하지 못함"으로 취급되어 같은 설정을 따릅니다. 존재하지 않는 경로만 건너뜁니다. Linux에서는 백업 파일이 삭제된 루프 장치를, 프로세스가 `/dev/loopN`을 열 수 있으면 커널 자체 기록(`LOOP_GET_STATUS64`)으로 대조하고, 열 수 없으면 대조하지 않습니다.

**확인이 보지 못하는 것**

- **공유가 이미지를 연 뒤에 이루어진 마운트.** 확인은 이미지를 열 때 실행됩니다. 시작 시, 관리 API로 만든 공유, 그리고 리로드가 공유를 다시 구성하면서(사람이나 목록 변경) 그동안 이미지가 마운트된 것을 발견했을 때입니다. 이 경우 해당 공유는 자체 읽기 전용 열기를 받습니다. 아무것도 바뀌지 않는 리로드는 다시 확인하지 않으며, 이후의 `hdiutil attach`나 `mount -o loop`를 감시하는 것도 없습니다. 쓰기 가능으로 제공 중인 이미지를 마운트하기 전에 공유를 중지하거나 서버를 재시작하십시오.
- **루프 장치나 `hdiutil`을 거치지 않는 마운트:** `qemu-nbd -c`(nbd 장치), FUSE 마운트(`fuse2fs`, `guestmount`), 이미지를 열고 있는 가상 머신이나 다른 프로그램.
- 다른 운영 체제(표 참조)와 다른 마운트 네임스페이스에서 연결된 루프 장치.
- 마운트 때문에 읽기 전용이 된 공유는 마운트가 사라져도 리로드 후에도 읽기 전용으로 남습니다. 다시 쓰기용으로 열려면 재시작하십시오.

## 어떤 파일 시스템이든 가능한 파티션 {#a-partition-for-any-filesystem}

FAT32, ext4, exFAT를 담은 디스크 이미지도 XFS를 담은 이미지만큼 자주 앞에
파티션 테이블이 있으며, 탐지는 오프셋 0을 읽는데 파티션된 이미지는 그곳에
*테이블*이 있습니다. 그래서 파티션을 먼저 선택하고, 드라이버에는 그 파티션의
뷰를 넘깁니다.

```hcl
share "photos" {
  image           = "/srv/disk.img"
  partition_label = "photos"          # what lsblk calls PARTLABEL
}
```

파티션을 지정하는 방법은 세 가지이며 **하나만 지정할 수 있습니다**. 서로 어긋나는
둘을 지정하면 코드가 먼저 시도한 쪽이 제공될 것이기 때문입니다.

| | |
|---|---|
| `partition = 2` | 파티셔닝 도구가 출력하는 방식대로 1부터 센 번호 |
| `partition_label = "photos"` | GPT 파티션 이름 |
| `partition_uuid = "…"` | GPT 고유 GUID — Linux의 `PARTUUID` |

{{< callout type="error" >}}
**인덱스는 움직입니다**

디스크를 다시 파티셔닝하거나, 항목을 다른 순서로 쓰는 도구를 사용하거나,
파티션이 하나 적은 상태로 이미지를 복원하면 `partition = 2`가 **조용히** 다른
것을 가리키게 됩니다. 그곳에서도 여전히 파일 시스템이 발견되기 때문입니다. 레이블이나
UUID는 파티션 자체를 가리키며, fstab이 수년 전에 인덱스 사용을 그만둔 이유가
이것입니다. 인덱스는 둘 다 없는 MBR 이미지를 위해 있습니다.
{{< /callout >}}

파티션을 선택한 공유는 **읽기 전용**이며 시작할 때 그렇게 알립니다. 드라이버의
오프셋은 파티션 기준인데 아래의 파일은 디스크 전체이므로, 쓰기는 *이미지*의
시작부터 그 오프셋에 도달하게 되고, 이는 대개 파티션 테이블 위입니다.

## 파일 시스템을 지정하면 탐지가 꺼집니다 {#naming-a-filesystem-turns-detection-off}

{{< callout type="error" >}}
**그것이 요점이자 위험입니다**

이미지는 *그것으로* 열리거나 거부됩니다. XFS라고 지정된 FAT32 이미지는 XFS
공유가 되지 않으며, 시작에 실패하고 무엇으로 열라는 요청을 받았는지를 알려 줍니다.
{{< /callout >}}

`filesystem`은 탐지할 수 있는 종류에도 허용되며, 이 경우 둘을 비교합니다.
FAT32 이미지에 `ext4`를 지정한 공유는 *the share says ext4 and the image holds
fat32*와 함께 거부됩니다. 사이트는 이렇게 오탐지를 나중에 발견하는 대신
거부합니다. [`check`]({{< relref "/configuration/check.md" >}})는 이름이 지정된
드라이버에 별표를 붙입니다. 그 행은 인식된 것이 아니라 지정된 것이기 때문입니다.

## 이미지뿐 아니라 장치도 {#a-device-not-only-an-image}

```hcl
share "photos" {
  image = "/dev/sda"        # a device node, not a file
}
```

**권한이 필요하지 않습니다.** 여기서는 아무것도 마운트하지 않습니다.
[`go-filesystems`](https://github.com/go-filesystems) 드라이버가 ext4, xfs 등을
사용자 공간에서 읽으므로 `mount(2)`가 없고, 따라서 `CAP_SYS_ADMIN`도 필요
없습니다. 장치를 여는 것은 노드의 권한이 좌우하는 평범한 `open(2)`입니다.

| | 소유자 | 모드 | 충분한 조건 |
|---|---|---|---|
| Linux `/dev/sda` | `root:disk` | 0660 | `disk` 그룹의 구성원 |
| macOS `/dev/disk0` | `root:operator` | 0640 | `operator` 그룹의 구성원 및 전체 디스크 접근 권한 |

거부될 때는 `permission denied` 대신 어느 그룹인지를 알려 줍니다. 답은 결코
`sudo`가 아닙니다.

⛔ **장치 공유는 읽기 전용입니다.** 파티션을 선택한 공유와 같은 이유입니다.
동작 중인 디스크에 쓰는 것은 사용자를 대신해 내릴 결정이 아닙니다.

{{< callout type="error" >}}
**장치는 배타적으로 열리며, 이는 신중함이 아니라 정확성의 문제입니다**

커널이 그 장치의 파일 시스템을 마운트하고 있다면 파일 시스템의 페이지 캐시가
장치보다 새로운 메타데이터를 가지고 있습니다. 그러면 raw 리더는 갱신 이전의
디렉터리 블록과 갱신 이후의 inode 블록을 나란히 보게 되며, 이는 디스크에
어느 한순간에도 존재한 적 없는 상태입니다. 블록 장치에 대한 `O_EXCL`은 "마운트를
포함해 나 외에는 아무도 없음"을 뜻하는 커널 자체의 원시 기능이며, `mkfs`와
`fsck`가 사용하는 것입니다. 사용 중인 장치는 이름과 이유와 함께 거부됩니다.
{{< /callout >}}

가정하지 않고 측정한 두 가지가 있습니다.

- 장치의 `Stat().Size()`는 **0**이므로 길이는 장치 자체에 묻습니다. 끝으로
  이동(seek)하면 Linux에서는 답하지만 macOS에서는 **오류 없이 0**을 반환하므로,
  Darwin은 대신 `DKIOCGETBLOCKCOUNT`를 사용합니다.
- **raw** 노드(`/dev/rdiskN`, 또는 Linux의 `O_DIRECT`)는 정렬되지 않은 읽기를
  거부하는데, FAT BPB를 파싱한다는 것은 곧 오프셋 11의 2바이트 필드를 읽는 것을
  *뜻합니다*. 장치 읽기는 전체 블록 단위로 확장됩니다. 이것이 없으면
  `/dev/rdisk4`는 `unknown filesystem`을 보고했습니다.

## 이미지뿐 아니라 디렉터리도 {#a-directory-not-only-an-image}

v0.8.0부터 공유는 이미지 대신 호스트의 디렉터리(예: 컨테이너의 볼륨)를 제공할 수
있습니다.

```hcl
share "photos" {
  directory = "/data/photos"
  allow     = ["@family"]
}
```

이는 [go-filesystems/osfs](https://github.com/go-filesystems/osfs)이며, 트리에
**`os.Root`**를 통해 접근합니다. 밖으로 올라가는 `..`, 절대 경로, 밖으로 향하는
심볼릭 링크는 문자열 검사가 아니라 커널이 뒷받침하는 루트가 거부합니다. 클라이언트가
`/etc`로의 링크를 *만들* 수는 있지만 아무것도 그것을 따라가지 않습니다. 트리에
심어 둔 FIFO는 서버를 멈추게 두지 않고 거부됩니다.

{{< callout type="error" >}}
**한 공유가 다른 공유를 포함할 수 없습니다 (v0.17.0부터)**

트리에 다른 공유의 이미지나 디렉터리가 들어 있는 디렉터리 공유는 **시작할 때
거부되며** 관리 API에서도 거부됩니다. 바깥 공유를 사용할 수 있는 사람이 허용되지
않은 안쪽 공유를 읽고 쓰게 되기 때문이며, 익명 NFS도 마찬가지입니다. 같은
소스에 대한 두 공유도 거부됩니다. 하나의 디스크 이미지의 서로 다른 두 파티션은
예외입니다.
{{< /callout >}}

{{< callout type="info" >}}
**이미지 잠금 뒤에 있지 않습니다**

이미지 드라이버는 파일 하나를 소유하며 동시에 두 번 호출했을 때의 동작을
보장하지 않습니다. 그래서 [모든 이미지는 하나의 잠금으로 감싸집니다]({{< relref "/operations/locking.md" >}}).
호스트의 트리는 커널의 것이며, 커널은 호출마다 직렬화가 필요한 것만 직렬화하고
그 이상은 하지 않습니다. 모든 클라이언트의 모든 파일 앞에 뮤텍스 하나를 두는 것은
파일 서버가 할 수 있는 가장 느린 일이 될 것입니다.
{{< /callout >}}

⛔ 공유는 `image` **또는** `directory`를 가지며, 둘 다 있거나 둘 다 없을 수
없습니다. `filesystem`과 `partition`은 이미지를 위한 것입니다. 이 중 하나를
지정한 디렉터리 공유는 거부되는데, 지정했다는 것은 공유가 이미지로 의도되었다는
뜻이기 때문입니다. `read_only = true`는 이미지와 마찬가지로 디렉터리에서도
동작합니다. 이를 지정하면서 `writers`도 나열한 공유는 `read_only`가 우선하게
되므로 거부됩니다.

클라이언트에 알려 주는 용량은 트리가 있는 파일 시스템의 용량입니다. 이를 알 수
없는 플랫폼은 0으로 두며, 프로토콜은 이를 "가득 참"이 아니라 "알 수 없음"으로
해석합니다.

## 저장할 수 없는 시간을 SMB가 처리하는 방식 {#what-smb-does-with-a-time-it-cannot-store}

클라이언트가 설정한 시간을 저장소가 보존할 수 없으면 기본적으로 **무시됩니다**.
v0.42.0부터 공유마다 `unstorable_times`로 선택할 수 있습니다. 저장할 수 없는 것은
보고된 값과 다른 CreationTime, 단독으로 전송된 ChangeTime, 이미지에 대한 접근 시간
또는 수정 시간 중 시간을 설정할 수 없는 드라이버(NTFS, APFS, ISO 9660, SquashFS)의
이미지에 대한 것입니다(호스트 디렉터리와 FAT, exFAT, ext4, HFS+ 이미지는 이를 저장합니다).

```hcl
share "photos" {
  directory        = "/data/photos"
  unstorable_times = "log"      # "ignore"(기본값), "refuse", "log"
}
```

`refuse`는 클라이언트에 오류로 응답하고, `log`는 공유 이름과 경로를 적은 한 줄을
기록합니다. 공유 종류별 정확한 경우는
[실제 파일 시간]({{< relref "/protocols/times.md#when-unstorable" >}})을 참고하세요.
