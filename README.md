# docs

[![docs](https://github.com/go-fileshare/docs/actions/workflows/docs.yml/badge.svg)](https://github.com/go-fileshare/docs/actions/workflows/docs.yml)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-0A6E96?style=flat-square)](LICENSE)

**The go-fileshare documentation site**, published at
<https://go-fileshare.github.io/docs/>.

MkDocs Material, versioned with `mike`. The content is derived from
[`go-fileshare/fileshare`](https://github.com/go-fileshare/fileshare) — its
README and `docs/plugins.md` — so a claim here should be traceable to something
the program does, not to something a documentation site wished it did.

## Local

```sh
pip install -r requirements.txt
mkdocs serve
```

CI runs `mkdocs build --strict` on every pull request, so a broken nav entry, a
dead internal link or a missing asset fails the PR rather than reaching the
published site.
