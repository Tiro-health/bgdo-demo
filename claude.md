# Tiro ChartReview — Coding Agent Specification

> **Purpose**: This document is the authoritative reference for a coding agent building `index.html`, a single-file clinical chart-review tool. Read it fully before writing any code.

---

## 1. Overview

Build a **single HTML file** (`index.html`) with no external bundler, no build step, and no Node.js. All logic lives in one self-contained file. The app enables a clinician to paste clinical free text, fill in a structured FHIR questionnaire (via the Tiro SDK), and produce a clean output text — all in one view.

### Primary layout: three columns (the "Capture" view)

| Column | Role |
|---|---|
| **Left** | Magic Clipboard — free-text input area |
| **Center** | Questionnaire renderer (FHIR SDC) with questionnaire selector dropdown and picture-in-picture (PiP) pop-out |
| **Right** | Rendered text result (plain text) with one-click copy-to-clipboard |

### Secondary view: "Data" tab

A tabular view of all saved `QuestionnaireResponse` FHIR resources, powered by SQL-on-FHIR ViewDefinitions.

---

## 2. Technology Stack

### Required scripts (load in `<head>`, in this order)

```html
<!-- Tiro Web SDK — provides tiro-form-filler, tiro-magic-clipboard, tiro-magic-clipboard-button -->
<script src="https://sdk.tiro.health/web"></script>

<!-- SheetJS — for CSV/XLSX export in the Data tab -->
<script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>

<!-- SQL-on-FHIR v2 — ViewDefinition evaluation -->
<script src="https://cdn.jsdelivr.net/npm/sof-js@latest/dist/index.umd.js"></script>
```

> **Important**: `https://sdk.tiro.health/web` is the canonical SDK URL. Do not substitute `https://cdn.tiro.health/sdk/latest/tiro-web-sdk.iife.js` — always use `https://sdk.tiro.health/web`.

### Fonts

```html
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link href="https://fonts.googleapis.com/css2?family=Open+Sans:ital,wght@0,300..800;1,300..800&display=swap" rel="stylesheet" />
```

### No other external dependencies

Do not add React, Vue, Alpine, Tailwind CDN, or any other framework. Vanilla JS and the above three scripts are sufficient.

---

## 3. Tiro SDK — Custom Elements Reference

The SDK registers these custom HTML elements:

### `<tiro-form-filler>`

Renders a FHIR SDC Questionnaire and manages its `QuestionnaireResponse`.

**Key attributes:**

| Attribute | Type | Description |
|---|---|---|
| `sdc-endpoint-address` | string | FHIR R5 SDC endpoint. Use `https://sdc.tiro.health/fhir/r5` |
| `questionnaire-url` | string | Canonical URL of the questionnaire template (e.g. `http://templates.tiro.health/templates/<id>|<version>`) |
| `questionnaire-json` | string | JSON-stringified FHIR Questionnaire resource (alternative to URL) |
| `compact-grouping` | boolean attr | Renders groups more compactly |
| `read-only` | boolean attr | Makes the form read-only (used in preview/design mode) |

**Key methods (call on the DOM element):**

```js
const filler = document.getElementById('my-form');

// Get the current QuestionnaireResponse as a FHIR resource object
const qr = await filler.getQuestionnaireResponse();

// Reset the form to a blank QuestionnaireResponse
await filler.reset();
```

**Key events (listen on the element):**

```js
filler.addEventListener('tiro-response-change', (e) => {
  // e.detail.questionnaireResponse — the updated QR object
  // fired whenever any answer changes
});
```

### `<tiro-magic-clipboard>`

Provides AI-powered autofill: takes free text from a textarea and populates the linked `<tiro-form-filler>`.

**Key attributes:**

| Attribute | Value | Description |
|---|---|---|
| `for` | string | The `id` of the `<tiro-form-filler>` to autofill |

**Inner child element:**

