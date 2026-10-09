---
title: "SFTP – Schlüssel oder Zertifikate"
linkTitle: "SFTP: Schlüssel oder Zertifikate"
weight: 30
description: "SFTP mit öffentlichen Schlüsseln oder SSH-Zertifikaten: lokale Stellen und die des Anbieters, OpenPubkey, Domain-Grants, source-address und Widerruf."
tags: [protokolle, sftp, ssh]
---

Jeder Client hat es bereits: `sftp` wird mit OpenSSH ausgeliefert, der Finder und GNOME
hängen es ein, Editoren sprechen es. Es ist zugleich das eine Protokoll hier, dessen Form
nicht passt – eine Person meldet sich an und landet in **einem** Dateisystem, nicht in einer Liste von
Freigaben.

Daher werden die Freigaben zu den Verzeichnissen der obersten Ebene eines Baums, der für denjenigen gebaut wird, der sich gerade
authentifiziert hat:

```
$ sftp -i ~/.ssh/id_ed25519 -P 2222 alice@attic
sftp> ls
photos  scratch
sftp> cd photos
sftp> get holiday.jpg
```

Eine Freigabe, die alice nicht nutzen darf, **ist kein Verzeichnis, das alice sehen kann**, und eine Freigabe, die sie
nur lesen darf, lehnt ihre Schreibvorgänge *im Baum* ab, vor einem Treiber, der sie
zugelassen hätte.

{{< callout type="info" >}}
**Ein Umbenennen über zwei Freigaben hinweg wird abgelehnt**

Es wäre ein Kopieren und ein Löschen über zwei Images, und das ist nicht, was
Umbenennen irgendwo verspricht.
{{< /callout >}}

## Konfiguration {#configuration}

```hcl
host_key_file        = "/etc/fileshare/ssh_host_ed25519_key"
trusted_user_ca_file = "/etc/fileshare/ca.pub"   # optional

user "alice" {
  authorized_keys_file = "/etc/fileshare/alice.pub"
}

user "carol" {}   # nothing here: her certificate is her credential
```

Mit `trusted_user_ca_file` genügt ein Zertifikat, das von dieser Stelle signiert ist und
den Benutzer unter seinen Principals nennt – so wird der Zugang einer Person **anderswo ausgestellt und
läuft anderswo ab**, und keine Datei hier wird bearbeitet, wenn jemand hinzukommt oder geht.

Die Signatur, das Gültigkeitsfenster und die Principals prüft der
`CertChecker` von `x/crypto/ssh`; geprüft mit **OpenSSHs eigenem Client**,
der denselben Schlüssel ebenfalls ablehnt, sobald sein Zertifikat beiseitegelegt wird.

{{< callout type="warning" >}}
**Ohne `host_key_file` wird bei jedem Start eine neue Identität erzeugt**

Der Server sagt das, und jeder Client, der ihn schon einmal gesehen hat, warnt
vor einem geänderten Schlüssel – womit der Client seine Aufgabe erfüllt.
{{< /callout >}}

## Zertifikate, die für diesen Host bestimmt sind: der Domain-Grant {#certificates-meant-for-this-host-the-domain-grant}

