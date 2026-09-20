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
| Telegraph | telegra.ph | Quill (contenteditable; title/author/body are `h1`/`address`/`p` children of one `.ql-editor`) | `execCommand('insertHTML')` (see gotcha) | **dofollow** (verified 2026-09-20: `rel` attribute absent on published links) | no login needed; save the published URL — edit access is a browser cookie, not a recoverable token, so it only works back in the same browser/profile that created it |
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
| JustPaste | justpaste.it | contenteditable | synthetic paste | nofollow | **CAPTCHA at publish** — user must solve it. Also seen fully down ("We are updating JustPaste.it") for a full day (2026-09-20) — if the homepage shows a maintenance page, don't retry, swap in the next pool platform |
| Google Docs | docs.google.com/document | Google's canvas editor | see note below | indexable once published to web | native "publish to web" confirm dialog blocks automation — user clicks OK |
| Notion | notion.so new page | Notion block editor | see note below | indexable once shared | must turn on Share → "Publish"/"Discoverable on web" or it's never crawled |
| HackMD | hackmd.io new note | CodeMirror (markdown) | `CodeMirror.setValue()` + `.save()` | indexable when public | set note visibility to public/published |
| Product Hunt Forums (p/self-promotion) | producthunt.com/forums → "Start new thread" → select forum "Self-Promotion" | TipTap / ProseMirror (`.tiptap.ProseMirror`) | `execCommand('insertHTML')` (see gotcha) | `rel="ugc"` (not plain nofollow — tagged as user-generated content) | new (added 2026-09-20). Genuine builder/maker audience, free, no login beyond an existing PH account. Every thread goes to a **"Pending Review" moderation queue** before it's publicly visible — don't report it as fully live until confirmed off pending. Title field only appears in the DOM (and only takes typed text reliably) after a forum is selected from "Select Forum" — filling it before selecting the forum can silently fail to save |

