---
title: "Stand"
weight: 60
description: "Welches fileshare-Release diese Seiten beschreiben, was jedes Release hinzugefügt hat, was mit echten Clients geprüft wurde und was noch nicht implementiert ist."
tags: [stand, releases]
---

Diese Seiten beschreiben **fileshare v0.28.0**.

## Was jedes Release hinzugefügt hat {#what-each-release-added}

| | |
|---|---|
| v0.7.0 | eine eingeschränkte Freigabe über NFS, mit einem [`kerberos`-Block]({{< relref "/protocols/nfs.md#the-refusal-that-used-to-be-permanent" >}}) |
| v0.8.0 | die [Admin-API]({{< relref "/administration/_index.md" >}}) über gRPC; [Verzeichnisfreigaben]({{< relref "/configuration/shares.md#a-directory-not-only-an-image" >}}); [`/healthz`, `/readyz`, `/metrics`]({{< relref "/administration/health.md" >}}) |
| v0.9.0 | [TLS]({{< relref "/security/tls.md" >}}) für WebDAV, S3 und NFS, aus Dateien oder per ACME; **WebDAV mit Passwörtern wird unverschlüsselt abgelehnt** |
| v0.10.0 | [das Verzeichnis erneut einlesen]({{< relref "/administration/reload.md" >}}): `reload = "5m"`, `SIGHUP`, `ReloadDirectory` |
| v0.11.0 | [SFTP für Personen, für die der Identitätsanbieter bürgt]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}): seine SSH-CA und opkssh |
| v0.12.0 | Widerruf: eine [KRL]({{< relref "/security/revocation-lists.md" >}}) für die SSH-Zertifikate des Anbieters, eine CRL für [NFS-Identitäten aus Zertifikaten]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.13.0 | [Shared Signals]({{< relref "/security/shared-signals.md" >}}): ein CAEP-`session-revoked` macht alles ungültig, was der Anbieter zuvor ausgestellt hat, auf jedem Protokoll |
| v0.14.0 | die Korrekturen aus dem Sicherheitsreview: [Freigabenamen, die ein Protokoll transportieren kann]({{< relref "/administration/_index.md#what-it-will-and-will-not-touch" >}}), keine Freigabe, die die Konfiguration oder ihre Geheimnisse enthält, `source_roots` beim Start erneut geprüft, SSF `max_age` ≥ 1m und `retain` ≥ 169h |
| v0.15.0 | eine [KRL aus `ssh_krl_url` muss signiert und datiert sein]({{< relref "/security/revocation-lists.md#signed-and-dated-from-ssh_krl_url" >}}) |
| v0.16.0 | Sperrlisten [gehen nie zurück]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}}), auch über einen Neustart hinweg, mit `ssh_krl_state_file` / `crl_state_file`; Vertrauensanker und Listendateien dürfen nicht in einer Freigabe liegen |
| v0.16.1 | erfordert Go 1.26.6: neun erreichbare Sicherheitshinweise der Standardbibliothek in 1.26.4 |
| v0.16.2 | die [Signatur einer NFS-CRL wird geprüft, bevor sie geparst wird]({{< relref "/security/nfs-certificates.md#revocation-the-crl" >}}) |
| v0.16.3 | [niemand wird über eine unbestätigte E-Mail-Adresse benannt]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}) |
| v0.16.4 | nur Tests |
| v0.16.5 | nur Tests ([#36](https://github.com/go-fileshare/fileshare/issues/36): ein Test las einen Transportfehler als Erfolg) |
| v0.16.6 | die NFS-Ablehnung nennt beide Auswege: einen `kerberos`-Block oder [`identity = "certificate"`]({{< relref "/security/nfs-certificates.md" >}}) |
| v0.16.7 | go-authn/oidc v0.2.2: die Größe eines RSA-Schlüssels wird in Bits gezählt. Zuvor verifizierte ein mit Null-Oktetten aufgefüllter 1024-Bit-Schlüssel ein WebDAV-Token oder ein opkssh-PK-Token |
| v0.17.0 | die Korrekturen eines Sicherheitsaudits: Eine Freigabe darf keine andere enthalten; Zugangsdateien (`authorized_keys_file`, `dsn_file`, …) dürfen nicht in einer Freigabe liegen; WebDAV wird nie anonym, weil das Lesen eines Verzeichnisses leer zurückkam; S3 beachtet `protocols`; eine Verbindung, die sich nicht innerhalb von 30 s authentifiziert hat, wird geschlossen; eine opkssh-Sitzung endet bei `opkssh_max_age`; eine widerrufene Sitzung bleibt widerrufen und wird geschlossen |
| v0.17.1 | go-authn/oidc v0.2.4 (kein Schlüsselsatz hinter einer Umleitung auf http), servercert v0.3.0 (der ACME-Cache-Pfad wird wie bei sshds StrictModes geprüft), krl v0.5.0, revocation v0.3.0 |
| v0.17.2 | go-filesystems/s3 v0.3.0: S3 verrät einem nicht authentifizierten Aufrufer nicht mehr, welche Access Keys existieren; vorsignierte URLs funktionieren, gelten höchstens eine Woche und werden vor ihrem eigenen Datum abgelehnt |
| v0.18.0 | die CI pinnt Go 1.27.1 statt `stable` |
| v0.18.1 | `fileshare check` warnt, wenn ein `oidc`-Block keine `domains` hat: [Der bloße Name beim Anbieter erreicht das lokale Konto gleichen Namens]({{< relref "/configuration/identity.md" >}}) |
| v0.19.0 | go-authn/directory v0.11.0: Ein `users "ldap"`-Block lehnt einen [unverschlüsselten Bind an eine andere Maschine]({{< relref "/configuration/identity.md" >}}) ab; Passwörter werden als Digests verglichen, in konstanter Zeit |
| v0.20.0 | [`local_names`]({{< relref "/configuration/identity.md" >}}): Die Namen des Anbieters sind nur dann lokale Namen, wenn der `oidc`-Block das sagt (Sicherheitsaudit F4) |
| v0.21.0 | [Volumes]({{< relref "/administration/volumes.md" >}}): Ein privilegierter `fileshare provisioner` legt ZFS-Datasets, btrfs-Subvolumes und XFS/ext4-Projekt-Quota-Verzeichnisse an, und die Admin-API stellt Freigaben daraus bereit; eine volle Verzeichnisfreigabe gibt auf jedem Protokoll dieselbe Antwort; `allowed_uids` des Admin-Sockets |
| v0.21.1 | eine [volle Freigabe]({{< relref "/administration/volumes.md#a-full-share" >}}) meldet das auch über SMB (`STATUS_DISK_FULL`) und NFS (`NFS3ERR_NOSPC` / `NFS3ERR_DQUOT`) |
| v0.22.0 | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}}): Ein SSH-Zertifikat wird nur dort akzeptiert, wo sein Domain-Grant (`ssh-domain-grant@core.aai.geant.org`, das SSH-CA-Profil von EuroHPC) diesen Host nennt, fail-closed; `ssh_accept_ungranted` lässt Zertifikate ohne Grant zu |
| v0.22.1 | go-filesystems/sftp v0.5.1: Ein [an eine Adresse gebundenes]({{< relref "/protocols/sftp.md#a-certificate-pinned-to-an-address" >}}) Zertifikat (`source-address`) wird von dieser Adresse aus unter `ssh_domains` akzeptiert, ebenso bei den Zertifikaten des Anbieters und von OpenPubkey; zuvor wurde es von jeder Adresse abgelehnt |
| v0.22.2 | nur Tests und Dokumentation: `source-address` wird von `::1` und von einer zweiten IPv4-Adresse (`127.0.0.2`) aus getestet, unter `ssh_domains` und mit einem OpenPubkey-Zertifikat; unter Linux schlägt mit `FILESHARE_REQUIRE_JUDGE` ein fehlender Judge fehl, statt übersprungen zu werden |
| v0.23.0 | ein [btrfs-Volume]({{< relref "/administration/volumes.md#the-size-a-client-sees" >}}) hat über NFS, SMB und WebDAV die Größe seiner Quota (go-filesystems/nfs v0.7.0, smb v0.5.0, webdav v0.3.0 melden die Größe bei jeder Abfrage); zuvor zeigte btrfs das ganze Dateisystem. Ein Server, dessen Freigaben aus der Admin-API kommen, hält beim Start nicht mehr an, wenn er NFS anbietet und noch keine Freigabe hat |
| v0.24.0 | schnelleres Lesen: Ein WebDAV-GET einer Verzeichnis- oder Volume-Freigabe geht über einfaches HTTP (nicht über TLS) mit `sendfile(2)` hinaus, NFS liest und schreibt in 1 MiB (rtmax/wtmax), und SMB-Clients lesen mehr als 64 KiB pro Anfrage (`CAP_LARGE_MTU`). Auf dem CI-Runner, eine 256-MiB-Datei: WebDAV 6,6 GB/s, SMB 0,84, NFS 1,47 (ein Client) |
| v0.24.1 | Sicherheit: Eine untätige NFS-Verbindung behält nicht mehr die 3 MiB ihres größten Aufrufs (24 untätige Verbindungen hielten 59 MiB); ein SMB-Frame-Header lässt den Server nicht mehr die angekündigten 8 MiB belegen, bevor die Bytes ankommen; eine SMB-Anfrage kann nicht mehr ein Credit für 4 GiB zahlen (go-filesystems/nfs v0.8.1, smb v0.6.1 und v0.6.2) |
| v0.25.0 | SFTP beantwortet `limits@openssh.com`, sodass OpenSSHs `sftp` bis zu 255 KiB liest (+25 %); eine Sitzung hält höchstens 1024 offene Handles (go-filesystems/sftp v0.6.0) |
| v0.26.0 | serverseitiges Kopieren bei jedem Protokoll, das es kennt: Ein WebDAV-`COPY`, ein SFTP-`copy-data` (OpenSSHs `sftp cp`) und ein SMB-COPYCHUNK (Explorer, Linux `copy_file_range`) werden auf dem Server ausgeführt, mit `copy_file_range(2)` zwischen Dateien des Hosts — ein Reflink auf btrfs und XFS, in der CI gemessen: 0 Byte belegt für eine 8-MiB-Kopie. Sicherheit: Ein WebDAV-`COPY` und jedes S3-`GET`/`HEAD` lesen nicht mehr die ganze Datei in den Speicher; S3 streamt Objekte jeder Größe. Korrektur: Der Sendfile-Weg für WebDAV-`GET` aus v0.24.0 griff nur bei schreibgeschützten Freigaben — eine Hülle verbarg ihn bei beschreibbaren; seit v0.26.0 greift er bei beiden |
| v0.27.0 | ACME: Das Zertifikat wird **beim Start** angefordert, nicht bei der ersten Verbindung – neuer Versuch nach 1 min, verdoppelt bis 1 h, während der Server weiter bedient; ein Client, der **kein SNI** sendet (Verbindung per IP-Adresse), erhält das Zertifikat der ersten Domain statt eines gescheiterten Handshakes (go-authn/servercert v0.5.0). [HARICA]({{< relref "/security/tls.md#harica-gant-tcs-which-account-and-caa" >}}): Nur ein Enterprise-**Admin**-Konto (OV) stellt ohne Challenge aus, und CAA muss `harica.gr` erlauben |
| v0.28.0 | die Admin-API wird auch [über HTTPS für OIDC-Tokens]({{< relref "/administration/_index.md#over-https-for-oidc-tokens" >}}) angeboten – Connect, gRPC-Web und gRPC auf einem Listener, eine Audience je Server, Aufrufer je Aussteller per Subject oder Gruppe benannt – der erste Schritt zu den Web- und nativen Oberflächen |

