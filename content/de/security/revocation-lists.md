---
title: "SSH-Zertifikate widerrufen: die KRL"
weight: 20
description: "Die SSH-Zertifikate des Identitätsanbieters mit einer signierten, datierten OpenSSH-KRL widerrufen, die bei der Anmeldung und bei jeder SFTP-Operation geprüft wird."
tags: [sicherheit, widerruf, sftp, ssh]
---

Ein Zertifikat, das die SSH-CA des Identitätsanbieters signiert hat – siehe
[SFTP für Personen, für die der Identitätsanbieter bürgt]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})
–, wird bei der Anmeldung geprüft. Jemanden in go-authn/bridge zu deaktivieren, widerruft
seine Tokens und löscht seine Anwendungspasswörter, was ein
[Neuladen des Verzeichnisses]({{< relref "/administration/reload.md" >}}) hier anwendet; ohne eine
Liste würde ein bereits ausgestelltes Zertifikat SFTP jedoch weiterhin öffnen, bis es abläuft, und
eine bereits geöffnete SFTP-Sitzung bliebe offen.

Seit v0.12.0 schließt eine Liste diese Lücke:

```hcl
oidc {
  issuer          = "https://login.example.org"
  audience        = "fileshare"
  ssh_ca_file     = "/etc/fileshare/bridge-ca.pub"
  ssh_krl_url        = "https://bridge.example.org/ssh/krl"   # or ssh_krl_file
  ssh_krl_state_file = "/var/lib/fileshare/ssh.krl.state"    # the order, across a restart
  ssh_krl_max_age    = "1h"                                  # default; ssh_krl_refresh = "1m"
}
```

