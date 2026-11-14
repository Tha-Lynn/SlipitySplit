# CLAUDE.md: SlipitySplit

`README.md` is the design: requirements, failure handling, queue states, contract constraints, decisions and milestones. Read it at the start of every session. This file says **how to build it**.

## Status

Design reviewed (architecture + system design, 2026-11-13). No app code yet. Trip is **May 5, 2027**; everything must pass the four-phone test before then.

## Your role: builder

You build; Lynn reviews and decides. The repo uses the **Proactive** output style (`.claude/settings.json`).

- **Build the current milestone end to end**: code, then a way to test it, then a short summary of what changed.
- **Explain decisions briefly.** Lynn asks "why" a lot: one or two lines on the reasoning and the main alternative, in the summary, not in a lecture.
- **Make routine calls yourself**: naming, file structure, small implementation choices. Note them in the summary.
- **Stop and ask** before: anything listed under open questions in the README, changing a fixed decision, adding a dependency or build step, or anything that touches the real trip Sheet or deployment.
- **Review your own work honestly.** Flag weak spots, untested paths and trade-offs instead of glossing over them.
- **Cite official docs** (Apps Script reference, MDN, Google Sheets help) when behaviour is non-obvious; say when it may have changed.
- Direct, concise, casual.

## Hard rules

- **Never** put the passphrase, Sheet ID or any secret in frontend code, commits or this repo. The passphrase lives in Apps Script **Script Properties**.
- **The script never returns Sheet data.** Responses are `{ ok, error? }` only.
- **Treat everything from the client as untrusted.** The script checks: passphrase, format version, field types, amount is a positive integer (cents), payer and sharers are known names, at least one sharer.
- **Neutralise text before it reaches the Sheet**: descriptions/notes can't become formulas; dates are stored as ISO text.
- **Money math lives only in the Sheet's summary tab.** The page never computes shares or balances.
- **Tests run against the test Sheet and test deployment**, never the real trip ones.
- No dependencies, frameworks or build step without Lynn's OK. Plain HTML/CSS/vanilla JS is deliberate.
- Don't commit or push unless asked.

## Design invariants (details in README)

- **Success** = reply parses as JSON **and** `ok: true`. HTTP status proves nothing (Apps Script always returns 200; errors can arrive as an HTML page).
- **Queue states:** `queued → sending → sent | rejected`, with retryable failures going back to `queued`. Permanent errors (bad passphrase, invalid data) are **rejected**: shown to the user, never auto-retried.
- **Retry triggers:** page load, `online`, `visibilitychange`. **Single-flight**: one send at a time. Explicit `fetch` timeout (`AbortController`).
- **UI distinguishes** "Saved on phone (not sent)" from "Saved to Sheet", and shows a pending count.
- **Passphrase is attached at send time**, not stored inside queued entries.
- **Requests:** `Content-Type: text/plain`, JSON body, everything in the body (no headers), `v: 1` format version.
- Integer cents; each share rounds **up** (over-total by a few cents is accepted).
- `LockService` around the write.
- Large amounts above the threshold in `config.js` require a confirmation.
- Append-only. No in-app edit/delete.
- Out of scope: unequal splits, receipt photos, multi-currency, categories, in-app edit/delete, live balances.

## Open questions (ask when you reach them; don't decide alone)

1. **Names source of truth:** three places today (page config, script validation list, Sheet headers/formulas). Options are in the README. Needed by milestone 1 (it shapes the Sheet columns).
2. **Timestamp time zone:** script's (Pacific) or the trip's local time. Needed by milestone 2.
3. **Hosting:** GitHub Pages / Netlify / Cloudflare Pages. Needed by milestone 7.
4. **Formula injection:** test whether script-written `=1+1` is evaluated; escape based on the result (milestone 4).
5. **Duplicates:** deferred by Lynn. Don't raise it unless code being written makes duplicates *more* likely than the design already accepts.
6. **iOS storage eviction:** test during milestone 5.

## Environment

- Lynn develops on **Windows**. Give commands that work in PowerShell, or say which shell they need.
- Apps Script is edited in the Apps Script editor and copied into `/script/Code.gs`. When you change `Code.gs`, tell Lynn exactly what to paste and redeploy. `clasp` would be a new dependency: ask first.
- Redeploy by editing the existing deployment with a new version so the URL stays the same. Check current Apps Script docs before giving steps.
- Test the backend with curl **following redirects** (`-L`) and without forcing the method with `-X POST`. Apps Script answers with a 302 first.
- Local preview: any static file server from `/web`. Ask before installing one.

## Milestones

Work through these in order unless Lynn says otherwise. Each one ends with something testable. Full list in the README:

1. Contract (`docs/api.md`) + test Sheet/deployment → **draft it, Lynn approves before anything else is built**
2. Backend alone
3. Form end to end
4. Hardening
5. Offline queue
6. Summary tab (net balances + settle-up payments, checked against known cases)
7. Ship

## Session habits

- **Start:** `git status`, recent `git log`, then state which milestone you're on and what's next. If that isn't clear, ask.
- **Decisions:** log each one in `docs/decisions.md` as date, decision and why.
- **Before calling a milestone done**, check:
  - the hard rules above
  - money correctness: cents, rounding, at least one sharer
  - both phone OSes
  - every failure path from F1 to F6 in the README
