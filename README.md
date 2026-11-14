# SlipitySplit

A deliberately tiny Splitwise-style web app for one trip: four friends, one link, one shared passcode. Everyone logs purchases from their phone during the trip; the Google Sheet behind it works out who owes whom.

> **Status:** planning done, no code yet. Target: fully tested on all four phones before the trip on **May 5, 2027**.

## How it works

```
Phone (static page)  ──POST──▶  Google Apps Script (doPost)  ──append──▶  Google Sheet
  form + offline queue            checks passcode, locks,                   raw entries tab
                                  adds server timestamp                     + summary tab (balances)
```

| Layer | Choice | Why |
|---|---|---|
| Frontend | Static HTML/JS page, mobile-first (Android + iOS) | Free hosting, no build step, works on any phone browser |
| Backend | Google Apps Script web app (`doPost`) | Free, sits right next to the Sheet |
| Storage | Google Sheet | Free, shared, doubles as the admin panel |
| Access | Unguessable link + one shared passcode, checked server-side | No logins for four friends, still not open to the world |
| Writes | Append-only | No in-app edit/delete; mistakes are fixed by hand in the Sheet |

## The form

| Field | Notes |
|---|---|
| Date | Entered by the user |
| Who paid | Dropdown of the four names |
| Amount (CAD) | Stored as integer cents |
| Description | Free text |
| Shared by | Four checkboxes, all ticked by default; untick to opt out. Split = amount ÷ number ticked |
| Notes | Optional |
| Needs review | Checkbox to flag an entry for checking later |

The script also adds a **server timestamp**, and the Sheet has a **reviewed** column that's toggled by hand.

Names are placeholders for now (`AA`, `BB`, `CC`, `DD`) and live in a single config spot so they're easy to swap.

## Decisions

- **Rounding:** each share rounds *up*. Shares can total a few cents more than the purchase; that's accepted.
- **Money:** integer cents everywhere, never floats.
- **Spotty signal:** unsent entries queue in `localStorage` and retry, with a clear success/failure message.
- **Simultaneous writes:** guarded with Apps Script `LockService`.
- **Passcode:** kept in Apps Script **Script Properties**, never in frontend code or this repo.
- **Backup:** Google Sheets version history; daily manual export if that turns out to be insufficient.

## Out of scope

Unequal splits · receipt photos · multi-currency · categories · in-app edit/delete · live balances in the app.

## Open questions

1. **Duplicates** (deferred). The retry queue can create duplicate rows if a request succeeds but its response is lost. Parked for now; a client-generated ID per entry is the likely fix if it comes back.
2. **Apps Script + `fetch`:** how to handle CORS and redirects (one option: POST as `text/plain` to skip the preflight).
3. **Hosting:** GitHub Pages, Netlify or Cloudflare Pages. (Pages from a *private* repo may need a paid GitHub plan; check.)
4. **Summary tab:** net balances only, or also the minimum set of payments to settle up.
5. **iOS:** Safari can clear `localStorage` for sites that haven't been used in a while; test it.
6. **End-to-end test** on all four phones, both OSes, including airplane mode → queue → retry.

## Repo layout (planned)

```
/web      static frontend (index.html, app.js, config.js for names)
/script   Apps Script source (Code.gs)
/docs     notes, decisions, Sheet setup
```

## Working with Claude

See [`CLAUDE.md`](CLAUDE.md). Short version: Claude is a tutor and reviewer on this project, not the builder.