```html
<tiro-magic-clipboard for="my-form">
  <tiro-magic-clipboard-button>Formulier automatisch invullen</tiro-magic-clipboard-button>
</tiro-magic-clipboard>
```

The `<tiro-magic-clipboard-button>` automatically manages loading states via `data-state` attribute (`pending`, `success`, `error`).

**Key events:**

```js
const mc = document.querySelector('tiro-magic-clipboard');
mc.addEventListener('tiro-populate-error', (e) => {
  console.error(e.detail?.message);
});
```

**How it works**: The SDK scans for `<textarea>` elements inside (or near) the `<tiro-magic-clipboard>` component and uses their text as context for AI-powered form filling.

### Custom element CSS

Style `tiro-magic-clipboard-button` directly by targeting the element. It supports `data-state` attribute values: `pending` (shows spinner), `success` (shows checkmark), `error` (shows red).

```css
tiro-magic-clipboard-button {
  display: block;
  width: 100%;
  padding: 0.55rem 1rem;
  font-family: "Open Sans", sans-serif;
  font-size: 0.8rem;
  font-weight: 700;
  color: #fff;
  background: #2563eb;
  border: none;
  border-radius: 0.4rem;
  cursor: pointer;
}

tiro-magic-clipboard-button[data-state="pending"] {
  opacity: 0.75;
  cursor: wait;
}

tiro-magic-clipboard-button[data-state="success"] {
  background: #7dd3fc;
  color: #0d2137;
}

tiro-magic-clipboard-button[data-state="error"] {
  background: #dc2626;
}
```

---

## 4. App Structure

### 4.1 Shell

```
<body>
  <header class="header">        <!-- Fixed top bar, dark navy -->
  <nav class="tab-nav">          <!-- Tab bar with "Invoer" and "Data" tabs -->
  <main class="main">
    <div id="panel-capture">     <!-- Default active panel -->
    <div id="panel-dataset">     <!-- Data tab panel, hidden by default -->
  </main>
  <div id="toast">               <!-- Toast notification overlay -->
</body>
```

### 4.2 Capture Panel (3-column layout)

```html
<div id="panel-capture" class="panel">
  <!-- Toolbar row: questionnaire dropdown + entry ID chip + action buttons -->
  <div class="toolbar-row">
    <select id="questionnaire-select">...</select>
    <span class="entry-id-chip" id="capture-entry-id"></span>
    <div class="btn-group">
      <button id="new-record-btn">Nieuw record</button>
      <button id="save-record-btn" disabled>Opslaan in dataset</button>
    </div>
  </div>

  <!-- Three-column split -->
  <div class="split-3col">

    <!-- Column 1: Magic Clipboard -->
    <div class="card col-clipboard">
      <div class="card__header">Klinische tekst</div>
      <div class="card__body">
        <tiro-magic-clipboard for="capture-form">
          <textarea id="clipboard-text" placeholder="Plak klinische tekst hier..."></textarea>
          <tiro-magic-clipboard-button>Formulier automatisch invullen</tiro-magic-clipboard-button>
        </tiro-magic-clipboard>
      </div>
    </div>

    <!-- Column 2: Questionnaire renderer -->
    <div class="card col-form">
      <div class="card__header">
        <span class="card__title">Formulier</span>
        <button id="pip-btn" title="Pop-out formulier" class="btn btn--ghost btn--icon">
          <!-- PiP icon -->
        </button>
      </div>
      <div class="form-filler-wrapper">
        <tiro-form-filler
          id="capture-form"
          sdc-endpoint-address="https://sdc.tiro.health/fhir/r5"
          compact-grouping
        ></tiro-form-filler>
      </div>
    </div>

    <!-- Column 3: Text result -->
    <div class="card col-result">
      <div class="card__header">
        <span class="card__title">Resultaat</span>
        <button id="copy-result-btn" class="btn btn--ghost btn--icon" title="Kopieer naar klembord">
          <!-- Copy icon -->
        </button>
      </div>
      <div class="card__body">
        <div id="result-text" class="result-output">
          <!-- Plain text result rendered here -->
        </div>
      </div>
    </div>

  </div>
</div>
```

