---
title: "Widerrufen, was keine Liste abdeckt: Shared Signals"
linkTitle: "Shared Signals"
weight: 40
description: "Access-Tokens, OpenPubkey-Zertifikate und andere föderierte Zugangsdaten mit CAEP-session-revoked-Ereignissen von einem Shared-Signals-Transmitter widerrufen."
tags: [sicherheit, widerruf, shared signals]
---

Ein Zertifikat hat eine Widerrufsliste – die [KRL]({{< relref "/security/revocation-lists.md" >}}), die
[CRL]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}). Ein **Access-Token**, das fileshare
selbst prüft, und ein **OpenPubkey-Zertifikat**, das der eigene Schlüssel der Person
signiert hat, haben keine: Sie sind gültig, bis sie ablaufen, was auch immer der
Person seitdem widerfahren ist.

Seit v0.13.0 schließt das [OpenID Shared Signals Framework](https://openid.net/specs/openid-sharedsignals-framework-1_0-final.html)
diese Lücke, mit einem [CAEP](https://openid.net/specs/openid-caep-1_0-final.html)-Ereignis
**`session-revoked`**:

```hcl
ssf {
  transmitter        = "https://bridge.example.org"          # its issuer
  audience           = "https://files.example.org"           # what this server is to it
  client_id          = "fileshare"                           # OAuth client credentials,
  client_secret_file = "/etc/fileshare/ssf.secret"           # scope "ssf" (or token_file)
  state_file         = "/var/lib/fileshare/revocations.json"
  max_age            = "10m"                                 # default; retain = "192h"
}
```

go-authn/bridge sendet eines, wenn eine Person oder ein IdP deaktiviert wird; fileshare
**fragt es ab** (Polling, RFC 8936, der SSF-Standard) und hält pro Person fest, **wann**
es geschah.

## Alles zuvor Ausgestellte wird abgelehnt, bei jedem Protokoll {#everything-issued-before-is-refused-on-every-protocol}

Ab diesem Moment werden alle föderierten Zugangsdaten abgelehnt, die **davor ausgestellt**
wurden – und **die Sitzungen, die damit geöffnet wurden, werden nicht mehr bedient**, über
dieselben Kontrollpunkte wie bei der KRL. Was „ausgestellt“ bedeutet, hängt davon ab, was
sie getragen hat:

| Protokoll | Zugangsdaten | ihr Ausstellungszeitpunkt |
|---|---|---|
| WebDAV | ein Access-Token | sein `iat` |
| SFTP | ein Zertifikat, das die SSH-CA des Anbieters signiert hat | sein Gültigkeitsbeginn (`ValidAfter`) |
| SFTP | ein OpenPubkey-Zertifikat (opkssh) | das `iat` des ID-Tokens |
| NFS | ein Client-Zertifikat ([identity = "certificate"]({{< relref "/security/nfs-certificates.md" >}})) | sein `NotBefore` |

Was der Anbieter **danach** ausstellt, ist seine Sache: Eine wieder aktivierte Person wird
nicht ausgesperrt.

{{< callout type="info" >}}
**Dieselbe Sekunde zählt als davor**

`event_timestamp` von CAEP und das `iat` eines Tokens sind ganze Sekunden, und
Zugangsdaten, die **zum Zeitpunkt des Widerrufs oder davor** ausgestellt wurden, werden
abgelehnt – daher werden auch solche abgelehnt, die in derselben Sekunde wie ein Widerruf
ausgestellt wurden, und eine Person, die direkt nach ihrer Deaktivierung wieder aktiviert
wird, erhält ab der nächsten Sekunde funktionierende Zugangsdaten. Das ist die vorsichtige
Seite, mit Absicht. Ein unbekannter Ausstellungszeitpunkt ist nicht „lange her“: Er wird
abgelehnt, sobald es irgendeinen Widerruf für diese Person gibt.
{{< /callout >}}

## Wen ein Ereignis betrifft {#who-an-event-is-about}

Das Subjekt ist **`account`** aus RFC 9493 (`acct:user@domain`, der Name, den die
Freigaben verwenden), **`iss_sub`**, **`email`** oder **`aliases`** davon.

**Ein vollständig deaktivierter IdP** kommt als CAEP-*tenant*-Subjekt mit den Scopes des IdP
im Ereignis an: Alle, deren Name `@` einer davon ist – als Ganzes verglichen,
`evil-univ-a.fr` ist nicht `univ-a.fr` – und deren Zugangsdaten davor ausgestellt wurden,
werden abgelehnt, **einschließlich Personen, an die sich der Anbieter nicht mehr erinnert**.

## Wie sich der Receiver authentifiziert {#how-the-receiver-authenticates}

Der Receiver (Empfänger) authentifiziert sich mit **OAuth-Client-Credentials** (RFC 6749 §4.4,
Scope `ssf`): `client_id` und `client_secret_file`, das Token wird vom eigenen
Token-Endpunkt des Transmitters abgerufen – gefunden in dessen OpenID-Konfiguration oder
über `token_url` – und vor seinem Ablauf erneuert. `token_file` gibt es stattdessen für
Transmitter, die ein langlebiges Bearer-Token ausgeben. Das eine oder das andere: beides
oder keines wird abgelehnt.

| Feld | Standard | |
|---|---|---|
| `transmitter` | | sein Issuer, **nur HTTPS** |
| `audience` | | was dieser Server für den Transmitter ist; erforderlich, sonst würde hier einem SET geglaubt, das an einen anderen Receiver adressiert ist |
| `client_id`, `client_secret_file` | | OAuth-Client-Credentials, zusammen |
| `token_url` | ermittelt | nur HTTPS, sonst geht das Geheimnis unverschlüsselt über das Netzwerk |
| `token_file` | | ein langlebiges Token statt Client-Credentials |
| `state_file` | | **erforderlich**: wo Widerrufe festgehalten werden |
| `ca_file` | die des Systems | legt die Zertifizierungsstellen des Transmitters fest |
| `max_age` | `10m` | wie lange ohne Nachricht vom Transmitter, bevor föderierte Zugangsdaten abgelehnt werden; mindestens eine Minute |
| `retain` | `192h` | wie lange ein Widerruf aufbewahrt wird; mindestens 169h (seit v0.14.0) |

Ein Widerruf wird **festgehalten, bevor er bestätigt wird**, sodass ein bestätigter einen
Neustart übersteht; und er wird für `retain` aufbewahrt – länger als alle Zugangsdaten,
die er ungültig machen könnte (192h liegt über den am längsten gültigen Zugangsdaten, die
der Anbieter ausstellt, einem SSH-Zertifikat mit 168h).

Ein `ssf`-Block ohne etwas Föderiertes zum Widerrufen – kein `oidc`-Block, kein NFS mit
`identity = "certificate"` – wird abgelehnt.

Der Transport ist [github.com/hstern/go-ssf](https://github.com/hstern/go-ssf):
Discovery, der Poller nach RFC 8936, die JWS-Schicht des SET. Was ein Ereignis
**bedeutet**, entscheidet fileshare.

## Er ist ausfallsicher geschlossen {#it-fails-closed}

{{< callout type="error" >}}
**Solange der Transmitter schweigt, werden föderierte Zugangsdaten abgelehnt**

Wie bei der KRL: Solange der Transmitter eine Abfrage nicht innerhalb von
`max_age` beantwortet hat, werden **alle föderierten Zugangsdaten** abgelehnt (fail closed),
weil „kein Widerruf ist angekommen“ und „keiner konnte ankommen“ gleich aussehen.
`fileshare_ssf_last_heard_seconds` ist die Metrik, auf die alarmiert werden sollte;
`fileshare_ssf_revoked_subjects` gibt an, wie viele Widerrufe aufbewahrt werden. Siehe
[Health und Metriken]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

## Durchgängig geprüft {#checked-end-to-end}

Die Interop-Lane von go-authn/bridge
([#12](https://github.com/go-authn/bridge/pull/12),
[#17](https://github.com/go-authn/bridge/pull/17)) prüft fileshare v0.13.0
gegen die echte Bridge: die KRL, die CRL über NFS, über SSF widerrufene WebDAV-Tokens,
einen per Scope deaktivierten IdP und opkssh über SSF – den Fall, den keine Widerrufsliste
erreicht.
