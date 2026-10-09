---
title: "TLS und Zertifikate per ACME"
weight: 10
description: "WebDAV, S3 und NFS über TLS bereitstellen, mit einem Zertifikat aus Dateien oder von einer ACME-CA, und warum WebDAV mit Passwörtern unverschlüsselt abgelehnt wird."
tags: [sicherheit, tls, acme]
---

Seit v0.9.0 werden **WebDAV und S3 über HTTPS bereitgestellt und NFS über
RPC-with-TLS** (RFC 9289), mit `tls = true` in ihrem `serve`-Block und einem
`tls`-Block, der angibt, woher das Zertifikat kommt.

## Aus Dateien {#from-files}

```hcl
tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true
}
```

Die Dateien werden **neu geladen, wenn sie sich ändern**, sodass eine Erneuerung, die
ein anderes Werkzeug schreibt, ohne Neustart übernommen wird. Das Zertifikat stammt aus
[go-authn/servercert](https://github.com/go-authn/servercert).

## Von einer ACME-CA {#from-an-acme-ca}

Standardmäßig Let's Encrypt; oder eine CA, die Sie über **External Account
Binding** kennt, etwa GÉANT TCS über HARICA:

```hcl
tls {
  acme {
    directory_url     = "https://acme-v02.harica.gr/acme/<uuid>/directory"   # the Server URL cm.harica.gr shows
    domains           = ["files.example.org"]
    cache_dir         = "/var/lib/fileshare/acme"
    eab_key_id        = "…"
    eab_hmac_key_file = "/etc/fileshare/tls/eab.key"   # a secret: a file, never the config
  }
}
```

| Feld | |
|---|---|
| `directory_url` | das ACME-Verzeichnis der CA; leer bedeutet Let's Encrypt |
| `domains` | die anzufordernden Namen (erforderlich) |
| `cache_dir` | wo das Konto und die Zertifikate aufbewahrt werden (erforderlich) |
| `email` | der Kontakt des Kontos, optional |
| `eab_key_id`, `eab_hmac_key_file` | External Account Binding; der HMAC-Schlüssel ist ein Geheimnis und wird daher aus einer Datei gelesen – base64url, wie die CA ihn ausgibt (Standard-base64 wird ebenfalls gelesen); alles andere wird abgelehnt |
| `http_challenge` | wo http-01 beantwortet wird, z. B. `"0.0.0.0:80"` |

### Ob ACME funktionieren kann, entscheidet die Art, wie die CA den Namen prüft {#whether-acme-can-work-is-decided-by-how-the-ca-checks-the-name}

| Challenge | die CA verbindet sich mit | also |
|---|---|---|
| tls-alpn-01 (RFC 8737) | Port **443** | auf 443 muss ein TLS-Protokoll bereitgestellt werden |
| http-01 (RFC 8555 §8.3) | Port **80** | `http_challenge = "0.0.0.0:80"` beantwortet sie – oder die Adresse, an die ein Port 80 weitergeleitet wird |
| keine | nichts | ein CA-Konto mit **vorab validierter** Domain verlangt keine Challenge: ein Enterprise-**Admin**-Konto von HARICA für GÉANT TCS (OV-Zertifikate) – daher erhält auch ein Server, **den von außen niemand erreichen kann**, sein Zertifikat |

Die letzte Zeile ist die entscheidende für einen Dateiserver in einem Campusnetz:
Mit einem GÉANT-TCS-Konto, dessen Domain die Einrichtung bei HARICA bereits
validiert hat, muss überhaupt kein Port zum Internet geöffnet werden.

#### HARICA (GÉANT TCS): welches Konto, und CAA {#harica-gant-tcs-which-account-and-caa}

HARICAs ACME-Konten gibt es in zwei Arten, und nur eine kommt ohne Challenge
aus:

- ein **Enterprise-Admin**-Konto stellt **OV**-Zertifikate für Domains aus, die
  die Einrichtung validiert hat: **keine Challenge** – das ist das Konto für
  einen Server, den von außen niemand erreichen kann;
- ein **Enterprise-User**-Konto stellt **DV**-Zertifikate aus, und DV heißt,
  dass die CA den Namen bei jeder Bestellung prüft, mit tls-alpn-01 oder
  http-01 wie oben.

In beiden Fällen muss der **CAA**-Eintrag der Domain, falls es einen gibt,
`harica.gr` erlauben, sonst wird die Bestellung abgelehnt.

#### Wann das Zertifikat angefordert wird

**Beim Start** (seit v0.27.0): Sobald die Listener laufen, wird das Zertifikat
jeder konfigurierten Domain geholt oder aus dem Cache-Verzeichnis gelesen. Ein
Fehlschlag hält den Server nicht an – er bedient weiter und versucht es nach
1 Minute erneut, verdoppelt bis 1 Stunde –, und die erste TLS-Verbindung, die
den Host benennt, fragt ohnehin erneut. Erneuert wird 30 Tage vor Ablauf, oder
im letzten Drittel der Laufzeit, wenn das früher ist.

**Ein Client, der kein SNI sendet** – einer, der sich per IP-Adresse verbindet –
erhält das Zertifikat der **ersten** Domain (seit v0.27.0); vorher scheiterte
der Handshake. Der Client prüft dieses Zertifikat weiterhin gegen die Adresse,
die er gewählt hat: Es hilft einem Client, dem gesagt wurde, diesem Namen zu
vertrauen, und täuscht niemanden.

{{< callout type="info" >}}
**ALPN**

Mit ACME enthält die ALPN-Liste des Servers `acme-tls/1`, und ein Go-Server
lehnt einen Client ab, dessen Liste nichts mit seiner eigenen gemeinsam hat. Daher
wird für WebDAV und S3 `http/1.1` **angehängt** – jeder Browser und jeder WebDAV-Client
bietet es an – und für NFS `sunrpc`, so wie RFC 9289 §5.2 RPC-with-TLS kennzeichnet.
Nicht `h2`: HTTP/2 wird über diese Listener nicht angeboten, und es anzubieten
wäre eine Lüge. `acme-tls/1` bleibt auf jedem Listener, sodass tls-alpn-01 überall
beantwortet wird, wo die CA hingelangt.
{{< /callout >}}

## WebDAV wird nicht unverschlüsselt bereitgestellt {#webdav-is-not-served-in-the-clear}

{{< callout type="error" >}}
**BREAKING in v0.9.0: WebDAV mit Passwörtern auf einer erreichbaren Adresse wird abgelehnt**

HTTP Basic ist das Passwort, base64-kodiert, bei jeder Anfrage, und ein Bearer-Token
ist ebenso viel wert wie eines. Eine Konfiguration, die WebDAV **ohne TLS** auf einer
Adresse bereitstellt, die andere Rechner erreichen können, während sich irgendjemand
authentifiziert, wird daher **abgelehnt, nicht bloß gewarnt** – eine Warnung scrollt
vorbei, und das Passwort kommt nicht zurück:

```
webdav on 0.0.0.0:8080 would carry passwords in the clear: serve it with
tls = true, or -- when TLS is terminated in front of it, by a proxy -- say
plaintext = true
```
{{< /callout >}}

Eine Konfiguration, die vor v0.9.0 mit
`serve "webdav" { addr = "0.0.0.0:8080" }` startete, startet danach nicht mehr. Zwei
Auswege:

```hcl
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true            # with a tls block
}
```

oder, wenn ein Reverse Proxy davor TLS terminiert und unverschlüsselt weiterleitet:

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true       # TLS is terminated in front of this, on purpose
}
```

**Loopback**-Adressen (`127.0.0.1`, `::1`, `localhost`) und ein Server, bei dem sich
**niemand authentifizieren muss** – kein `user`, kein `users`, kein `oidc`-Block –, sind
nicht betroffen. Ein `serve`-Block ohne `addr` landet auf Loopback und ist ebenfalls
nicht betroffen.

`plaintext` gilt nur für WebDAV, das Protokoll, das ein Passwort sendet, und
`plaintext` zusammen mit `tls` wird abgelehnt: das eine oder das andere.

## Was `check` meldet {#what-check-says}

[`fileshare check`]({{< relref "/configuration/check.md" >}}) gibt pro Protokoll aus, was
verschlüsselt ist und was **absichtlich** unverschlüsselt ist:

```
webdav: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem
nfs: TLS, certificate from the files /etc/fileshare/tls/fullchain.pem and /etc/fileshare/tls/key.pem; clients present one signed by /etc/fileshare/tls/clients.pem (the machine, not the person)
```

und, für WebDAV hinter einem Proxy,

```
webdav: in the clear on 10.0.0.5:8080, on purpose (plaintext = true): TLS must be terminated in front of it
```

## Pro Protokoll {#per-protocol}

| Protokoll | `tls = true` | |
|---|---|---|
| WebDAV | ja | HTTPS |
| S3 | ja | HTTPS |
| NFS | ja | RFC 9289; siehe unten |
| SMB | **abgelehnt** | SMB 3 verschlüsselt mit eigenen Schlüsseln |
| SFTP | **abgelehnt** | SFTP ist SSH |

## NFS über TLS weist den Rechner nach, nicht die Person {#nfs-over-tls-proves-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

`client_ca_file` lässt einen Client ein Zertifikat dieser Stelle vorlegen –
**welcher Host einhängt** –, und RFC 9289 lässt die Benutzerauthentifizierung, wie sie war:
Die uid darin ist weiterhin die, die `AUTH_SYS` behauptet. Eine Freigabe, die benennt, wer
sie nutzen darf, wird daher über NFS **weiterhin abgelehnt**, ohne einen `kerberos`-Block –
oder ohne [Identitäten aus Zertifikaten]({{< relref "/security/nfs-certificates.md" >}}), was eine andere Verwendung
desselben Zertifikats ist.

TLS wird **angeboten, nicht verlangt**: Ein Client, der nie danach fragt, erhält
weiterhin die offenen Freigaben. `client_ca_file` ohne `tls = true` wird abgelehnt – es
würde nichts prüfen –, und `client_ca_file` bei jedem anderen Protokoll als NFS wird
ebenfalls abgelehnt.

## Was sonst abgelehnt wird {#what-else-is-refused}

- `tls = true` ohne `tls`-Block: Nichts gibt an, woher das Zertifikat kommt.
- Ein `tls`-Block, den kein `serve`-Block verwendet: Nichts würde ihn nutzen.
- Eine `http_challenge`, die keine Adresse ist, auf der gelauscht werden kann.