### 4.3 Questionnaire Selector

The toolbar contains a `<select>` listing all configured questionnaires. The list is defined in a `QUESTIONNAIRES` constant at the top of the script. When the selection changes, update `<tiro-form-filler>`'s `questionnaire-url` attribute and reset the form.

```js
const QUESTIONNAIRES = [
  {
    id: 'ibd',
    name: 'IBD Consultatie',
    code: 'IBD',
    url: 'http://templates.tiro.health/templates/def02a77523d472fabfa949e9522e651|1.0.1',
  },
  // Additional questionnaires added here by the user at runtime
  // or hardcoded for known deployments
];
```

When the user switches questionnaires:
1. Update `capture-form` attribute `questionnaire-url`
2. Call `captureForm.reset()` to clear the current response
3. Clear the result text panel
4. Update the active entry ID

### 4.4 Picture-in-Picture (PiP) for the Form Column

Use the [Document Picture-in-Picture API](https://developer.chrome.com/docs/web-platform/document-picture-in-picture/) to pop out the questionnaire column.

```js
async function openPiP() {
  if (!('documentPictureInPicture' in window)) {
    showToast('Picture-in-Picture wordt niet ondersteund in deze browser.');
    return;
  }

  const pip = await window.documentPictureInPicture.requestWindow({
    width: 480,
    height: 700,
  });

  // Copy stylesheets into the PiP window
  [...document.styleSheets].forEach(sheet => {
    try {
      const style = pip.document.createElement('style');
      style.textContent = [...sheet.cssRules].map(r => r.cssText).join('\n');
      pip.document.head.appendChild(style);
    } catch {}
  });

  // Move the form-filler DOM node into PiP window
  const formWrapper = document.getElementById('pip-content-slot');
  pip.document.body.appendChild(formWrapper);

  pip.addEventListener('pagehide', () => {
    // Return the element to the main page on PiP close
    document.getElementById('col-form-inner').appendChild(formWrapper);
  });
}
```

**Implementation notes:**
- Wrap the `<tiro-form-filler>` in a dedicated `<div id="pip-content-slot">` so it can be moved in/out of the PiP window without destroying the element.
- The `<tiro-form-filler>` custom element remains the same DOM node; only its parent changes. This preserves state.
- Show/hide the PiP button depending on API availability: `'documentPictureInPicture' in window`.
- If the API is unavailable (Firefox, Safari), fall back to opening the questionnaire URL in a new browser tab at `https://forms.tiro.health/<questionnaire-url>` (configure this fallback per questionnaire entry if known).

---

## 5. Result Text Generation

The right column shows a rendered plain-text summary of the current `QuestionnaireResponse`. This is generated every time a `tiro-response-change` event fires.

### Approach

1. Listen for `tiro-response-change` on `capture-form`.
2. Retrieve the QR: `const qr = e.detail.questionnaireResponse`.
3. Walk `qr.item` recursively to build a text summary.
4. Render into `#result-text` as `<pre>` or `white-space: pre-wrap` text.

### Text rendering algorithm

```js
function renderQRToText(qr) {
  if (!qr || !qr.item) return '';
  const lines = [];

  function walk(items, depth = 0) {
    for (const item of items) {
      const indent = '  '.repeat(depth);
      if (item.item) {
        // Group item
        lines.push(`${indent}### ${item.text || item.linkId}`);
        walk(item.item, depth + 1);
      } else if (item.answer && item.answer.length) {
        const values = item.answer.map(a => getAnswerDisplayValue(a)).filter(Boolean).join(', ');
        lines.push(`${indent}${item.text || item.linkId}: ${values}`);
      }
    }
  }

  walk(qr.item);
  return lines.join('\n');
}

