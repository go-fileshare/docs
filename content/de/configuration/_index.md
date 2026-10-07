---
title: "Die Konfiguration"
linkTitle: "Konfiguration"
weight: 10
description: "Wie die HCL-Konfiguration aufgebaut ist, was ein user-Block enthält und was beim Start abgelehnt wird."
tags: [konfiguration]
---

Eine Datei oder ein Verzeichnis kleiner Dateien. `--config /etc/fileshare.d` führt
sie zusammen, sodass ein Benutzer in einer Datei und eine Freigabe in einer anderen zusammengehören.

```hcl
name = "ATTIC"

user "alice" { password_file = "/etc/fileshare/alice.pw" }
user "bob"   { password_file = "/etc/fileshare/bob.pw" }

group "family" { members = ["alice", "bob"] }

share "photos" {
  image   = "/srv/photos.img"
  allow   = ["@family"]        # a group, or a person, in either list
  writers = ["alice"]          # bob gets it read-only
}

share "scratch" {
  image = "/srv/scratch.img"   # anyone who authenticates, read-write
}

tls {
  cert_file = "/etc/fileshare/tls/fullchain.pem"
  key_file  = "/etc/fileshare/tls/key.pem"
}

serve "smb"    { addr = "0.0.0.0:445" }
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true                  # HTTP Basic is the password: never in the clear
}
serve "sftp"   { addr = "0.0.0.0:2222" }
serve "nfs"    { addr = "0.0.0.0:2049" }
serve "s3"     { addr = "0.0.0.0:9000" }
```

{{< callout type="error" >}}
**Seit v0.9.0 wird WebDAV mit Passwörtern nicht unverschlüsselt bereitgestellt**

`serve "webdav" { addr = "0.0.0.0:8080" }` – das Beispiel, das diese Seite
bis dahin zeigte – wird jetzt **beim Start abgelehnt**, sobald sich jemand authentifiziert:
HTTP Basic ist das Passwort bei jeder Anfrage. Setzen Sie `tls = true` mit einem `tls`-Block,
wie oben, oder `plaintext = true`, wenn ein Proxy davor TLS
terminiert. Siehe [TLS]({{< relref "/security/tls.md" >}}).
{{< /callout >}}

Eine Freigabe kann statt eines Images auch ein [Gerät]({{< relref "/configuration/shares.md#a-device-not-only-an-image" >}}) oder ein
[Verzeichnis des Hosts]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}})
sein.

## Was ein `user`-Block enthalten kann {#what-a-user-block-can-carry}

Jedes Beispiel oben verwendet `password_file`; das ist der übliche Fall und war
eine Zeit lang der einzige, den dieses Programm las. Es ist nicht der ganze Block, und
der Unterschied entscheidet, **über welche Protokolle eine Person bedient werden kann** – es
lohnt sich also, das an einer Stelle zu haben, statt es beim Einhängen herauszufinden.

| Feld | was es ist | was es bringt |
|---|---|---|
| `password` | ein Passwort, in der Datei | jedes Protokoll, das ein Passwort beantwortet |
| `password_file` | dasselbe, in einer eigenen Datei | dasselbe, ohne das Geheimnis in der Konfiguration |
| `nt_hash` | `MD4(UTF16LE(password))`, 32 Hex-Zeichen | **SMB**, für eine Person, deren Passwort diese Installation nicht hat |
| `authorized_keys` | `authorized_keys`-Zeilen, inline | **SFTP** |
| `authorized_keys_file` | dasselbe, aus einer Datei | **SFTP** |
| `totp_secret` | ein Base32-Geheimnis für Einmalcodes | noch nichts: Es wird gelesen und geprüft, und kein Protokoll hier fragt nach einem zweiten Faktor |

⛔ `password` und `password_file` zusammen werden beim Start abgelehnt, unter Nennung der
Person: Zwei Antworten auf „Wie lautet das Passwort?" werfen die Frage auf, welche
gewinnt, und eine Konfiguration sollte nicht zweimal gelesen werden müssen, um das herauszufinden. Dasselbe gilt für
`authorized_keys` und `authorized_keys_file` zusammen.

⛔ Ein `user`-Block ohne `password`, `password_file`, `authorized_keys`
oder `authorized_keys_file` wird ebenfalls abgelehnt – *has no way to authenticate* –,
es sei denn, `trusted_user_ca_file` ist gesetzt. `nt_hash` und `totp_secret` zählen
hier nicht.

{{< callout type="default" >}}
**Ein inline definierter Benutzer KANN über SMB bedient werden**

