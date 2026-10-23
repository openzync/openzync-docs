<p align="center"><img src="./.github/assets/banner.png" width="100%" alt="OpenZync"></p>

# OpenZync Documentation

Guides, domain references, and ADRs for the OpenZync memory platform.

<p align="center">
  <a href="https://docs.openzync.tech/en/latest/"><img src="https://img.shields.io/badge/docs-docs.openzync.tech-blue" alt="Documentation"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey" alt="License: CC BY 4.0"></a>
  <img src="https://img.shields.io/badge/sphinx-7.0%2B-green" alt="Sphinx 7.0+">
</p>

Source for the documentation published at [docs.openzync.tech](https://docs.openzync.tech/en/latest/). Built with Sphinx, the Furo theme, and MyST for embedded Markdown.

[docs.openzync.tech](https://docs.openzync.tech/en/latest/) · [openzync-core](https://github.com/openzync/openzync-core) · [GitHub org](https://github.com/openzync) · [Contact](mailto:rohnsha0@gmail.com)

## What goes where

| You are writing | Add it here |
|---|---|
| A getting-started or operations guide | `guides/*.rst` |
| An explanation of one backend domain | `domains/*.rst` |
| API reference | `api/*.rst`, generated from core docstrings — do not hand-edit |
| An architecture decision record | `adr/NNN-slug.{md,rst}` — 005 to 008 exist today |

A new file also needs an entry in the appropriate toctree in `index.rst`, or it will not appear in navigation or search. ADRs are mixed format — 005 and 006 are Markdown, 007 and 008 are reStructuredText; MyST renders both.

## Build locally

The build needs a sibling `openzync-core/` checkout: `conf.py` prepends `../openzync-core` to `sys.path` so autodoc can import core modules. This repository does not build standalone.

```bash
pip install -r requirements.txt
sphinx-build -b html . _build/html
```

Use `sphinx-autobuild . _build/html` for a live-reloading server, and open `_build/html/index.html` to read the result.

## Layout

- `guides/` — overview, quickstart, architecture, deployment, rotating keys, contributing
- `domains/` — one page per backend domain, from `core` and `llm` through `mcp_server` and `frontend`
- `api/` — auto-generated reference for routers, models, schemas, services, repositories, and packages
- `adr/` — architecture decision records; the source of truth for cross-cutting backend design

`_build/` is build output and is gitignored.

## Contributing

Corrections and new pages are welcome. Read the [contributing guide](https://docs.openzync.tech/en/latest/guides/contributing.html), open an issue, or send a pull request against `main`.

## License

CC BY 4.0 — reuse with attribution. See [LICENSE](./LICENSE).

If these docs helped you, star the repo.
