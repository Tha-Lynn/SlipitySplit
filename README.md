# SlipitySplit

A deliberately tiny Splitwise-style web app for one trip: four friends, one link, one shared passphrase. Everyone logs purchases from their phone during the trip; the Google Sheet behind it works out who owes whom.

> **Status:** design reviewed, no code yet. Target: fully tested on all four phones before the trip on **May 5, 2027**.

## Requirements

| | Target |
|---|---|
| Users | 4, all trusted |
| Load | ~10–30 entries/day, a few hundred rows total. Scaling is a non-issue; don't design for it |
| Latency | A few seconds per save is fine |
| Availability | Must work offline; the backend may be unreachable for hours |
| **Durability** | **Once the app says "Saved to Sheet", the entry is never lost.** "Saved on phone" is weaker and the UI says so |
| Delivery | At-least-once: an entry may arrive twice (duplicates deferred), never zero times |
| Cost | $0 |
| Data lifetime | Until everyone has settled up |

## How it works

```
 [Phone]                                  [Google]
 form ─▶ queue (localStorage) ─▶ fetch ─▶ /exec ─302─▶ doPost ─▶ Sheet ─▶ summary tab
          ▲ F1                   ▲ F2      ▲ F3         ▲ F4      ▲ F5     ▲ F6
```

| Layer | Choice | Why |
|---|---|---|
| Frontend | Static HTML/CSS/vanilla JS, mobile-first (Android + iOS) | Free hosting, no build step, works in any phone browser |
| Backend | Google Apps Script web app (`doPost`) | Free, sits right next to the Sheet |
| Storage | Google Sheet | Free, shared, doubles as the admin panel and the monitoring |
| Access | One long shared passphrase, checked server-side | No logins for four friends |
| Writes | Append-only | No in-app edit/delete; mistakes are fixed by hand in the Sheet |

### Failure points and how each is handled

| | Failure | Handling |
|---|---|---|
| F1 | Phone storage cleared (iOS can evict idle sites) or phone lost | No recovery. The UI shows "not sent yet" clearly so people know what's at risk |
| F2 | No signal / timeout | Entry stays queued; retried later. `fetch` has an explicit timeout |
| F3 | Google returns an HTML error page instead of JSON | Treated as a retryable failure |
| F4 | Script rejects the entry (bad passphrase, invalid data) | Marked **rejected**, shown to the user, never auto-retried |
| F5 | Two people save at once / manual edits during writes | `LockService` in the script; formulas tolerate hand edits |
| F6 | Summary formula breaks after hand edits | Formulas cover whole columns; checked against a test tab with known answers |

### Offline queue

```
queued ──▶ sending ──▶ sent (removed from queue)
   ▲          │
   └─ retry ──┤  network error / timeout / non-JSON reply / retryable error code
              └─▶ rejected  (bad passphrase / invalid data) → shown to user, never auto-retried
```

