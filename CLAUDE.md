# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`rsvn-admin` is the **admin frontend** for a hotel reservations-team knowledge base
(訂房組知識庫). It is a single static HTML file (`index.html`, ~2,700 lines: inline
`<style>` + inline `<script>`, no external JS/CSS files beyond two CDN libraries).
There is no build step, no package manager, no framework, and no test suite — the
entire "codebase" is that one file.

The UI text, code comments, and domain vocabulary are almost entirely **Traditional
Chinese (zh-TW)**. Match that when editing user-facing strings or comments.

## Deployment / workflow

- `.github/workflows/static.yml` deploys the **entire repo root as-is** to GitHub
  Pages on every push to `main` (or manual `workflow_dispatch`). There is no build
  step in CI — whatever is committed is what gets served.
- "Development" means editing `index.html` directly and pushing to `main`; there is
  no local dev server, linter, or test command to run.
- The file starts with `<!-- VERSION-CHECK: admin-v3-clean-YYYY-MM-DD -->`. Bump this
  comment when making a meaningful change, since the page is also loaded inside a
  Google Apps Script (GAS) iframe where cache-busting is otherwise hard to verify.

## Architecture: this repo is only half the system

This page is a **client for a separate Google Apps Script (GAS) web app backend**
that is *not* part of this repository. The backend's source lives elsewhere (a GAS
project); this repo only knows its deployed `/exec` URL, hardcoded as `GAS_URL` near
the top of the `<script>` block.

- All backend calls go through `gasCall(action, payload)`, which does a single
  `fetch(GAS_URL, { method: 'POST', headers: {'Content-Type': 'text/plain'}, body:
  JSON.stringify({action, ...payload}) })` and parses the JSON response. `text/plain`
  is deliberate — it avoids a CORS preflight against the GAS endpoint.
- The action names called from this file **are** the API contract with the backend:
  `getCategories`, `addCategory`, `renameCategory`, `deleteCategory`,
  `moveCategoryOrder`, `getData`, `getExistingTitlesAndKeywords`,
  `getExistingForSemanticCheck`, `saveKnowledge`, `getDeleteLog`,
  `deleteKnowledgeBatch`. If you rename/add a payload shape here, the GAS side must
  change in lockstep (out of reach in this repo — flag this to the user).
- Gemini AI calls (`analyzeDirectly`) go **directly from the browser** to
  `generativelanguage.googleapis.com` using `gemini-2.5-flash`, bypassing the GAS
  backend entirely (a comment notes `fetch()` used to be blocked inside the GAS
  iframe sandbox, hence the direct-from-browser design). The user pastes their own
  Gemini API key into the UI; it is kept only in `sessionStorage` (`gemini_api_key`)
  and never sent to the GAS backend.

## Client-side document ingestion (why mammoth.js / SheetJS are here)

`.docx` and `.xlsx` files are converted to plain text **in the browser** (via
`mammoth.js` and `SheetJS`/`xlsx.full.min.js`, both loaded from CDN) before being
sent to Gemini as prompt text, instead of being sent as base64 file/inline_data.
This is intentional: Gemini's free tier commonly has a **zero quota for
multimodal/file input**, but a normal quota for plain-text prompts. Images
*embedded inside* a `.docx` are extracted separately and still sent as
`inline_data` parts (small volume, not treated as "file input"). Preserve this
split when touching file-upload/analyze code — don't "simplify" it by sending
docx/xlsx as raw file parts.

## Data model: the "knowledge card"

The unit of data is a knowledge card row, shaped like this when submitted
(`submitAll()` → `rows`):

```js
{ date, category, title, content, image, imageName, link, keywords, memo, expiry }
```

- `category` is a comma-joined string of one or more **hierarchical category
  paths** in `"Parent/Child"` form (e.g. `"交通/接機"`); categories with no
  meaningful child use the parent name alone. The full valid category list is
  fetched from the backend (`getCategories`) — never invent new category strings
  in code, they must come from that list or from the category-management UI.
- `expiry` has no fallback: an empty string means "no expiry," and the per-card
  value shown on screen is submitted verbatim (it is *not* silently overwritten by
  the memo-level default expiry after card creation — that default is only used
  once, at `addItem()` time, to pre-fill a new blank card).
- Reference images are per-card (`item{index}_{originalFilename}`) as well as
  memo-level (bulk-uploaded) files; the backend matches uploaded files back to
  rows by filename, so filename collisions between the two must be avoided (hence
  the `item{i}_` prefix).

## Page structure (single file, three tabs, no router)

`switchTab('add' | 'cat' | 'delete')` toggles visibility between three panels in
the same DOM — there is no routing/history, just show/hide:

1. **新增知識卡 (Add)** — `#tabPanelAdd`: two-column layout. Left = the list of
   knowledge cards being drafted (`items` array + `renderItems()`). Right =
   optional "Memo" helper that can be analyzed by Gemini to auto-generate cards,
   or filled in manually.
2. **類別管理 (Category management)** — `#tabPanelCat`: add / rename / move / delete
   categories, gated by a hardcoded client-side password constant
   (`CAT_MANAGE_PASSWORD`) checked against `sessionStorage` — this is a soft UI
   gate only, not real access control (the check and the password both ship to
   every browser).
3. **知識卡管理 (Card management)** — `#tabPanelDelete`: filterable table of
   existing cards (category/staff/date-range/expiry-status filters), batch
   delete, CSV export, and a deletion log sub-tab (`getDeleteLog`).

`goAddCategory(idx)` / `returnToTriggerCard()` implement a "jump to category tab,
then jump back to the card you came from" flow — when editing this, keep the
return-banner state (`returnToCardIdx`) in sync or the return trip breaks.

## State & persistence conventions

- `localStorage.kb_last_staff_name` — remembers the last-used staff name across
  sessions (prefilled, always editable, never hidden).
- `sessionStorage.gemini_api_key` — the user's pasted Gemini key; per-tab, cleared
  on browser close, never transmitted to the GAS backend.
- `sessionStorage.kb_catmanage_authed` — category-management gate flag.
- `localStorage` draft autosave (`saveDraft`/`tryRestoreDraft`/`clearDraft`,
  `DRAFT_KEY`) — periodically persists in-progress card text fields (title,
  content, keywords, dates, categories) so a refresh doesn't lose typed work.
  Deliberately **excludes** file/image binary data (base64 is large and is input
  data the user can just re-attach, not something worth persisting).

## Conventions to preserve when editing

- Comments in this file are almost all "why," not "what" — many document a
  specific bug or constraint that was worked around (e.g. GAS iframe fetch
  blocking, Gemini free-tier file-input quota, flatpickr being unreliable in the
  GAS iframe sandbox → replaced with native `<input type=date>` + a custom
  `date-combo` text field). Don't remove these without understanding the
  constraint they record, and don't reintroduce the same pattern (e.g. flatpickr).
- AI category assignment is always filtered against the real category set
  (`catSet.has(c)`) before being trusted, to guard against model hallucination;
  keep that guard if touching `analyzeDirectly`.
- Conflict detection before submit (`checkConflicts`) is a local, heuristic
  title/keyword similarity check (cheap, synchronous) distinct from
  `checkSemanticConflicts` (an optional Gemini-backed deeper check) — both exist
  because the cheap check can't catch "same idea, different words."