{{< callout type="warning" >}}
**Upgrade auf v0.21.0**

Eine **volle Verzeichnisfreigabe** antwortet anders. `EDQUOT` (eine Quota) und
`ENOSPC` sind jetzt ein einziger Fehler, „no space left on device (the share is full)",
auf jeder Verzeichnisfreigabe, nicht nur auf Volumes: Ein `EDQUOT`, das bei WebDAV
`500` und bei NFS `NFS3ERR_IO` war, ist jetzt `507` und `NFS3ERR_NOSPC`, und der Text bei SFTP
ist dieser Satz, ohne den Pfad. SMB antwortet weiterhin mit „Zugriff verweigert". Siehe
[eine volle Freigabe]({{< relref "/administration/volumes.md#a-full-share" >}}).
{{< /callout >}}

{{< callout type="warning" >}}
**Upgrade auf v0.20.0**

Die `alice` des Anbieters erhält nicht mehr die Freigaben der lokalen `alice`. Eine Installation,
deren Namen beim Anbieter ihre lokalen Namen sind, ergänzt `local_names = true` im
`oidc`-Block (und setzt `domains`); der Server protokolliert diese Zeile für jeden Namen,
den er mangels dieser Einstellung ablehnt.
{{< /callout >}}

{{< callout type="warning" >}}
**Upgrade auf v0.19.0**