- **Success means one thing:** the reply parses as JSON **and** says `ok: true`. Apps Script always answers HTTP 200 (it can't set error codes), so the status code proves nothing.
- **Retry triggers:** page load, the browser's `online` event, and the tab becoming visible again. iOS has no background sync, so retries only happen while the app is open; a visible **pending count** tells people to reopen it.
- **Single-flight:** only one send runs at a time, so triggers firing together can't send the same entry twice.
- **Passphrase is attached at send time**, not when the entry is queued, so changing it mid-trip doesn't strand queued entries.

## Data

### The form

| Field | Notes |
|---|---|
| Date | Entered by the user; stored as ISO text (`2027-05-05`) so Sheets doesn't reinterpret it |
| Who paid | Dropdown of the four names |
| Amount (CAD) | The page converts to **integer cents** before sending |
| Description | Free text; neutralised so it can't become a formula in the Sheet |
| Shared by | Four checkboxes, all ticked by default; untick to opt out. At least one required |
| Notes | Optional |
| Needs review | Checkbox to flag an entry for checking later |

The script adds a **server timestamp**. The Sheet has a **reviewed** column toggled by hand. Large amounts (over a configurable threshold) ask "Are you sure? $1,250.00" before queuing, to catch an extra zero.

Names are placeholders for now (`AA`, `BB`, `CC`, `DD`).

### Request contract

Defined in [`docs/api.md`](docs/api.md) (to be written in milestone 1). Fixed constraints:

- Sent as `Content-Type: text/plain` with a JSON body. A JSON content type triggers a CORS preflight that Apps Script answers with 405.
- Everything (passphrase included) goes in the **body**; don't rely on HTTP headers reaching `doPost`.
- Every request carries a format version (`v: 1`) so entries queued before a format change still work.
- Responses are JSON `{ ok, error? }`. Error codes say whether an error is **retryable** or **permanent**.
- The script never returns Sheet data, only `ok` or an error.

### Money

- Integer cents end to end. The script rejects amounts that aren't positive integers.
- Each share rounds **up**; a purchase's shares can total a few cents more than it. Accepted.
- **All split and balance math lives in the Sheet's summary tab**, not in the page. One source of truth.
- The summary shows **net balances** and **the payments needed to settle up**.

## Decisions

- **Security:** the link is *not* secret: the page and the Apps Script URL are publicly readable. The passphrase is the gate, so it's long (a phrase, not a PIN) and lives only in Apps Script **Script Properties**. Worst case for an attacker who guesses it: junk rows, since nothing can be read back.
- **Concurrency:** `LockService` around the write.
- **Backup:** Google Sheets version history; daily manual export if that proves insufficient.
- **Testing:** a separate test Sheet and test deployment, so test rows never touch trip data.

## Trade-offs

| Decision | What it costs | Revisit if… |
|---|---|---|
| Apps Script + Sheet | No real HTTP error codes, slow first call, little control | Free-tier quirks start blocking features |
| Shared passphrase, no accounts | No record of who submitted what | An audit trail becomes necessary |
| Append-only | No fixing mistakes in the app | Manual fixes become a daily chore |
| Duplicates deferred | An entry may arrive twice; the review flag won't catch it | Duplicates show up in testing |
| Queue only on the phone | Unsent entries die with the phone's storage | Testing shows entries getting lost |
| Names as Sheet columns | Renaming touches several places | The group changes |

If this is ever reused for more trips: add a trip ID column, a Config tab for names, and a per-entry ID.

## Out of scope

Unequal splits · receipt photos · multi-currency · categories · in-app edit/delete · live balances in the app.

## Open questions

1. **Names source of truth.** Names currently live in three places: page config, the script's validation list, and the Sheet's headers/formulas. Options: (a) accept three places plus a rename checklist; (b) a Config tab in the Sheet that the script reads and the page fetches; (c) store sharers as one text field so names aren't columns.
2. **Timestamp time zone.** The server timestamp uses the script's time zone setting, not the phone's. Pacific (account) or the trip's local time?
3. **Hosting:** GitHub Pages, Netlify or Cloudflare Pages. (Pages from a *private* repo may need a paid GitHub plan.)
4. **Formula injection:** confirm by test whether a description like `=1+1` gets evaluated when written by the script, and escape accordingly.
5. **Duplicates:** deferred.
6. **iOS storage eviction:** test whether Add to Home Screen helps.

## Milestones

1. **Contract:** `docs/api.md` (request fields, response shape, error codes and which are retryable, Sheet columns) + test Sheet and deployment.
2. **Backend alone:** `doPost` checks the passphrase and appends a row; tested with curl (follow redirects).
3. **Form:** the page posts one entry end to end and shows the result.
4. **Hardening:** validation, `LockService`, Script Properties, formula/date neutralising.
5. **Offline:** queue states, retry triggers, single-flight, timeout, pending count; airplane-mode test on Android and iOS.
6. **Summary tab:** net balances + settle-up payments, checked against known test cases.
7. **Ship:** pick hosting, deploy, real names, four-phone end-to-end test.

## Repo layout (planned)

```
/web      index.html, app.js, config.js (names, amount-confirm threshold)
/script   Code.gs (copy of the Apps Script source)
/docs     api.md, decisions.md, Sheet setup notes
```

## Working with Claude Code

See [`CLAUDE.md`](CLAUDE.md).
