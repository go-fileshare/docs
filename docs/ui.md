# Web UIs — the portal, the console, the explorer

fileshare is configured with a file and changed live through an admin API. Two
UIs sit on top of that, for **many servers at once**, and one server behind them
holds the login:

| | |
|---|---|
| [**portal**](https://github.com/go-fileshare/portal) | Logs a person in at an OpenID provider, keeps the session in encrypted cookies, holds a token per fileshare server, and passes the admin API (Connect) and WebDAV through. The page never sees a token. |
| [**console**](https://github.com/go-fileshare/console) | The admin console: shares, grants, volumes and server state, on every server the portal knows. |
| [**explorer**](https://github.com/go-fileshare/explorer) | The file explorer: browse, upload, download, rename, copy and delete. |

Both UIs are written once in Go with [go-widgets](https://github.com/go-widgets):
the same code is a page the portal serves and a native window. The design, and
what each phase delivered, is in fileshare's
[docs/ui.md](https://github.com/go-fileshare/fileshare/blob/main/docs/ui.md).

## What the servers need

```hcl
# fileshare.hcl: the admin API, for tokens of the portal's provider
admin {
  listen     = "unix:///run/fileshare/admin.sock"
  state_file = "/var/lib/fileshare/shares.json"
  web {
    listen = "0.0.0.0:8443"
    issuer "https://id.example.org" {
      audience = "https://fs-paris.example.org/"
      groups   = ["fileshare-admins"]
    }
  }
}
# and WebDAV, for the explorer: a bearer token over TLS
oidc {
  issuer   = "https://id.example.org"
  audience = "https://fs-paris.example.org/"
}
serve "webdav" {
  addr = "0.0.0.0:443"
  tls  = true
}
```

```hcl
# portal.hcl
server "paris" {
  url      = "https://fs-paris.example.org:8443"
  issuer   = "https://id.example.org"
  resource = "https://fs-paris.example.org/"
  dav_url  = "https://fs-paris.example.org"
}
ui "console"  { dir = "/usr/share/fileshare-console" }
ui "explorer" { dir = "/usr/share/fileshare-explorer" }
```

The explorer needs **fileshare v0.29.0** or later: earlier versions answer a
`PROPFIND` of the root with an HTML page, and refuse the first piece of an
upload.

## What it promises

- **No token in the page.** Tokens live in the portal's sealed session; a page
  calls `/api/<server>/…` and `/dav/<server>/…` and the portal adds the token
  addressed to that server (RFC 8707 `resource`).
- **A file on a share is served inert.** It is whatever somebody put there,
  served from the portal's origin, so every `/dav` answer carries
  `Content-Security-Policy: sandbox`, `nosniff`, and a download is an
  attachment.
- **Nothing is replaced without asking.** A rename, a copy or an upload onto a
  name that is taken is refused (`Overwrite: F`); a delete takes two presses.
- **One refresh at a time.** A provider that rotates refresh tokens ends the
  login when one is spent twice; the UIs take one browser lock across all their
  tabs, the console's and the explorer's alike.

Each UI is tested in headless Chrome behind the real portal binary, in front of
real fileshare binaries, every step checked on the server's disk.