go-authn/bridge veröffentlicht die SSH-Zertifikate, die es widerrufen hat – eine Person
deaktiviert, ein IdP deaktiviert –, als OpenSSH-**KRL** (`PROTOCOL.krl`, die Liste, die
`RevokedKeys` von `sshd` liest), und fileshare hält eine Kopie davon, gelesen mit
[go-authn/krl](https://github.com/go-authn/krl).

## Abgelehnt bei der Anmeldung und mitten in einer Sitzung {#refused-at-login-and-in-the-middle-of-a-session}

Ein widerrufenes Zertifikat wird bei der Anmeldung abgelehnt, und **eine Sitzung, die es
bereits geöffnet hat, wird nicht mehr bedient**. SSH prüft ein Zertifikat einmal, beim
Handshake; daher fragt jede Operation einer föderierten SFTP-Sitzung erneut die KRL,
**geöffnete Dateien eingeschlossen**. Die erste Operation, nachdem sich die Kopie der Liste
auf diesem Server geändert hat, wird abgelehnt.

## Felder {#fields}

| Feld | Standard | |
|---|---|---|
| `ssh_krl_url` | | wo die Liste abgerufen wird, **nur HTTPS** |
| `ssh_krl_file` | | oder eine Datei, per absolutem Pfad – das eine oder das andere, nie beides |
| `ssh_krl_ca_file` | die des Systems | legt die Zertifizierungsstellen fest, gegen die der HTTPS-Server der Liste geprüft wird |
| `ssh_krl_refresh` | `1m` | wie oft sie abgerufen wird, mindestens eine Sekunde |
| `ssh_krl_max_age` | `1h` | wie alt die letzte gute Kopie sein darf, bevor die Zertifikate, die sie regelt, abgelehnt werden |
| `ssh_krl_state_file` | | wo die letzte geprüfte Liste aufbewahrt wird, damit sich ein Neustart an die Reihenfolge erinnert |

Ein `max_age`, das kürzer ist als `refresh`, wird abgelehnt – jede Kopie wäre vor dem
nächsten Abruf veraltet. Eine KRL ohne `ssh_ca_file` wird ebenfalls abgelehnt: Sie
widerruft die Zertifikate des Anbieters, und ohne diese CA gibt es keine.

## Sie ist ausfallsicher geschlossen {#it-fails-closed}

{{< callout type="error" >}}
**Solange die KRL nicht abgerufen werden kann, werden die Zertifikate des Anbieters abgelehnt**

Solange die Liste nicht abgerufen werden kann oder ihre letzte gute Kopie älter ist als
`ssh_krl_max_age`, wird **jedes** Zertifikat abgelehnt, das die CA des Anbieters
signiert hat (fail closed): Eine Widerrufsprüfung, die alles durchlässt, wenn die Liste
nicht erreichbar ist, ist genau die, die ein Angreifer aushebelt, der die Liste blockieren
kann. Die Liste muss dann so zuverlässig bereitgestellt werden wie die Anmeldungen, die
sie regelt. `fileshare_revocation_list_age_seconds{list="ssh_krl"}` ist die Metrik,
auf die alarmiert werden sollte, bevor `max_age` erreicht ist; siehe
[Health und Metriken]({{< relref "/administration/health.md#what-to-alert-on" >}}).
{{< /callout >}}

## Signiert und datiert, von `ssh_krl_url` {#signed-and-dated-from-ssh_krl_url}

Seit v0.15.0 **muss eine KRL von `ssh_krl_url` signiert sein und angeben, wann sie
abläuft**, wie es [go-authn/revocation](https://github.com/go-authn/revocation)
beschreibt und go-authn/bridge ab v0.10.0 bereitstellt:

- **Die Signatur:** `<url>.sig` ist eine abgetrennte SSHSIG-Signatur der SSH-CA des
  Anbieters (`ssh_ca_file`, das `ssh_krl_url` deshalb voraussetzt) im
  Namensraum `krl@go-authn.github.io`. Sie wird mit `If-Match` auf das ETag der
  Liste abgerufen und sofort erneut abgerufen, wenn die Liste zwischenzeitlich neu ausgestellt wurde.
- **Der Ablauf:** die Erweiterung `expires@go-authn.github.io`. Danach ist die Liste
  nicht aktuell, wie kürzlich auch immer ein `304` sie bestätigt hat.

HTTPS (`ssh_krl_ca_file` legt seine Zertifizierungsstellen fest) authentifiziert den
Server, der geantwortet hat, nicht die Liste: Ein Spiegel, ein Cache oder ein
kompromittierter Webserver könnte eine leere ausliefern, und nichts in einer KRL allein
sagt, dass sie veraltet ist. Eine Liste, die die Prüfung nicht besteht, wird abgelehnt,
und die letzte gute Kopie bleibt erhalten. Eine mit `ssh_krl_file` von Hand abgelegte Liste
genießt dasselbe Vertrauen wie das Dateisystem, das sie enthält, und braucht weder Signatur
noch Ablaufdatum; trägt sie eines, wird es beachtet.

{{< callout type="warning" >}}
**Aktualisierung auf v0.15.0**

Eine `ssh_krl_url`, die eine unsignierte KRL oder eine ohne Ablaufdatum ausliefert, wird
nicht mehr übernommen: Die Zertifikate des Anbieters werden dann abgelehnt, wie bei jeder
Liste, die nicht zu bekommen ist. Betreiben Sie zuerst go-authn/bridge v0.10.0 oder neuer.
{{< /callout >}}

## Nie rückwärts, auch nicht über einen Neustart hinweg {#never-backwards-across-a-restart-too}

Seit v0.16.0 wird eine Liste abgelehnt, wenn sie älter ist als die gehaltene: eine
niedrigere Version (die Version der KRL, die Nummer einer CRL) oder dieselbe Version,
früher ausgestellt. Mit `ssh_krl_state_file` (`crl_state_file` für
[NFS]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}})) wird die letzte geprüfte Liste
auf der Festplatte aufbewahrt, beim Start erneut geprüft und bestimmt die Reihenfolge für die
nächste. Für sich allein gilt sie nicht als aktuell, bis eine Liste abgerufen wird. Ohne sie
vergisst ein Neustart die Reihenfolge, und fileshare meldet das beim Start: Eine ältere
Liste, noch signiert und nicht abgelaufen, würde übernommen.

Diese Dateien dürfen, wie `ssh_ca_file`, die CA-Dateien, die Listendateien und (seit
v0.17.0) `authorized_keys_file`, `dsn_file`, `bind_password_file` und die
`ssf`-`ca_file`, **nicht innerhalb einer Freigabe liegen**: Wer dort schreibt, würde
entscheiden, wer hineinkommt.

## Ein einfaches `sshd` neben fileshare {#a-plain-sshd-next-to-fileshare}

Dieselbe Prüfung gibt es auch ohne fileshare: `authn-revokd` aus go-authn/revocation
ruft die Listen ab, behält nur die, die die Prüfung bestehen, hält ihre Reihenfolge auf
der Festplatte fest, schreibt `RevokedKeys` für `sshd` und schreibt, wenn eine Liste
verfällt, eine, die die CA selbst widerruft, sodass auch `sshd` geschlossen fehlschlägt.

## Was sie nicht abdeckt {#what-it-does-not-cover}

**OpenPubkey-Zertifikate (opkssh) stehen in keiner KRL** – nichts hat sie ausgestellt
außer dem eigenen Schlüssel der Person. Sie sind durch `opkssh_max_age` begrenzt und werden
durch [Shared Signals]({{< relref "/security/shared-signals.md" >}}) zurückgenommen. Ebenso ein Access-Token über WebDAV.

Derselbe Mechanismus – abgerufen, aufbewahrt, ausfallsicher geschlossen – trägt die **X.509-CRL**
von [NFS-Identitäten aus Zertifikaten]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}).
