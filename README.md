# website

Source for https://aakashsinghal.io, built with [Hugo](https://gohugo.io/) and the [Coder](https://github.com/luizdepra/hugo-coder) theme. GitHub Pages serves the generated site from `docs/`.

## Layout

| Path | What it is |
|---|---|
| `hugo.toml` | Site config |
| `content/posts/<slug>/index.md` | One folder per blog post |
| `static/` | Files copied as-is into the site (images, `CNAME`) |
| `themes/hugo-coder/` | Vendored copy of the Coder theme |
| `docs/` | Generated output. Do not edit by hand |

## Writing a post

1. Create `content/posts/<slug>/index.md` with front matter (`title`, `date`, `description`, `tags`, `draft`).
2. Preview with `hugo server -D` and open http://localhost:1313/.
3. Set `draft: false`, run `hugo`, and commit both the source and the regenerated `docs/`.

A post dated in the future is skipped by the build, so keep the date in the past.

## Building

Last built with Hugo extended v0.168.0 (`brew install hugo`). A different Hugo version can change the hashed CSS filenames, which touches every page in `docs/`.

```bash
hugo            # writes the site to docs/
hugo server -D  # local preview, includes drafts
```
