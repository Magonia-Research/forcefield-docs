# forcefield-docs

Publishes the [ForceField](https://github.com/Magonia-Research/ForceField) documentation as a
GitHub Pages site: <https://magonia-research.github.io/forcefield-docs/>

Built by GitHub Pages' native Jekyll with `remote_theme: just-the-docs/just-the-docs@v0.12.0`.
There is no build step and no Actions workflow — pushing to `main` publishes.

## Do not edit the docs here

The documentation is maintained in the main repo, under
[`ForceField/docs/`](https://github.com/Magonia-Research/ForceField/tree/main/docs). This
repository holds a copy of those files with Just the Docs front matter added on top.

**The two copies are currently synced by hand.** Nothing detects drift. A change made here and not
made in `ForceField/docs/` will be silently overwritten by the next sync; a change made there and
not copied here will simply never appear on the site. Edit `ForceField/docs/`, then re-copy.

| Here | Source of truth |
|---|---|
| `index.md` | `ForceField/README.md` (adapted — repo-only sections trimmed) |
| `threat-model.md` | `ForceField/docs/threat-model.md` |
| `hooks.md` | `ForceField/docs/hooks.md` |
| `configuration.md` | `ForceField/docs/configuration.md` |
| `architecture.md` | `ForceField/docs/architecture.md` |
| `logging/index.md` | `ForceField/docs/logging/README.md` (renamed to serve as the directory index) |
| `logging/0*.md` | `ForceField/docs/logging/0*.md` |

Apart from the front-matter block, the file bodies are byte-identical to their sources, with one
exception: `architecture.md`'s `<details>` opening tag carries `markdown="1"`. Kramdown, which
GitHub Pages uses, otherwise emits everything inside a `<details>` block as raw text — code fences
come out as literal backticks. GitHub's own renderer strips the attribute, so the same line works
in both places and the source file can safely adopt it.

The `jekyll-relative-links` plugin resolves the relative `.md` cross-links inside these files, so
the same link text works in both places.

## License

The documentation is part of ForceField and is GPL-3.0-or-later. The Just the Docs theme is MIT.
