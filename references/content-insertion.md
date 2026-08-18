# Content Insertion Techniques

Getting formatted text into a web editor reliably is the single most fiddly part of
this whole workflow. Every editor is built differently, and the naive approaches
fail silently. This file is the playbook. Match the editor's architecture (from
`platforms.md`) to one of the three techniques below.

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
the *native value setter* for framework textareas, and the editor's own API
(`CodeMirror.setValue`) for CodeMirror. Always screenshot afterward to confirm.

## Technique 1 — Synthetic paste (contenteditable rich editors)

Use for: Medium, Tumblr, Quora, HackerNoon, IndieHackers (body), JustPaste, and any
ProseMirror/TipTap/Quill-style contenteditable surface.

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
  the editor handled the paste — verify visually regardless.
- Build the `html` string from your markdown → HTML so headings, bold, lists, and
  links all come through. Keep links absolute.

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

Use for: Telegraph, Rentry, Blogger (HTML view), HackMD, and anything else backed by
CodeMirror. CodeMirror keeps its own document model, so reach its instance and set
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
  the instance isn't exposed this way — fall back to synthetic paste or typing.
- `.save()` matters on form-backed editors (Rentry, Blogger HTML view): without it,
  the visible editor has your text but the form still submits empty, which is what
  causes "This field is required" on first submit.

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
retry the matching technique rather than switching approaches blindly.
