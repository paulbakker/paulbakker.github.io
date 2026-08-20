# paulbakker.io

Paul Bakker's professional website and technical publication, built with [Astro](https://astro.build/) and deployed as a static site.

## Local development

Requires Node.js 22 or newer.

```sh
npm install
npm run dev
```

The local server runs at `http://localhost:4321` by default.

## Writing

Add Markdown or MDX files to `src/content/writing`. Article frontmatter uses this shape:

```yaml
---
title: Article title
description: A concise description for article lists and metadata.
publishedDate: 2026-08-12
updatedDate: 2026-08-13 # optional
tags: [Java, Platforms]
draft: false
image: /images/article-image.jpg # optional
canonicalUrl: https://example.com/original # optional
---
```

## Validation and deployment

```sh
npm run build
```

Pushes to `master` build and deploy through GitHub Actions. The custom domain is configured by `public/CNAME`.
