# Content Insertion Techniques

Getting formatted text into a web editor reliably is the single most fiddly part of
this whole workflow. Every editor is built differently, and the naive approaches
fail silently. This file is the playbook. Match the editor's architecture (from
`platforms.md`) to one of the four techniques below.

## Why the obvious approaches don't work

- **`element.value = text`** on a React/Preact/Vue textarea does nothing lasting:
  the framework re-renders from its own state and overwrites you. You must use the
  *native* value setter and then fire `input`/`change` so the framework notices.
- **`navigator.clipboard.write()` and `document.execCommand('copy')`** are blocked by
  the browser's user-activation policy in this automation context — `clipboard.write`
  hangs the call and `execCommand('copy')` returns false. Don't rely on the real
  clipboard.
- **Typing character-by-character** into rich editors is slow and mangles formatting.

The reliable moves are: a *synthetic* paste event for rich contenteditable editors,
the *native value setter* for framework textareas, the editor's own API
(`CodeMirror.setValue`) for CodeMirror, and `execCommand('insertHTML')` as a fallback
when the synthetic paste event gets silently ignored. Always screenshot afterward to
confirm.

## Technique 1 — Synthetic paste (contenteditable rich editors)

Use for: Medium, Tumblr, Quora, HackerNoon, JustPaste, and any
ProseMirror/TipTap/Quill-style contenteditable surface — **try this first**, but
verify the result rather than trusting `defaultPrevented`, since it has been
observed to silently no-op on Quill (Telegraph) and TipTap/ProseMirror (Product
Hunt Forums, and possibly IndieHackers) in this environment. See Technique 4 for
the fallback when that happens.

The trick is to construct a `ClipboardEvent('paste')` carrying a `DataTransfer` with
both `text/html` (so formatting/links survive) and `text/plain` (fallback), then
dispatch it into the focused editor. The editor's own paste handler ingests it.

First click into the editor so it's focused and has a selection, then run:

```javascript
(() => {
  const html = `<h1>Title</h1><p>First paragraph with a <a href="https://example.com/">link</a>.</p>`; // your article HTML
  const plain = "Title\n\nFirst paragraph with a link (https://example.com/).";   // plain fallback
  const el = document.activeElement;               // the focused editor
  const dt = new DataTransfer();
  dt.setData('text/html', html);
  dt.setData('text/plain', plain);
  const ev = new ClipboardEvent('paste', { bubbles: true, cancelable: true, clipboardData: dt });
  el.dispatchEvent(ev);
  return ev.defaultPrevented; // true means the editor consumed it (good sign)
})();
```

Notes:
- Make sure the editor is actually focused first (click into the body area). Some
  editors require a non-collapsed selection or a click before they'll accept paste.
- `defaultPrevented === true` (or `dispatched=false` in some wrappers) usually means
  the editor handled the paste — but `false` does not always mean it failed, and
  `true` does not always mean it fully landed either. Always re-read the editor's
  DOM (e.g. `el.innerHTML.length`, or check for your known link URLs) after
  dispatching, not just the event's own flags. If the DOM didn't change at all,
  move straight to Technique 4 rather than retrying the same dispatch.
- Build the `html` string from your markdown → HTML so headings, bold, lists, and
  links all come through. Keep links absolute.
- Also verify on the **published** page, not just the editor: some editors (Quora)
  convert `<a href>` into their own internal representation during editing and
  don't expose the href again until the content is rendered live.

## Technique 2 — Native value setter (framework-controlled textareas)

Use for: Dev.to, Hashnode, Bear Blog, Pastebin, Mataroa, Write.as (`#writer`), and
the IndieHackers **title** field. These are `<textarea>` (or `<input>`) elements
whose value the framework controls, and they usually take **markdown or plain text**
(not HTML), so paste the markdown source.

```javascript
(() => {
  const el = document.querySelector('textarea');   // narrow this selector to the real editor
  const value = "# Title\n\nFirst paragraph with a [link](https://example.com/).";  // markdown source
  const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
  setter.call(el, value);
  el.dispatchEvent(new Event('input', { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
  return el.value.length;
})();
```

Notes:
- Replace `document.querySelector('textarea')` with a selector that hits the actual
  body editor — pages often have several textareas. Use the `find` tool or inspect
  `id`/`class` first. Write.as is `#writer`; IndieHackers title is
  `textarea.newlines-disabled` (single-line — set the title string only there).
- For an `<input>` element, swap `HTMLTextAreaElement` for `HTMLInputElement`.
- Fire both `input` and `change`; some frameworks listen for one, some the other.

## Technique 3 — CodeMirror API (`setValue`)

