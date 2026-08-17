# IndieVolve

An open knowledge base about **AI + Education** — exploring how AI can solve education inequality.

This repository is an [OKF (Open Knowledge Format) v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog) compatible knowledge bundle: structure is carried by the filesystem, hierarchy, and plain Markdown links. No database, no build step.

## Structure

| Path | Contents |
|------|----------|
| [`index.md`](index.md) | Bundle root — navigation entry point |
| [`vision/`](vision/) | The problem domain: education inequality and the core AI+education thesis |
| [`concepts/`](concepts/) | Domain concepts (adaptive learning, AI tutors, …) |
| [`product/`](product/) | The IndieVolve major product |
| [`industry/`](industry/) | Industry entities: organizations, competitors, partners |
| [`research/`](research/) | Research notes, data, and references |
| [`log.md`](log.md) | Content change history |

## OKF conventions

- One `.md` file = one Concept; the Concept ID is its path relative to the bundle root, minus `.md`.
- Every Concept carries frontmatter with a top-level `type:` field.
- Relationships are plain Markdown links; links to not-yet-created Concepts are tolerated.
- `index.md` files are navigation maps; only the bundle root `index.md` carries frontmatter (`okf_version`).
- Content history lives in `log.md` and Git.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

This work is licensed under [CC BY-SA 4.0](LICENSE).