Ein `users "ldap"`-Block, der ohne `start_tls = true` auf `ldap://` einer anderen Maschine
zeigt, **startet nicht mehr**: Verwenden Sie `ldaps://` oder StartTLS.
{{< /callout >}}

{{< callout type="warning" >}}
**Upgrade auf v0.17.0**

Eine Konfiguration mit einer Freigabe innerhalb einer anderen, zwei Freigaben auf einer Quelle
oder einer Zugangsdatei (`authorized_keys_file`, `dsn_file`, `bind_password_file`,
die `ca_file` von `ssf`) innerhalb einer Freigabe **startet nicht mehr**; der Fehler nennt
die Freigabe und den Pfad.
{{< /callout >}}

{{< callout type="warning" >}}
**Upgrade über v0.9.0 hinaus**

Eine Konfiguration, die WebDAV ohne TLS auf einer Adresse bereitstellt, die andere Maschinen
erreichen können, und bei der sich jemand authentifizieren muss, **startet nicht mehr**. Ergänzen Sie `tls = true`
und einen `tls`-Block, oder `plaintext = true` hinter einem Proxy, der TLS terminiert. Siehe
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

## Geprüft {#verified}

Dies sind Dinge, die ein Client, den dieses Projekt nicht geschrieben hat, tatsächlich tun musste, keine
Behauptungen über den Code.