Seit v0.22.0. Die [SSH-CA der EuroHPC Federation Platform](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-overview/)
(GÉANT / MyAccessID) signiert Zertifikate, denen jede Site der Föderation
vertraut. Jedes Zertifikat gibt in der Erweiterung
`ssh-domain-grant@core.aai.geant.org` an, für welche Hosts es bestimmt ist. sshd ignoriert diese Erweiterung,
und fileshare tat das bis v0.22.0 ebenfalls: Ein Zertifikat, das für die Maschinen von jemand anderem
ausgestellt war, galt hier genauso. Mit `ssh_domains` wird der Domain-Grant von
[go-authn/sshcert](https://github.com/go-authn/sshcert) gelesen, geparst und abgeglichen, wie
die Spezifikation es vorgibt. Ein Zertifikat, dessen Domain-Grant keinen der Namen dieses Hosts nennt,
wird abgelehnt.

```hcl
trusted_user_ca_file = "/etc/fileshare/efp-ssh-ca.pub"
ssh_domains          = ["files.example.org", "sftp.example.org"]
# ssh_accept_ungranted = true   # only if a trusted CA never writes a grant
```

| Schlüssel | |
|---|---|
| `ssh_domains` | die Namen dieses Hosts, so wie ein Domain-Grant sie nennt. Ein Zertifikat wird akzeptiert, wenn ein Muster seines Domain-Grants auf einen davon passt. `*` steht für **ein oder mehrere Zeichen innerhalb eines Labels**: `*.example.org` deckt `files.example.org` ab, aber nicht `a.files.example.org`, und `files-*.example.org` deckt `files-.example.org` nicht ab. Der Vergleich ignoriert Groß- und Kleinschreibung. Dies sind Hostnamen, keine Muster, jeder einmal aufgeführt, und mindestens einer Stelle (`trusted_user_ca_file` oder dem `ssh_ca_file` des `oidc`-Blocks) muss vertraut werden. |
| `ssh_accept_ungranted` | lässt ein Zertifikat **ohne** Domain-Grant zu. Standardmäßig aus; ohne `ssh_domains` wird es als bedeutungslos abgelehnt. |

| Zertifikat | `ssh_domains` | `+ ssh_accept_ungranted` | kein `ssh_domains` |
|---|---|---|---|
| deckt einen der `ssh_domains` ab | zugelassen | zugelassen | zugelassen |
| deckt nur andere Hosts ab, oder `[]` | abgelehnt | abgelehnt | zugelassen |
| kein Domain-Grant | abgelehnt | zugelassen, protokolliert | zugelassen |
| ein Domain-Grant, der sich nicht parsen lässt (`null`, `[1]`, nicht kompakt...) | abgelehnt | **abgelehnt** | zugelassen |

Ohne `ssh_domains` wird nichts gelesen, und nichts ändert sich. Der Domain-Grant wird bei
den Zertifikaten gelesen, die eine **Stelle** signiert hat: denen von `trusted_user_ca_file` und denen des
Anbieters (`ssh_ca_file`). Ein OpenPubkey-Zertifikat ist mit dem eigenen Schlüssel des Benutzers
signiert, sodass ein Domain-Grant darin das Wort des Benutzers über sich selbst wäre. Seine Zielgruppe ist
stattdessen `opkssh_client_id`. Einfache Schlüssel tragen keinen Domain-Grant und sind nicht betroffen.

Jede Ablehnung wird mit der Seriennummer des Zertifikats, seiner Schlüssel-ID und dem Grund protokolliert:

```
sftp: alice: certificate 42 ("alice@efp") refused: its domain grant [login.example.eu] names none of this host's names [files.example.org sftp.example.org]
sftp: alice: certificate 43 ("staff") refused: it has no domain grant (ssh-domain-grant@core.aai.geant.org), and ssh_domains requires one
```

Jede Entscheidung wird außerdem in `fileshare_sftp_domain_grant_total{result}` gezählt,
mit dem Ergebnis `granted`, `accepted_ungranted`, `refused_not_granted`,
`refused_absent` oder `refused_malformed`. Ein Client bietet ein Zertifikat an, bevor er
nachweist, dass er den Schlüssel besitzt; diese Zähler zählen also Versuche, nicht Personen.

{{< callout type="error" >}}
**Warum fail closed**

Nehmen Sie einen Host, der einer zweiten Stelle vertraut: seiner eigenen CA für Mitarbeitende, einer Test-CA,
der an dieselbe Datei angehängten Staging-CA von EFP oder einem go-authn/bridge-Client
ohne Domain-Grants. Die Zertifikate dieser Stelle tragen keinen Domain-Grant. Bedeutete ein fehlender
Domain-Grant „keine Einschränkung“, würde der Filter auf diesem Host nichts filtern.

Hier wird ein Zertifikat ohne Domain-Grant abgelehnt, sofern `ssh_accept_ungranted`
nichts anderes sagt. Ein vorhandener Domain-Grant, der sich nicht parsen lässt, wird abgelehnt,
was auch immer er besagt: Eine Stelle, die einen geschrieben hat, wollte das
Zertifikat einschränken. go-authn/sshcert erläutert das
[Szenario mit mehreren CAs](https://github.com/go-authn/sshcert#why-fail-closed-the-multi-ca-scenario).
{{< /callout >}}

### Mit der CA von EFP {#with-efps-ca}

`trusted_user_ca_file` entspricht `TrustedUserCAKeys` von OpenSSH. Legen Sie darin den Schlüssel ab, den EFP
unter <https://sshca.my-eurohpc.eu/config> veröffentlicht, wie es die
[Vertrauensseite](https://integration.docs.my-eurohpc.eu/aai/ssh-ca-trust/) von EFP beschreibt.
Diese URL liefert `{"PublicKey": "ssh-ed25519 ..."}`; die Datei enthält den Wert
von `PublicKey` auf einer eigenen Zeile:

```sh
curl -s https://sshca.my-eurohpc.eu/config | jq -r '.PublicKey' > /etc/fileshare/efp-ssh-ca.pub
```

Setzen Sie dann `ssh_domains` auf die Namen, die der Domain-Grant Ihrer Hosting-Einrichtung verwendet. Der eine Principal eines EFP-Zertifikats
ist der MyAccessID-Bezeichner
(`<id>@myaccessid.org`), das ist also der Benutzername, unter dem es sich anmeldet. Ein `user`-Block
dieses Namens, ohne eigene Zugangsdaten, nimmt es entgegen:

```hcl
user "u1234@myaccessid.org" {}
```

### Mit go-authn/bridge {#with-go-authnbridge}

Ein [go-authn/bridge](https://github.com/go-authn/bridge)-Client (v0.19.0 oder
neuer) mit `ssh_certificates = true` und
`ssh_domain_grants = ["files.example.org"]` stellt Zertifikate aus, deren Domain-Grant
diesen Host abdeckt. Mit `ssh_principal_claim = "voperson_id"` ist sein einer Principal die
`voperson_id` der Person, wie im EFP-Profil, und das ist der `user`-Block, den Sie
hier schreiben. Vertrauen Sie der CA von bridge so, wie dieser Server jeder vertraut:

- als `trusted_user_ca_file`, für lokale Konten. bridge stellt seinen Schlüssel im Format von EFP
  unter `/ssh/config` bereit, sodass die `curl | jq`-Zeile oben mit der URL von bridge funktioniert;
- oder als `ssh_ca_file` des `oidc`-Blocks, für
  [Personen, für die der Anbieter bürgt](#people-the-identity-provider-vouches-for).

Die anderen Clients von bridge schreiben keinen Domain-Grant. Wenn deren Zertifikate hier weiter funktionieren
müssen, ist `ssh_accept_ungranted` dafür da. Es lässt jedoch **jedes**
Zertifikat ohne Domain-Grant jeder vertrauenswürdigen Stelle zu; geben Sie diesen
Clients daher lieber einen Domain-Grant.

### Ein an eine Adresse gebundenes Zertifikat {#a-certificate-pinned-to-an-address}

`ssh_source_address` von bridge schreibt die kritische Option `source-address`, wie
es `ssh-keygen -O source-address=...` tut. Ein Zertifikat, das eine solche trägt, wird
**nur von diesen Adressen** akzeptiert, mit `ssh_domains` wie ohne. Eine
Anmeldung von einer Adresse, die das Zertifikat erlaubt, wird zugelassen, sofern sein Domain-Grant
bei gesetztem `ssh_domains` auch diesen Host nennt. Eine Anmeldung von jeder anderen Adresse wird
abgelehnt.

| fileshare | `trusted_user_ca_file` | `oidc` `ssh_ca_file` und OpenPubkey |
|---|---|---|
| vor v0.22.0 | durchgesetzt | **von jeder Adresse abgelehnt** |
| v0.22.0, ohne `ssh_domains` | durchgesetzt | **von jeder Adresse abgelehnt** |
| v0.22.0, mit `ssh_domains` | **von jeder Adresse abgelehnt** | **von jeder Adresse abgelehnt** |
| v0.22.1, mit oder ohne `ssh_domains` | durchgesetzt | durchgesetzt |

„Von jeder Adresse abgelehnt“ ist ausfallsicher geschlossen (fail closed): Die Einschränkung des Zertifikats wurde
nie gelockert, nur die Anmeldung ging verloren. Vor v0.22.1 lehnte der sshd von go-filesystems/sftp
jede kritische Option bei Zertifikaten ab, die er fileshare zur Entscheidung übergab; das
betrifft die des Anbieters, die von OpenPubkey und, unter `ssh_domains`,
jedes Zertifikat. v0.22.1 baut auf go-filesystems/sftp v0.5.1 auf, dessen sshd
`source-address` dort akzeptiert und auf jedem Zertifikatspfad durchsetzt,
gegen die Adresse, von der die Verbindung kommt, IPv4 oder IPv6.

Jede andere kritische Option (`force-command`, `verify-required`, ...) wird abgelehnt,
da dieser Server nicht danach handelt.

## Personen, für die der Identitätsanbieter bürgt {#people-the-identity-provider-vouches-for}

Seit v0.11.0 reicht ein `oidc`-Block auch bis SFTP – nicht mit einem Token, für das SSH
keinen Platz hat, sondern mit einem **Zertifikat**:

```hcl
oidc {
  issuer           = "https://login.example.org"
  audience         = "fileshare"
  ssh_ca_file      = "/etc/fileshare/bridge-ca.pub"   # go-authn/bridge's ssh_ca
  opkssh_client_id = "opkssh"                         # OpenPubkey logins
  opkssh_max_age   = "24h"                            # 12h, 24h, 48h, 1week
}
```

- **Ein Zertifikat, das die SSH-CA des Anbieters signiert hat** – `authn-bridge ssh-cert` schreibt eines
  nach einer Anmeldung über die Föderation. Sein Principal ist die Person, seine
  Erweiterung `groups@go-authn.org` deren Gruppen, sodass
  [`oidc:groups:`-Regeln]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}})
  greifen. Es ist **nicht** `trusted_user_ca_file`, dessen Zertifikate sich auf
  lokale Konten beziehen: Das Zertifikat einer lokalen Stelle, das die Gruppen des Anbieters beansprucht,
  wird als das lokale Konto gelesen, das es nennt.
- **Ein OpenPubkey-Zertifikat**, wie `opkssh login` es schreibt: signiert mit dem eigenen
  Schlüssel des Benutzers, mit einem ID-Token, das sich auf diesen Schlüssel festlegt. Es wird so geprüft, wie
  `opkssh verify` es prüft – mit dem Verifier von
  [openpubkey](https://github.com/openpubkey/openpubkey) (die Signatur des
  Anbieters gegen seine veröffentlichten Schlüssel, die Nonce-Bindung, die
  Client-ID, das Alter), dann den Schlüssel des Zertifikats gegen den des Tokens –, und der
  SSH-Benutzername muss der Benutzername des Tokens sein, weil es hier keine `auth_id`-Datei gibt,
  die das eine auf das andere abbildet: `sftp alice@univ-example.fr@files.example.org`.
  `opkssh_client_id` sollte eine eigene Client-ID sein, nie die einer anderen
  Anwendung: Das ID-Token wandert zu jedem Server, an dem sich die Person anmeldet.

Um die Gruppen zu sehen, die ein Zertifikat trägt:

```sh
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub     # the Extensions section
```

und der Server meldet bei jeder föderierten Anmeldung
`sftp: alice@univ-a.fr, vouched for by the provider, in groups [...]`.

{{< callout type="error" >}}
**Verbürgt heißt nicht zugelassen**

Jemand, für den der Anbieter bürgt, ist hier dennoch ein Fremder, es sei denn, eine Regel
benennt die Person, `trust_all` gibt an, dass der Anbieter das Verzeichnis ist, oder
`local_names = true` gibt an, dass die Namen des Anbieters die dieses Servers sind und ein lokales
Konto den ihren trägt – derselbe Test, den ein Token über WebDAV besteht. Ein Zertifikat des Anbieters **ohne Principal**,
nach der eigenen Definition des Formats für *jeden* gültig, wird abgelehnt.
{{< /callout >}}

`-tags noopenpubkey` lässt den OpenPubkey-Verifier weg (etwa 2 MB); eine
Konfiguration mit `opkssh_client_id` wird in einem solchen Build abgelehnt.

## Ein Zertifikat zurücknehmen {#taking-a-certificate-back}

{{< callout type="error" >}}
**Ein Zertifikat wird bei der Anmeldung geprüft**

Die Person beim Anbieter zu widerrufen, widerruft für sich allein kein
bereits ausgestelltes Zertifikat: Es öffnet SFTP, bis es abläuft, und eine bereits offene SFTP-Sitzung
bleibt bis dahin offen – diese Personen stehen nicht im Verzeichnis, daher
betrifft sie kein [Neuladen]({{< relref "/administration/reload.md" >}}). Das Fenster ist die
Lebensdauer des Zertifikats: `ssh_ca { validity }` von bridge (standardmäßig 12h, höchstens
168h, nie über das Ende der IdP-Sitzung hinaus) und hier `opkssh_max_age`. Halten Sie
es so kurz, wie die erneute Anmeldung der Clients es zulässt. Seit v0.17.0 endet auch eine bereits offene
opkssh-Sitzung bei `opkssh_max_age`, was auch immer das
Zertifikat (das die Person selbst signiert) sagt.
{{< /callout >}}

Zwei Mechanismen schließen dieses Fenster:

- die **[KRL]({{< relref "/security/revocation-lists.md" >}})** des Anbieters (`ssh_krl_url`):
  Ein widerrufenes Zertifikat wird bei der Anmeldung abgelehnt, und jede Operation einer Sitzung, die es
  geöffnet hat, fragt die Liste erneut, offene Dateien eingeschlossen;
- **[Shared Signals]({{< relref "/security/shared-signals.md" >}})** (`ssf`): Alles,
  was der Anbieter der Person vor einem CAEP-`session-revoked` ausgestellt hat, wird abgelehnt –
  der einzige Weg, ein OpenPubkey-Zertifikat zu erreichen, das in keiner Liste steht.

Beide schlagen geschlossen fehl.

## Kein Passwort {#no-password}

Ein Client, der nach einem fragt, tut genau das, was Schlüssel vermeiden sollen.

## Übertragungsgröße und offene Dateien {#transfer-size-and-open-files}

fileshare beantwortet OpenSSHs `limits@openssh.com` (seit v0.25.0), sodass
OpenSSHs `sftp` bis zu 255 KiB pro Anfrage liest und schreibt statt 32 KiB:
etwa +25 %, gemessen mit OpenSSH 10.3. Eine Sitzung darf höchstens 1024
Dateien und Verzeichnisse gleichzeitig offen halten; bei einer
Verzeichnisfreigabe ist jede ein Deskriptor des Hosts, und die Grenze hält
einen Benutzer davon ab, alle zu verbrauchen.
