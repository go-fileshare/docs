---
title: "Una imagen, varios protocolos, un bloqueo"
weight: 30
description: "Por qué una imagen servida por varios protocolos se envuelve en un único bloqueo compartido, y lo que conserva ese envoltorio."
tags: [operación, bloqueo]
---

Cada servidor de esta familia serializa el propio controlador, porque
[`go-filesystems/interface`](https://github.com/go-filesystems/interface) no promete
nada sobre llamadas concurrentes basadas en rutas, y cada uno de ellos tiene **su
propio** bloqueo, sin saber nada de los demás.

Servir una misma imagen por tres protocolos a la vez no tendría, por tanto, ningún
bloqueo común en ningún sitio. Así que la imagen se envuelve **una sola vez**, aquí,
y el mismo envoltorio se entrega a todos ellos.

## Presentado como medido, para no exagerarlo {#stated-as-measured-so-as-not-to-overstate-it}

Ocho goroutines que escriben a través de un controlador fat32 **sin envolver**, con
`-race`, **no producen hoy ninguna condición de carrera**.

Es un seguro sobre un contrato, no la reproducción de un defecto: ext4 y ntfs no son
fat32, y una actualización de un controlador no es algo que este programa debiera
tener que volver a auditar.

## El envoltorio deja pasar las capacidades {#the-wrapper-carries-capabilities-through}

No las oculta:

- un controlador que responde a `Opener` recibe a cambio un **`File` con bloqueo**;
- uno cuyo `File` responde a `WritableFile` conserva sus **escrituras posicionales**.

La diferencia entre estas y la alternativa de archivo completo se midió en **~70×**
en otro lugar de la familia, así que un envoltorio que las borrara sería un defecto
de rendimiento disfrazado de seguridad.

## Con `--isolate` el bloqueo no puede ayudar {#under-isolate-the-lock-cannot-help}

Un proceso hijo abre la imagen él mismo, así que no hay envoltorio compartido entre
hijos. Por eso una imagen servida en escritura por dos protocolos se
[rechaza]({{< relref "/operations/isolation.md#the-rule-that-makes-it-honest" >}}) en lugar de
servirse discretamente sin bloqueo.
