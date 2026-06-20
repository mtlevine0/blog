# blog.mtlevine0.com

Personal blog built with Jekyll, hosted on GitHub Pages at [blog.mtlevine0.com](https://blog.mtlevine0.com).

## Run locally

```bash
bundle install
bundle exec jekyll serve
```

Then visit `http://localhost:4000`.

## Writing a post

Create a file in `_posts/` named `YYYY-MM-DD-slug.md` with front matter:

```markdown
---
layout: post
title: "Post Title"
date: YYYY-MM-DD
tags: [tag1, tag2]
---

Post content here.
```

Push to `main` — GitHub Pages rebuilds the site automatically in ~30 seconds.

## Images

Store images in `assets/images/` and reference them with:

```markdown
![Alt text](/assets/images/filename.jpg)
```

For a media-heavy blog, host images on a CDN and link to them to keep the repo lean.

## Embedding video

Paste YouTube/Vimeo iframe embed code directly into any Markdown post — Jekyll passes raw HTML through unchanged.
