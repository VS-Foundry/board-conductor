# Pitch Room — Product Specification

Status: **planned** · Owner: V.S. Foundry · Executed by: overnight autonomous run (see `PLAN.md`)

---

## 1. Problem & current state

Today, board-conductor is a single-page static site (`index.html`) whose only path to value is:

1. Copy the B0.1b skill prompt to clipboard
2. Open Claude / ChatGPT in another tab
3. Paste the prompt
4. Attach a deck
5. Read the review in the other tool

This is the "onboarding process" we want to remove. There is no pitch room today —
no upload surface, no document checklist, no persistent workspace, no layout guidance.
The site captures emails and logs events to Supabase (`e_skill_events`), but a startup
gets nothing interactive on the page itself.

**Goal:** a startup lands on the main page, clicks one button, and is inside their own
Pitch Room — no signup, no email, no copy-paste ritual. They add their materials, and
AI tells them (a) the best layout for their pitch room and (b) which documents are missing.

## 2. Non-goals (v1)

- No user accounts, passwords, or OAuth.
- No multi-user collaboration or investor-facing share links (v2 candidate).
- No server-side file storage in Phase 1 (browser-only; Phase 2 adds Supabase Storage).
- No payment or gating.
- No editing of skill file `skills/B0.1b-*.md` shared sections (sync convention in that file header).

## 3. Entry point (main page changes)

`index.html` gets:

- A primary hero CTA above the existing content: **"Set up your Pitch Room →"**
  linking to `pitch-room.html`. The existing copy-skill funnel stays intact below it.
- A short 3-bullet section explaining the Pitch Room (instant, private, AI-guided).
- New analytics events via the existing `logEvent()` helper:
  `pitch_room_cta_click` on the CTA.

## 4. Pitch Room page (`pitch-room.html`)

Self-contained single file (inline CSS/JS), same design system as `index.html`
(reuse the `:root` CSS variables verbatim). No build step, no framework, no external
CDNs (GitHub Pages + privacy posture).

### 4.1 Room lifecycle — zero onboarding

- On first visit, a room is created instantly: `crypto.randomUUID()`, stored in
  `localStorage` under `bc_pitch_room_v1` together with all room state.
- Returning visits restore the room automatically. A "Start over" control clears it
  (with confirm).
- Optional, inline, skippable intake (4 fields, all optional): company name, stage
  (pre-seed / seed / series-a), one-line description, wedge mode
  (Discovery / Execution / not sure). These sharpen the checklist and the AI prompt.

### 4.2 Materials upload

- Drag-and-drop zone + file picker, multiple files.
- **Phase 1: files never leave the browser.** Only metadata is kept (name, size, type,
  user-assigned category). State the privacy guarantee prominently — it is a feature.
- Each file gets a category from the taxonomy (§5). Auto-suggest by filename heuristics
  (e.g. `/deck|pitch/i → pitch_deck`, `/cap.?table/i → cap_table`,
  `/model|financ|budget/i → financial_model`, `/transcript|call|notes/i → customer_transcripts`);
  the user can override via a select. "Mark as present without uploading" is allowed per
  category (a doc may live elsewhere).
- File cap: 25 files, 50 MB each (metadata only in Phase 1, so limits are advisory UX).

### 4.3 Document checklist ("what's missing") — deterministic layer

Rendered live from the taxonomy (§5) filtered by stage. Three states per category:
**present** (uploaded or marked), **missing — required**, **missing — recommended**.
A readiness meter: `present required / total required` as a percentage, styled like the
existing scorecard language ("Room readiness: 67%").

This layer works with zero AI and zero network — it is the baseline guarantee that
"which documents are missing" always has an answer.

### 4.4 AI layout recommendation

Two modes, one contract:

**Phase 1 — BYO-AI (ships first, no backend):**
1. "Get AI recommendation" builds a tailored prompt embedding: intake answers, the
   document inventory (categories present/missing), the condensed four-lens framework,
   and the required output contract (§6). Copies to clipboard, same UX as the existing
   Copy Full Skill button; user pastes into Claude/ChatGPT and attaches their files.
2. A "Paste the AI's response" panel accepts the response. The page extracts the fenced
   ```json block, validates it against the contract, and renders the recommended room
   layout visually (§4.5). Invalid/absent JSON → show a friendly error + the raw text.

**Phase 2 — one-click (backend, behind flag):** the same contract is fulfilled by a
Supabase Edge Function calling the Claude API directly (files uploaded to Supabase
Storage first). Controlled by `const BACKEND_ENABLED = false;` until deployed.
The Phase 1 path remains as fallback forever.

### 4.5 Layout rendering

The recommended layout renders as an ordered list of section cards: section title,
one-line curator note, chips for the documents assigned to it (present = normal,
missing = dashed/amber with "add this" affordance). Below it: the missing-documents
list from the AI (merged with the deterministic checklist — AI can only add reasons
and priorities, never mark a missing required doc as fine) and up to 3 red flags.

A deterministic **baseline layout** (from §7) is shown even before any AI call, so the
page is never empty.

### 4.6 Analytics

Reuse `logEvent()` posting to `e_skill_events` with `skill_id: "PITCH-ROOM"`:
`pitch_room_view`, `room_created`, `file_added` (category, no filename),
`checklist_complete`, `prompt_copied`, `response_parsed`, `layout_rendered`.
Never log filenames, file contents, or intake free-text.

## 5. Document taxonomy

Category ids are stable API — used in code, prompts, and the AI contract.

| id | Label | pre-seed | seed | series-a | Lens |
|---|---|---|---|---|---|
| `pitch_deck` | Pitch deck | required | required | required | Wedge |
| `one_pager` | One-pager / exec summary | required | required | required | Wedge |
| `financial_model` | Financial model / budget | recommended | required | required | Board Conductor |
| `cap_table` | Cap table | recommended | required | required | Board Conductor |
| `traction_metrics` | Traction / KPI summary | recommended | required | required | Learning Engine |
| `team_bios` | Team & founder bios | required | required | required | Board Conductor |
| `customer_transcripts` | Customer / pitch call notes | recommended | recommended | recommended | Wedge |
| `learning_log` | Experiment / learning log | recommended | recommended | recommended | Learning Engine |
| `data_assets` | Data asset inventory | — | recommended | required | Data Laboratory |
| `competitive_landscape` | Competitive analysis | recommended | recommended | required | Wedge |
| `round_terms` | Round details / terms | — | recommended | required | Board Conductor |
| `use_of_funds` | Use of funds | recommended | required | required | Board Conductor |
| `product_demo` | Product demo (video/link) | recommended | recommended | recommended | Wedge |
| `legal_pack` | Legal pack (incorporation, IP) | — | recommended | required | Board Conductor |

The `customer_transcripts` / `learning_log` / `data_assets` rows are the
Christensen × Lakhani differentiator vs. a generic data room — surface them with a
"most rooms skip these; boards notice" note.

## 6. AI output contract

The prompt (Phase 1) and the Edge Function (Phase 2) both require this exact JSON,
returned in a fenced ```json block:

