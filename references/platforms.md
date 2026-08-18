# Platform Roster

Every platform this campaign has published on, with the one thing that matters most
for each: how you get text into its editor, whether its outbound links help SEO
(dofollow) or not (nofollow), and the specific trap it throws.

## Loading the browser tools

If the Claude-in-Chrome tools are deferred, load them in a single ToolSearch call
before touching any site:

```
ToolSearch "select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__find,mcp__claude-in-chrome__form_input"
```

Always call `tabs_context_mcp` first, and open a **new** tab per site rather than
reusing an existing one unless the user asks.

## Quick-reference table

| Platform | New-post URL | Editor architecture | Insertion technique | Links | Notable gotcha |
|---|---|---|---|---|---|
| Medium | medium.com/new-story (or /@handle → write) | contenteditable (classic) | synthetic paste | nofollow | tag selection is a separate step |
| Dev.to | dev.to/new | Preact-controlled `<textarea>` (markdown) | native value setter | nofollow | discard any pre-existing draft first |
| Hashnode | your blog → new article | React-controlled `<textarea>` (markdown) | native value setter | nofollow | blog must be created/activated; public URL can 404 until visible |
| Telegraph | telegra.ph | CodeMirror / Quill | `CodeMirror.setValue()` + `.save()` | nofollow | no login needed; save the edit URL or it's unrecoverable |
| Bear Blog | bearblog.dev dashboard → new post | plain `<textarea>` (markdown) | native value setter | **dofollow** | high-value link; confirm blog is public |
| Blogger | blogger.com → new post | CodeMirror (HTML view) + Compose iframe | `CodeMirror.setValue()` in HTML view | **dofollow** | toggling HTML/Compose can reset body; check privacy + "visible to search engines" |
| Mataroa | mataroa.blog/new/post | plain `<textarea>` (markdown) | native value setter | **dofollow** | clean, reliable; good dofollow target |
| Write.as | write.as/new | `<textarea id="writer">` | native value setter | nofollow | anonymous post; save the edit token |
| Rentry | rentry.co | CodeMirror (markdown) | `CodeMirror.setValue()` + `.save()` | nofollow | first submit fails if editor still "empty" — set via CodeMirror; save edit code |
| Tumblr | tumblr.com/new/text | block editor (contenteditable) | synthetic paste | nofollow | block editor; paste into the focused text block |
| IndieHackers | indiehackers.com/product/<slug> (timeline) or a group | TipTap / ProseMirror | synthetic paste for body; native value setter for title | nofollow | general posting is privilege-gated; the product page timeline works. Title is a `textarea.newlines-disabled`; body is `.tiptap.ProseMirror`. Publish = blue check on the draft card |
| HackerNoon | hackernoon.com/new | ProseMirror | synthetic paste | nofollow | editorial review queue; must flag original + attest disclosure of affiliation |
| Quora | quora.com (write/post) | contenteditable | synthetic paste | nofollow | best as an answer or Space post; disclose affiliation |
| Pastebin | pastebin.com | plain `<textarea>` | native value setter | nofollow | plain text only — no markdown rendering; paste the .txt version |
| JustPaste | justpaste.it | contenteditable | synthetic paste | nofollow | **CAPTCHA at publish** — user must solve it |
| Google Docs | docs.google.com/document | Google's canvas editor | see note below | indexable once published to web | native "publish to web" confirm dialog blocks automation — user clicks OK |
| Notion | notion.so new page | Notion block editor | see note below | indexable once shared | must turn on Share → "Publish"/"Discoverable on web" or it's never crawled |
| HackMD | hackmd.io new note | CodeMirror (markdown) | `CodeMirror.setValue()` + `.save()` | indexable when public | set note visibility to public/published |

