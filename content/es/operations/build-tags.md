---
title: "Compilar solo lo que necesita"
weight: 10
description: "Las etiquetas de compilación que dejan fuera del binario protocolos, directorios de usuarios, la API de administración y el aprovisionador, y lo que ahorra cada una."
tags: [operación, compilación]
---

Cada protocolo está detrás de una etiqueta de compilación (build tag), y una etiqueta
lo deja fuera **por completo**: ni escucha, ni analizador, ni dependencia, ni código.

```sh
go install -tags nonfs,nowebdav,nosftp,nos3 github.com/go-fileshare/fileshare@latest   # SMB only
go build   -tags nosmb,nonfs,nosftp,nos3 .                                            # WebDAV only
```

De dónde proceden las **personas** depende de etiquetas propias: `nosql` deja fuera
los tres controladores de bases de datos, `noldap` el cliente LDAP.

| compilación | tamaño |
|---|---|
| todo | 31,2 MB |
| `-tags noldap` | 30,9 MB |
| `-tags nosftp` | 30,6 MB |
| `-tags noopenpubkey` (sin inicios de sesión opkssh por SFTP) | unos 2 MB menos |
| `-tags nos3` | 31,1 MB |
| `-tags nonfs,nowebdav,nosftp,nos3` (solo SMB) | 28,1 MB |
| `-tags nopartitioned` (sin apfs, btrfs, xfs, zfs) | 29,2 MB |
| `-tags nosql` | 20,2 MB |
| `-tags nosql,noldap` | 19,8 MB |
| `-tags nosql,noldap,nonfs,nowebdav,nosftp,nos3` | 16,2 MB |

Son las cifras que da el README de fileshare, medidas antes de que llegara la API de
administración; la tabla de abajo es la medición posterior.

## La API de administración, y `nogrpc` {#the-admin-api-and-nogrpc}

La [API de administración]({{< relref "/administration/_index.md" >}}) trajo gRPC y protobuf,
y están detrás de `nogrpc` (linux/amd64, medido el 2026-09-29, cuando las filas de
arriba habían crecido hasta 34,1 MB para todo):

| compilación | tamaño |
|---|---|
| todo, con la API de administración | 46,1 MB |
| `-tags nogrpc` | 34,4 MB |
| `-tags nosql,noldap,nogrpc` | 22,5 MB |

gRPC cuesta **11,7 MB**: la [medición de los plugins](#why-not-subprocess-plugins)
de más abajo, repetida ahora que está aquí por un motivo propio. Un sitio que
gestiona sus recursos compartidos en archivos no necesita cargar con él.

{{< callout type="warning" >}}
**Un bloque `admin` en una compilación `nogrpc` se rechaza**

En lugar de servirse sin él: una configuración que pide la API y arranca
sin ella no hace lo que dice.
{{< /callout >}}

`nogrpc` deja fuera la API de administración y, con ella, el
[aprovisionador]({{< relref "/administration/volumes.md" >}}). [Salud y métricas]({{< relref "/administration/health.md" >}})
son HTTP simple y se quedan.

`-tags noprovisioner` deja fuera solo el aprovisionador, el rol privilegiado
`fileshare provisioner`, y con él zfs, btrfs y projquota de go-fsctl: un servidor
que nunca sirve volúmenes no necesita cargar con el código que los crea.
`fileshare provisioner` en una compilación así lo indica, en lugar de decir
«comando desconocido».

`nosql` es con diferencia la palanca más grande: PostgreSQL, MySQL y SQLite juntos
pesan **11,7 MB**, más que todos los protocolos de este programa juntos. Un sitio
cuyos usuarios están en el archivo la quiere. Los controladores los importa el
comando y no la biblioteca que los usa, y eso es lo que convierte la elección en una
etiqueta de compilación y no en un fork.

{{< callout type="info" >}}
**Un protocolo con el que este binario no se compiló**

A una configuración que nombra uno se le dice *eso*, y no «no existe tal
protocolo»: la diferencia entre una errata y una etiqueta de compilación. Lo
mismo vale para `opkssh_client_id` con `noopenpubkey` y para un bloque
`admin` con `nogrpc`.
{{< /callout >}}

## Por qué no plugins en subprocesos {#why-not-subprocess-plugins}

Se **midió en lugar de argumentarse**: `hashicorp/go-plugin` trae gRPC y protobuf,
que cuestan **13,2 MB por sí solos**, más que todo este binario con los tres
protocolos y todos los controladores. Un host de plugins tendría el doble de tamaño
antes de cargar nada, y cada binario de plugin volvería a llevar gRPC.

En cuanto a superficie de ataque, una etiqueta es también la herramienta más sólida
para todo lo que no ejecuta: **el código que nunca se compiló no se puede
alcanzar**, esté aislado o no.

Lo que las etiquetas *no* dan es aislamiento entre los protocolos que **sí** ejecuta,
ni una forma de añadir un protocolo sin recompilar. Ambas cosas son reales, y ambas
son argumentos a favor de una frontera de proceso y no de un binario más pequeño, que
es [un proceso por protocolo]({{< relref "/operations/isolation.md" >}}), con este mismo binario, no un
framework de plugins.
