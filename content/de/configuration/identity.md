---
title: "Benutzer, Gruppen und Verzeichnisse"
weight: 20
description: "Woher die Personen kommen (user- und group-Blöcke, SQL, LDAP, ein Identitätsanbieter) und was jede Quelle sie nutzen lässt."
tags: [konfiguration, identität, oidc, ldap, sql]
---

Ein `user`-Block ist das ganze Verzeichnis für einen Haushalt. Eine Installation, deren Personen
bereits in einer Datenbank oder in LDAP stehen, sollte sie nicht an einen zweiten Ort kopieren, der
veraltet; also liest ein `users`-Block sie dort, wo sie sind.

```hcl
users "sql" {
  driver   = "postgres"                 # or sqlite, or mysql
  dsn_file = "/etc/fileshare/dsn"       # a DSN holds a password: it lives in a file
  users    = "select login, null, nt_hash, ssh_keys from staff"
  groups   = "select team, member from team_members"
}

users "ldap" {
  url                = "ldaps://ldap.example.org"
  base_dn            = "ou=people,dc=example,dc=org"
  bind_dn            = "cn=reader,dc=example,dc=org"
  bind_password_file = "/etc/fileshare/bind.pw"
}
```

Die **Abfragen sind Ihre**, denn die Personen einer Installation liegen bereits in deren
Form vor; ein hier erfundenes Schema würde bedeuten, sie in ein zweites zu kopieren. Die
Spalten der Personenabfrage werden **nach Position** gelesen, nicht nach Name: ein Name, dann
beliebige von Passwort, NT-Hash, SSH-Schlüsseln (einer pro Zeile) und TOTP-Geheimnis, in
dieser Reihenfolge. Ein NULL sind Zugangsdaten, die diese Person nicht hat; deshalb
wählt das Beispiel `null` für das Passwort. Eine Passwortspalte wird als das
Passwort selbst genommen: Es gibt hier keine Einstellung für ein gehashtes.

Die LDAP-Seite liest, was ein Samba-fähiges Verzeichnis bereits veröffentlicht –
`sambaNTPassword`, `sshPublicKey`, `memberUid`. Wo die Personen und Gruppen
liegen und welche Attribute sie benennen, ist konfigurierbar: `user_filter`
(Standard `(objectClass=posixAccount)`), `user_attribute` (`uid`),
`group_base_dn` (`base_dn`), `group_filter` (`(objectClass=posixGroup)`),
`group_attribute` (`cn`), `group_member_attribute` (`memberUid`) und
`totp_attribute`, das keinen Standard hat. `sambaNTPassword` und `sshPublicKey`
werden unter diesen Namen gelesen.

{{< callout type="error" >}}
**Kein unverschlüsselter Bind an eine andere Maschine (seit v0.19.0)**

Ein `users "ldap"`-Block lehnt `ldap://` zu einer anderen Maschine ab, es sei denn,
`start_tls = true` ist gesetzt: Jedes Passwort, das ein Bind prüft, ginge im Klartext
über das Netz. `ldaps://` wird akzeptiert, ebenso `ldap://` zu einer Loopback-Adresse
und `ldapi://`, wo niemand dazwischen ist. Es gibt keinen Schalter, um
das abzuschalten (go-authn/directory v0.10.0).
{{< /callout >}}

## Reihenfolge, und die eine Ausnahme {#order-and-the-one-exception}

Quellen werden **in der Reihenfolge gefragt, in der sie geschrieben sind**, und die erste, die
einen Namen kennt, besitzt ihn. Die `user`- und `group`-Blöcke kommen zuerst, sodass ein lokal
eingetragenes Dienstkonto nicht von jemandem mit demselben Namen
in LDAP überschrieben wird.

**Gruppen sind die Ausnahme**: Die Mitglieder einer Gruppe sind die **Vereinigung** aller
Quellen, denn ein Team kann Personen in einer Datei und in einer Datenbank haben.

```hcl
share "photos" {
  allow   = ["@engineers", "alice"]
  writers = ["@owners"]
}
```

{{< callout type="info" >}}
**Beim Start aufgelöst, und erneut bei jedem Neuladen**

Seit v0.10.0 werden die `users`-Blöcke – SQL, LDAP – alle
`reload = "5m"`, bei `SIGHUP` und beim `ReloadDirectory` der Admin-API erneut gelesen; eine
Mitgliedschaft, die sich in LDAP ändert, wartet nicht mehr auf einen Neustart. Was ein Neuladen
an Ort und Stelle anwendet und was es neu startet, und warum eine Gruppe, die sich leert,
ihre Freigabe **nicht** für alle öffnet, steht unter
[das Verzeichnis erneut einlesen]({{< relref "/administration/reload.md" >}}). Das Verzeichnis
wird weiterhin nicht bei jeder Verbindung gefragt: Das wäre ein anderes Design, und dieses
ist es nicht.
{{< /callout >}}

## Was eine Quelle beweisen kann und was jedes Protokoll braucht {#what-a-source-can-prove-and-what-each-protocol-needs}

Das ist der Teil, den eine Installation sonst beim Einhängen entdeckt, also sagt [`check`]({{< relref "/configuration/check.md" >}})
ihn zuerst:

| die Quelle hat | SMB | S3 | WebDAV | SFTP |
|---|---|---|---|---|
| ein Passwort (Datei oder eine Klartextspalte) | ja | ja | ja | — |
| einen NT-Hash (`sambaNTPassword`, `nt_hash`) | ja | **nein** | — | — |
| nur einen Bind (LDAP) | **nein** | **nein** | ja | — |
| öffentliche Schlüssel oder eine vertrauenswürdige CA | — | — | — | ja |

