---
title: "WebDAV – Basic oder ein Bearer-Token"
linkTitle: "WebDAV: Basic oder ein Bearer-Token"
weight: 20
description: "WebDAV mit HTTP Basic oder einem OIDC-Bearer-Token, das TLS, das es verlangt, und wie hier aus einem Token eine Person wird."
tags: [protokolle, webdav, oidc]
---

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

HTTP Basic oder ein Bearer-Token – über [TLS]({{< relref "/security/tls.md" >}}), aus Dateien oder
über ACME.

{{< callout type="error" >}}
**Nicht unverschlüsselt, seit v0.9.0**

HTTP Basic ist das Passwort bei jeder Anfrage, und ein Bearer-Token ist so gut
wie eines. WebDAV **ohne TLS**, auf einer Adresse, die andere Maschinen erreichen können,
während sich irgendjemand authentifiziert, wird **beim Start abgelehnt** –
`serve "webdav" { addr = "0.0.0.0:8080" }`, das Beispiel, das diese Seite früher
zeigte, startet nicht mehr. Wenn ein Proxy davor TLS terminiert, geben Sie das an:

```hcl
serve "webdav" {
  addr      = "10.0.0.5:8080"
  plaintext = true
}
```

Loopback und ein Server, bei dem sich niemand authentifiziert, sind nicht betroffen. Siehe
[TLS]({{< relref "/security/tls.md#webdav-is-not-served-in-the-clear" >}}).
{{< /callout >}}

{{< callout type="info" >}}
**404, nicht 403**

Eine Freigabe, die eine Person nicht nutzen darf, antwortet mit **404**. Ihre Existenz wird nicht bestätigt.
{{< /callout >}}

## Ein Token, über das eine Protokoll, das eines tragen kann {#a-token-over-the-one-protocol-that-can-carry-one}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
}
```

Ein Browser hat ein Token und kein Passwort. Daher akzeptiert WebDAV `Authorization:
Bearer`, und wo es auch lokale Personen gibt, bietet die gesendete Challenge
**beides** an – ein Client wählt das, worauf er antworten kann. Das Token wird von
[go-authn/oidc](https://github.com/go-authn/oidc) geprüft: Signatur, Aussteller, Zielgruppe,
Ablauf.

{{< callout type="error" >}}
**Ein Bearer-Token: nur WebDAV**

SMB authentifiziert mit NTLMv2, SFTP mit einem Schlüssel oder einem Zertifikat, NFS mit
gar nichts und S3 mit einer SigV4-Signatur: Keines davon hat einen Platz für
einen `Authorization`-Header. Das ist eine Tatsache der Protokolle, keine
Grenze dieses Programms.

Das Wort des Anbieters erreicht **SFTP** stattdessen in einem Zertifikat – siehe
[SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}). Eine Konfiguration,
die einen Anbieter nennt und weder WebDAV noch ein solches SFTP bereitstellt, wird **abgelehnt
statt gestartet**.
{{< /callout >}}

{{< callout type="error" >}}
**Ein Token sagt, für wen der Anbieter jemanden hält. Es sagt nicht, dass dieser Server eine Freigabe für ihn hat.**

Ein gültiges Token für einen Namen, den keine Quelle hier kennt, wird abgelehnt – die sichere Lesart
von *Ich kenne Sie nicht* ist nicht *Sie dürfen*. Ebenso wenig erhält ein Name, den auch ein lokales
Konto trägt, dieses Konto, sofern nicht `local_names = true` angibt, dass die
Namen des Anbieters die dieses Servers sind (standardmäßig aus, seit v0.20.0); ohne
diese Einstellung wird ein solches Token ebenfalls abgelehnt, es sei denn, eine `oidc:`-Regel benennt die Person.
{{< /callout >}}

Eine Site, bei der der Anbieter das Verzeichnis **ist**, gibt das an:

```hcl
oidc {
  issuer    = "https://login.example.org"
  audience  = "fileshare"
  trust_all = true      # everybody that provider vouches for, not just these people
}
```

Hier gibt es **keinen Anmeldeablauf**: keine Weiterleitung, kein Client-Secret, keine Cookies.
Dies ist der Ressourcenserver.

**Der Name**, den ein Token liefert, ist `preferred_username` oder der Claim, den
`username_claim` nennt. Ohne `preferred_username` ist es die E-Mail-Adresse nur, wenn
der Anbieter `email_verified: true` angibt, und andernfalls `sub`. Mit
`username_claim = "email"` wird ein Token, dessen E-Mail-Adresse nicht verifiziert ist, abgelehnt –
über WebDAV ebenso wie für eine opkssh-Anmeldung über SFTP. Bis sie verifiziert ist,
ist die E-Mail-Adresse nur das, was die Person eingetippt hat, und ein signiertes Token, das sie trüge,
würde an eine Adresse binden, die niemand geprüft hat (OpenID Connect Core 5.1;
go-authn/oidc v0.2.0, fileshare v0.16.3).

Welche Personen des Anbieters eine Freigabe erhalten – Gruppen, benannte Personen, ganze
Einrichtungen –, wird mit `oidc:`-Regeln und `domains` festgelegt; siehe
[Personen, die der Identitätsanbieter benennt]({{< relref "/configuration/identity.md#people-the-identity-provider-names-not-this-file" >}}).

## Ein Token überdauert die Person, sofern nichts anderes es sagt {#a-token-outlives-the-person-unless-something-says-otherwise}

Ein Token wird hier für sich allein geprüft und ist gültig, bis es abläuft – was auch immer
der Person seitdem widerfahren ist. Keine Sperrliste kann es benennen. Ein
[`ssf`-Block]({{< relref "/security/shared-signals.md" >}}) sorgt dafür, dass ein CAEP-`session-revoked`
vom Anbieter jedes zuvor (`iat`) ausgestellte Token ungültig macht.