- **macOS** hängt eine Freigabe über SMB ein, während `curl` über WebDAV in dasselbe Image
  schreibt, und liest zurück, was WebDAV geschrieben hat. Bobs WebDAV-Schreibzugriff auf eine Freigabe,
  die er nur lesen darf, ergibt `403`; ein falsches Passwort ergibt `401`.
- **go-smb2** – ein Client, den dieses Projekt nicht geschrieben hat – führt in der CI unter `-race` zwanzig
  gleichzeitige Lesezugriffe über SMB aus, während zwanzig über WebDAV laufen.
- Die Zugriffsregeln werden **über jedes Protokoll** geprüft statt nur in der
  Konfiguration: alice schreibt und bob nicht, über SMB und über WebDAV,
  und NFS wird die Freigabe gar nicht erst angeboten.
- SFTP-Zertifikate werden gegen den **eigenen Client von OpenSSH** geprüft, der denselben
  Schlüssel auch ablehnt, sobald sein Zertifikat beiseitegelegt wird.
- NFS-Identitäten aus Zertifikaten wurden mit einem **echten Linux-Client**
  (Kernel 6.17, ktls-utils 0.9) in der CI von go-filesystems/nfs gemessen – so wurden
  [`MNT` im Klartext und die Fallstricke von `tlshd`]({{< relref "/security/nfs-certificates.md#measured-with-a-real-linux-client" >}})
  gefunden.