function getAnswerDisplayValue(answer) {
  if (answer.valueCoding)   return answer.valueCoding.display || answer.valueCoding.code;
  if (answer.valueString)   return answer.valueString;
  if (answer.valueDecimal !== undefined) return String(answer.valueDecimal);
  if (answer.valueInteger !== undefined) return String(answer.valueInteger);
  if (answer.valueBoolean !== undefined) return answer.valueBoolean ? 'Ja' : 'Nee';
  if (answer.valueDate)     return answer.valueDate;
  if (answer.valueDateTime) return answer.valueDateTime;
  return '';
}
```

### Copy to clipboard

```js
document.getElementById('copy-result-btn').addEventListener('click', async () => {
  const text = document.getElementById('result-text').innerText;
  if (!text.trim()) return;
  await navigator.clipboard.writeText(text);
  showToast('Gekopieerd naar klembord.');
});
```

---

## 6. Data Tab — FHIR ViewDefinitions with SQL-on-FHIR

### 6.1 SQL-on-FHIR v2 library

Load from CDN:

```html
<script src="https://cdn.jsdelivr.net/npm/sof-js@latest/dist/index.umd.js"></script>
```

This exposes `window.sofjs` (or check the actual export name after loading — it may be `window.sof` or a named export depending on the UMD bundle). Adjust after inspecting the actual export.

Reference implementation: https://github.com/FHIR/sql-on-fhir-v2/blob/master/sof-js/src/index.js

The key function is `runViewDefinition(viewDefinition, resources)`:

```js
// viewDefinition: a FHIR ViewDefinition resource (JSON object)
// resources: array of QuestionnaireResponse FHIR resources
const rows = sofjs.runViewDefinition(viewDefinition, resources);
// rows: array of plain objects, one per resource, with columns as defined in the ViewDefinition
```

### 6.2 ViewDefinition structure

Each questionnaire entry in `QUESTIONNAIRES` should carry an optional `viewDefinition` property. This is a FHIR ViewDefinition resource that describes how to flatten a `QuestionnaireResponse` for that questionnaire into tabular rows.

Example ViewDefinition for IBD:

```json
{
  "resourceType": "ViewDefinition",
  "name": "IBDConsultatie",
  "resource": "QuestionnaireResponse",
  "select": [
    { "column": [{ "path": "id", "name": "id" }] },
    { "column": [{ "path": "authored", "name": "authored" }] },
    {
      "forEach": "item.where(linkId='patientennummer-8').answer",
      "column": [{ "path": "valueString", "name": "patientennummer" }]
    },
    {
      "forEach": "item.where(linkId='geboortedatum-12').answer",
      "column": [{ "path": "valueDate", "name": "geboortedatum" }]
    }
  ]
}
```

**Note for the coding agent**: The exact FHIRPath syntax for `item.descendants()` vs nested `forEach` depends on the sof-js implementation. Refer to the reference implementation at https://github.com/FHIR/sql-on-fhir-v2/blob/master/sof-js/src/index.js to confirm supported FHIRPath expressions. When in doubt, use simple `forEach` with `item.where(linkId='...')` rather than `descendants()`.

### 6.3 Fallback: generic row extractor

If no `viewDefinition` is defined for the active questionnaire, fall back to the generic row extractor:

```js
function extractGenericRow(qr) {
  const row = { 'ID': qr.id || qr._entryId || '', 'Aangemaakt': qr.authored || '' };
  collectAnswers(qr.item, row, '');
  return row;
}

