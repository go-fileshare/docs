---
title: "프로토콜마다 하나의 프로세스"
weight: 20
description: "--isolate로 각 프로토콜을 자체 프로세스에서 실행하는 방법과 이를 안전하게 유지하는 규칙."
tags: [운영, 격리]
---

```sh
fileshare --config /etc/fileshare.d --isolate
```

```
smb    on 0.0.0.0:445  — process 19810, serving public, photos and scratch
webdav on 0.0.0.0:8080 — process 19811, serving public and photos
nfs    on 0.0.0.0:2049 — process 19812, serving public
```

부모는 리스너를 바인딩한 다음 프로토콜마다 **자기 자신을** 한 번씩 exec하며,
각 자식에게는 해당 프로토콜이 제공할 수 있는 공유만 넘깁니다.

## 코드를 읽어서가 아니라 `lsof`로 확인했습니다 {#checked-with-lsof-not-by-reading-the-code}

```
smb     (pid 19810) has open: scratch.img photos.img
webdav  (pid 19811) has open: photos.img
nfs     (pid 19812) has open: photos.img
```

쓰기 가능한 이미지는 **정확히 하나의 프로세스**에서만 열려 있으며, WebDAV
자식은 이를 전혀 열지 않습니다.

이것이 [빌드 태그]({{< relref "/operations/build-tags.md" >}})가 줄 수 없는 것입니다.
한 프로토콜의 패닉이나 고갈된 힙은 그 프로토콜 하나만 중단시키고, 각 자식은
운영 체제가 제공하는 수단으로 가둘 수 있으며, 한 파서의 버그는 그 프로세스가
열어 본 적 없는 이미지에는 닿을 수 없습니다.

## 권한이 필요한 포트 {#privileged-ports}

Unix에서는 부모가 바인딩한 리스너를 자식에게 넘기므로, **권한이 필요한 포트도
권한 없는 자식으로 동작합니다**. 부모가 445를 바인딩하고 자식은 그 권한이 전혀
필요 없습니다.

Windows에는 `ExtraFiles`가 없으므로 자식이 직접 주소를 바인딩합니다. 격리는
같지만, 권한이 필요한 포트에 관한 절반은 해당하지 않습니다.

## 정직하게 만드는 규칙 {#the-rule-that-makes-it-honest}

자식은 이미지를 직접 열므로, 두 프로토콜이 *쓰기 가능하게* 제공하는 이미지는
**서로 간에 잠금이 없는** 하나의 파일 위의 두 드라이버가 됩니다. 이는 한
프로세스 안에서 [공유 잠금]({{< relref "/operations/locking.md" >}})이 막아 주는
바로 그 상황입니다.

이는 거부되며, 거부 메시지가 해결 방법을 알려 줍니다.

```
scratch is writable over smb and webdav, and one process per protocol means
that many drivers writing one file with no lock between them. Say
`protocols = ["smb"]` on the share, or make it read_only, or do not isolate.
```

공유의 `protocols = [...]`는 `--isolate`에서만이 아니라 그 자체로도 유용합니다.
SMB 전용으로 선언된 공유는 WebDAV 프로세스가 전혀 알지 못하는 공유입니다.
결국 아무것도 제공하지 않게 될 `serve` 블록도 이미지가 열리기 전에 거부되며, 그
블록에서 제외된 공유와 이유가 함께 표시됩니다.

## 아직은 `admin` 또는 `metrics` 블록과 함께 쓸 수 없습니다 {#not-with-an-admin-or-a-metrics-block-yet}

`--isolate`를 [`admin`]({{< relref "/administration/_index.md" >}}) 또는
[`metrics`]({{< relref "/administration/health.md" >}}) 블록과 함께 쓰면 거부됩니다.
자식이 이미지를 열고 부모는 아무것도 열지 않으므로, API 변경을 적용할 단일
프로세스도, 프로브가 준비 상태를 물을 단일 프로세스도 없기 때문입니다.