Use for: Rentry, Blogger (HTML view), HackMD, and anything else backed by
CodeMirror 5. CodeMirror keeps its own document model, so reach its instance and set
the value directly, then `.save()` to flush it back into the underlying textarea the
form submits.

```javascript
(() => {
  const cm = document.querySelector('.CodeMirror').CodeMirror;   // CM5 instance
  cm.setValue("# Title\n\nFirst paragraph with a [link](https://example.com/).");
  if (cm.save) cm.save();   // flush to the backing <textarea> so the form sees it
  return cm.getValue().length;
})();
```

Notes:
- This is CodeMirror 5 (`.CodeMirror` on the DOM node). If a site uses CodeMirror 6,
  the instance isn't exposed this way — fall back to Technique 1 or Technique 4.
- `.save()` matters on form-backed editors (Rentry, Blogger HTML view): without it,
  the visible editor has your text but the form still submits empty, which is what
  causes "This field is required" on first submit.
- **Telegraph is not CodeMirror**, despite earlier notes here — its whole editable
  surface (title/author/body) is one Quill instance. Use Technique 4 for it.

## Technique 4 — `execCommand('insertHTML')` fallback (Quill, TipTap/ProseMirror)

Use for: Telegraph (Quill), Product Hunt Forums (TipTap/ProseMirror), and any other
Quill or TipTap/ProseMirror surface where Technique 1's synthetic paste dispatches
without error but leaves the editor unchanged (`defaultPrevented: false` and no DOM
change is the tell). `execCommand('insertHTML')` performs a real DOM insertion that
these editors' own mutation observers pick up and normalize into their internal
document model correctly, where a synthetic `ClipboardEvent` sometimes doesn't reach
their paste handler at all.

```javascript
(() => {
  const html = `<h1>Title</h1><p>First paragraph with a <a href="https://example.com/">link</a>.</p>`;
  const el = document.querySelector('.ql-editor, .tiptap.ProseMirror'); // narrow to the real editor
  // IMPORTANT: anchor the selection inside a plain block line (a <p>), never inside
  // a heading (h1/h3/etc) — see gotcha 1 below.
  const target = el.querySelector('p') || el; // a <p> line to place the caret in
  const range = document.createRange();
  range.selectNodeContents(target);
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
  const ok = document.execCommand('insertHTML', false, html);
  return { ok, len: el.innerHTML.length };
})();
```

Two gotchas, both seen in live runs:

1. **Never anchor the selection inside a heading (`h1`/`h3`/etc.) before calling
   `insertHTML` with multi-block content.** Inserting a `<p>` or another heading
   *inside* an existing heading element produces malformed nesting, and the
   editor's own normalization can turn that into duplicated or repeated content
   across the whole body (this has produced a document with the same paragraphs
   inserted 2–3 times over). Always place the caret inside a plain `<p>` line
   first — clear the editor down to `title + author + one empty <p>` before
   inserting if you're not sure what's currently focused, e.g.:
   ```javascript
   const editor = document.querySelector('.ql-editor'); // or .tiptap.ProseMirror
   const keep = new Set(['H1', 'ADDRESS']); // adjust to whatever must survive
   [...editor.children].filter(c => !keep.has(c.tagName)).forEach(n => n.remove());
   const p = document.createElement('p');
   p.innerHTML = '<br>';
   editor.appendChild(p);
   // now select p's contents and insertHTML as above
   ```
2. **Both Quill and TipTap insert an empty spacer block after each block-level
   element** they recognize from the inserted HTML (`<p class="empty"><br></p>` on
   Quill, a plain empty `<p>` on TipTap). Harmless to the published output, but
   clean it up if a tidy DOM matters:
   ```javascript
   [...document.querySelectorAll('.ql-editor p.empty, .tiptap.ProseMirror p')]
     .filter(p => p.textContent.trim() === '')
     .forEach(p => p.remove());
   ```

After using this technique, verify the same way as Technique 1: re-read the
editor's DOM for your links and text, and check the **published** page's `<a>`
tags afterward too.

## Blogger's special case

Blogger's HTML/Compose toggle can wipe the body. The robust pattern:
1. Switch to HTML view and set content via the CodeMirror technique, OR
2. Set the content directly in the Compose-mode iframe body, then make a no-op edit
   (type a space and delete it) to trigger Blogger's autosave/sync so the change
   sticks. Avoid flipping views again after inserting.

## Always verify

After any insertion, take a screenshot and confirm: the body text is there, headings
render as headings, bold is bold, and — most importantly for this campaign — every
backlink is present and points at the right absolute URL. Editors drop pasted links
more often than any other element. If content didn't land, re-focus the editor and
retry the matching technique — and if Technique 1 visibly no-op'd (no error, but no
DOM change either), move to Technique 4 rather than re-dispatching the same paste
event again.
