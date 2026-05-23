# substack-toolkit

> Programmatic Substack publishing that works around the private-API gotchas Substack will not document for you.

`substack-toolkit` is a Claude Skill and a small standalone Python library for posting rich drafts to Substack from Markdown or a Python builder API. It exists because Substack's web editor mangles anything you paste in from Notion, Obsidian, or Google Docs, and the official write path is undocumented.

## What it does

- Posts **native ProseMirror drafts** using only the editor-validated node types, so drafts actually open cleanly.
- Uploads images via the **correct JSON base64-data-URI form** (multipart 400s).
- Converts a useful subset of **Markdown** — headings, paragraphs, bold, italic, links, bullet/ordered lists, blockquotes (with inline marks preserved), images.
- Handles draft **create / update / list / delete** with the `draft_bylines: null` workaround built in.
- Sets **cover images** for inline display and the social card.

What it doesn't do yet: read posts, search archive, summarize posts, publish Notes, restack. Those are on the roadmap.

## Install

```bash
git clone https://github.com/mphinance/substack-toolkit
cd substack-toolkit
pip install requests markdown-it-py
```

Grab your Substack session cookie (DevTools → Application → Cookies → `substack.sid` on substack.com), then:

```bash
export SUBSTACK_SID="s%3A..."
export SUBSTACK_PUB="yourname.substack.com"
python scripts/substack_draft.py auth
```

## Use it from the CLI

Write your post as a Markdown file with frontmatter:

```markdown
---
title: Your Title
subtitle: Your subtitle
hero: ./hero.png
---

Body content in **Markdown**.

- Lists
- Work
- Fine

![Inline images](./diagram.png) get auto-uploaded.
```

Then:

```bash
python scripts/substack_draft.py post --body your-post.md
```

You get back a draft URL. Review in the Substack dashboard, hit Publish.

## Use it from Python

```python
from scripts.substack_draft import Client, Doc

client = Client(pub="yourname.substack.com")
hero_url = client.upload_image("hero.png")

doc = Doc()
doc.h(1, "My Essay")
doc.p("With ", Doc.strong("bold"), " and ", Doc.em("italic"), ".")
doc.image(hero_url, caption="Inline hero")

draft_id = client.create_draft(
    title="My Essay",
    subtitle="A subtitle",
    doc=doc,
    cover_image=hero_url,
)
print(client.edit_url(draft_id))
```

## Use it as a Claude Skill

This repo is structured as a [Claude Skill](https://github.com/anthropics/skills). Copy the repo into your Claude Code skills directory (or install via the awesome-claude-skills index) and Claude will trigger on prompts like *"post this markdown to Substack as a draft"*.

The skill definition is in [SKILL.md](./SKILL.md). The body explains exactly when Claude should and should not trigger it, what to run, and how to interpret the output.

## Why this exists

Substack's editor is built on ProseMirror with a schema-strict validator. Three common dead ends:

1. **The `rawHtml` node.** The most obvious shortcut. The API accepts it. The editor crashes opening any draft that contains it with the message *"Something has gone wrong. Please refresh the page and try again."* No useful error.
2. **Multipart image upload.** The first thing every developer tries. The endpoint rejects it with a 400 and a generic *"Invalid value"* message. The endpoint actually wants a JSON body with a base64 data URI.
3. **The `draft_bylines: null` race.** A `GET` on an unpublished draft returns bylines as null. A `PUT` to update it requires non-null bylines. The fix is to inject them before sending the update.

Each of these took an afternoon to discover the first time. This repo packages the workarounds so nobody else has to.

## Roadmap

- [ ] Read posts and archive from a publication
- [ ] Search archive
- [ ] Summarize a post
- [ ] Post Substack Notes (with link attachments)
- [ ] Restack a post programmatically
- [ ] Scheduled-publish helper

If you want any of these enough to PR them, the contributing bar is low.

## License

MIT. Use it, fork it, sell a wrapper for it, whatever earns its keep.

## Hall of Shame

For completeness, the API gotchas that this library handles for you:

| Trap | Symptom | Built-in fix |
|---|---|---|
| `rawHtml` node | API 200, editor crashes opening the draft | Use validated node types only |
| `body_html` field | Empty draft created | Use `draft_body` with ProseMirror JSON |
| Multipart image upload | 400 "Invalid value" on `image` field | JSON `{"image": "data:...;base64,..."}` |
| `draft_bylines: null` on PUT | 400 on update of unpublished draft | Inject `[{"id": user_id, "is_guest": False}]` before PUT |
| Tables / code blocks | Render as collapsed text or nothing | Out of scope — render to image and embed |

If you hit one not on this list, open an issue with the request body, the response, and the draft ID.

## Acknowledgements

Built by [Michael Hanko](https://mphinance.substack.com) with [Claude Code](https://www.anthropic.com/claude-code). The library exists because the Substack editor and a long-form draft pipeline kept fighting each other and the editor was winning.