{{< callout type="error" >}}
**NTLMv2 braucht das Passwort oder seinen MD4, und nichts anderes genügt**

Ein Client sendet einem SMB-Server nie ein Passwort – er sendet einen daraus
berechneten Nachweis –, sodass ein Verzeichnis, das Passwörter nur *prüft*, SMB nicht
beantworten kann, so gut die Prüfung auch ist. Das ist eine Eigenschaft des Protokolls,
keine Einschränkung dieses Programms, und keine Konfiguration ändert
daran etwas. WebDAV fragt nur „Ist das das richtige Passwort?", was ein Bind beantwortet.
{{< /callout >}}

Eine Person, die ein Verzeichnis nennt, für die es aber nichts beweist, ist legitim – eine Auflistung
mit den Geheimnissen anderswo –, und `check` sagt das in einer Zeile, statt
sie es selbst herausfinden zu lassen.

## Personen, die der Identitätsanbieter nennt, nicht diese Datei {#people-the-identity-provider-names-not-this-file}

Eine Föderation – etwa RENATER über
[go-authn/bridge](https://github.com/go-authn/bridge) – weiß, wer zu einem
Projekt gehört, und eine Freigabe kann **sie** fragen, statt die Liste zu kopieren. Mit einem
[`oidc`-Block]({{< relref "/protocols/webdav.md#a-token-over-the-one-protocol-that-can-carry-one" >}}):

```hcl
share "photos" {
  image   = "/srv/photos.img"
  allow   = ["oidc:groups:urn:mace:univ-example.fr:photos", "oidc:user:bob@univ-example.fr", "alice"]
  writers = ["oidc:groups:urn:mace:univ-example.fr:photos"]
}
```

`oidc:groups:<value>` ist jemand, dessen Groups-Claim im Token den Wert enthält;
`oidc:user:<name>` ist jemand, den das Token nennt. Die Schreibweise ist die von opkssh, damit
ein einziges Vokabular sagt, wer eine Shell erreicht und wer eine Freigabe. Jemand,
den eine Regel nennt, ist diesem Server bekannt, soweit der Anbieter reicht – auch für die offenen
Freigaben –, und jemand, den keine Regel nennt, bleibt ein Fremder. Dieselben Regeln
gelten über [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}),
wo die Gruppen statt im Token im Zertifikat reisen.

{{< callout type="error" >}}
**Eine Regel betrifft nur die Personen des Anbieters**

`oidc:user:bob` ist der bob, für den der Anbieter bürgt; ein lokales Konto namens
bob, mit einem Passwort, ist jemand anderes und erfüllt keine Regel. Eine Regel ohne
`oidc`-Block, eine fehlerhafte oder eine, die schreiben darf, ohne sich verbinden
zu dürfen, wird **beim Start abgelehnt**.
{{< /callout >}}

{{< callout type="error" >}}
**Die `alice` des Anbieters ist nicht die lokale `alice`, es sei denn, Sie sagen es (seit v0.20.0)**

Ein einfacher Name in `allow` oder `writers` ist ein lokales Konto; ein Token oder ein
Zertifikat des Anbieters wird nur von den `oidc:`-Regeln einer Freigabe erreicht. Eine Installation,
deren Namen beim Anbieter ihre lokalen Namen SIND, schreibt `local_names = true` in
den `oidc`-Block – und sollte dazu `domains` setzen oder den Namen aus
einem Claim nehmen, den der Anbieter kontrolliert: Ein Anbieter, bei dem Personen ihren
`preferred_username` selbst wählen, würde sonst jedem, der sich registriert, die Freigaben eines lokalen
Kontos übergeben, Schreibzugriff eingeschlossen. Das war vor v0.20.0 der Standard
(Sicherheitsaudit F4). Ein föderierter Name, der mangels `local_names` abgelehnt wird,
wird mit der zu ergänzenden Zeile protokolliert, und `fileshare check` sagt, welche Regel
gilt.

NFS-Client-Zertifikate (`identity = "certificate"`) sind die Ausnahme:
Ihre CA ist die, die `client_ca_file` genau dafür festlegt, und einfache
Namen sind die einzige Möglichkeit, mit der eine NFS-Freigabe jemanden benennen kann, also sind ihre Namen
lokale Namen, was auch immer `local_names` sagt.
{{< /callout >}}

### Welche Einrichtungen {#which-institutions}

```hcl
oidc {
  issuer   = "https://login.example.org"
  audience = "fileshare"
  domains  = ["univ-a.fr", "univ-b.fr"]   # nobody else from the federation gets in
}

share "projet-x" {
  image   = "/srv/projet-x.img"
  allow   = ["oidc:groups:urn:mace:univ-a.fr:projet-x", "oidc:domain:univ-b.fr"]
  writers = ["oidc:groups:urn:mace:univ-a.fr:projet-x"]
}
```

`domains` wird bei der Authentifizierung geprüft, über SFTP und WebDAV gleichermaßen, und für
NFS-Client-Zertifikate: Ein Name muss
`<something>@<one of them>` sein, als Ganzes verglichen (`evilunivb.fr` ist nicht
`univb.fr`). `oidc:domain:` ist derselbe Test für eine einzelne Freigabe. Der Domain kann man
so weit vertrauen wie dem Anbieter: go-authn/bridge verwirft einen eppn oder eine subject-id,
deren Scope die Föderationsmetadaten des IdP ihm nicht zugestehen.

**Gruppen** sind das, was der IdP der Einrichtung freigibt, durch
`claims { groups = [...] }` von go-authn/bridge in den `groups`-Claim umgewandelt: standardmäßig `eduPersonEntitlement`
(die Gruppen eines Labors oder einer VO) oder `eduPersonScopedAffiliation`
(`staff@univ-a.fr`, `student@univ-b.fr`).
