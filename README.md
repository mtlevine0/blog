# blog.mtlevine0.com

Personal blog built with Hugo + PaperMod, hosted on GitHub Pages at [blog.mtlevine0.com](https://blog.mtlevine0.com).

## Prerequisites

```bash
brew install hugo
```

## Local development

```bash
git clone --recurse-submodules https://github.com/mtlevine0/blog.git
cd blog
hugo server -D
```

Then visit `http://localhost:1313`. The server live-reloads on file changes.

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

Enable the pre-commit hook that strips EXIF/GPS metadata from staged images (one-time per clone; requires Python 3 + Pillow):

```bash
git config core.hooksPath .githooks
pip install Pillow
```

## Writing a post

Each post is a [page bundle](https://gohugo.io/content-management/page-bundles/): a folder holding `index.md` and the post's images and videos.

```bash
hugo new content posts/my-title/index.md
```

A multi-part series groups its posts in a subfolder, one bundle per part:

```
content/posts/in-9-spectrum-analyzer/
└── part-1-driver-prototype/
    ├── index.md
    ├── breadboard-prototype-lit.jpg
    └── three-band-demo.mp4
```

To also list a post under other groups in the post tree, add `groups: ["pcb-design"]` to its front matter. Each name is a series folder; a name with no folder creates a new group.

Reference assets by bare filename; the image render hook and `video` shortcode resolve them to the bundle's URL. Older posts still use `/images/...` paths under `static/`, which keep working.

```markdown
![Alt text](photo.jpg "Optional caption")

{{</* video src="demo.mp4" poster="demo-poster.jpg" caption="Optional caption" */>}}
```

`video` also takes `loop="true"` (autoplay, muted, looping, like a GIF) and `class="compact"` (caps the height of portrait clips). For a cover image, set `cover.image` to the bundle filename.

Edit `index.md`, set `draft: false`, then push:

```bash
git add . && git commit -m "add post: title" && git push
```

GitHub Actions builds and deploys to GitHub Pages automatically (~1 min).

### Drafts

A post with `draft: true` is excluded from the build entirely: no page, and it's left out of the home page, post list, tag pages, RSS, and search index.

- **Locally**, `hugo server -D` builds drafts, so they show up at `localhost:1313`.
- **In production**, the deploy workflow runs `hugo --minify` (no `-D`) and `hugo.toml` sets `buildDrafts = false`, so drafts never reach the live site — even if they're committed and pushed.

To publish, set `draft: false` (or remove the line) and push. Hugo also skips posts with a `date` in the future (`buildFuture` defaults to `false`), so a future-dated post won't appear until that date has passed and the site is rebuilt.

## Front matter reference

```yaml
---
title: "Post Title"
date: YYYY-MM-DD
draft: false
tags: ["tag1", "tag2"]
description: "Optional summary shown in post list"
---
```

## Home page and socials

The intro block at the top of the home page comes from `[params.homeInfoParams]` in `hugo.toml`. Keep it: without it, PaperMod renders the newest post as a borderless hero that doesn't look clickable.

`Content` is rendered as Markdown, so don't indent lines inside a `"""` string by 4+ spaces — that turns them into a non-wrapping code block.

Social icons are `[[params.socialIcons]]` entries with a `name` and `url`. Supported names are the icon keys in `themes/PaperMod/layouts/_partials/svg.html` (e.g. `github`, `linkedin`, `bluesky`, `mastodon`, `youtube`, `printables`, `email` with a `mailto:` URL).
