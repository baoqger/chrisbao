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

Watches files and regenerates on change (`hexo-server`). Refresh the browser manually after edits.

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

Published site repo: `https://github.com/baoqger/baoqger.github.io.git` (branch `master`), configured in `_config.yml`. The live site uses custom domain `organicprogrammer.com` via [`source/CNAME`](source/CNAME).

### Automated (GitHub Actions)

Workflow: [`.github/workflows/deploy-pages.yml`](.github/workflows/deploy-pages.yml)

- **On push** to `master` when paths under `source/`, site or theme config, `package.json` / lockfile, or the workflow file change.
- **Manual:** GitHub → **chrisbao** → **Actions** → **Deploy to GitHub Pages** → **Run workflow**.

The job runs `npm ci`, `hexo clean`, `hexo generate`, then publishes `public/` to `baoqger/baoqger.github.io` with [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages).

**One-time setup:**

1. Generate a dedicated ed25519 SSH key pair (no passphrase).
2. **`baoqger/baoqger.github.io`** → Settings → Deploy keys → add the **public** key → enable **Allow write access**.
3. **`baoqger/chrisbao`** → Settings → Secrets and variables → Actions → secret **`DEPLOY_KEY`** = full **private** key.
4. Confirm Pages on `baoqger.github.io` still serves branch **`master`** / root with custom domain **organicprogrammer.com**.

### Manual (local)

Recommended (clean, generate, then deploy):

```bash
npm run deploy:site
```

Deploy only (uses existing `public/`; run `build` first if needed):

```bash
npm run deploy
```

You need permission to push to `baoqger.github.io`. Use HTTPS with a personal access token or SSH as configured in Git. Local deploy remains useful if Actions is unavailable.

## Create content

New blog post:

```bash
npm run new -- "Post title"
# or: hexo new "Post title"
```

### Publishing from Obsidian

To convert an OrgPro vault note into a Hexo post (images, structure, and code preserved; About the Author removed), use the **obsidian2hexo** Cursor skill: [.cursor/skills/obsidian2hexo/SKILL.md](.cursor/skills/obsidian2hexo/SKILL.md).

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
