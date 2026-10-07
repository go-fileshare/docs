---
title: "NFS mit Identitäten aus Zertifikaten"
weight: 30
description: "Eingeschränkte Freigaben über NFS für Personen bereitstellen, die ein X.509-Client-Zertifikat benennt, widerrufen durch eine CRL, und was das auf einem gemeinsam genutzten Client nicht versprechen kann."
tags: [sicherheit, nfs, tls, widerruf]
---

NFSv3 allein authentifiziert niemanden, weshalb eine Freigabe, die benennt, wer sie
nutzen darf, [nicht über NFS exportiert wird]({{< relref "/protocols/nfs.md" >}}) – es sei denn, etwas kann
Personen unterscheiden. Ein `kerberos`-Block ist eine Antwort. Seit v0.12.0 ist ein
Client-Zertifikat, das **eine Person benennt**, eine weitere:

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file; required
  crl_state_file = "/var/lib/fileshare/nfs.crl.state"     # the order, across a restart
}
```

## Was das Zertifikat enthält {#what-the-certificate-carries}

Nach einer Anmeldung beim Identitätsanbieter stellt go-authn/bridge ein **kurzlebiges
X.509-Client-Zertifikat** aus, das die Person so benennt, wie `rpc.tlsservd -u` von FreeBSD
sie liest:

- die Person: der SubjectAltName **otherName `1.3.6.1.4.1.2238.1.1.1`**, ein
  UTF8String `user@domain`
  ([draft-cel-nfsv4-rpc-tls-othername](https://www.ietf.org/archive/id/draft-cel-nfsv4-rpc-tls-othername-04.html)
  beschreibt ihn; seine OIDs sind noch nicht vergeben);
- ihre Gruppen: je eine SubjectAltName-URI,
  `tag:go-authn.github.io,2026:group:<group>` – eine **Tag-URI** (RFC 4151: keine
  Registrierung, genau dafür gemacht). `rpc.tlsservd` von FreeBSD liest den ersten
  Identitäts-otherName und überspringt jeden anderen SAN-Eintrag.

{{< callout type="info" >}}
**Warum eine Tag-URI und keine eigene Erweiterung**

Eine wurde entworfen, unter einem aus einer UUID abgeleiteten Bogen (`2.25.<128-bit number>`), und
**gemessen**: `x509.ParseCertificate` von Go lehnt das ganze Zertifikat ab –
*malformed extension OID field* –, weil ein Bogen eines ASN.1-Objektbezeichners dort
ein `int` ist. Jeder TLS-Server in Go hätte das Zertifikat zurückgewiesen.
Ein URI-SAN braucht keine OID, und jeder Parser liest ihn. Auch nicht `urn:x-…`:
RFC 8141 hat die experimentellen URN-Namensräume abgeschafft, und ein strenger Parser
lehnt einen solchen ab.
{{< /callout >}}

## Was bei jedem Aufruf geprüft wird {#what-every-call-is-asked}

Mit `identity = "certificate"` wird eine Freigabe, die Personen benennt, über NFS
**ohne Kerberos** bereitgestellt. Jeder Aufruf darauf muss über TLS ankommen, mit einem
Zertifikat, das:

1. **nicht in der CRL** steht;
2. **genau eine** Person benennt;
3. jemanden benennt, den dieser Server **so zulässt, wie er ein Token zulässt** – eine Regel,
   `trust_all`, `domains`, siehe
   [Personen, die der Identitätsanbieter benennt]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}});
4. jemanden benennt, **den die Freigabe erlaubt**.

Und mit einem [`ssf`-Block]({{< relref "/security/shared-signals.md" >}}) muss das Zertifikat
**nach** jedem Widerruf dieser Person ausgestellt worden sein (sein `NotBefore`).

## Widerruf: die CRL {#revocation-the-crl}

**Eine Sperrliste (CRL) ist erforderlich** – `crl_url` oder `crl_file`. Das Zertifikat einer
Person, das nichts widerrufen kann, überdauert ihre Entfernung um seine ganze Lebensdauer.

| Feld | Standard | |
|---|---|---|
| `crl_url` | | nur HTTPS |
| `crl_file` | | oder ein absoluter Pfad – das eine oder das andere |
| `crl_ca_file` | die des Systems | legt die Zertifizierungsstellen des HTTPS-Servers der CRL fest |
| `crl_refresh` | `1m` | wie oft sie abgerufen wird |
| `crl_max_age` | `1h` | wie alt die letzte gute Kopie sein darf |
| `crl_state_file` | | wo die letzte geprüfte CRL aufbewahrt wird, damit sich ein Neustart erinnert, welche neuer ist |

Sie wird wie die [KRL]({{< relref "/security/revocation-lists.md" >}}) abgerufen und aufbewahrt und ist auf
dieselbe Weise **ausfallsicher geschlossen (fail closed)**; eine CRL jenseits ihres eigenen
**`NextUpdate`** gilt als unbekannt, was auch immer `crl_max_age` erlaubt. Eine CRL muss
von der Client-CA signiert sein – eine CRL, die jeder hätte schreiben können, widerruft,
was immer er will, und, schlimmer noch, hebt den Widerruf wieder auf. Ist die CRL erreichbar,
wird **der nächste Aufruf nach ihrer Änderung abgelehnt**; die Lebensdauer des Zertifikats
begrenzt einen Widerruf nur, wenn die CRL nicht erreichbar ist.

Seit v0.16.2 wird die Signatur über [go-authn/revocation](https://github.com/go-authn/revocation)
an den Rohbytes geprüft, **bevor** die CRL geparst wird: Wurde zuerst geparst, kostete eine große,
von irgendjemandem signierte CRL Speicher und Sekunden, bevor sie abgelehnt wurde. Was dieser
Server nicht als vollständige Liste lesen kann, wird ebenfalls abgelehnt: eine Delta-CRL,
eine unbekannte kritische Erweiterung oder Eintragserweiterung, keine CRL-Nummer, kein
`NextUpdate`. Eine CRL mit einer niedrigeren Nummer als die gehaltene oder mit derselben
Nummer, früher ausgestellt, wird abgelehnt wie eine ältere
[KRL]({{< relref "/security/revocation-lists.md#never-backwards-across-a-restart-too" >}}).

Ein CRL-Feld ohne `identity = "certificate"` wird abgelehnt, ebenso
`identity = "certificate"` ohne `tls = true` und `client_ca_file`: Die
Identität steht in dem Zertifikat, das der Client vorlegt.

## Was sie nicht versprechen kann {#what-it-cannot-promise}

{{< callout type="error" >}}
**Das Zertifikat eines Linux-Clients gehört zu einem MOUNT, nicht zu einer Person**

Der Server sieht das Zertifikat einer **Verbindung**, und ein Linux-Client verknüpft
eines mit einem Mount (`tlshd`, die Keyring-Seriennummer bei
`mount -o xprtsec=mtls,...`). **Jeder Benutzer dieses Mounts handelt als die Person,
die das Zertifikat benennt.** Auf einem Arbeitsplatzrechner, den eine Person nutzt, ist das
genau sie; auf einem Rechner, an dem sich mehrere Personen anmelden, ist es, wer auch immer
eingehängt hat – verwenden Sie dort `sec=krb5`.

Deshalb lehnt es RFC 9289 allein ab, eine Benutzerauthentifizierung zu versprechen, und deshalb
weist einfaches [NFS über TLS]({{< relref "/security/tls.md#nfs-over-tls-proves-the-machine-not-the-person" >}})
weiterhin nur den Rechner nach.
{{< /callout >}}

## Gemessen mit einem echten Linux-Client {#measured-with-a-real-linux-client}

Kernel 6.17, ktls-utils 0.9, in der CI von go-filesystems/nfs:

- **`MNT` von MOUNT kommt unverschlüsselt an**, was auch immer `xprtsec=` sagt: Der
  Mount-Client des Kernels hat kein TLS. Er wird beantwortet, und jeder NFS-Aufruf ohne
  TLS wird danach abgelehnt – eine Person, die die Freigabe nicht erlaubt, sieht daher
  *access denied* beim ersten Zugriff statt bei `mount`. **Nichts von einer
  Freigabe geht unverschlüsselt über die Leitung.**
- `tlshd` prüft das Zertifikat dieses Servers nur gegen den Vertrauensspeicher des
  **Systems** – es ignoriert `x509.truststore` – und verlangt, dass Zertifikat und
  Schlüssel des Clients **root gehören, der Schlüssel mit Modus 600**. Andernfalls schlägt es mit
  *gnutls: Error in the certificate (-43)* fehl, ohne eines von beiden zu nennen.
