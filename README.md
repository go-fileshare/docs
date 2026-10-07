# docs

[![docs](https://github.com/go-fileshare/docs/actions/workflows/docs.yml/badge.svg)](https://github.com/go-fileshare/docs/actions/workflows/docs.yml)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-0A6E96?style=flat-square)](LICENSE)

**The go-fileshare documentation site**, published at
<https://go-fileshare.github.io/docs/>.

A [Hugo](https://gohugo.io/) site with the [tannevaled/hextra](https://github.com/tannevaled/hextra)
fork of the [Hextra](https://github.com/imfing/hextra) theme, laid out like the
[Claimward documentation](https://github.com/claimward/docs), with
go-fileshare's branding. The content is derived from
[`go-fileshare/fileshare`](https://github.com/go-fileshare/fileshare), and
checked against its code, so a claim here should be traceable to something the
program does, not to something a documentation site wished it did.

## Versions

A version of the documentation is **the minor version of fileshare it
describes**:

| Source | Published under |
| --- | --- |
| branch `main`, with `params.fileshare.version: v0.22.2` in `hugo.yaml` | `https://go-fileshare.github.io/docs/0.22/` |
| the newest version | also `https://go-fileshare.github.io/docs/latest/` |

`https://go-fileshare.github.io/docs/` redirects to `latest/`. Each push to
`main` replaces the directory of the version it describes, and nothing else.

To document a new fileshare release, change `params.fileshare.version` with the
pages. A patch release (`v0.22.3`) updates `0.22`; a minor release (`v0.23.0`)
publishes `0.23` beside it, `latest` moves to it, and `0.22` stays as it was last
published.

The version selector in the navbar lists every version from
`docs/versions.json` and opens the same page in the version chosen; if it is
not there, the same page in English, else that version's home.

Versions 0.4 to 0.21 were built with MkDocs and mike before this site moved to
Hugo. Their directories on `gh-pages` are kept as they were published, and they
read the same `versions.json` (mike's format).

## Languages

English is the default and sits at the root of each version
(`/docs/<version>/protocols/sftp/`). The site is laid out for other languages,
which will be under their code (`/docs/<version>/fr/protocols/sftp/`); only
English is declared so far. The language switch at the foot of the sidebar
opens the same page in the other language, in the same version.

To add a language `<lang>`:

1. `content/<lang>/`: a translation of every page, with the same file names
   (that is how a page and its translation are paired). A translated heading
   keeps the English one's anchor as an explicit id
   (`## Certificats destinés à cet hôte : la délégation de domaine {#certificates-meant-for-this-host-the-domain-grant}`),
   so the relrefs resolve in every language and links into a page keep their
   fragment.
2. `i18n/<lang>.yaml`: the site's own strings (copyright, version).
3. A `languages.<lang>` entry in `hugo.yaml`: label, `contentDir`, weight,
   `params.flag`, the translated `params.description`, and a `dateFormat`.
4. Its flag in `static/images/flags/`, from
   [lipis/flag-icons](https://github.com/lipis/flag-icons) (4x3, MIT).

Declare a language only with its content: a declared language without pages
is an empty entry in the switch.

## Layout

| Path | What |
| --- | --- |
| `content/en/` | the pages; `_index.md` is a section's own page, `weight` orders the sidebar |
| `i18n/<lang>.yaml` | the site's strings, beside the theme's |
| `static/images/flags/` | the language switch's flags (lipis/flag-icons, MIT) |
| `hugo.yaml` | `baseURL`, the fileshare version described, the languages, the navbar (version › theme › search › GitHub), the brand mounts |
| `layouts/_partials/custom/version-select.html` | the version selector (reads `versions.json`) |
| `layouts/_partials/favicons.html` | the brand's favicons |
| `assets/css/custom.css` | the brand cyan as Hextra's primary colour, and the table headers |
| `themes/hextra` | submodule: [tannevaled/hextra](https://github.com/tannevaled/hextra), pinned to a release tag (upstream Hextra plus opt-in features: partial menu items, related sites, page subtitle and history) |
| `branding` | submodule: [go-fileshare/brand](https://github.com/go-fileshare/brand) (the mark, its PNGs and ICO) |
| `scripts/publish-version.sh` | puts one build into a `gh-pages` checkout and rewrites `versions.json`, `latest` and the root redirect |

## Writing a page

Each page starts with a title, a one-sentence description, and tags:

```yaml
---
title: "SFTP — keys, or certificates"
linkTitle: "SFTP: keys, or certificates"
weight: 30
description: "…"
tags: [protocols, sftp, ssh]
---
```

Link to other pages with `{{< relref "/protocols/sftp.md#a-certificate-pinned-to-an-address" >}}`:
a broken reference, or an anchor that does not exist, fails the build. A
heading's anchor is a URL other sites link to: when a heading's words change,
keep the old anchor with an explicit `{#id}`. A warning or a note is a
`{{< callout type="warning" >}}` (`error` for what MkDocs called danger, `info`
for a note), its title in bold on the first line.

The page history at the foot of each page (created, modified, by whom) comes
from git: `themes/hextra/scripts/page-history.sh` writes
`data/pagehistory.json` before the build.

## Working locally

```sh
git clone --recurse-submodules https://github.com/go-fileshare/docs.git
cd docs
mkdir -p data && sh themes/hextra/scripts/page-history.sh > data/pagehistory.json
hugo server --baseURL http://localhost:1313/docs/0.22/
```

then open <http://localhost:1313/docs/0.22/>. Without a `versions.json` the
selector shows only the current version. Hugo 0.146 or later (CI uses the
version pinned in the workflow); no Python, no Node.

To upgrade the theme: `git -C themes/hextra checkout <tag>` (a tag of the
fork), commit the submodule, and run the production build of the workflow:
the fork's features are opt-in and its own CI does not build them, so this
site's `--panicOnWarning` build is their test.

## Publication

`.github/workflows/docs.yml`:

| Job | When | What |
| --- | --- | --- |
| `build` | pull requests, `main` | builds with `--baseURL …/docs/<version>/`, checks internal links and anchors (lychee, offline), uploads the site |
| `links` | pull requests, `main` | fetches every external URL the pages name |
| `deploy` | `main` | `scripts/publish-version.sh` into `gh-pages`, then a commit and a push; one deploy at a time |

GitHub Pages serves the `gh-pages` branch.

## Licence

BSD-3-Clause.
