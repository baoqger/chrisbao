# obsidian2hexo examples

## Obsidian tags → front matter

**Note (first line):**

```markdown
#AI #onnx #edge-ai

## Background
...
```

**Hexo:**

```yaml
---
title: "Breaking Into Edge AI: Deconstructing the Transformer Autoregressive Loop with ONNX Small Models"
date: 2026-09-26 10:00:00
tags: AI, ONNX, Edge AI
keywords: edge AI, ONNX Runtime GenAI, Phi-3, transformer inference, autoregressive loop
---
```

(First hashtag line removed from body; tags normalized in YAML.)

## Image embeds → HTML

Post slug: `breaking-into-edge-ai-onnx-transformer-autoregressive-loop`  
Image folder on disk: `source/images/edge-ai-onnx/` (slug chosen for this article; use the same string in paths below).

| Obsidian | Hexo |
|----------|------|
| `![[Local_Edge_AI_Processing_Loop.png]]` | `<img src="/images/edge-ai-onnx/Local_Edge_AI_Processing_Loop.png" alt="" >` |
| `![[Pasted image 20260916153212.png\|523]]` | `<img src="/images/edge-ai-onnx/Pasted%20image%2020260916153212.png" alt="" width="523px">` |
| `![[Pasted image 20260916155838.png\|700]]` | `<img src="/images/edge-ai-onnx/Pasted%20image%2020260916155838.png" alt="" width="700px">` |

Files copied from:

`C:\Users\JBao6\Documents\obsidian-vault\OrgPro\images\`

to:

`source/images/edge-ai-onnx/`

## About the Author (strip)

**Note (end of file — omit from post):**

```markdown
Build and run it, and you will see a small model answering your questions locally on ordinary CPU hardware. Great, right?

![[Pasted image 20260915135410.png]]

---

## About the Author

I am Chris Bao, a software engineer focused on Azure and AWS AI platforms...
```

**Post ends after the last content image and closing paragraph**; no `---`, no About the Author block.

## What stays verbatim

Code blocks, heading levels, and prose remain as in the note. Example — keep `c#` and content exactly:

```c#
using Microsoft.ML.OnnxRuntimeGenAI;

var path = "pathtoonnxmodel"
var config = new Config(path);
```

Do not “fix” typos or add missing variables unless the user asks for a separate edit pass.