"Links" = the status of the outbound links *to the company* in the published post.
Dofollow links pass SEO authority; nofollow ones mostly build presence/referral
traffic and a natural-looking link profile. Prioritise getting your strongest,
most keyword-relevant angles onto the **dofollow** platforms (Bear Blog, Blogger,
Mataroa, and the company's own domains).

## Per-platform notes

### Medium
Classic contenteditable editor. Paste the HTML version so headings, bold, and links
render. Tags/topics are chosen in a separate publish dialog — pick 3–5 relevant
ones. Links are nofollow but Medium has strong domain authority and readership.

### Dev.to
Preact app; the markdown body is a controlled `<textarea>`, so use the native value
setter (React/Preact ignore plain `.value =`). If a stale draft (e.g. a template)
is open, discard it before inserting. Front-matter tags go at the top.

### Hashnode
Controlled `<textarea>` markdown editor. The account needs a blog created and
activated first. Known issue: the public post URL can 404 briefly or until the blog
is fully visible — verify the live URL after publishing and check blog visibility
settings if it 404s.

### Telegraph (telegra.ph)
No account required — which also means the *only* way back to edit is the URL it
gives you, so capture it. CodeMirror-backed; use `setValue()` then `.save()`.

### Bear Blog
Dofollow — one of the most valuable targets. Plain markdown `<textarea>`. After
publishing, confirm the post and blog are public (not draft/unlisted).

### Blogger
Two-mode editor. Insert via the **HTML view** CodeMirror, but beware: switching
between HTML and Compose can wipe the body. A reliable pattern is to set the content
in the Compose iframe body directly and make a tiny no-op edit (type a space,
delete it) to trigger Blogger's sync. Check the blog's privacy is Public and
"Visible to search engines" is on (the private-blog banner is sometimes stale —
verify the actual settings).

### Mataroa
Dofollow, minimal, and reliable. Plain markdown `<textarea>`, native value setter.

### Write.as
Anonymous-friendly; body is `<textarea id="writer">`. Save the edit token it
returns. Nofollow.

### Rentry
CodeMirror markdown. The first submit can fail with "This field is required" if the
editor looks empty to the form — set the content via `CodeMirror.setValue()` so the
backing field updates. Save the edit code it issues.

### Tumblr
`/new/text` opens the block editor. Focus the body text block and dispatch a
synthetic paste of the HTML. Reblog-friendly, casual audience — match the angle.

### IndieHackers
General post creation is often privilege-gated ("you can't create posts yet") for
new accounts. The **product page timeline** (indiehackers.com/product/<slug>) lets
you post updates directly. The title field is a single-line `textarea` with class
`newlines-disabled` — set it with the native value setter. The body is a separate
`.tiptap.ProseMirror` contenteditable — synthetic paste into it. Publish by clicking
the blue checkmark on the "UNPUBLISHED DRAFT" card (a red X discards).

### HackerNoon
ProseMirror editor, then an editorial review queue — the post won't be instantly
live. At submit it asks whether the piece is original (yes) and requires a
disclosure/attestation about rights and any vested interest. Add a one-line
affiliation disclosure to the article body first, then attest truthfully.

### Quora
Works best as a genuinely useful answer to a relevant question, or a post in a
Space, rather than a bare ad. Contenteditable; synthetic paste. Disclose the
affiliation. Nofollow.

### Pastebin
Plain `<textarea>`, no markdown rendering — paste a plain-text version of the
article with URLs written out in full so they're clickable/visible. Nofollow.

### JustPaste
Contenteditable; synthetic paste works for the body, but publishing triggers a
**CAPTCHA**. Do not attempt to solve it — pause and ask the user to complete it,
then confirm the live URL (e.g. justpaste.it/xxxxx).

### Google Docs
Write the doc, then File → Share → Publish to web to make it indexable. The publish
confirmation is a **native browser dialog** that blocks automation — ask the user to
click "OK". Set link sharing to "anyone with the link" as well.

### Notion
Create the page, then Share → Publish, and enable "Search engine indexing" /
"Discoverable on web" (a paid-plan setting on some accounts). Without this the page
exists but is never crawled, so the backlink is worthless for SEO.

### HackMD
CodeMirror markdown note. Set the note's visibility/permission to public or
published so it's readable and indexable, then use the published/shared URL.
