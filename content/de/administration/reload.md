---
title: "Das Verzeichnis erneut einlesen"
weight: 30
description: "Wie und wann der Server seine Benutzerverzeichnisse erneut einliest und was ein Neuladen mit Freigaben und offenen Verbindungen macht."
tags: [administration, identität, neuladen]
---

```hcl
reload = "5m"   # and on SIGHUP, and on the admin API's ReloadDirectory
```

Die Personen kommen aus Verzeichnissen, die andere verändern – go-authn/bridge schreibt
ein Anwendungspasswort in eine Tabelle und löscht die Zeile, wenn die Person
deaktiviert wird. Ohne Neuladen bedient der Server, wer beim Start da war.
Seit v0.10.0 liest er sie erneut ein:

- in jedem `reload`-Intervall, einer Go-Dauer von **mindestens einer Sekunde** – ein Verzeichnis,
  das viele Male pro Sekunde gelesen wird, ist eine Last für die Datenbank von jemandem, kein aktuellerer
  Server;
- bei **`SIGHUP`** (Unix);
- bei **`ReloadDirectory`** der [Admin-API]({{< relref "/administration/_index.md" >}}), das antwortet, wer
  hinzugefügt, entfernt und geändert wurde, ob eine neue Generation gestartet ist, und Hinweise gibt, die
  man lesen sollte – eine Gruppe, die es nicht mehr gibt, jemand, den eine Freigabe nennt,
  der nicht mehr im Verzeichnis steht.

Ohne `reload` wird das Verzeichnis nur beim Start, bei `SIGHUP` und bei
`ReloadDirectory` gelesen.

Es sind die `users`-Blöcke – SQL, LDAP –, die erneut gelesen werden. Die `user`- und
`group`-Blöcke der Konfigurationsdatei **sind** die Konfiguration und werden beim
Start gelesen.

## Was ein Neuladen tut, hängt davon ab, was sich geändert hat {#what-a-reload-does-depends-on-what-changed}

Die Grenze verläuft beim **Widerruf**:

| was sich geändert hat | was geschieht |
|---|---|
| **nur Hinzufügungen** – jemand Neues, dessen Ankunft die aufgelösten Listen keiner Freigabe ändert (ein neues Anwendungspasswort, eine Person, die die Freigaben über `oidc:`-Regeln oder gar nicht erreichen) | **an Ort und Stelle** hinzugefügt, einschließlich des laufenden SMB-Servers, und **keine Verbindung wird angetastet**. Eine Bridge, die den ganzen Tag Anwendungspasswörter anlegt, stört niemanden. |
| **alles, was entzogen wird** – jemand ist weg, Zugangsdaten haben sich geändert, eine Freigabe, deren aufgelöste Listen sich geändert haben | eine **neue Generation**, und die Verbindungen der alten werden **geschlossen**, genau wie bei einer [Admin-Änderung]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}}). Eine Sitzung, die die Entfernung ihrer Person überdauert hat, ist genau das, was ein Neuladen beenden soll. |
| **ein Verzeichnis, das nicht gelesen werden kann** – die Datenbank nicht erreichbar, eine Abfrage schlägt fehl | **nichts ändert sich**. Ein Ausfall darf keinen Dateiserver leeren, und „niemand" ist genau das, wie ein fehlgeschlagenes Lesen aussieht. |

Jemand Neues, der einer Gruppe beitritt, die eine Freigabe nennt, *ändert* diese Freigabe sehr wohl und durchläuft
eine neue Generation wie jede andere Änderung an ihr: SMB legt die Listen einer Freigabe
bei deren Start fest.

Eine Freigabe, die jemanden nennt, der weg ist, oder eine Gruppe, die es nicht mehr gibt, wird
**ohne sie** bereitgestellt – die sichere Richtung – und das wird gemeldet, statt sie abzulehnen, wie es
der Start tut: Ablehnen würde die alten Listen behalten, und die alten Listen sind der
Zugriff, der gerade entzogen wurde.

## Eine leere Gruppe ist nicht jeder {#an-empty-group-is-not-everyone}

{{< callout type="error" >}}
**Eine für `@engineers` geschriebene Freigabe, deren letzter Engineer gegangen ist, wird niemandem bereitgestellt**

Ein leeres `allow` bedeutet „jeder, der sich authentifiziert". Was also darüber entscheidet, ob
eine Freigabe offen ist, ist das, was **geschrieben** wurde, nicht das, wozu es jetzt aufgelöst wird: Eine
für eine Gruppe geschriebene Freigabe bleibt geschlossen, wenn diese Gruppe sich leert. Eine solche Freigabe wird
über SMB gar nicht erst angeboten, wo ein leeres `AllowUsers` als
jeder gelesen würde.
{{< /callout >}}

## Es beobachten {#watching-it}

`fileshare_directory_reloads_total{result}` zählt Neuladevorgänge, die nichts gefunden haben
(`unchanged`), an Ort und Stelle hinzugefügt haben (`added`), eine Generation gestartet haben (`swapped`) oder
das Verzeichnis nicht lesen konnten (`failed`); `fileshare_directory_people` ist die Anzahl
der Personen beim letzten Lesen. Siehe [Health und Metriken]({{< relref "/administration/health.md" >}}).

{{< callout type="info" >}}
**Was ein Neuladen nicht erreicht**

Personen, für die der Identitätsanbieter über ein Token oder ein Zertifikat bürgt,
stehen nicht im Verzeichnis, also betrifft sie kein Neuladen. **Sie** zurückzunehmen
ist die Aufgabe von [Sperrlisten und Shared Signals]({{< relref "/security/_index.md" >}}).
{{< /callout >}}
