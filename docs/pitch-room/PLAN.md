# Pitch Room — Overnight Execution Plan

This is the task list for the autonomous overnight run. Read `SPEC.md` first — it is
the source of truth for behavior, taxonomy, contracts, and copy. This file is the
source of truth for **order of work, acceptance criteria, and the loop protocol**.

## Loop protocol (read every iteration)

1. Work tasks **strictly top to bottom**. One task per iteration.
2. Before starting a task, re-read its acceptance criteria and the SPEC sections it cites.
3. After finishing: run the task's verification, tick its checkbox here, append one
   line to `docs/pitch-room/EXECUTION-LOG.md` (`T1.2 done — <one-line note>`), then
   commit **only the files that task touched** with message
   `feat(pitch-room): T<id> <short description>` and push with
   `git push -u origin claude/pitch-room-setup-planning-ku79fw`.
4. If a task is blocked after 2 honest attempts: log `T<id> BLOCKED — <reason>` in the
   execution log, tick it as `[b]` (blocked) here, commit, and move to the next task.
   Never let one task stall the run.
5. **Stop conditions:** all tasks ticked; or 3 consecutive blocked tasks; or any task
   would require a secret, credential, or action outside this repo.

## Guardrails (hard rules)

- Never commit secrets. The Supabase anon key already in `index.html` is public by
  design; the `ANTHROPIC_API_KEY` must never appear anywhere in this repo.
- Do not modify `skills/B0.1b-christensen-lakhani-lite.md` (sync convention).
- Do not remove or break anything existing in `index.html`: the copy-skill funnel,
  email capture, `logEvent`, footer, meta tags. Additions only.
- No frameworks, no build step, no external CDN scripts. Vanilla HTML/CSS/JS only.
- Do not deploy anything to Supabase. Phase 2 produces files + docs only.
- Push only to `claude/pitch-room-setup-planning-ku79fw`.

## Verification harness

Static site — verify with a local server and DOM checks:

```bash
cd /home/user/board-conductor && python3 -m http.server 8020 &
curl -s http://localhost:8020/pitch-room.html | grep -c "<section"   # smoke
```

Preferred when available: Playwright (Chromium preinstalled at
`/opt/pw-browsers/chromium`) — write throwaway scripts in the scratchpad, not the repo.
Each task lists the specific assertions. If Playwright is unavailable, fall back to
`curl` + `grep` structural checks and note it in the execution log.

---

## Phase 1 — Working pitch room, zero backend

- [ ] **T1.1 Scaffold `pitch-room.html`** — page shell with nav, footer, hero
  ("Your Pitch Room"), copied `:root` design tokens from `index.html`, empty
  placeholder sections in order: intake, upload, checklist, layout, AI panel.
  *Accept:* page renders locally, no console errors, visually consistent dark theme,
  nav links back to `index.html`.
  *Verify:* serve + open; assert title, 5 placeholder sections, zero console errors.

- [ ] **T1.2 Room lifecycle + intake** — SPEC §4.1. UUID room in
  `localStorage.bc_pitch_room_v1`; restore on revisit; "Start over" with confirm;
  optional intake fields (company, stage, one-liner, wedge mode) persisted on change.
  *Accept:* reload preserves state; Start over clears it; all intake fields optional.
  *Verify:* Playwright — set fields, reload, assert values; click Start over, assert reset.

- [ ] **T1.3 Taxonomy module + checklist UI** — SPEC §5, §4.3. Embed the taxonomy as
  a JS const (ids exactly as SPEC). Render checklist filtered by stage with
  present / missing-required / missing-recommended states, "mark as present" toggle
  per row, and readiness meter.
  *Accept:* changing stage re-renders requirements per SPEC table; readiness math =
  present-required / total-required; states distinguishable without color (icons/labels).
  *Verify:* Playwright — pre-seed shows financial_model as recommended, series-a as
  required; toggling 4 required docs at seed moves the meter accordingly.

- [ ] **T1.4 Upload zone** — SPEC §4.2. Drag-drop + picker, metadata-only storage,
  filename→category auto-suggest with override select, per-file remove, file list UI,
  prominent "files never leave your browser" note.
  *Accept:* dropping `acme-pitch-deck.pdf` auto-suggests `pitch_deck`; category
  override updates checklist; remove works; state survives reload.
  *Verify:* Playwright — programmatic file input with 3 named fixtures, assert
  suggested categories and checklist updates.

- [ ] **T1.5 Baseline layout rendering** — SPEC §7, §4.5. Deterministic layout from
  wedge mode + present docs, rendered as ordered section cards with doc chips
  (present vs missing styling).
  *Accept:* Discovery vs Execution produce the SPEC §7 orders; "not sure" = Execution;
  chips reflect live checklist state.
  *Verify:* Playwright — switch wedge mode, assert section titles and order.

- [ ] **T1.6 AI prompt generator** — SPEC §4.4 mode 1, §6. Build the prompt string
  (intake + inventory + condensed four-lens framing + the §6 JSON contract verbatim),
  copy to clipboard with the same success-state UX as `copySkill()` in `index.html`.
  *Accept:* prompt contains all present AND missing category ids, the JSON contract,
  and instructions to return a fenced json block; works with empty intake.
  *Verify:* expose the generated string (e.g. `window.__lastPrompt` in a debug hook or
  read clipboard via Playwright permissions) and assert required substrings.

