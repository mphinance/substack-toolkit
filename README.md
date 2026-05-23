# substack-toolkit

> Full read-and-write Substack toolkit that works around the private-API gotchas Substack will not document for you.

`substack-toolkit` is a Claude Skill and a small standalone Python library for working with Substack from the outside. Post drafts from Markdown or a Python builder. Read your reader feed, the published archive of any publication, individual posts, profiles, and other users' Notes. Publish Substack Notes. Like and comment with explicit, side-effect-aware methods. Single file, two deps, MIT.

## What it does

**Write:**

- Posts **native ProseMirror drafts** using only the editor-validated node types, so drafts actually open cleanly.
- Uploads images via the **correct JSON base64-data-URI form** (multipart 400s).
- Converts a useful subset of **Markdown** — headings, paragraphs, bold, italic, links, bullet/ordered lists, blockquotes (with inline marks preserved), images.
- Handles draft **create / update / list / delete** with the `draft_bylines: null` workaround built in.
- Sets **cover images** for inline display and the social card.

**Read:**

- Walks the **authenticated user's reader feed** with built-in 429 retry and cursor pagination.
- Lists the **published archive** of a publication, with pagination and an iterator helper.
- Fetches **a single post** with full `body_html`, cover image, audience, dates, comment count.
- Looks up **any user's public profile** by handle (id, name, bio, subscriber count).
- Paginates **any user's published Notes** with engagement metrics (likes, comments, restacks).
- Reads **publication metadata** — name, hero text, custom domain, bylines.
- **Exports subscribers** to CSV (owner only; direct API plus a documented Playwright fallback for gated accounts).

**Publish Notes:**

- Publishes plain or rich Substack Notes (`client.publish_note(text)` or `client.publish_note(doc)`).
- Publishes Notes with a **link attachment** (two-step attachment-then-publish flow that produces a rich link card).

**Engage** (library-only, no CLI on purpose):

- Like a post or a Note.
- Comment on a post.

What it doesn't do yet: restack a post, search archive, scheduled publishing, comments-read, following iteration. Those are on the roadmap.

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

## Read the archive

```python
from scripts.substack_draft import Client

client = Client(pub="yourname.substack.com")

# Publication metadata
pub = client.publication()
print(pub["name"], pub.get("custom_domain"))

# Listings
for post in client.archive(sort="new", limit=10):
    print(post["id"], post["title"], post["audience"], post["post_date"])

# Walk the whole archive across pages
for post in client.iter_archive(page_size=25):
    ...

# Full post body
post = client.get_post(12345678)
with open("post.html", "w") as f:
    f.write(post["body_html"])

# Subscribers (owner only)
client.subscribers_csv(out_path="subs.csv")
```

From the CLI:

```bash
python scripts/substack_draft.py publication
python scripts/substack_draft.py archive --limit 25
python scripts/substack_draft.py get 12345678 --body | pandoc -f html -o post.md
python scripts/substack_draft.py feed --limit 20
python scripts/substack_draft.py whois @someone
python scripts/substack_draft.py notes --user @someone --limit 10
python scripts/substack_draft.py subscribers --out subs.csv
```

## Publish Notes and engage

The Notes / like / comment methods are library-only — no CLI on purpose. Side-effects on your account should be explicit code, not casual shell commands.

```python
from scripts.substack_draft import Client, Doc

client = Client(pub="yourname.substack.com")

# Plain Note
client.publish_note("Working on a draft about why everyone gets the VIX wrong.")

# Note with link attachment (rich card preview)
client.publish_note(
    "Just shipped this — MIT, single file:",
    attachment_url="https://github.com/mphinance/substack-toolkit",
)

# Rich Note built with the Doc API
note = (Doc()
        .p("Three things I learned this week:")
        .p("1. ", Doc.strong("VIX <17"), " is not always a buy signal.")
        .p("2. ", Doc.em("Patience"), " is a position size.")
        .p("3. The most expensive trade is the one you take to feel something."))
client.publish_note(note)

# Engagement — call explicitly, no auto-loops
client.like_post(198832063)
client.like_note(2814629384)
client.comment_on_post(198832063, "This is the post I needed today.")

# Walk the feed and do something with it
for item in client.iter_feed(max_items=50):
    if item["type"] == "post":
        full = client.get_post(item["post"]["id"])
        # ... summarize, save, like, whatever ...
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

- [x] Read posts and archive from a publication (v0.2)
- [x] Fetch publication metadata (v0.2)
- [x] Export subscribers as CSV (v0.2)
- [x] Reader feed for the authenticated user (v0.3)
- [x] Profile lookup by handle (v0.3)
- [x] Any user's published Notes with engagement metrics (v0.3)
- [x] Publish Substack Notes (plain, with link attachment, or rich Doc) (v0.3)
- [x] Like a post, like a Note, comment on a post (v0.3)
- [x] `safe_text()` helper for accounts where the editor still chokes on Unicode (v0.3)
- [ ] Restack a post programmatically
- [ ] Search archive (titles, full-text)
- [ ] Scheduled-publish helper
- [ ] Read comments on a post
- [ ] Following / followers iteration

If you want any of these enough to PR them, the contributing bar is low.

## Subscriber export — Playwright fallback

`client.subscribers_csv()` calls `/api/v1/subscribers/export` directly. For most owner accounts that returns CSV. Some accounts gate it behind dashboard JavaScript; you'll see a non-CSV content-type and a `RuntimeError` from the method. In that case, this snippet (requires `playwright` + a browser) injects your SID and downloads from the dashboard:

```python
import os, asyncio
from urllib.parse import unquote
from playwright.async_api import async_playwright

PUB = "yourname.substack.com"
SID = unquote(os.environ["SUBSTACK_SID"])

async def export():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        ctx = await browser.new_context()
        await ctx.add_cookies([{"name": "substack.sid", "value": SID,
                                 "domain": ".substack.com", "path": "/",
                                 "httpOnly": True, "secure": True}])
        page = await ctx.new_page()
        r = await page.goto(f"https://{PUB}/api/v1/subscribers/export")
        body = await r.body()
        open("subs.csv", "wb").write(body)
        await browser.close()

asyncio.run(export())
```

If the page redirects to login, your SID has expired — refresh it from DevTools and try again.

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
| `user/self` returns 403 | Some accounts always 403 on this endpoint | Use `client.user_id` (resolves via drafts/archive bylines) |
| `get_post` returns wrapper | Top-level keys are `post`, `publication`, ... | This library unwraps to the inner post dict |
| `subscribers_csv` returns HTML | Account gated behind dashboard JS | Documented Playwright fallback above |
| `notes_for_user` with `?types[]=comment` | Returns zero items even for prolific Note authors | Drop the filter — every item already has `type="comment"`. This library omits it automatically |
| Reader feed 429 mid-pagination | Throttled | `iter_feed()` retries once with a 10s sleep; tune `inter_page_sleep` for long backfills |
| Note publish silently no-ops | `bodyJson.attrs.schemaVersion` missing | `publish_note()` injects `{"schemaVersion": "v1"}` when you pass a string or `Doc` |

If you hit one not on this list, open an issue with the request body, the response, and the draft ID.

## Acknowledgements

Built by [Michael Hanko](https://mphinance.substack.com) with [Claude Code](https://www.anthropic.com/claude-code). The library exists because the Substack editor and a long-form draft pipeline kept fighting each other and the editor was winning.
