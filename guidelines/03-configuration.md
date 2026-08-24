# 03 — Configuration Reference

Every constant, identifier, repo, and config file the app depends on. When rebuilding, these are the values to restore or re-issue.

## In-file constants (`default.asp`)

Search `default.asp` for each; they live in the VBScript region.

| Constant / var | Value (at time of writing) | Purpose |
|----------------|----------------------------|---------|
| `PROXY_SECRET` | `R6108d421-…-928345111400` (GUID) | Shared secret matched against the `X-BCC-PROXY-SECRET` header injected by App Proxy. Rotate in both places together. |
| `CC_DATA_DIR` | `C:\inetpub\serv\` | Base dir for editable/cache data (device list, POC roster, AI logs). |
| `CC_ICS_URL` | `https://outlook.office365.com/owa/calendar/…/calendar.ics` | Published Outlook calendar feed for the At-a-Glance "Upcoming" list. A second ICS URL (~line 3466) feeds another calendar view. |
| `GH_OWNER_REPO` | `BaldwinCountyCommission/toolkit` | Repo holding `default.asp`, link JSONs, `version.json`. |
| `GH_BRANCH` | `main` | Branch used for raw reads and edits. |
| `AL_REPO` / `AL_PATH` | `BaldwinCountyCommission/aiwarninglogs` / `logs` | AI usage logs source. |
| `AF_REPO` | `BaldwinCountyCommission/aiwarninglogs` | AI logs raw-file fetch. |
| `TF_REPO` / `TF_PATH` | `BaldwinCountyCommission/ticketfeedback` / `reports` | Ticket-feedback reports source. |
| `CF_ACCOUNT` | `15f12352613f2eb0f9d4c975da0653c8` | Cloudflare account id (DNS/analytics handlers). |
| `APP_VERSION_FALLBACK` | `4.0` | Footer version if `version.json` can't be read. |
| `ghAllowed` | array incl. `links-tools.json`, `links-quicklinks.json`, `links-infographics.json`, `links-secops.json`, `links-sysadmin.json`, `scripts.json`, `ninja.json`, … | Whitelist of files the in-app editor (`ghEdit`) may commit. Anything not listed is rejected. |

## GitHub repositories

| Repo | Role |
|------|------|
| `BaldwinCountyCommission/toolkit` | Source of truth for `default.asp`, all link/`scripts` JSON tiles, `version.json`. The app reads these via `raw.githubusercontent.com` and edits them via the GitHub contents API. |
| `BaldwinCountyCommission/aiwarninglogs` | AI usage log files (`logs/…`). Read by the AI Reporting pane (server-side) and mirrored to disk by `Build-AiLogsCache.ps1`. |
| `BaldwinCountyCommission/ticketfeedback` | Per-ticket feedback JSON under `reports/`. Read by the Ticket Feedback report; also the target of the weekly `Send-TicketFeedbackReport.ps1`. |

There is also a status banner read from `raw.githubusercontent.com/ericbcc/cisstatus/…/banner.txt` — a small text banner shown in the UI. If rebuilding under new ownership, repoint this.

## Config files in the `toolkit` repo

All are plain JSON committed to the repo; the app fetches them at load and the editable ones can be changed through the in-app editor (which commits via `ghEdit`).

### Link-tile files
`links-tools.json`, `links-quicklinks.json`, `links-infographics.json`, `links-secops.json`, `links-sysadmin.json`

Each is an array of tile objects. A tile supports these `action` types:

| `action` | Behavior | Extra keys |
|----------|----------|-----------|
| `newtab` | Opens `url` in a new tab | `url` |
| `protocol` | Launches a protocol/URI handler | `url` (the protocol URI) |
| `shadowbox` | Renders a button whose `id` = `target`; app JS wires it to open an in-page modal/tool | `target` |

Common keys on every tile: `label`, `icon` (Font Awesome class, e.g. `fa-solid fa-tower-broadcast`). Example (the Phonetic Alphabet tool tile in `links-tools.json`):

```json
{
  "label": "Phonetic Alphabet",
  "icon": "fa-solid fa-tower-broadcast",
  "action": "shadowbox",
  "target": "btnPhonetic"
}
```

Order in the array is display order (first = top-left).

### `scripts.json`
Entries for the Scripts pane (script name/description/link tiles). Same general shape as the link files.

### `version.json`
```json
{ "version": "4.40" }
```
Shown in the footer. Bump on release.

### `ninja.json`
A legacy link list kept in the `ghAllowed` whitelist. **Not** the Ninja Reporting data — that lives on disk at `C:\inetpub\reports\ninja\` (see below). Don't confuse the two.

## Config/data files on the server (not in Git)

| File | Format | Feeds |
|------|--------|-------|
| `C:\inetpub\serv\list.csv` | `Name,Type,IP` | At-a-Glance Systems card, Internal Systems Status ping monitor |
| `C:\inetpub\serv\poc-people.csv` | roster columns | At-a-Glance POC roster (`action=pocRoster`) |
| `C:\inetpub\wwwroot\printlist.csv` | printer rows | Printer Pass tool |
| `C:\inetpub\wwwroot\tools\orgchart.csv` | name,title,manager,ext,cell,email | Org Chart pane and global people search |
| `C:\inetpub\reports\warranty.json` | NinjaOne export | Computer Warranty report |
| `C:\inetpub\reports\ninja\ninja-YYYY-MM.json` | schema-2 day buckets | Ninja Reporting (see [06](06-powershell-jobs.md)) |
| `C:\inetpub\serv\ai-logs\ai-logs-index.json` + `ai-logs-YYYY-MM-DD.json` | AI usage cache | AI Reporting pane |

## Secrets (token files)

Under `C:\inetpub\secrets\` — see [02-deployment.md](02-deployment.md#secrets) for the table. Each is a single-line text file. The Ninja scripts also accept an inline fallback variable but the file is preferred.

## Brand / asset constants

- **County navy**: `#0C2340`. **Accent blue**: `#2563EB`. Used in emailed HTML reports.
- **Logo** (white, for dark headers): `https://res.cloudinary.com/baldwincounty/image/upload/v1781701739/Full_White_2x_gm5n2g.png`
- **Seal**: `https://res.cloudinary.com/baldwincounty/image/upload/…/Seal_na8i8w.png`
- **NinjaOne ticket URL base**: `https://baldwincountyal.rmmservices.net/#/ticketing/ticket/` (append ticket id).
- **Report sender address**: `DSNinjaFeedback@baldwincountyal.gov` (used by `tfShare` and the emailed reports). Must be permitted on the Exchange connector.