- [ ] **T1.7 Response parser + AI layout rendering** — SPEC §4.4 mode 1, §4.5, §6.
  Paste panel; extract first fenced json block; validate per §6 rules; on success
  render AI layout (replacing baseline, with "AI-recommended" badge), merged missing
  docs with priorities/reasons, red flags (max 3), readiness. On failure: friendly
  error, keep baseline.
  *Accept:* valid sample renders fully; invalid json and json violating rules
  (unknown category id, 12 sections) are rejected gracefully; deterministic
  missing-required entries always survive the merge.
  *Verify:* Playwright — paste 3 fixtures (valid / malformed / rule-violating),
  assert rendered vs error states.

- [ ] **T1.8 Main page entry point** — SPEC §3. Hero CTA button to `pitch-room.html`
  above existing content, 3-bullet explainer section, `pitch_room_cta_click` event.
  *Accept:* existing funnel untouched (copy button, email form, all events still
  present in source); CTA visible above the fold.
  *Verify:* diff shows only additions to `index.html`; serve + click CTA lands on
  pitch room; grep confirms `copySkill`, `submitEmail`, `logEvent` intact.

- [ ] **T1.9 Analytics events** — SPEC §4.6. Port `logEvent()` into `pitch-room.html`
  with `skill_id: "PITCH-ROOM"`; fire the 7 listed events; never log filenames or
  free-text.
  *Accept:* events fire at the right moments (inspect network or a debug hook);
  payload metadata contains only category ids / counts.
  *Verify:* Playwright request interception on `/rest/v1/e_skill_events`, assert
  event types and absence of filename strings.

- [ ] **T1.10 Mobile + accessibility pass** — SPEC §9.
  *Accept:* usable at 375px width (single column, no horizontal scroll); keyboard-only
  walkthrough reaches every control; dropzone and selects have aria-labels; focus
  visible.
  *Verify:* Playwright at 375×812 — assert no horizontal overflow; tab-order script
  touches upload, checklist toggles, copy button, paste panel.

## Phase 2 — Backend scaffold (files only, no deployment)

- [ ] **T2.1 SQL migration** — `supabase/migrations/0001_pitch_rooms.sql` per SPEC §8:
  three tables, indexes on `token` and `room_id`, RLS enabled, no anon policies,
  comment header explaining the token-through-edge-function model.
  *Accept:* valid PostgreSQL (parseable), matches SPEC schema exactly.
  *Verify:* `psql` unavailable — self-review + comment; log as verified-by-review.

- [ ] **T2.2 Edge Function `pitch-room`** — `supabase/functions/pitch-room/index.ts`
  per SPEC §8: create room, get by token, add/delete document with signed upload URLs.
  Deno-style imports, CORS for `board.vsfoundry.com` + localhost, input validation,
  no secrets.
  *Accept:* type-consistent TypeScript, all routes token-gated except create,
  errors are JSON with status codes.
  *Verify:* `deno check` if deno available; otherwise careful self-review; log which.

- [ ] **T2.3 Edge Function `pitch-room-review`** — SPEC §8 + §6: fetch room docs,
  call Claude API `claude-sonnet-5` with tool-forced output matching the §6 schema,
  persist to `pr_reviews`, return JSON. Reads `ANTHROPIC_API_KEY` from env only.
  *Accept:* schema in the tool definition matches §6 field-for-field; timeout and
  API-error paths return structured errors.
  *Verify:* as T2.2.

- [ ] **T2.4 Frontend flag wiring** — in `pitch-room.html`: `BACKEND_ENABLED = false`;
  when true, "Get AI recommendation" calls the Edge Functions (upload → review →
  render via the same §6 validator) with the BYO-AI path kept as visible fallback.
  *Accept:* with flag false (default), behavior is byte-identical to Phase 1 UX; the
  backend code path is reachable only via the flag; no dead console errors.
  *Verify:* Playwright — full Phase 1 regression (T1.5–T1.7 assertions) with flag false.

- [ ] **T2.5 `DEPLOYMENT.md`** — `docs/pitch-room/DEPLOYMENT.md`: exact manual steps —
  `supabase db push`, create private bucket `pitch-rooms`, deploy both functions,
  `supabase secrets set ANTHROPIC_API_KEY=...`, flip `BACKEND_ENABLED`, smoke-test
  checklist. Address it to a human.
  *Accept:* a person who has never seen this repo could deploy from it.

## Phase 3 — Docs + wrap-up

- [ ] **T3.1 README update** — add a "Pitch Room" section (what it is, link,
  Phase 1 privacy model, pointer to docs/pitch-room/). Do not disturb the skills
  table or publishing arc.
- [ ] **T3.2 Final regression + log** — run every Phase 1 verification once more
  end-to-end; write a closing summary in `EXECUTION-LOG.md` (what shipped, what's
  blocked, what a human must do next); final commit + push.

---

## Review of current setup (context for the executor and reviewers)

Findings from the planning pass (2026-07-08):

1. **There is no pitch room today.** The nearest thing is the copy-paste skill funnel
   on `index.html`, which requires leaving the site — the friction this project removes.
2. **Supabase is already wired** (`e_skill_events`, anon key in `index.html`) — reuse
   it for analytics now and as the Phase 2 backend home.
3. **The B0.1b skill is the moat** — the taxonomy and layout logic in SPEC §5/§7 are
   derived from its four lenses; keep that linkage when writing copy.
4. **The site is deliberately dependency-free** — preserve that; it is why an
   overnight autonomous run on a static host is feasible at all.
5. Improvement backlog beyond this build is in SPEC §10 — do not build it overnight.
