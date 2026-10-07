---
title: "Ein Image, mehrere Protokolle, eine Sperre"
weight: 30
description: "Warum ein über mehrere Protokolle bereitgestelltes Image in eine gemeinsame Sperre gehüllt wird und was diese Hülle bewahrt."
tags: [betrieb, sperren]
---

Jeder Server dieser Familie serialisiert den Treiber selbst, weil
[`go-filesystems/interface`](https://github.com/go-filesystems/interface)
nichts über gleichzeitige pfadbasierte Aufrufe zusichert – und jeder von ihnen hält
**seine eigene** Sperre, ohne etwas von den anderen zu wissen.

Ein Image gleichzeitig über drei Protokolle bereitzustellen, hätte daher nirgends eine gemeinsame
Sperre. Also wird das Image **einmal** umhüllt, hier, und dieselbe Hülle wird
allen übergeben.

## Als gemessen ausgewiesen, um nicht zu übertreiben {#stated-as-measured-so-as-not-to-overstate-it}

Acht Goroutinen, die unter `-race` über einen **nicht umhüllten** fat32-Treiber schreiben,
erzeugen **heute keinen Race**.

Das ist eine Absicherung eines Vertrags, keine Reproduktion eines Fehlers – ext4 und ntfs
sind nicht fat32, und ein Treiber-Update ist nichts, was dieses Programm erneut
prüfen müssen sollte.

## Die Hülle reicht Fähigkeiten durch {#the-wrapper-carries-capabilities-through}

Sie verbirgt sie nicht:

- ein Treiber, der `Opener` beantwortet, erhält eine **gesperrte `File`** zurück;
- einer, dessen `File` `WritableFile` beantwortet, behält seine **positionellen Schreibzugriffe**.

Der Unterschied zwischen diesen und dem Rückgriff auf die ganze Datei wurde an anderer Stelle der Familie mit
**~70×** gemessen, sodass eine Hülle, die sie verwischt, ein als Sicherheit
getarnter Leistungsfehler wäre.

## Unter `--isolate` kann die Sperre nicht helfen {#under-isolate-the-lock-cannot-help}

Ein Kindprozess öffnet das Image selbst, also gibt es keine gemeinsame Hülle zwischen
den Kindprozessen. Deshalb wird ein Image, das zwei Protokolle beschreibbar bereitstellen,
[abgelehnt]({{< relref "/operations/isolation.md#the-rule-that-makes-it-honest" >}}), statt stillschweigend
ohne Sperre bereitgestellt zu werden.
