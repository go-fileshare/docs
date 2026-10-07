---
title: "Betrieb"
weight: 50
description: "Wie ein Binary gebaut wird, wie seine Protokolle laufen und wie sie sich ein Image teilen."
tags: [betrieb]
---

{{< cards >}}
  {{< card link="build-tags/" title="Nur bauen, was Sie brauchen" icon="cube" subtitle="Ein Build-Tag lässt ein Protokoll oder die Admin-API ganz aus dem Binary weg." >}}
  {{< card link="isolation/" title="Ein Prozess pro Protokoll" icon="view-grid" subtitle="--isolate: jedes Protokoll in einem eigenen Prozess." >}}
  {{< card link="locking/" title="Ein Image, mehrere Protokolle, eine Sperre" icon="lock-closed" subtitle="Schreibzugriffe mehrerer Protokolle auf ein Image laufen über eine gemeinsame Sperre." >}}
{{< /cards >}}
