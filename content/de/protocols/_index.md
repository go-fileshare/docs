---
title: "Was ein Protokoll versprechen kann"
linkTitle: "Protokolle"
weight: 20
description: "Warum jedes Protokoll erkennen kann oder nicht, wer anfragt, und was das dafür bedeutet, welche Freigaben es bereitstellt."
tags: [protokolle]
---

Die Protokolle sind sich in dem einen Punkt nicht einig, den die Zugriffskontrolle braucht: ob
der Server erkennen kann, **wer** anfragt.

| | |
|---|---|
| **[SMB]({{< relref "/protocols/smb.md" >}})** | NTLMv2. Das Passwort geht nie über die Leitung, und die Freigabe teilt einer lesenden Person mit, dass sie nur liest – in der Zugriffsmaske, bevor sie es versucht. |
| **[WebDAV]({{< relref "/protocols/webdav.md" >}})** | HTTP Basic, über [TLS]({{< relref "/security/tls.md" >}}): unverschlüsselt auf einer erreichbaren Adresse abgelehnt, sofern nicht `plaintext = true` angibt, dass ein Proxy davor TLS terminiert. Eine Freigabe, die eine Person nicht nutzen darf, antwortet mit 404, nicht 403: Ihre Existenz wird nicht bestätigt. |
| **[SFTP]({{< relref "/protocols/sftp.md" >}})** | Ein **öffentlicher Schlüssel** oder ein **SSH-Zertifikat** von einer Stelle, der Sie vertrauen: Der Server hält nie das Geheimnis, und mit einem Zertifikat wird der Zugang einer Person anderswo ausgestellt und läuft anderswo ab. Kein Passwort: Ein Client, der danach fragt, tut genau das, was Schlüssel vermeiden sollen. |
| **[OIDC]({{< relref "/protocols/webdav.md" >}})** (über WebDAV) | Ein **Bearer-Token**, das ein Identitätsanbieter signiert hat. Geprüft von [go-authn/oidc](https://github.com/go-authn/oidc): Signatur, Aussteller, Zielgruppe, Ablauf. Kein anderes Protokoll hier hat einen Platz dafür – S3 signiert mit SigV4, das kein Feld für ein Bearer-Token hat. Über [SFTP]({{< relref "/protocols/sftp.md#people-the-identity-provider-vouches-for" >}}) kommt das Wort des Anbieters stattdessen in einem Zertifikat an. |
| **[S3]({{< relref "/protocols/s3.md" >}})** | **SigV4**, im Header oder vorsigniert. Das Geheimnis beweist sich, indem es einen HMAC berechnet, und geht nie über die Leitung – daher muss das Verzeichnis, wie bei NTLMv2, das Passwort HALTEN, statt es nur zu prüfen. Eine Freigabe ist ein Bucket. |
| **[NFSv3]({{< relref "/protocols/nfs.md" >}})** | Für sich allein **nichts**. `AUTH_UNIX` ist eine Behauptung – der Client sagt „uid 501“, und die Leitung kann nicht widersprechen. Ein `kerberos`-Block hebt das auf, ebenso [Identitäten aus Zertifikaten]({{< relref "/security/nfs-certificates.md" >}}). [TLS]({{< relref "/security/tls.md" >}}) verschlüsselt es und weist die Maschine nach, nicht die Person. |

## Die Konsequenz, einmal ausgesprochen {#the-consequence-stated-once}

**Eine Freigabe, die benennt, wer sie nutzen darf, wird nicht über NFS exportiert** – es sei denn,
Kerberos ist konfiguriert oder NFS bezieht
[Identitäten aus Zertifikaten]({{< relref "/security/nfs-certificates.md" >}}).

Keine Warnung, keine Option: Eine Konfiguration, die sagt *photos gehört alice*,
und ein Protokoll, das photos jedem aushändigt, der sich verbindet, können nicht beide eingehalten werden,
und den Zugriff stillschweigend auszuweiten ist der schlimmere der beiden Fehler. Die Ablehnung wird
beim Start und in [`check`]({{< relref "/configuration/check.md" >}}) ausgegeben, mit Begründung.

## Was jedes davon von einem Verzeichnis braucht {#what-each-one-needs-from-a-directory}

Welche Protokolle eine bestimmte Person bedienen können, entscheidet sich danach, was ihre Quelle
beweisen kann, nicht allein nach der Konfiguration. Siehe
[was eine Quelle beweisen kann]({{< relref "/configuration/identity.md#what-a-source-can-prove-and-what-each-protocol-needs" >}}).

## Was verschlüsselt ist {#what-is-encrypted}

WebDAV, S3 und NFS akzeptieren `tls = true`; SMB 3 verschlüsselt mit eigenen Schlüsseln, und SFTP
ist SSH. Siehe [TLS und Zertifikate über ACME]({{< relref "/security/tls.md" >}}).
