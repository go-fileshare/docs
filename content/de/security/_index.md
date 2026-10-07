---
title: "Was was widerruft"
linkTitle: "Sicherheit"
weight: 40
description: "Welcher Mechanismus jede Art von Zugangsdaten zurücknimmt, die fileshare akzeptiert, und warum jeder davon ausfallsicher geschlossen ist."
tags: [sicherheit, widerruf]
---

Zugangsdaten werden geprüft, wenn sie vorgelegt werden. Einer Person den Zugang
danach wieder zu entziehen – einem ausgeschiedenen Mitarbeiter, einem in
go-authn/bridge deaktivierten Konto, einem ganzen abgeschalteten IdP –, muss
**alle** Zugangsdaten erreichen, die ihr ausgestellt wurden, und die Sitzungen,
die damit bereits geöffnet sind. Jede Art von Zugangsdaten wird hier über einen
anderen Mechanismus erreicht, weil jede anders ausgestellt und geprüft wird:

| Zugangsdaten | Protokoll | was sie zurücknimmt | hinzugefügt in |
|---|---|---|---|
| ein Passwort, NT-Hash oder Anwendungspasswort in einem `users`-Verzeichnis (SQL, LDAP) | SMB, WebDAV, S3 | die entfernte Zeile, dann ein [Neuladen des Verzeichnisses]({{< relref "/administration/reload.md" >}}): eine neue Generation, und die damit geöffneten Verbindungen werden geschlossen | v0.10.0 |
| eine Berechtigung für eine Freigabe | jedes Protokoll | `Revoke` oder `DisableShare` der [Admin-API]({{< relref "/administration/_index.md" >}}): eine neue Generation, Verbindungen geschlossen | v0.8.0 |
| ein SSH-Zertifikat, das die CA des Anbieters signiert hat | SFTP | seine [KRL]({{< relref "/security/revocation-lists.md" >}}), geprüft bei der Anmeldung **und bei jeder Operation**; und [Shared Signals]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |
| ein OpenPubkey-Zertifikat (opkssh) | SFTP | keine Liste kann es benennen; [Shared Signals]({{< relref "/security/shared-signals.md" >}}) und `opkssh_max_age` | v0.13.0 |
| ein Access-Token | WebDAV | keine Liste kann es benennen; [Shared Signals]({{< relref "/security/shared-signals.md" >}}) | v0.13.0 |
| ein X.509-Client-Zertifikat, das eine Person benennt | NFS | seine [Sperrliste (CRL)]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}), beim nächsten Aufruf; und [Shared Signals]({{< relref "/security/shared-signals.md" >}}) | v0.12.0, v0.13.0 |

Ohne den Mechanismus in der dritten Spalte bleiben Zugangsdaten gültig, bis sie
ablaufen – und bei einem Zertifikat oder einem Token ist dieses Fenster seine
gesamte Lebensdauer.

## Jeder davon ist ausfallsicher geschlossen {#every-one-of-them-fails-closed}

Eine Widerrufsliste, die nicht abgerufen werden kann, eine Kopie, die älter ist als
ihr `max_age`, eine CRL jenseits ihres eigenen `NextUpdate`, ein Shared-Signals-Transmitter,
von dem innerhalb von `max_age` nichts gekommen ist: Nichts davon wird als „nichts ist
widerrufen“ gelesen. Es bedeutet „dieser Server weiß es nicht“, und die Zugangsdaten, die
dieser Mechanismus regelt, werden **abgelehnt**, bis er es wieder weiß (fail closed).

{{< callout type="error" >}}
**Der Preis ist die Verfügbarkeit, und er wird absichtlich gezahlt**

Eine Widerrufsprüfung, die alles durchlässt, wenn die Liste nicht erreichbar
ist, ist genau die, die ein Angreifer aushebelt, der die Liste blockieren kann –
der Grund, warum das Soft-Fail-OCSP der Browser „ein Sicherheitsgurt, der beim
Aufprall reißt“ genannt wurde. Die Liste oder der Transmitter muss also so
zuverlässig bereitgestellt werden wie die Anmeldungen, die sie regelt, und die
Metriken, die anzeigen, dass sie veraltet, sind diejenigen, auf die
[alarmiert werden sollte]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

Eine Kopie wird nur durch eine Liste ersetzt, die sich parsen ließ: Eine Liste, die
beschädigt ankommt, lässt die letzte gute an ihrem Platz, deren `max_age` weiterhin
ab dem Zeitpunkt zählt, zu dem **sie** abgerufen wurde. Und eine Liste wird
**nur über HTTPS** abgerufen – eine Liste, die ein Angreifer auf dem Pfad ersetzen
kann, widerruft nichts –, daher wird eine reine HTTP-URL beim Start abgelehnt.

## Transport {#transport}

Die Zugangsdaten selbst dürfen kein Netzwerk unverschlüsselt durchqueren. Das ist
[TLS]({{< relref "/security/tls.md" >}}): WebDAV, S3 und NFS über TLS, ein Zertifikat aus Dateien oder per ACME,
und – seit v0.9.0 – **WebDAV mit Passwörtern, unverschlüsselt abgelehnt**.