function collectAnswers(items, result, prefix) {
  if (!items) return;
  for (const item of items) {
    const key = prefix ? `${prefix} / ${item.text || item.linkId}` : (item.text || item.linkId);
    if (item.answer && item.answer.length) {
      result[key] = item.answer.map(getAnswerDisplayValue).filter(Boolean).join(' | ');
    }
    if (item.item) collectAnswers(item.item, result, key);
  }
}
```

### 6.4 Data tab UI

```html
<div id="panel-dataset" class="panel" hidden>
  <div class="toolbar-row">
    <select id="dataset-questionnaire-select"></select>
    <span id="dataset-count" class="count-label"></span>
    <div class="btn-group">
      <button id="export-csv-btn" class="btn btn--secondary">CSV exporteren</button>
      <button id="export-xlsx-btn" class="btn btn--secondary">Excel exporteren</button>
      <button id="clear-dataset-btn" class="btn btn--danger">Wissen</button>
    </div>
  </div>
  <div class="card table-card">
    <div class="table-wrapper">
      <div id="dataset-empty" class="empty-state" hidden>Geen records.</div>
      <table id="data-table">
        <thead id="table-head"></thead>
        <tbody id="table-body"></tbody>
      </table>
    </div>
  </div>
</div>
```

---

## 7. Storage

Use `localStorage` for all persistence. No IndexedDB or server-side storage.

### Keys

```js
const RECORDS_PFX   = 'tiro_cr_records_';    // + questionnaire id -> JSON array of QR objects
const COUNTER_PFX   = 'tiro_cr_counter_';    // + questionnaire id -> integer counter
const ACTIVE_KEY    = 'tiro_cr_active';      // active questionnaire id
const QS_CONFIG_KEY = 'tiro_cr_questionnaires'; // user-added questionnaire configs
```

### Entry ID generation

```js
function generateEntryId(questionnaire) {
  const key = COUNTER_PFX + questionnaire.id;
  const count = parseInt(localStorage.getItem(key) || '0', 10) + 1;
  localStorage.setItem(key, String(count));
  const year = new Date().getFullYear();
  return `${questionnaire.code}-${year}-${String(count).padStart(5, '0')}`;
}
```

### Saving a record

```js
async function saveCurrentRecord() {
  const filler = document.getElementById('capture-form');
  const qr = await filler.getQuestionnaireResponse();

  // Attach metadata
  qr._entryId = currentEntryId;
  if (!qr.meta) qr.meta = {};
  if (!qr.meta.extension) qr.meta.extension = [];
  qr.meta.extension.push({
    url: 'http://hl7.org/fhir/StructureDefinition/firstCreated',
    valueInstant: new Date().toISOString(),
  });

  const records = getRecords(activeQuestionnaireId);
  const existingIdx = records.findIndex(r => r._entryId === currentEntryId);
  if (existingIdx >= 0) {
    records[existingIdx] = qr;
  } else {
    records.push(qr);
  }
  saveRecords(activeQuestionnaireId, records);
  updateBadge();
  showToast('Record opgeslagen.');
}
```

---

## 8. Design System

### Colors

```css
--bg:        #e8ecf2;   /* page background */
--surface:   #ffffff;   /* card surface */
--navy:      #0d2137;   /* header, tab bar */
--blue:      #2d7bf4;   /* primary action, accents */
--border:    #dde3ec;   /* card borders */
--text:      #1e293b;   /* body text */
--text-muted: #64748b;  /* secondary text */
--text-faint: #94a3b8;  /* placeholders */
--success:   #16a34a;
--danger:    #dc2626;
```

### Layout

- `body`: `display: flex; flex-direction: column; height: 100%; overflow: hidden`
- Header: `height: 56px; background: var(--navy); flex-shrink: 0`
- Tab nav: `height: 44px; background: var(--navy); flex-shrink: 0`
- Main: `flex: 1; min-height: 0; overflow: hidden; display: flex; flex-direction: column`
- Panel: `flex: 1; min-height: 0; overflow: hidden; display: flex; flex-direction: column; padding: 1rem 1.25rem; gap: 0.875rem`

### Three-column grid

```css
.split-3col {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: 340px 1fr 320px;
  gap: 0.875rem;
}
```

Adjust column widths as needed. The center column (questionnaire) should get the most space. All three columns are `card` components with internal scroll.

### Cards

```css
.card {
  background: #fff;
  border: 1px solid #dde3ec;
  border-radius: 0.625rem;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}
