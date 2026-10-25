---
title: "운영"
weight: 50
description: "바이너리를 빌드하는 방법, 프로토콜을 실행하는 방법, 그리고 프로토콜들이 이미지를 공유하는 방법."
tags: [운영]
---

{{< cards >}}
  {{< card link="build-tags/" title="필요한 것만 빌드하기" icon="cube" subtitle="빌드 태그는 프로토콜이나 관리 API를 바이너리에서 완전히 제외합니다." >}}
  {{< card link="isolation/" title="프로토콜마다 하나의 프로세스" icon="view-grid" subtitle="--isolate: 각 프로토콜을 자체 프로세스에서 실행합니다." >}}
  {{< card link="locking/" title="이미지 하나, 여러 프로토콜, 잠금 하나" icon="lock-closed" subtitle="여러 프로토콜에서 하나의 이미지로 가는 쓰기는 하나의 공유 잠금을 거칩니다." >}}
{{< /cards >}}