- Die unsignierte Form der KRL ist eine Messung: OpenSSH 9.6 und 10.3 lesen eine signierte
  KRL und überspringen die Signatur (go-authn/krl, gegen beide).
- **Die Interop-Strecke von go-authn/bridge** ([#12](https://github.com/go-authn/bridge/pull/12),
  [#17](https://github.com/go-authn/bridge/pull/17)) prüft fileshare v0.13.0
  gegen die echte Bridge: die KRL, die CRL über NFS, über Shared Signals widerrufene
  WebDAV-Tokens, ein per Scope deaktivierter IdP und opkssh über Shared Signals.

## Noch nicht {#not-yet}

**Schreibzugriffe über S3.** `PUT` und `DELETE` antworten mit 403. Die Bibliothek kann schreiben, aber
eine Freigabe, die eine Person nur lesen darf, muss an derselben Stelle ablehnen wie SFTP, und
das ist nicht verdrahtet. Ablehnen ist besser als ein halb geschriebenes Objekt.

**Ein OIDC-Token über S3.** Tokens funktionieren über [WebDAV]({{< relref "/protocols/webdav.md" >}}) und
nirgendwo sonst. Der übliche Weg für S3 ist STS `AssumeRoleWithWebIdentity`,
das das Token gegen temporäre Zugangsdaten tauscht, mit denen der Client dann signiert –
ein anderer Mechanismus als das Akzeptieren eines Bearer-Tokens, und nicht
implementiert.

**`--isolate` mit einem `admin`- oder einem `metrics`-Block.** Abgelehnt: Es gibt keinen einzelnen
Prozess, auf den eine Änderung über die API angewendet werden könnte. Siehe
[ein Prozess pro Protokoll]({{< relref "/operations/isolation.md#not-with-an-admin-or-a-metrics-block-yet" >}}).

**Eine Person pro Benutzer auf einem gemeinsam genutzten Linux-NFS-Client, aus einem Zertifikat.** Ein Linux-Client
bindet das Zertifikat an einen Mount, sodass jeder Benutzer des Mounts die Person ist, die es
nennt. Das ist eine Eigenschaft des Clients, und die Antwort dort ist
`sec=krb5`. Siehe [was es nicht zusichern kann]({{< relref "/security/nfs-certificates.md#what-it-cannot-promise" >}}).

[S3 selbst]({{< relref "/protocols/s3.md" >}}) ist ausgeliefert: Eine Freigabe ist ein Bucket, bereitgestellt über
denselben Baum pro Benutzer, den SFTP verwendet.

## Gemessen, und als gemessen ausgewiesen {#measured-and-stated-as-measured}

Aussagen auf diesen Seiten, die Absicherung oder Messung sind statt
Reproduktionen eines Fehlers, und die auch im Quellcode so formuliert sind:

- Acht Goroutinen, die unter `-race` über einen nicht umhüllten fat32-Treiber schreiben,
  erzeugen heute keinen Race. Die [gemeinsame Sperre]({{< relref "/operations/locking.md" >}}) ist eine Absicherung
  eines Vertrags – `go-filesystems/interface` sichert nichts über gleichzeitige
  pfadbasierte Aufrufe zu –, keine Korrektur eines beobachteten Fehlers.
- Die [Größen nach Build-Tag]({{< relref "/operations/build-tags.md" >}}) wurden gemessen, einschließlich der
  13,2 MB, die `hashicorp/go-plugin` allein kostet – das gab den Ausschlag
  gegen ein Plugin-Framework –, und der 11,7 MB, die gRPC kostet, seit die Admin-API
  es mitbringt; deshalb gibt es `nogrpc`.
- Eine eigene Zertifikatserweiterung für Gruppen-Tags wurde entworfen und gegen den X.509-Parser von Go
  **gemessen**, der das ganze Zertifikat ablehnte; die Gruppen
  reisen stattdessen als [Tag-URIs]({{< relref "/security/nfs-certificates.md#what-the-certificate-carries" >}}).
