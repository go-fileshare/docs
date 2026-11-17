---
title: "automount"
weight: 10
description: "Monter un partage SMB dans le gestionnaire de fichiers du système sans boîte de dialogue de mot de passe, et le démonter."
---

[go-fileshare/automount](https://github.com/go-fileshare/automount) v0.1.2
monte un partage SMB dans le gestionnaire de fichiers du système **sans aucune
boîte de dialogue de mot de passe**, et le démonte. C'est une petite façade en
Go pur (sans cgo) au-dessus de trois moteurs existants. Cette page est écrite à
partir de son
[README](https://github.com/go-fileshare/automount/blob/main/README.md) ; là où
elle diffère du dépôt, le dépôt fait foi.

| Système | Moteur | Ce qui apparaît |
|---|---|---|
| macOS | [go-macos/netfs](https://github.com/go-macos/netfs) (NetFS, `NoUI`) | un volume sous `/Volumes`, dans la barre latérale du Finder |
| Linux (GNOME) | [go-freedesktop/gvfs](https://github.com/go-freedesktop/gvfs) (répond à l'`AskPassword` de GVfs) | un emplacement gvfs, dans Fichiers |
| Windows | [go-mswin/win32](https://github.com/go-mswin/win32) (`MSFT_SmbMapping.Create`, tout port TCP, utilisateur standard) | une lettre de lecteur, dans « Ce PC » |

Tout autre GOOS se compile et répond `automount.ErrUnsupported`.

```go
u, _ := url.Parse("smb://127.0.0.1:4455/theca")
m, err := automount.Mount(ctx, automount.Share{URL: u, User: "alice", Password: pw}, automount.Options{})
if err != nil { /* errors.Is(err, netfs.ErrAuth) etc. */ }
fmt.Println(m.Path())   // /Volumes/theca, Z:\, /run/user/1000/gvfs/...
defer m.Unmount(ctx)
```

`Mount` refuse, avant de toucher au système, une URL nulle ou non `smb`, un hôte
ou un partage manquant, un partage avec un sous-chemin, un mauvais port, des
informations d'utilisateur dans l'URL, un lecteur Windows invalide ou utilisé,
et `ReadOnly` là où il ne peut pas être honoré (seulement macOS). Le mot de
passe ne passe jamais dans un vecteur d'arguments, l'environnement, une URL, un
message d'erreur ou un journal.

## Mesuré, et où {#measured}

- **macOS 27.2.1 (arm64) : mesuré.** Le serveur SMB de
  [go-filesystems/smb](https://github.com/go-filesystems/smb) tourne dans le
  processus de test sur `127.0.0.1:<port aléatoire>` ; la façade le monte, un
  fichier est lu et écrit via `Path()`, `Unmount` retire le volume. Un mauvais
  mot de passe échoue avec `netfs.ErrAuth`.
- **Linux (Ubuntu 24.04, gvfs 1.54.4) : mesuré** dans une session sans écran
  (`dbus-run-session`, gvfsd, gvfsd-smb et gvfsd-fuse) : la même séquence que sous
  macOS, un mauvais mot de passe aboutissant à `gvfs.ErrAuthFailed`. Non mesuré :
  une vraie session de bureau GNOME. L'essai a trouvé un défaut dans
  go-freedesktop/gvfs v0.1.0 (`/` rendu comme chemin quand gvfs-fuse ne sert pas la
  session), corrigé en v0.1.1 ; `Path()` est alors l'URI `smb://`, que GIO sait
  ouvrir et pas les appels `os`.
- **Windows 11 25H2 (arm64, QEMU) : mesuré en utilisateur standard et en
  administrateur** (v0.1.2 ajoute l'essai en utilisateur standard) : un mauvais
  mot de passe aboutit à `*win32.SMBPortError`, `Z:` est associé, lu et écrit,
  démonté, `Z:` disparaît. Non mesuré : l'apparition de la lettre dans « Ce PC »
  de l'Explorateur (une connexion ssh a sa propre session ; la vérification passe
  par des appels `os`).

Elle est utilisée par [theca](https://github.com/go-theca/theca) `serve --mount`.
