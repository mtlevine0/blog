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

## Writing a post

```bash
hugo new content posts/YYYY-MM-DD-my-title.md
```

Edit the generated file in `content/posts/`, set `draft: false`, then push:

```bash
git add . && git commit -m "add post: title" && git push
```

GitHub Actions builds and deploys to GitHub Pages automatically (~1 min).

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
