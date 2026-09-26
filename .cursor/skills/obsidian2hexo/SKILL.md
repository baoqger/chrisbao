---
name: obsidian2hexo
description: >-
  Converts Obsidian vault notes into Hexo blog posts for Chris Bao's blog.
  Copies embedded images, preserves article text/code/structure, strips About
  the Author. Use when publishing from Obsidian, obsidian2hexo, OrgPro vault,
  or converting a note to source/_posts.
disable-model-invocation: true
---

# Obsidian to Hexo

Convert an Obsidian note in the OrgPro vault into a post under `source/_posts/` for this Hexo blog.

## Content policy

1. **Remove** the trailing **About the Author** section if present: any heading whose text matches `About the Author` (typically `##` or `###`), from that heading through end of file. Do not include content after that heading.
2. **Keep unchanged**: prose, code fences (including language tags such as `c#`), heading levels, lists, and section order. No rewrites, typo fixes, or heading renumbering for “blog style.”
3. **Images**: copy only embed-referenced files from the vault `images/` folder into `source/images/<post-slug>/`, **keep original filenames**, and reference them as `/images/<post-slug>/<filename>` in HTML `<img>` tags. URL-encode spaces in `src` as `%20`.

## Default paths

Override only when the user specifies another vault or note path.

| Role | Path |
|------|------|
| Obsidian vault (OrgPro) | `C:\Users\JBao6\Documents\obsidian-vault\OrgPro` |
| Vault images | `{vault}/images/` |
| Hexo posts | `source/_posts/` (repo root) |
| Post images | `source/images/<post-slug>/` |

Permalinks follow `_config.yml`: `:year/:month/:day/:title/`.

## Inputs

Collect or infer:

- **Note**: full path or title to search under `{vault}`.
- **Post slug**: kebab-case (e.g. `breaking-into-edge-ai-onnx-transformer-autoregressive-loop`). Used for `source/_posts/<post-slug>.md` and `source/images/<post-slug>/`.
- **title**: note filename without `.md`, or first `#` heading if the note uses one.
- **date**: user-provided or today (site timezone).
- **tags**: from first line Obsidian hashtags (`#AI #onnx` → `tags: AI, onnx` in YAML), or user-provided.
- **keywords**: optional; user or derived from tags/topic.

## Workflow checklist

```
- [ ] Locate and read the note
- [ ] Choose post slug (confirm with user if ambiguous)
- [ ] Truncate body at About the Author
- [ ] List image embeds and copy files to source/images/<post-slug>/
- [ ] Write source/_posts/<post-slug>.md with front matter + transformed body
- [ ] npm run build and spot-check public/ HTML
```

### 1. Read and truncate

Read the note file. Find the first markdown heading line matching (case-insensitive):

```regex
^#{1,6}\s+About the Author\s*$
```

Drop that line and everything after it.

### 2. Discover and copy images

Find Obsidian embeds:

```regex
!\[\[([^\]|]+)(?:\|(\d+))?\]\]
```

For each unique basename `filename`:

- Source: `{vault}/images/{filename}`
- Dest: `source/images/<post-slug>/{filename}` (create directory if needed)

Copy **only** referenced files, not the entire vault `images/` folder. If a file is missing, report the path and stop or ask the user.

### 3. Transform body (mechanical)

- If the first line is only Obsidian tags (`#word` tokens), remove it; put equivalent tags in YAML front matter.
- Replace each `![[file.png]]` with:

```html
<img src="/images/<post-slug>/file.png" alt="" >
```

Use percent-encoding in `src` for spaces and special characters (e.g. `Pasted%20image%2020260916153212.png`).

- If the embed has a width pipe `![[file.png|523]]`, add `width="523px"` to the `<img>` tag.

Do not change any other markdown or code.

### 4. Front matter

```yaml
---
title: "Post title matching the note"
date: YYYY-MM-DD HH:mm:ss
tags: Tag1, Tag2
keywords: optional comma-separated keywords
---
```

Match style of existing posts in `source/_posts/`.

### 5. Write output

Write `source/_posts/<post-slug>.md`: front matter, blank line, then transformed body.

### 6. Verify

From repo root:

```bash
npm run build
```

Confirm:

- `public/<year>/<month>/<day>/<post-slug>/index.html` exists
- All `<img src="/images/<post-slug>/...">` paths resolve (images copied under `public/images/<post-slug>/`)
- Generated HTML does not contain “About the Author”

Do not `git commit` or `npm run deploy` unless the user asks.

## Reference conversion

| Artifact | Location |
|----------|----------|
| Source note | `OrgPro/Breaking Into Edge AI - Deconstructing the Transformer Autoregressive Loop with ONNX Small Models.md` |
| Post | `source/_posts/breaking-into-edge-ai-onnx-transformer-autoregressive-loop.md` |
| Images | `source/images/edge-ai-onnx/` |

More snippets: [examples.md](examples.md).