```json
{
  "wedge_mode": "discovery | execution",
  "stage_note": "one sentence on whether materials match the claimed stage",
  "layout": [
    {
      "section": "wedge_proof",
      "title": "Wedge Proof",
      "docs": ["pitch_deck", "customer_transcripts"],
      "note": "one-line curator note for this section"
    }
  ],
  "missing_documents": [
    { "category": "financial_model", "priority": "high | medium", "why": "one sentence" }
  ],
  "red_flags": ["at most 3 short strings"],
  "readiness": 0
}
```

Rules enforced by the client-side validator: `layout` is 4–8 sections; every `docs`
entry must be a known category id; `readiness` is 0–100; unknown fields ignored;
`missing_documents` is merged with (never replaces) the deterministic checklist.

## 7. Baseline layouts (deterministic, no AI)

**Discovery mode:** 1. The Question (`one_pager`) · 2. Wedge Candidates & Market Pull
(`pitch_deck`, `customer_transcripts`, `competitive_landscape`) · 3. Learning Engine
(`learning_log`, `traction_metrics`) · 4. Team (`team_bios`, `product_demo`) ·
5. The Ask (`use_of_funds`, `round_terms`) · 6. Data Room (`financial_model`,
`cap_table`, `data_assets`, `legal_pack`)

**Execution mode (and "not sure"):** 1. Overview (`one_pager`, `pitch_deck`) ·
2. Wedge Proof (`traction_metrics`, `customer_transcripts`) · 3. Learning Engine &
Data Moat (`learning_log`, `data_assets`) · 4. Financials (`financial_model`,
`use_of_funds`) · 5. Team (`team_bios`, `product_demo`) · 6. The Round (`round_terms`,
`cap_table`) · 7. Data Room (`competitive_landscape`, `legal_pack`)

## 8. Phase 2 backend (scaffolded, deployed manually later)

- **Schema** (`supabase/migrations/0001_pitch_rooms.sql`):
  - `pr_rooms(id uuid pk, token text unique, company_name text, stage text, wedge_mode text, created_at timestamptz)`
  - `pr_documents(id uuid pk, room_id fk, category text, filename text, storage_path text, size_bytes bigint, created_at)`
  - `pr_reviews(id uuid pk, room_id fk, output jsonb, model text, created_at)`
  - RLS enabled on all three with **no anon policies** — all access goes through Edge
    Functions using the service role; the client authenticates with the room `token`.
- **Storage:** private bucket `pitch-rooms`, path `{room_id}/{document_id}/{filename}`.
- **Edge Functions** (Deno / TypeScript):
  - `pitch-room` — POST create room (returns id+token), GET room by token,
    POST/DELETE documents (signed upload URLs), all token-gated.
  - `pitch-room-review` — assembles document text (PDF text extraction best-effort,
    otherwise metadata only), calls the Claude API (`model: "claude-sonnet-5"`,
    tool-forced JSON matching §6), stores in `pr_reviews`, returns the JSON.
- **Secrets:** `ANTHROPIC_API_KEY` via `supabase secrets set` — **manual step**, never
  committed. See `DEPLOYMENT.md` (written in Phase 2).
- Frontend flag `BACKEND_ENABLED` stays `false` until a human completes DEPLOYMENT.md.

## 9. Design constraints

- Match `index.html`: dark theme tokens, same fonts, `--radius`, `.btn` patterns.
- Self-contained files; no external requests except the existing Supabase REST calls
  and (Phase 2) Edge Functions.
- Mobile: single column below 640px; drag-drop degrades to file picker.
- Accessibility: all interactive elements keyboard-reachable, focus styles, aria-labels
  on the dropzone and category selects, checklist states not conveyed by color alone.
- Copy tone: matches the site — plain, board-level, no hype.

## 10. Suggested improvements beyond v1 (backlog, do NOT build overnight)

1. Shareable read-only investor link (needs Phase 2 + signed URLs).
2. Client-side PDF text extraction so the BYO-AI prompt can include deck excerpts.
3. Per-lens readiness sub-scores tied to the B0.1b scorecard.
4. Email-capture hook: "email me my missing-docs list" (reuses existing capture).
5. A B-series skill file (`skills/PR0.1-pitch-room.md`) so the pitch room prompt is
   also distributable as a standalone skill, consistent with the publishing arc.
