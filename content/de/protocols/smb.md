---
title: "SMB – NTLMv2"
linkTitle: "SMB: NTLMv2"
weight: 10
description: "SMB mit NTLMv2: was ein Verzeichnis enthalten muss, der privilegierte Port unter Isolierung und was ein Neuladen des Verzeichnisses mit offenen Verbindungen macht."
tags: [protokolle, smb]
---

```hcl
serve "smb" { addr = "0.0.0.0:445" }
```

Das Passwort **geht nie über die Leitung**: Ein Client sendet einen daraus berechneten
Nachweis. Die Freigabe teilt einer lesenden Person mit, dass sie nur liest – in der Zugriffsmaske, bevor sie
es versucht, statt durch einen später fehlschlagenden Schreibvorgang.

## Was ein Verzeichnis enthalten muss {#what-a-directory-must-hold}

NTLMv2 braucht das Passwort selbst oder seinen MD4-Hash (`sambaNTPassword`, eine `nt_hash`-Spalte).
**Nichts anderes genügt.**

Ein Verzeichnis, das Passwörter nur *prüft* – ein LDAP-Bind, eine bcrypt-Spalte –,
kann SMB nicht beantworten, wie gut die Prüfung auch sein mag, weil der Server denselben
Nachweis berechnen muss wie der Client. Das ist eine Eigenschaft des Protokolls, keine
Einschränkung dieses Programms.

[`check`]({{< relref "/configuration/check.md" >}}) gibt für eine solche Person ein `-` in der SMB-Spalte aus,
zum Zeitpunkt der Konfiguration, statt sie es erst beim Einhängen
entdecken zu lassen.

## Ein privilegierter Port ohne privilegierten Server {#a-privileged-port-without-a-privileged-server}

Unter [`--isolate`]({{< relref "/operations/isolation.md" >}}) bindet unter Unix der Elternprozess Port 445
und übergibt den Listener an den Kindprozess, sodass der Prozess, der tatsächlich SMB spricht,
das Privileg nie braucht. Windows hat keine `ExtraFiles`, daher bindet dort der Kindprozess
die Adresse selbst; die Isolierung ist dieselbe, die Hälfte mit dem privilegierten Port
nicht.

## Geprüft mit einem Client, den dieses Projekt nicht geschrieben hat {#verified-against-a-client-this-project-did-not-write}

[go-smb2](https://github.com/cloudsoda/go-smb2) führt in der CI zwanzig gleichzeitige
Lesevorgänge über SMB aus, während zwanzig über WebDAV laufen, unter `-race`. macOS
hängt eine Freigabe über SMB ein, während `curl` über WebDAV in dasselbe Image schreibt, und
liest zurück, was WebDAV geschrieben hat.

## Wenn sich die Personen ändern {#when-the-people-change}

SMB legt fest, wer sich mit einer Freigabe verbinden darf, **wenn die Freigabe hinzugefügt wird**, und prüft das
einmal pro Tree Connect. Wer also neu ist und von einem [Neuladen des Verzeichnisses]({{< relref "/administration/reload.md" >}})
gefunden wird, wird an Ort und Stelle hinzugefügt, einschließlich des laufenden SMB-Servers, ohne dass eine Verbindung
angetastet wird; alles hingegen, was entzogen wird – eine Person, Zugangsdaten, die expandierten Listen einer Freigabe –,
ist eine neue Generation, und die offenen SMB-Verbindungen werden geschlossen, sodass ein
Widerruf die bereits offenen Sitzungen erreicht. Eine Freigabe, deren `allow`-Gruppe sich
geleert hat, wird **überhaupt nicht über SMB angeboten**: Ihr leeres `AllowUsers` würde als
„alle“ gelesen.
