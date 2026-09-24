# Chris Bao's Blog (Hexo)

Source for [organicprogrammer.com](https://organicprogrammer.com). Built with [Hexo](https://hexo.io/) 8 and the [NexT](https://theme-next.js.org/) theme.

## Prerequisites

- [Node.js](https://nodejs.org/) 20.19 or newer (required for Hexo 8)
- Git access to push the generated site to `baoqger/baoqger.github.io` (for deploy)

## Setup

```bash
npm install
```

## Run locally (dev server)

Watches files, regenerates on change, and reloads the browser (via `hexo-browsersync`).

```bash
npm run server
```

Open http://localhost:4000/

Same as above, but opens the browser automatically:

```bash
npm run dev
```

Stop the server with `Ctrl+C`. If port 4000 is in use, stop the old Node process or run `npx hexo server -p 4001`.

## Build

Generate static files into `public/`:

```bash
npm run build
```

Clean generated output and cache:

```bash
npm run clean
```

Full rebuild:

```bash
npm run clean && npm run build
```

## Deploy to GitHub Pages

Published site repo: `https://github.com/baoqger/baoqger.github.io.git` (branch `master`), configured in `_config.yml`.

Recommended (clean, generate, then deploy):

```bash
npm run deploy:site
```

Deploy only (uses existing `public/`; run `build` first if needed):

```bash
npm run deploy
```

You need permission to push to `baoqger.github.io`. Use HTTPS with a personal access token or SSH as configured in Git.

## Create content

New blog post:

```bash
npm run new -- "Post title"
# or: hexo new "Post title"
```

New page (e.g. under `source/my-page/index.md`):

```bash
hexo new page my-page
```

## Configuration

| File | Purpose |
|------|---------|
| `_config.yml` | Site URL, Hexo settings, deploy target |
| `_config.next.yml` | NexT theme (menu, scheme, etc.) |
| `source/` | Posts, pages, images |
