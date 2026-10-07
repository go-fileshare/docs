---
title: "Ein Prozess pro Protokoll"
weight: 20
description: "Jedes Protokoll mit --isolate in einem eigenen Prozess ausführen, und die Regeln, die das sicher machen."
tags: [betrieb, isolierung]
---

```sh
fileshare --config /etc/fileshare.d --isolate
```

```
smb    on 0.0.0.0:445  — process 19810, serving public, photos and scratch
webdav on 0.0.0.0:8080 — process 19811, serving public and photos
nfs    on 0.0.0.0:2049 — process 19812, serving public
```

Der Elternprozess bindet die Listener und führt dann **sich selbst** einmal pro Protokoll aus (exec),
wobei er jedem Kindprozess nur die Freigaben übergibt, die dieses Protokoll bereitstellen darf.

## Mit `lsof` geprüft, nicht durch Lesen des Codes {#checked-with-lsof-not-by-reading-the-code}

```
smb     (pid 19810) has open: scratch.img photos.img
webdav  (pid 19811) has open: photos.img
nfs     (pid 19812) has open: photos.img
```

Das beschreibbare Image ist in **genau einem Prozess** geöffnet, und der WebDAV-Kindprozess
öffnet es überhaupt nie.

Das ist es, was [Build-Tags]({{< relref "/operations/build-tags.md" >}}) nicht leisten können: Ein Panic oder ein erschöpfter
Heap in einem Protokoll legt ein Protokoll lahm, jeder Kindprozess kann mit allem eingeschränkt werden,
was das Betriebssystem bietet, und ein Fehler in einem Parser kann kein
Image erreichen, das dieser Prozess nie geöffnet hat.

## Privilegierte Ports {#privileged-ports}

Unter Unix übergibt der Elternprozess den gebundenen Listener an den Kindprozess, sodass **ein privilegierter
Port mit unprivilegierten Kindprozessen funktioniert** – der Elternprozess bindet 445, der Kindprozess braucht
das Privileg nie.

Windows hat keine `ExtraFiles`, daher bindet der Kindprozess dort die Adresse selbst; die
Isolierung ist dieselbe, die Hälfte mit dem privilegierten Port nicht.

## Die Regel, die es ehrlich macht {#the-rule-that-makes-it-honest}

Ein Kindprozess öffnet das Image selbst, sodass ein Image, das zwei Protokolle *beschreibbar* bereitstellen,
zwei Treiber über einer Datei wäre, **ohne Sperre zwischen ihnen** – genau das, was
die [gemeinsame Sperre]({{< relref "/operations/locking.md" >}}) innerhalb eines Prozesses verhindert.

Das wird abgelehnt, und die Ablehnung sagt, wie es zu beheben ist:

```
scratch is writable over smb and webdav, and one process per protocol means
that many drivers writing one file with no lock between them. Say
`protocols = ["smb"]` on the share, or make it read_only, or do not isolate.
```

`protocols = [...]` an einer Freigabe ist für sich allein nützlich, nicht nur unter
`--isolate`: Eine als nur SMB deklarierte Freigabe ist eine Freigabe, von der der WebDAV-Prozess nie
erfährt. Ein `serve`-Block, der am Ende nichts transportieren würde, wird ebenfalls abgelehnt,
bevor ein Image geöffnet wird, unter Nennung der Freigaben, die ihm vorenthalten wurden, und warum.

## Noch nicht mit einem `admin`- oder einem `metrics`-Block {#not-with-an-admin-or-a-metrics-block-yet}

`--isolate` zusammen mit einem [`admin`]({{< relref "/administration/_index.md" >}})- oder einem
[`metrics`]({{< relref "/administration/health.md" >}})-Block wird abgelehnt: Die Kindprozesse öffnen
die Images und der Elternprozess öffnet nichts, also gibt es keinen einzelnen Prozess, auf den eine Änderung
über die API angewendet werden könnte oder nach dessen Bereitschaft eine Probe fragen würde.