"Links" = the status of the outbound links *to the company* in the published post.
Dofollow links pass SEO authority; nofollow ones mostly build presence/referral
traffic and a natural-looking link profile. Prioritise getting your strongest,
most keyword-relevant angles onto the **dofollow** platforms (Bear Blog, Blogger,
Mataroa, Telegraph, and the company's own domains).

## Known platform bugs / quirks

Things that have shown up in live runs that are worth recognizing on sight rather
than re-diagnosing from scratch:

- **Bear Blog can silently ignore the title/slug you set.** Even with the
frontmatter typed in the documented format and verified byte-for-byte correct
in the textarea's actual DOM value before submitting, a post has published
titled "New post" with an auto-generated slug instead — on both a fresh post
and an edited existing one. This is a platform/account-side bug, not a
formatting error, so re-typing it a different way doesn't fix it. Verify the
insertion, retry once cleanly, and if it still comes out wrong, stop and flag
it to the user instead of creating more attempts — each retry that makes a new
post risks leaving a stray duplicate draft, and drafts must never be deleted
without the user asking.
- **Native browser dialogs (Bear Blog's delete confirm, Google Docs' "Publish to
the web") hang automation for ~30s** rather than failing fast — see the
"Safety boundaries" section of `SKILL.md` for the required response (stop
driving that tab, hand off to the user).
- **A synthetic `ClipboardEvent('paste')` can silently no-op on Quill (Telegraph)
and TipTap/ProseMirror (Product Hunt Forums, IndieHackers).** The dispatch
succeeds without throwing, but `defaultPrevented` comes back `false` and the
editor body never changes — there's no error to catch, so this fails silently
unless you verify the DOM afterward. `document.execCommand('insertHTML', false,
html)` is a reliable fallback for both editor families; see
`content-insertion.md` Technique 4 for the exact pattern and its own gotchas
(cursor placement, leftover empty paragraphs).

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
gives you (edit access is a browser cookie tied to the browser/profile that created
the post, not a portable token), so capture the published URL right after publishing.

Contrary to earlier notes, it is **not** literally CodeMirror. The whole editable
area is one Quill instance (`.ql-editor`), and the title, author, and body are
just its first three block children — an `h1[data-placeholder="Title"]`, an
`address[data-placeholder="Your name"]`, and one or more `p`/`h3`/etc. blocks.
A synthetic `ClipboardEvent('paste')` dispatched at this editor did not get
processed in a live run (`defaultPrevented: false`, no DOM change). What worked
reliably: click into the target line so the caret is inside it, then
`document.execCommand('insertHTML', false, html)`.

Two gotchas: (1) never call `insertHTML` while the selection is anchored inside
the `h1` or an `h3` — inserting block-level content (another heading, a
paragraph) *inside* a heading produces malformed nesting that Quill's own
normalization can turn into duplicated/repeated content. Always place the
caret in a plain `<p>` line first (create one if needed) before inserting
multi-block HTML. (2) Quill inserts an empty `<p class="empty">` spacer after
each block-level element it recognizes from the pasted HTML — harmless but
messy; select and remove `p.empty` elements after insertion if a clean DOM
matters (it doesn't affect the rendered/published output, just DOM tidiness).

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
the blue checkmark on the "UNPUBLISHED DRAFT" card (a red X discards). If a
synthetic paste no-ops here too (see the Product Hunt Forums note and the
"Known platform bugs" section), fall back to `execCommand('insertHTML')`.

### HackerNoon
ProseMirror editor, then an editorial review queue — the post won't be instantly
live. At submit it asks whether the piece is original (yes) and requires a
disclosure/attestation about rights and any vested interest. Add a one-line
affiliation disclosure to the article body first, then attest truthfully.
Story Settings has several gates that trip up a first pass: the TL;DR/description
field must be filled in before the submit control even enables; tags must be
**committed by clicking the autocomplete suggestion chip**, not by pressing
Return (Return alone can leave a typed tag uncommitted); and there's both a
panel-level Continue/Submit button and a separate toolbar-level Publish/Submit
button — clicking one without the other can leave the story half-submitted, so
check both landed.

### Quora
Works best as a genuinely useful answer to a relevant question, or a post in a
Space, rather than a bare ad — searching Quora for the question first and
answering an existing (ideally unanswered) one beats writing a generic "Post".
Contenteditable; synthetic paste worked cleanly here (unlike Quill/TipTap — see
"Known platform bugs"), converting pasted `<a href>` tags into Quora's internal
`div.span.link` representation; verify by reading the **published** page's
`<a>` tags afterward, not just the editor DOM, since Quora's editor doesn't
expose the href directly on the span. Disclose the affiliation. Nofollow
(verified: `rel="noopener nofollow"` on published links).

### Pastebin
Plain `<textarea>`, no markdown rendering — paste a plain-text version of the
article with URLs written out in full so they're clickable/visible. Nofollow.

### JustPaste
Contenteditable; synthetic paste works for the body, but publishing triggers a
**CAPTCHA**. Do not attempt to solve it — pause and ask the user to complete it,
then confirm the live URL (e.g. justpaste.it/xxxxx). The whole site has also been
seen fully unavailable behind a "We are updating JustPaste.it" maintenance page
for an entire day — treat that the same as any other reachability failure in the
Phase 0 sweep and swap in the next platform rather than waiting/retrying.

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

### Product Hunt Forums (p/self-promotion)
New platform, added 2026-09-20. `producthunt.com/forums` → "Start new thread" in
the left rail → a "Select Forum" dropdown appears first; search/pick
**Self-Promotion** *before* typing the title — the title `input` only reliably
keeps typed text once a forum is selected (filling it first can leave the field
empty after selection, so verify with `input.value` before submitting either way).
The body is `.tiptap.ProseMirror`; a synthetic paste no-op'd here exactly like
Telegraph's Quill (see "Known platform bugs"), so use
`execCommand('insertHTML')` — same two gotchas as Telegraph (anchor the
selection in a `<p>`, not a heading; strip the empty spacer `<p>` elements it
leaves behind). Links on the published thread come back `rel="ugc"`, not plain
`nofollow` — Product Hunt tags forum-post links as user-generated content.
Real, engaged builder/maker audience and this specific forum exists *for*
self-promotion, so a first-person founder/build-story angle with an upfront
one-line affiliation disclosure fits the room better than a generic pitch.
Every new thread lands in a **"Pending Review" queue** — it has a real URL and
is readable immediately, but treat it as "submitted, pending moderation" rather
than "confirmed live" until you can verify the Pending Review badge is gone.
