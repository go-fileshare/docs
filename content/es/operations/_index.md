---
title: "Operación"
weight: 50
description: "Cómo se compila un binario, cómo se ejecutan sus protocolos y cómo comparten una imagen."
tags: [operación]
---

{{< cards >}}
  {{< card link="build-tags/" title="Compilar solo lo que necesita" icon="cube" subtitle="Una etiqueta de compilación deja un protocolo, o la API de administración, completamente fuera del binario." >}}
  {{< card link="isolation/" title="Un proceso por protocolo" icon="view-grid" subtitle="--isolate: cada protocolo en un proceso propio." >}}
  {{< card link="locking/" title="Una imagen, varios protocolos, un bloqueo" icon="lock-closed" subtitle="Las escrituras de varios protocolos en una misma imagen pasan por un único bloqueo compartido." >}}
{{< /cards >}}
