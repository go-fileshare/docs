---
title: "Operations"
weight: 50
description: "How a binary is built, how its protocols are run, and how they share an image."
tags: [operations]
---

{{< cards >}}
  {{< card link="build-tags/" title="Building only what you want" icon="cube" subtitle="A build tag leaves a protocol, or the admin API, out of the binary entirely." >}}
  {{< card link="isolation/" title="One process per protocol" icon="view-grid" subtitle="--isolate: each protocol in a process of its own." >}}
  {{< card link="locking/" title="One image, several protocols, one lock" icon="lock-closed" subtitle="Writes from several protocols to one image go through one shared lock." >}}
{{< /cards >}}
