# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Vorpic Flow is a **zero-dependency, single-file web application** for SOP generation and knowledge base management — the entire app lives in `index.html` (~1,070 lines). No build step, no package manager, no framework. Part of the Vorpic Operations Suite by Hagans Consulting.

## Running the App

Open `index.html` directly in a browser, or serve with any static server:

```
npx serve .
# or
python -m http.server
```

There are no tests, no lint commands, and no build process.

## Architecture

### Single-File Structure

All HTML, CSS, and JavaScript is in one `index.html` file. Sections are delimited by comments:
- `<!-- STYLES -->` — CSS custom properties + all component styles
- HTML body — header, tabs nav, generator panel, library panel, modals, toast container
- `<!-- SCRIPT -->` — all application logic

### Global State

```js
let darkMode       = true;
let currentSOPRaw  = '';    // raw AI text of the active SOP
let currentSOPMeta = null;  // { id, title, category, created_at, ... } if loaded from library
let attachedFile   = null;  // { name, text } from file upload
```

### Persistence

**IndexedDB** (`vorpic_flow_db`, object store `sops`) — accessed exclusively through `SOPStorageService`. Never write to IndexedDB directly.

| Key | Contents |
|-----|----------|
| `vorpic_flow_api_key` | Anthropic API key (localStorage) |
| `vf-dark` | Dark mode boolean (localStorage) |

`SOPStorageService` is an IIFE exposing `saveSOP(sop)`, `getSOP(id)`, `getAllSOPs()`, `deleteSOP(id)`. All write operations are wrapped in try/catch; any failure (including `QuotaExceededError`) calls `showQuota()` to surface the storage-limit modal.

### SOP Record Schema

```js
{
  id:         crypto.randomUUID(),  // uuid
  title:      string,
  category:   string,               // General | HR | IT | Operations | Safety | Finance | Custom
  content:    string,               // raw AI-generated text
  created_at: ISO string,
  updated_at: ISO string,
  owner_id:   null                  // reserved for future cloud sync
}
```

### Tab System

Two tabs: **SOP Generator** and **Document Library**. `switchTab(name)` toggles `.active` on the pane and button. The generator tab uses a CSS grid split: 360px input panel left, flexible output panel right.

### Anthropic API Integration

`handleGenerate()` calls `https://api.anthropic.com/v1/messages` directly from the browser using the `anthropic-dangerous-direct-browser-access: true` header. Model: `claude-sonnet-4-20250514`, max_tokens: 1000. The system prompt (`SYSTEM_PROMPT` constant) instructs Claude to output exactly five bold-header sections: **Purpose**, **Scope**, **Responsibilities**, **Procedure**, **Notes**.

`renderSOPDoc()` parses that output with a regex (`/\*\*([^*]+)\*\*\s*\n([\s\S]*?)(?=\n\*\*|$)/g`) and renders each section into `.sop-section` divs. The Procedure section is always rendered as an `<ol>`.

### Show/Hide Pattern

Elements are toggled with the `.hidden` CSS class (`display: none !important`) — never via inline `style=""`. Action buttons (Save, Copy, Print) start hidden and are revealed by `showOutputActions()` after a successful generation.

## Key Conventions

- **XSS prevention**: All user-supplied strings rendered into HTML must go through `esc(str)`. Never interpolate raw data into innerHTML.
- **No inline styles**: Design tokens belong in the `<style>` block; functional show/hide uses the `.hidden` class, not `style=""` attributes.
- **Prefer str_replace patches over full rewrites** when editing `index.html`.
- **`currentSOPMeta`**: When `null`, `handleSave()` creates a new record. When set (loaded from library or after first save), subsequent saves update the existing record in place.

## Design System

CSS custom properties on `:root`. Light mode via `[data-theme="light"]` attribute swap. Primary accent is green (`--accent: #00e5a0`). Typography: IBM Plex Sans (body), IBM Plex Mono (code/textarea/inputs). Full token list: `--bg`, `--bg2`, `--bg3`, `--border`, `--border2`, `--text`, `--text2`, `--text3`, `--accent`, `--accent2`, `--accent-dim`, `--amber`, `--red`, `--blue`, `--purple`.

## Conventions & Constraints

- **No inline styles** — all design tokens in the `<style>` block; show/hide via `.hidden` class only.
- **No modal close on outside click** — modals close via close button or Cancel only.
- **Prefer str_replace patches over full rewrites** when editing `index.html`.
- **Suite context**: Part of the Vorpic Operations Suite. HomeDeck (`vorpic-homedeck`) is the suite hub. All inter-app navigation is same-tab (no `target="_blank"`).
- **Commit messages**: Include version number and a brief changelog summary (e.g. `v0.2 — export to PDF, folder tags`).
