---
title: "NFS – nichts, außer mit Kerberos oder einem Zertifikat"
linkTitle: "NFS: nichts, außer mit Kerberos oder einem Zertifikat"
weight: 50
description: "NFSv3 authentifiziert für sich allein niemanden; wie Kerberos oder Client-Zertifikate es eingeschränkte Freigaben bereitstellen lassen und was TLS hinzufügt."
tags: [protokolle, nfs, kerberos]
---

```hcl
serve "nfs" { addr = "0.0.0.0:2049" }
```

{{< callout type="error" >}}
**NFSv3 authentifiziert für sich allein niemanden**

`AUTH_UNIX` ist eine **Behauptung**: Der Client sagt „uid 501“, und die Leitung kann
ihm nicht widersprechen. Verschlüsselung gibt es ebenfalls nicht.
{{< /callout >}}

Daher **wird eine Freigabe, die benennt, wer sie nutzen darf, nicht über NFS exportiert**. Die Ablehnung
wird beim Start und in [`check`]({{< relref "/configuration/check.md" >}}) ausgegeben, mit
Begründung:

```
nfs    on 0.0.0.0:2049 — scratch
       photos is not served over nfs: it is restricted to alice and bob, and
       NFSv3 on its own has no authentication: AUTH_UNIX is a claim the client
       makes about itself and the wire cannot disagree with it. A kerberos
       block lifts this: sec=krb5 carries a principal a ticket proves; so
       does identity = "certificate" on the nfs serve block: RPC-over-TLS
       with a client certificate naming the person
```

Eine Freigabe ohne `allow` und ohne `writers` – jeder, der sich verbindet, mit Lese- und Schreibzugriff – wird
über NFS exportiert, weil es dort keine Regel gibt, an deren Einhaltung das Protokoll
scheitern könnte.

## Die Ablehnung, die früher endgültig war {#the-refusal-that-used-to-be-permanent}

Ein `kerberos`-Block macht NFS fähig, Personen zu unterscheiden, und eine eingeschränkte Freigabe
wird dann darüber bereitgestellt wie überall sonst:

```hcl
kerberos {
  realm  = "EXAMPLE.ORG"
  keytab = "/etc/fileshare/krb5.keytab"
}
```

`sec=krb5` überträgt einen Principal, den ein **Ticket beweist**, statt einer uid, die der
Client behauptet.

{{< callout type="info" >}}
**Der Realm wird verglichen, nicht nur der Name vor dem `@`**

Zwei Realms können jeweils eine `alice` haben, und nur einer davon ist Ihrer.
{{< /callout >}}

## Über TLS: die Maschine, nicht die Person {#over-tls-the-machine-not-the-person}

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/tls/clients.pem"
}
```

Seit v0.9.0 wird NFS mit `tls = true` und einem
[`tls`-Block]({{< relref "/security/tls.md" >}}) über RPC-with-TLS (RFC 9289) bereitgestellt. Es verschlüsselt; mit `client_ca_file` verlangt es von einem
Client ein Zertifikat dieser Stelle – das belegt, welcher **Host** einhängt.
Die uid darin ist weiterhin die, die `AUTH_SYS` behauptet, daher wird eine eingeschränkte Freigabe darüber
**weiterhin abgelehnt**. TLS wird angeboten, nicht verlangt: Ein Client, der nie
danach fragt, erhält weiterhin die offenen Freigaben.

## Eine Person, aus ihrem Zertifikat {#a-person-from-their-certificate}

Seit v0.12.0 kann ein Zertifikat eine **Person** benennen – so wie go-authn/bridge eines
nach einer Anmeldung beim Identitätsanbieter ausstellt –, und dann wird eine Freigabe, die Personen benennt,
ohne Kerberos über NFS bereitgestellt:

```hcl
serve "nfs" {
  tls            = true
  client_ca_file = "/etc/fileshare/bridge-x509-ca.pem"
  identity       = "certificate"
  crl_url        = "https://bridge.example.org/x509/crl"   # or crl_file: one is required
}
```

{{< callout type="error" >}}
**Das Zertifikat eines Linux-Clients gehört zu einem Mount**

Jeder Benutzer dieses Mounts handelt als die Person, die das Zertifikat benennt. Richtig auf
einem Arbeitsplatzrechner, den eine Person nutzt; falsch auf einer Maschine, an der sich mehrere Personen anmelden –
verwenden Sie dort `sec=krb5`.
{{< /callout >}}

Was das Zertifikat enthält, die erforderliche Sperrliste (CRL) und was mit einem
echten Linux-Client gemessen wurde (`MNT` unverschlüsselt, die Fallstricke von `tlshd`), steht in
[NFS, mit Identitäten aus Zertifikaten]({{< relref "/security/nfs-certificates.md" >}}).
