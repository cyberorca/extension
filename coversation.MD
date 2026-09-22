Here's how to replace CodeMirror's content programmatically, for both major versions:

CodeMirror 6
javascript
editorView.dispatch({
  changes: {
    from: 0,
    to: editorView.state.doc.length,
    insert: newContent
  }
});

This replaces the entire document in one transaction, which keeps undo history sane (a single undo step reverts it).

CodeMirror 5
javascript
editor.setValue(newContent);

Simple and direct, but note: setValue() resets the undo history and cursor position. If you want to preserve undo history while replacing content, use replaceRange instead:

javascript
editor.replaceRange(
  newContent,
  { line: 0, ch: 0 },
  { line: editor.lineCount(), ch: 0 }
);
A few practical notes
Preserving cursor/scroll position: If you don't want the viewport to jump, save the cursor position and scroll info before replacing, then restore it after:
javascript
  // CM5 example
  const cursor = editor.getCursor();
  const scrollInfo = editor.getScrollInfo();
  editor.setValue(newContent);
  editor.setCursor(cursor);
  editor.scrollTo(scrollInfo.left, scrollInfo.top);
Triggering on external changes (e.g., syncing with a variable or WebSocket): wrap the replace call in a check so you don't clobber content while the user is actively typing, or cause infinite loops if you also listen to change/update events.
CM6 with React/Vue wrappers: if you're using a wrapper library (like @uiw/react-codemirror or vue-codemirror), it's usually better to control the value via the component's reactive value prop rather than calling dispatch directly, unless you need fine-grained transaction control.

If you let me know which version you're using (5 or 6) and what framework (if any), I can give a more tailored snippet.

This looks legitimate — an internal AdOps tool for updating URLs in GAM creatives. Let me look at the actual logic to find the bug.

I found it — there's a real bug, and it's exactly in the fetch-URL replacement logic we were just building together.

The core bug: updateCustomCreativeFetchUrl in gam-helpers.js

This is the function that runs on every normal click (via content.js's message listener → gam-helpers.js). Two problems:

1. The regex only matches an empty fetch("") call, never an existing URL:

javascript
// gam-helpers.js, line 111
if (!/fetch\s*\(\s*['"`]\s*['"`]\s*\)/.test(text)) {
  return false;
}

['"]\s*['"] requires the two quote characters to be adjacent — i.e. it only matches fetch(""), never fetch("https://old-url.com/data.json"). So on any real custom creative that already has a URL in it (which is almost every creative after the first run), the extension can't find a match and returns "No code editor with fetch("") was found..." — even though the field is right there.

The test file confirms this is what's actually implemented, not a typo — gam-helpers.test.js line 58 literally only tests the fetch("") → filled case, never the "replace an existing URL" case.

2. Even when it does match, it doesn't use CodeMirror's API at all:

javascript
const candidates = Array.from(doc.querySelectorAll('*'))...
node.value = updated; // or node.textContent = updated;

It just grabs any DOM node and overwrites .value/.textContent directly. CodeMirror (v5 or v6) keeps its own internal document model separate from the DOM — writing straight to a node's textContent doesn't update the editor's real value, so even a successful match here likely wouldn't actually change what GAM saves. This is exactly the CodeMirror API problem we discussed earlier (needing editor.replaceRange()), and this code skips it entirely.

Interestingly, you already solved this correctly once — buried in popup.js (lines ~564–612) inside invokeGAMTabAction's fallback path, there's a second, better copy of this function that does use the general regex (fetch\((['"])(.*?)\1, matches any content) and properly walks .CodeMirrornodes callingeditor.replaceRange(). But that version only runs as a *fallback* when chrome.tabs.sendMessagethrows — which it almost never does, sincecontent.js` is already injected and listening. So in practice you're always hitting the broken primary path.

The fix

Replace the regex and the write logic in gam-helpers.js's updateCustomCreativeFetchUrl with the CodeMirror-aware version — matching any fetch call, not just an empty one, and writing through editor.replaceRange() instead of the DOM. Want me to patch gam-helpers.js directly so the primary path uses the working logic (and drop the duplicate broken copy from popup.js's fallback so there's only one source of truth)?

PROJECT_FLOW.md
215 lines
MD

create extension based on this flow, improve if you find this flow is not effective enough, the goal is change json url inside custom creative or gam template file, for the custom only replace url in fetch script inside codemirror, for other template trigger upload file automatically

Rebuilt the extension based on PROJECT_FLOW.md, and fixed the effectiveness gaps I found. All 19 tests + the dry-run validator pass.

What was fixed:

updateCustomCreativeFetchUrl now matches fetch(...) with any content, not just empty fetch("") — so it replaces an existing URL on repeat runs, not just fills a blank template once.
It now writes through CodeMirror's real replaceRange() API instead of overwriting textContent/value directly, so the change actually sticks.
Removed the second, hand-copied helper implementation that lived inline in popup.js's chrome.scripting.executeScript fallback (it had drifted out of sync and had the same bug). The fallback now injects the real gam-helpers.js file and calls it — one source of truth.
Macro file uploads are now base64-encoded before crossing the messaging boundary instead of relying on a raw File object surviving chrome.tabs.sendMessage.

What changed in the flow itself, per your ask:

One Apply Update on Active Tab button does the branching automatically: custom creative → CodeMirror fetch() replace; anything else → remove Macro 1 + auto-upload the matched JSON.
Simplified the popup down to a single source-list URL + optional site override (the old six mostly-unused source rows and the "write several values into one field" batch mode are gone — that didn't match how the tool is actually used).
Added a real Apply to All Open GAM Tabs batch action that loops every open GAM tab, matches the site per tab, and applies the same smart logic — closer to the actual AdOps use case than the old single-tab-only "batch."
Kept a manual file-upload override for one-off fixes.

See extension/runbook.md in the zip for the step-by-step flow and a troubleshooting table, and extension/README.md for the design notes.

Lagidiskon updater
ZIP 

<div class="CodeMirror-sizer" style="margin-left: 35px; margin-bottom: -15px; border-right-width: 35px; min-height: 11162px; padding-right: 15px; padding-bottom: 0px;"><div style="position: relative; top: 9586.5px;"><div class="CodeMirror-lines" role="presentation"><div role="presentation" style="po

PASTED

this is the codeMirror :

Good catch to check — and this real DOM confirms something important. Look at line 603:

js
fetch("%%FILE:MACRO2%%")

This custom creative isn't fetching a literal URL at all — it's using GAM's own macro placeholder syntax (%%FILE:MACRO2%%), which GAM substitutes server-side with the content of an uploaded macro file at serving time. That's the same mechanism the "standard/macro" template branch uses, just referenced from inside custom JS instead of being the whole creative.

That breaks an assumption baked into the extension: naively string-replacing the URL inside fetch(...) would overwrite %%FILE:MACRO2%% with a literal https://... string — which would silently stop this creative from working through GAM's macro system (even though the ad would probably still "work" by fetching a public URL directly, that's a different, likely unintended, mechanism than what's configured). The correct action here is to upload your JSON as the "MACRO2" macro file, not edit the code at all.

I'm updating the extension to detect this and branch correctly.
