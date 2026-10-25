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
