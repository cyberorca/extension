# GAM JSON Updater Extension Flow

## 1. Project Goal
This project is a Chrome extension built for internal AdOps work inside Google Ad Manager (GAM). The extension helps users update JSON URL values for creatives without depending on the GAM API.

The key idea is:
- read the active GAM page DOM
- detect the active creative context
- resolve a matched JSON source from a list or URL
- update the correct field or custom creative code editor
- apply the macro upload flow when needed

---

## 2. Core Architecture
The extension runs as a Chrome MV3 extension with a popup and content script.

### Main files
- `manifest.json` — registers extension metadata and permissions
- `popup.html` — UI shown in the extension popup
- `popup.js` — main UI logic and browser-side orchestration
- `content.js` — listens for messages from popup and runs DOM actions in the active tab
- `gam-helpers.js` — GAM DOM-specific helpers for detection and mutation
- `source-list.js` — resolves source URLs from Google Drive or JSON payloads
- `batch-helpers.js` — batch processing support for multiple creatives
- `background.js` — background runtime utilities and config defaults

---

## 3. Key Behavior
The tool is designed around the active authenticated browser session. It does not rely on the GAM API.

### Runtime assumptions
- user is already logged in to GAM in the active Chrome tab
- the target page is a GAM creative or edit screen
- source data comes from external JSON URLs or Drive URLs

### Core tasks
1. Read the active page and detect whether it is a GAM creative environment.
2. Resolve the source JSON list and find the matching site or creative record.
3. Read the matched `json_url` field from the payload.
4. Populate the UI with the matched JSON URL.
5. Allow refresh from a fixed Google Drive source.
6. On custom creative flows, replace `fetch("")` or an existing fetch URL in the code editor with the matched JSON URL.
7. On standard macro flow, remove Macro 1, upload the prepared file, and apply it to the active GAM form.

---

## 4. Source Resolution Flow
This step decides what value should ultimately be used as the target URL.

### Source model
The extension supports a list-based source from Google Drive, where each item looks like:

```json
[
  {
    "name": "bola.net",
    "json_url": "https://cdn.example.com/bola.json"
  }
]
```

### Flow
1. The popup receives a fixed Google Drive source URL.
2. The URL is normalized into a downloadable file URL.
3. The extension fetches that file.
4. The payload is parsed.
5. It extracts all `name` and `json_url` combinations.
6. The current creative or site name is matched against the payload.
7. The correct `json_url` is selected and shown in the UI.

### Important rule
The value inserted into the UI or used in custom creative logic is not the Google Drive share URL itself. It is the actual matched `json_url` from the payload.

---

## 5. GAM DOM Interaction Flow
The extension uses DOM and browser automation instead of API access.

### Primary DOM strategy
- identify important inputs, labels, and controls in GAM
- find the most likely JSON field or data entry target
- detect the creative type in the subheader
- locate macro rows and upload elements

### Helper patterns
The `gam-helpers.js` module contains helpers such as:
- `resolveWriteTarget()`
- `validateTargetState()`
- `removeMacroByName()`
- `uploadMacroFile()`
- `updateCustomCreativeFetchUrl()`
- `getCreativeTypeLabel()`

These functions inspect the live DOM and perform targeted element actions.

---

## 6. Custom Creative Flow
This is the branch for custom creatives.

### Trigger
When the active creative is detected as custom, the extension should update the active code editor.

### Requirement
In a custom code editor, the extension searches for a `fetch()` call and replaces the URL with the matched JSON URL.

### Example logic
```js
const editor = document.querySelector('.CodeMirror').CodeMirror;
replaceFetchUrl(editor, 'https://matched-json-url.example/data.json');
```

### Actual flow
1. Detect the creative type from the GAM DOM.
2. If type matches custom, continue.
3. Find the active code editor (`CodeMirror` preferred).
4. Search for a `fetch(...)` statement.
5. Replace the URL inside the fetch call with the matched JSON URL.
6. Do this before the macro upload step completes.

### Important implementation detail
The extension must use the editor API, not raw `textContent` mutation, because CodeMirror stores the document internally and updating the DOM text alone does not update the actual editor state.

---

## 7. Macro Upload Flow
This branch is used for standard macro-based creative upload handling.

### Standard flow
1. Query the active tab.
2. Check whether a custom creative fetch update was needed.
3. If not custom, prepare the matched JSON as a File object.
4. Remove Macro 1 from the GAM form.
5. Upload the file to the file input.
6. Confirm the upload succeeded.

### Why remove Macro 1 first
The GAM form expects a macro row to be removed before the new file is uploaded as the replacement macro. This is a form-specific workflow, not a generic file upload.

---

## 8. UI Flow
The extension popup has a small, compact interface.

### Main UI elements
- fixed Google Drive source input
- matched site field
- matched JSON URL field
- refresh button
- custom URL toggle / override area
- apply macro button
- status area
- creative type badge

### User behavior
1. User opens the popup while on a GAM creative page.
2. Source data is fetched from the fixed Drive source.
3. Matching logic resolves the correct JSON URL.
4. The popup shows the matched result.
5. User clicks Apply Macro 1.
6. The extension updates the GAM page using the correct DOM logic.

---

## 9. Error Handling Pattern
The extension uses defensive logic to avoid hard failures.

### Examples
- no active tab found
- content script not connected
- no GAM page detected
- no field or editor found
- creative type not custom
- no `fetch()` found in editor
- no file input found on the form

### Fallback approach
When a direct message call fails, the extension can retry by running a script in the active page context using `chrome.scripting.executeScript()`. This is useful when message passing is unavailable due to timing or injection issues.

---

## 10. Validation Strategy
The project includes Node-based tests for the parser and helper logic.

### Example checks
- average site matching logic
- Google Drive URL normalization
- JSON source extraction
- custom creative fetch replacement
- macro cleanup and upload input logic

### Current validation command
```bash
cd "/Users/kly/Documents/GAM-JSON-UPDATER" && node --test extension/gam-helpers.test.js extension/source-list.test.js extension/batch-helpers.test.js
```

This ensures major helper behaviors still work after each adjustment.

---

## 11. Final Summary
This extension development flow is essentially:

1. Read and normalize external source JSON data.
2. Match the correct site or creative record.
3. Determine the actual `json_url` to use.
4. Update the active GAM page without using the GAM API.
5. Handle both standard and custom creative cases.
6. Remove and replace macro assets through the actual GAM DOM and file inputs.
7. Keep the popup compact and use the live page context for the final write operations.

The project is intentionally DOM-first and browser-session-based, because the requirement is to operate inside the authenticated GAM UI instead of calling any external backend or API.
