---
title: "Exploitation"
weight: 50
description: "Comment un binaire est compilé, comment ses protocoles sont exécutés, et comment ils partagent une image."
tags: [exploitation]
---

{{< cards >}}
  {{< card link="build-tags/" title="Ne compiler que ce que l'on veut" icon="cube" subtitle="Une étiquette de compilation laisse un protocole, ou l'API d'administration, entièrement hors du binaire." >}}
  {{< card link="isolation/" title="Un processus par protocole" icon="view-grid" subtitle="--isolate : chaque protocole dans son propre processus." >}}
  {{< card link="locking/" title="Une image, plusieurs protocoles, un verrou" icon="lock-closed" subtitle="Les écritures de plusieurs protocoles sur une même image passent par un seul verrou partagé." >}}
{{< /cards >}}
