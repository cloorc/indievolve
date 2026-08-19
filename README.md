# Indievolve

An open knowledge base about **AI + Education** — exploring how AI can solve education inequality.

This repository is an [OKF (Open Knowledge Format) v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog) compatible knowledge bundle: structure is carried by the filesystem, hierarchy, and plain Markdown links. No database, no build step.

## Structure

| Path | Contents |
|------|----------|
| [`index.md`](index.md) | Bundle root — navigation entry point |
| [`vision/`](vision/) | The problem domain: education inequality and the core AI+education thesis |
| [`concepts/`](concepts/) | Domain concepts (adaptive learning, AI tutors, …) |
| [`product/`](product/) | The Indievolve major product |
| [`industry/`](industry/) | Industry entities: organizations, competitors, partners |
| [`research/`](research/) | Research notes, data, and references |
| [`log.md`](log.md) | Content change history |

## OKF conventions

- One `.md` file = one Concept; the Concept ID is its path relative to the bundle root, minus `.md`.
- Every Concept carries frontmatter with a top-level `type:` field.
- Relationships are plain Markdown links; links to not-yet-created Concepts are tolerated.
- `index.md` files are navigation maps; only the bundle root `index.md` carries frontmatter (`okf_version`).
- Content history lives in `log.md` and Git.

## Cross-bundle references (xOKF extension)

The official OKF v0.1 spec defines no inter-bundle addressing, so cross-bundle links (e.g. from this repo into a larger `~/knowledge`-style federation) use the community `xokf://<bundleID>/<conceptID>` scheme. Two things worth knowing before relying on it:

- **It's an open convention, not a registry extension.** `xokf://` requires no package install, no registry lookup, and no network call — it is resolved purely by walking the filesystem up to the nearest `xokf.md` sentinel file (the federation root) and treating everything after `xokf://` as a path relative to it. Any tool that understands this convention can resolve it; nothing needs to be "installed" for a link to be valid.
- **Generic Markdown/OKF tooling treats it as inert text.** Because `xokf://` isn't part of the official spec, standard viewers and the reference `viz` tool simply skip it as an unrecognized scheme rather than erroring — so using it here doesn't break compatibility with plain OKF consumers, it just means only xOKF-aware tools will follow those particular links.
- Within this repository alone (a single, non-federated bundle) you generally won't need `xokf://` — sibling links (`./file.md`) and root-relative links (`/path`) cover everything. It becomes relevant only if this bundle is mounted as a leaf under a larger federated knowledge tree.

### Recommended reader

The [xOKF extension](https://open-vsx.org/extension/cloorc/xokf) (Open VSX) is built for browsing OKF/xOKF bundles like this one — it understands the Concept/frontmatter model and resolves `xokf://` links, and it's designed to be AI-native/AI-friendly, so coding agents and IDE-integrated assistants can navigate this knowledge base as easily as a human reader. It's optional — any plain Markdown viewer or Git host renders this repository fine — but recommended if you want full cross-bundle link resolution.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

This work is licensed under [CC BY-SA 4.0](LICENSE).
