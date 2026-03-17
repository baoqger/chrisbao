# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hexo static site generator blog (Chris Bao's Blog) that deploys to GitHub Pages. The blog uses the Landscape theme and includes various plugins for content generation, math rendering, and deployment.

## Common Commands

### Development
- `npm run server` or `hexo s` - Start local development server (default: http://localhost:4000)
- `npm run build` or `hexo g` - Generate static files to public/ directory
- `npm run clean` or `hexo clean` - Clean generated files (public/ and db.json)
- `npm run new <title>` or `hexo new <title>` - Create a new blog post

### Deployment
- `npm run deploy` or `hexo d` - Deploy to GitHub Pages
- Full deployment workflow: `hexo clean && hexo g && hexo d`

### Content Creation
- Create new blog post: `hexo new <blog_title>` (creates in source/_posts/)
- Create new page: `hexo new page <page_name>` (creates in source/)
- Drafts can be stored in draft/ directory and published later

## Architecture

### Directory Structure
- `source/_posts/` - Published blog posts (markdown files with front-matter)
- `draft/` - Draft blog posts (not published unless configured otherwise)
- `source/` - Additional site content (pages, images, static files)
- `themes/landscape/` - Theme templates and assets
- `scaffolds/` - Templates for new content (post.md, page.md, draft.md)
- `public/` - Generated static site (do not edit manually)
- `node_modules/hexo-*/` - Hexo core and plugins

### Configuration Files
- `_config.yml` - Main Hexo configuration (site settings, URL structure, plugins)
- `themes/landscape/_config.yml` - Theme-specific settings (navigation, widgets, sidebar)
- `package.json` - Project dependencies and npm scripts

### Theme Architecture (Landscape)
The theme uses EJS templating with a modular structure:
- `layout/layout.ejs` - Main layout wrapper
- `layout/index.ejs` - Homepage template
- `layout/post.ejs` - Blog post template
- `layout/page.ejs` - Static page template
- `layout/_partial/` - Reusable partial templates (header, footer, sidebar, etc.)
- `layout/_widget/` - Sidebar widgets (archive, tag cloud, recent posts, etc.)

### Content Front-Matter
Blog posts use YAML front-matter with fields:
- `title` - Post title
- `date` - Publication date
- `tags` - Comma-separated tags
- `keywords` - SEO keywords (optional)

Example:
```yaml
---
title: My Post Title
date: 2024-03-17 10:00:00
tags: programming, tutorial
keywords: hexo blog tutorial
---
```

### Key Plugins
- `hexo-deployer-git` - Git deployment (configured for GitHub Pages)
- `hexo-generator-archive/category/tag/sitemap` - Content generators
- `hexo-generator-feed` - RSS/Atom feed generation
- `hexo-math` - MathJax/LaTeX math rendering
- `hexo-renderer-marked` - Markdown rendering
- `hexo-server` - Development server
- `prismjs` - Syntax highlighting

### Asset Management
- `post_asset_folder: true` - Each post can have a companion folder for assets
- Images in source/_posts/ are referenced relative to the post
- Theme assets are in themes/landscape/source/

### Deployment Configuration
- Target repository: https://github.com/baoqger/baoqger.github.io.git
- Branch: master
- Generated from db.json database file

## Writing Style and Tone

When creating new blog content, maintain the established writing style and tone that characterizes this blog:

### Educational and Learning-Focused Approach
- Write from a learning perspective: "In this article, I will share what I learned..."
- Frame topics as knowledge sharing rather than authoritative presentation
- Use phrases like "After reading this post, you can understand..." to set learning objectives

### Technical Depth with Systematic Structure
Follow a consistent structure for technical articles:
- **Background/Introduction** - Set context and motivation for the topic
- **Conceptual Foundation** - Explain underlying principles and theory
- **Implementation Details** - Provide step-by-step technical breakdown
- **Code Examples** - Include substantial, well-commented code samples
- **Summary** - Recapitulate key learning points

### Interactive and Conversational Style
Engage directly with readers using mentorship-style phrases:
- "Let's see in the following section"
- "As we mentioned above"
- "Next step, let's examine"
- "Great, right?" and "Smart design, right?"
- Create a conversational learning atmosphere that guides readers through complex material

### Code-Heavy with Thorough Explanations
- Provide complete working implementations in multiple languages (C, Golang, JavaScript, C#)
- Include detailed line-by-line or section-by-section code walkthroughs
- For low-level topics, provide assembly-level explanations
- Add mathematical/algorithmic breakdowns where relevant

### Practical and Hands-On Orientation
- Emphasize real-world implementation and applications
- Reference actual GitHub repositories and projects
- Include hands-on project demonstrations
- Cover tool usage tutorials (WinDbg, tcpdump, etc.)
- Share production troubleshooting scenarios and lessons learned

### Progressive Learning Philosophy
Build articles progressively from foundations to advanced implementations:
- Start with basic concepts and definitions
- Progressively add complexity
- Connect concepts to real-world applications
- Reference future learning opportunities

### Cross-Referencing and Connected Learning
Create a learning ecosystem by:
- Referencing previous articles: "As I mentioned in my previous article..."
- Teasing future content: "I will share about this in coming articles"
- Creating interconnected knowledge paths across the blog

### Professional yet Accessible Balance
Maintain this balance in all technical writing:
- **Approachable** through conversational, mentorship style
- **Thorough** without being academic or dry
- **Practical** with real applications and implementations
- **Encouraging** to readers' learning journey

### Unique Voice Elements
- **Personal authenticity**: Share own learning experiences ("I came across this question," "This confuses me for a while")
- **Enthusiastic technical discovery**: Use phrases like "Great, right?" and "That's an interesting point"
- **Mentoring approach**: Guide readers through complex concepts step-by-step
- **Humble expertise**: Admit when topics are complex or when you don't know everything
