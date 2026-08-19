# Contributing to Indievolve

Thanks for your interest in contributing! This repository is an OKF v0.1 knowledge bundle — contributions are Markdown files, so no build tooling is required.

## How to contribute

1. Fork the repository and create a branch from `main`.
2. Add or edit Concepts following the conventions below.
3. Update the parent `index.md` to link any new Concept.
4. Append a dated line to `log.md` describing your change.
5. Open a pull request.

## OKF conventions (required)

- **One file = one Concept.** The Concept ID is the file path relative to the repo root, minus `.md`. Choose filenames carefully — renaming changes the ID.
- **Frontmatter:** every Concept must have YAML frontmatter with a top-level `type:` field (e.g. `type: concept`). Never nest it under a `metadata:` key.
- **`index.md` files** are navigation pages (heading + links). Only the bundle root `index.md` may carry frontmatter.
- **Reserved names:** `index.md` (navigation) and `log.md` (change history) — don't repurpose them.
- **Links:** relationships are plain Markdown links with semantics in the surrounding prose. Broken links to planned Concepts are tolerated.
- **Placement:**
  - Abstract/domain knowledge → `concepts/`, `vision/`
  - Industry entities (companies, organizations) → `industry/<category>/<org>/`
  - Product-specific material → `product/`
  - Non-Markdown resources (images, data) → an `assets/` folder beside the Concept that references them

## Style

- Write in English or Chinese, matching the surrounding files.
- Keep Concepts focused; prefer several small files over one large one.
- Use sentence-style headings and standard Markdown.

## License

By contributing, you agree that your contributions are licensed under [CC BY-SA 4.0](LICENSE).