Bis vor Kurzem las dieser Block nur die Passwortfelder, sodass SMB für
jemanden, der hier eingetragen ist, bedeutete, dieser Datei sein Passwort im
Klartext zu geben – oder ihn in eine Datenbank zu verschieben. Mit `nt_hash` ist das nicht mehr so:
Eine Installation, die speichert, was Samba speichert, kann stattdessen das eintragen. Der Block braucht
weiterhin eines der Felder oben daneben – etwa Schlüssel – oder
`trusted_user_ca_file`: `nt_hash` allein wird beim Start abgelehnt. Siehe
[was eine Quelle beweisen kann]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}),
dieselbe Tabelle eine Ebene höher.
{{< /callout >}}

Eine Gruppe wird `@name` geschrieben, überall dort, wo eine Person stehen könnte.

## Was beim Start abgelehnt wird statt später {#what-is-refused-at-startup-rather-than-later}

{{< callout type="warning" >}}
**Ein Name, der niemandem gehört**

`allow = ["alise"]` würde Alice sonst aus ihrer eigenen Freigabe aussperren und
fröhlich starten. Ein Name, den kein `user`-Block und kein [`users`-Verzeichnis]({{< relref "/configuration/identity.md" >}})
kennt, wird abgelehnt, ebenso eine `@group`, die keiner von ihnen hat.
{{< /callout >}}

{{< callout type="warning" >}}
**Eine Freigabe, die nennt, wer sie nutzen darf, über NFS**

Eine Konfiguration, die sagt *photos gehört alice*, und ein Protokoll, das
photos jedem übergibt, der sich verbindet, können nicht beide eingehalten werden. Eine solche Freigabe wird
nicht über NFS exportiert – es sei denn, ein `kerberos`-Block oder `identity = "certificate"`
lässt NFS Personen unterscheiden –, und ein `nfs`-`serve`-Block, dem nichts zu
transportieren bleibt, wird abgelehnt. Siehe [NFS]({{< relref "/protocols/nfs.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Ein Anbieter, dessen Wort nirgends ankommt**

Ein [`oidc`]({{< relref "/protocols/webdav.md" >}})-Block wird abgelehnt, es sei denn, WebDAV wird
bereitgestellt – das eine Protokoll, das einen `Authorization`-Header transportiert –, oder SFTP
wird mit eingeschalteten Zertifikaten des Anbieters bereitgestellt (`ssh_ca_file` oder
`opkssh_client_id`, siehe [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}})).
SMB und NFS haben für keines von beiden einen Platz.
{{< /callout >}}

{{< callout type="warning" >}}
**Passwörter im Klartext**

WebDAV auf einer Adresse, die andere Maschinen erreichen können, ohne `tls = true` oder
`plaintext = true`, während sich jemand authentifiziert. Siehe
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Ein Block, der nicht sicher bereitgestellt werden kann**

Ein `admin`-Block ohne `state_file` oder einer, der auf TCP ohne gegenseitiges
TLS lauscht; ein `tls`-Block, den kein `serve`-Block verwendet; ein `reload`, das kürzer als eine
Sekunde ist; eine Sperrliste, die über einfaches HTTP abgerufen wird. Jedes davon ist dort beschrieben,
wo es hingehört: [Administration]({{< relref "/administration/_index.md" >}}),
[TLS]({{< relref "/security/tls.md" >}}), [Neuladen]({{< relref "/administration/reload.md" >}}),
[Widerruf]({{< relref "/security/_index.md" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Ein Protokoll, ohne das dieses Binary gebaut wurde**

Die Meldung sagt *genau das*, statt „es gibt kein solches Protokoll" – der Unterschied
zwischen einem Tippfehler und einem [Build-Tag]({{< relref "/operations/build-tags.md" >}}).
{{< /callout >}}

Ein `serve`-Block ohne `addr` lauscht auf Loopback, `127.0.0.1`, auf einem Port,
der kein Privileg braucht: 4445 für SMB, 8080 für WebDAV, 2222 für SFTP, 2049
für NFS, 9000 für S3.

## `protocols` an einer Freigabe {#protocols-on-a-share}

```hcl
share "photos" {
  image     = "/srv/photos.img"
  protocols = ["smb"]
}
```

Sagt, welche Protokolle sie transportieren. Für sich allein nützlich – eine als nur SMB deklarierte Freigabe ist
eine Freigabe, von der der WebDAV-Prozess nie erfährt – und in manchen
Konfigurationen unter [`--isolate`]({{< relref "/operations/isolation.md" >}}) erforderlich. Ein `serve`-Block,
der am Ende nichts transportieren würde, wird ebenfalls abgelehnt, bevor ein Image geöffnet wird,
unter Nennung der Freigaben, die ihm vorenthalten wurden, und warum – es sei denn, ein `admin`-Block ist
da, um ihm später Freigaben zu geben.
