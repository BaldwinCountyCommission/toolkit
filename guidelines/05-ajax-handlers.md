# 05 — AJAX Handler Reference

Every server endpoint in `default.asp`. All live in the VBScript region above the HTML; each matches an `action` (mostly `Request.Form`, a few `Request.QueryString`), does its work, writes a response, and calls `Response.End`. JSON unless noted.

Role gate legend: **SecOps** = requires `SecOps.View`; **Auth** = any authenticated user (relies on the App Proxy secret + Entra); **Open-ish** = present but not role-checked (see notes / security doc).

| `action` | Method | Role | Purpose |
|----------|--------|------|---------|
| `ccBoard` | POST | Auth | At-a-Glance board/alerts data. |
| `ccCalendar` | POST | Auth | Fetch + parse the Outlook ICS feed; returns upcoming events. |
| `ccGetRaw` | POST | Auth | Read the editable raw notes/board content. |
| `ccSaveRaw` | POST | Auth | Save the editable raw notes/board content. |
| `ccNinja` | POST | Auth | NinjaOne device lookup for the Tools pane. |
| `ccPocRoster` / `pocRoster` | POST | Auth | Read `poc-people.csv` → people list. |
| `ccSystemsStatus` / `systemsStatus` | POST | Auth* | Read `list.csv`, ping devices, return status. Used by At-a-Glance and the SecOps-gated Internal Systems Status pane. |
| `ccWriteTest` (`?diag=writetest`) | GET/POST | **Ungated** | Diagnostic: probes writable dirs, discloses app-pool identity. **Has a destructive bug** (overwrites `poc.txt`/POC data). See security doc — fix or remove. |
| `cloudflareDns` | POST | SecOps | List Cloudflare zones + DNS records (token from `clouddnsviewapi.txt`, account `CF_ACCOUNT`). |
| `cloudflareAnalytics` | POST | SecOps | Cloudflare analytics. |
| `warrantyReport` | GET | SecOps | Stream `C:\inetpub\reports\warranty.json`. |
| `ghAiLogsIndex` | POST | SecOps | Stream `ai-logs-index.json` from `C:\inetpub\serv\ai-logs\`. |
| `ghAiLogsDay` | POST | SecOps | Stream `ai-logs-YYYY-MM-DD.json` for a given day. |
| `ghAiLogsList` | POST | SecOps | Enumerate AI log files from the GitHub `aiwarninglogs` repo. |
| `ghAiLogsFetch` | POST | SecOps | Fetch a specific AI log file (server-side from GitHub). |
| `ghEdit` | POST | SecOps | Commit an edit to a whitelisted link/scripts JSON in the `toolkit` repo (token from `gh-edit-token.txt`; file must be in `ghAllowed`). |
| `ticketFeedback` | POST | SecOps | Read ticket-feedback JSON from the `ticketfeedback` repo. |
| `tfShare` | POST | SecOps | Email a branded HTML summary of a feedback entry (validated recipients, CR/LF-stripped, 10-recipient cap, ReplyTo = sender, From = `DSNinjaFeedback@…`). |
| `orgChart` | POST | Auth* | Serve the org chart roster from `orgchart.csv`. *Not role-checked; roster (incl. cell/email) reachable by any authenticated user, and `orgchart.csv` is also directly fetchable from the web root.* |
| `reportPrinterPass` | POST | Auth | Printer Pass lookup from `printlist.csv`. |
| `reportMissingBuilding` | POST | Auth | Flag a missing building in the Building Locator. |
| `ninjaRptIndex` | POST | SecOps | Enumerate `C:\inetpub\reports\ninja\ninja-*.json`; return months newest-first with labels + modified time. |
| `ninjaRptMonth` | POST | SecOps | Validate `month=YYYY-MM` (`^\d{4}-\d{2}$`) and stream that month's file. |

\* Handlers marked with an asterisk are reachable by any authenticated user; whether that's acceptable depends on the sensitivity of the data. `orgChart` in particular is flagged in [09-security.md](09-security.md).

## Conventions every handler follows

- **Role check** (privileged handlers): re-read `HTTP_X_MS_APP_ROLES`, lower-case, `InStr(... "secops.view") = 0` → `403` + JSON error + `Response.End`. Never trust the server-side markup gating alone.
- **Content type**: `Response.ContentType = "application/json"` + `Response.CharSet = "utf-8"`.
- **File streaming**: use `StreamCacheFile(path)` (UTF-8/BOM aware, returns `True`/`False`); on `False`, return `{"ok":false,"msg":"…"}`.
- **Input validation**: any value used to build a file path must be regex-validated to prevent traversal (e.g. `ninjaRptMonth` requires `^\d{4}-\d{2}$`).
- **Terminate**: always `Response.End` so the page HTML isn't appended to the JSON.

## Adding a new cache-backed report (recipe)

1. Write a scheduled PowerShell job that authenticates to the source and writes JSON to a dir under `C:\inetpub\reports\` (temp-file-then-rename so readers never see a half-written file).
2. Add an index handler (enumerate the dir with `Scripting.FileSystemObject`, regex-match filenames) and a fetch handler (validate the key, `StreamCacheFile`). Gate both with the role check.
3. Add a Report Center tile (`.rc-open` + `data-rc-target`), a sub-pane inside the `If canSeeSecOps` block, and a JS IIFE that loads on the tile's click.
4. Register the pane slug for deep-linking (see [08-frontend.md](08-frontend.md)).

Ninja Reporting is the reference implementation of this whole recipe.