.card__header {
  flex-shrink: 0;
  padding: 0.625rem 0.875rem;
  border-bottom: 1px solid #eef0f5;
  background: #f8fafd;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
.card__body {
  flex: 1;
  min-height: 0;
  overflow: auto;
  padding: 0.875rem;
}
```

### Buttons

```css
.btn { height: 34px; padding: 0 1rem; border-radius: 6px; font-size: 13px; font-weight: 500; cursor: pointer; display: inline-flex; align-items: center; gap: 0.375rem; }
.btn--primary   { background: #2d7bf4; color: #fff; border: 1px solid #2d7bf4; }
.btn--secondary { background: #fff; color: #374151; border: 1px solid #dde3ec; }
.btn--success   { background: #16a34a; color: #fff; border: 1px solid #16a34a; }
.btn--danger    { background: #fff; color: #dc2626; border: 1px solid #fecaca; }
.btn--ghost     { background: transparent; color: #64748b; border: 1px solid transparent; }
.btn--icon      { width: 30px; height: 30px; padding: 0; justify-content: center; }
.btn:disabled   { opacity: 0.45; cursor: not-allowed; }
```

### Tab navigation

Active tab: `background: #e8ecf2; color: #0d2137; font-weight: 600; border-radius: 6px 6px 0 0`
Inactive tab: `color: rgba(255,255,255,0.52); background: transparent`

The Dataset tab should show a badge with the total record count across all questionnaires.

### Toast

```css
#toast {
  position: fixed; bottom: 1.5rem; right: 1.5rem;
  background: #1e293b; color: #fff;
  padding: 0.625rem 1rem; border-radius: 8px;
  font-size: 13px; font-weight: 500;
  opacity: 0; transform: translateY(8px);
  transition: opacity 0.2s, transform 0.2s;
  pointer-events: none; z-index: 9999;
}
#toast.show { opacity: 1; transform: translateY(0); }
```

---

## 9. Questionnaire Configuration

### Built-in questionnaires (hardcoded)

These are always available and cannot be deleted:

```js
const BUILTIN_QUESTIONNAIRES = [
  {
    id: 'ibd-demo',
    name: 'IBD Consultatie',
    code: 'IBD',
    url: 'http://templates.tiro.health/templates/def02a77523d472fabfa949e9522e651|1.0.1',
    viewDefinition: { /* IBD ViewDefinition object */ },
  },
  // Additional built-in questionnaires will be added by providing their URL
];
```

### User-added questionnaires (runtime)

Provide a small "Add questionnaire" UI (e.g. a modal or inline form in the toolbar) where the user can paste a Tiro template URL and give it a name and code. Save to `localStorage` under `QS_CONFIG_KEY`.

---

## 10. Event Flow Summary

```
User pastes text into <tiro-magic-clipboard> textarea
  -> User clicks <tiro-magic-clipboard-button>
    -> SDK sends text + questionnaire context to AI
    -> SDK autofills <tiro-form-filler id="capture-form">
    -> tiro-form-filler fires "tiro-response-change"
      -> JS renders updated QR to plain text in #result-text

User edits form manually
  -> tiro-form-filler fires "tiro-response-change"
    -> JS renders updated QR to plain text in #result-text

User clicks "Opslaan in dataset"
  -> JS calls filler.getQuestionnaireResponse()
  -> Attaches _entryId and meta.extension
  -> Saves to localStorage
  -> Updates tab badge
  -> Shows toast

User switches questionnaire via dropdown
  -> Updates filler.setAttribute('questionnaire-url', ...)
  -> Calls filler.reset()
  -> Clears result text
  -> Generates new entry ID

User clicks PiP button
  -> Opens Document PiP window
  -> Moves #pip-content-slot (containing tiro-form-filler) into PiP window
  -> On PiP close: returns element to main page

User opens Data tab
  -> Reads records from localStorage for active questionnaire
  -> Runs sof-js ViewDefinition or generic extractor
  -> Renders table
```

---

## 11. Implementation Checklist

- [ ] Single `index.html` file, no build step
- [ ] All three required scripts loaded from correct CDNs
- [ ] Open Sans font loaded
- [ ] Header with Tiro logo (navy background, blue logo square)
- [ ] Tab nav: "Invoer" (default active) and "Data" tabs
- [ ] Three-column capture layout (clipboard | form | result)
- [ ] Questionnaire selector dropdown in toolbar
- [ ] `<tiro-magic-clipboard for="capture-form">` wrapping a `<textarea>` and `<tiro-magic-clipboard-button>`
- [ ] `<tiro-form-filler id="capture-form">` with correct `sdc-endpoint-address`
- [ ] `tiro-response-change` listener driving the result text panel
- [ ] Copy-to-clipboard button in result column header
- [ ] PiP button in form column header (graceful degradation if API unsupported)
- [ ] "Nieuw record" button resets form and generates new entry ID
- [ ] "Opslaan in dataset" button saves QR to localStorage
- [ ] `tiro-populate-error` event handler shows toast
- [ ] Data tab with questionnaire selector, table, CSV/XLSX export, and clear buttons
- [ ] sof-js ViewDefinition evaluation for known questionnaires
- [ ] Generic row extractor fallback for questionnaires without ViewDefinition
- [ ] Dataset tab badge showing total record count
- [ ] Toast notification system
- [ ] Responsive within a desktop viewport (no mobile layout required)
- [ ] Dutch language throughout the UI (`lang="nl-BE"` on `<html>`)

---

## 12. Known Patterns from the Existing Codebase

The existing `index.html` demonstrates several patterns to reuse:

- **FHIRPath evaluation** for ViewDefinition-style column extraction (see `evalPath()` function and `IBD_COLUMNS` array) — the new implementation should use sof-js instead, but the column definitions can be preserved as ViewDefinition JSON.
- **Tab switching** via `data-target` attributes on `.tab-btn` elements and `[hidden]` on panel divs.
- **`tiro-form-filler` attribute mutation** to switch questionnaires: `el.setAttribute('questionnaire-url', newUrl)` triggers the SDK to reload.
- **Entry ID chip** as a styled `<span>` with font-variant-numeric for tabular display.
- **`localStorage` prefix pattern** (`tiro_cr_records_` + study ID) to namespace per-questionnaire storage.
- **SheetJS** is already included and used for XLSX export in the existing file — reuse the same pattern.
- **Spinner animation** on the autofill button is handled automatically by the SDK via `data-state="pending"` — just style the CSS states.

---

## 13. What NOT to Do

- Do not use `localStorage` for API keys or sensitive data.
- Do not add a "Design" or "Ontwerp" tab for questionnaire creation — this app is capture-only. Questionnaires are configured via the dropdown with known URLs.
- Do not attempt to host or proxy the FHIR SDC endpoint — always point `sdc-endpoint-address` to `https://sdc.tiro.health/fhir/r5`.
- Do not use `innerHTML` injection for user-provided content without escaping.
- Do not use inline `style=""` attributes for layout — use CSS classes.
- Do not use `em-dash` (`—`) characters in any UI-visible text strings.
- Do not use `document.write()`.
- Do not hardcode questionnaire responses or mock FHIR data.

---

## 14. Testing Notes

Since this is a browser-only app with external SDKs, manual testing is required:

1. Open `index.html` directly in Chrome (required for Document PiP API).
2. Paste a clinical note into the left column textarea.
3. Click the autofill button and verify the form fills.
4. Edit a field manually and verify the result text updates.
5. Save a record and verify the Data tab shows a row.
6. Switch questionnaires and verify the form resets and the new questionnaire loads.
7. Test the PiP button in Chrome; verify graceful degradation in Firefox.
8. Export CSV and verify columns match the ViewDefinition.
