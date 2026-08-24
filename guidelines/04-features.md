# 04 — Features (every pane & tool)

Panes are the `.tool-pane` divs in `default.asp`; the sidebar switches between them. Each pane below lists its id, sidebar group, role gate, data source, and behavior.

## Sidebar groups

- **General**: At a Glance, Tools, Building Locator, Infographics, Org Chart
- **IT Tools**: Password Generator, Printer Pass, Quick Links, Scripts
- **Admin**: SecOps, SysAdmin, Report Center

Role gating: SecOps, SysAdmin, and Report Center (with all its sub-panes) require `SecOps.View`. Everything else is visible to all authenticated users unless noted.

---

## General

### At a Glance — `pane-ataglance`
The landing dashboard. Composed of cards:

- **Clock/date** — live, updates each second.
- **Systems card** — reads `list.csv` (via `action=systemsStatus`/`ccSystemsStatus`), pings devices, shows "All Systems Normal" or down tiles. Pinned footer row: **Patch Tuesday countdown** (second Tuesday of month, computed locally; blue normally, amber ≤3 days, green "TODAY"; recomputes each minute).
- **Upcoming (calendar)** — fetches the Outlook ICS feed (`action=ccCalendar`), lists upcoming events. Non-all-day events show a **duration** line ("30 min", "1 hr 30 min", "2 days"); all-day events don't. Refreshes on a timer.
- **POC roster** — reads `poc-people.csv` (`action=ccPocRoster`).
- **Board / raw data** — `action=ccBoard`, `ccGetRaw`, `ccSaveRaw` support an editable notes/board area.
- Page footer shows "Last checked …" and refresh intervals.

### Tools — `pane-tools`
"Hardware & Connection Tools" card with a NinjaOne device lookup (`action=ccNinja`) and a **dynamic tile grid** (`#dyn-tools`) loaded from `links-tools.json`. Tiles can open new tabs, launch protocols, or open in-page shadowbox tools. The **Phonetic Alphabet** tool is a shadowbox launched from a `links-tools.json` tile (`target: btnPhonetic`) — NATO/ICAO converter with preserve-case and show-original-character toggles; deep-linkable via `?sb=phonetic`.

### Building Locator — `pane-locator`
Search county buildings/locations by name, department, or city. `action=reportMissingBuilding` lets users flag a missing entry.

### Infographics — `pane-infographics`
Tile grid from `links-infographics.json`.

### Org Chart — `pane-orgchart`
Renders an interactive org chart from `orgchart.csv`. Lazy-loaded on first open. Exposes `window.ocEnsureData()` (memoized roster load) and `window.ocFocusPerson(key)` (expand ancestors, center, flash, open the person's contact card) so the **global search** can find and jump to people. `action=orgChart` serves the roster. Reload button refreshes and fires `ocDataReloaded`.

---

## IT Tools

### Password Generator — `pane-pass`
Client-side generator with Easy / Standard / Complex modes.

### Printer Pass — `pane-printpass`
Looks up printer access info from `printlist.csv` (`action=reportPrinterPass` / server read via `Server.MapPath`).

### Quick Links — `pane-quicklinks`
Tile grid from `links-quicklinks.json`.

### Scripts — `pane-scripts`
Entries from `scripts.json`.

---

## Admin (all require `SecOps.View`)

### SecOps — `pane-secops`
Security operations launchpad: tiles from `links-secops.json` plus a Microsoft Defender device search. See the security doc regarding roster/data exposure.

### SysAdmin — `pane-sysadmin`
Tiles from `links-sysadmin.json`. Visibility rides on `canSeeSecOps`.

### Report Center — `pane-reportcenter`
Hub of report tiles (`.rc-open` buttons with `data-rc-target`). Each opens a sub-pane; sub-panes have a `.rc-back` button returning to the hub. All sub-panes are `SecOps.View`-gated:

- **AI Reporting** — `pane-aiwarning`. Reads AI usage logs. Data flows GitHub `aiwarninglogs` → `Build-AiLogsCache.ps1` → `C:\inetpub\serv\ai-logs\` → `action=ghAiLogsIndex` / `ghAiLogsDay`. If it stops updating, the cache builder job stopped.
- **Computer Warranty** — `pane-warranty`. Reads `C:\inetpub\reports\warranty.json` via `action=warrantyReport`. A scheduled NinjaOne export writes the file.
- **Ticket Feedback** — `pane-ticketfeedback`. Reads per-ticket JSON from the `ticketfeedback` repo (`action=ticketFeedback`). **Share** button (`action=tfShare`) emails a branded HTML summary of a feedback entry to chosen recipients (validated, CR/LF-stripped, 10-recipient cap, ReplyTo = sender).
- **Cloudflare DNS** — `pane-cloudflaredns`. `action=cloudflareDns` lists zones and DNS records.
- **Cloudflare Analytics** — `pane-cloudflareanalytics`. `action=cloudflareAnalytics`.
- **Internal Systems Status** — `pane-systemsstatus`. Full ping-monitor view of `list.csv` (`action=systemsStatus`), auto-refreshing every 60s, with a fullscreen mode.
- **Ninja Reporting** — `pane-ninjarpt`. Closed-ticket rollups by organization **and** agent. See below.

---

## Ninja Reporting (detail) — `pane-ninjarpt`

The most involved feature. Icon `fa-solid fa-user-ninja`.

**Data**: per-month JSON files at `C:\inetpub\reports\ninja\ninja-YYYY-MM.json`, written by the two scheduled PowerShell jobs (see [06](06-powershell-jobs.md)). **Schema 2** stores a per-day breakdown, and within each day both a per-organization and per-technician (agent) rollup:

```json
{
  "schema": 2, "month": "2026-08", "label": "August 2026",
  "generatedLocal": "…", "days": {
    "2026-08-01": {
      "orgs":   { "CIS": { "t": 3, "b": 1800, "n": 600 } },
      "agents": { "Eric Drinkard": { "t": 2, "b": 1200, "n": 300 } }
    }
  }
}
```
`t` = tickets, `b` = billable seconds, `n` = non-billable seconds. Each closed ticket is attributed to its close day (its `lastUpdated` date).

**Handlers** (both `SecOps.View`-gated): `action=ninjaRptIndex` enumerates the `ninja` directory (regex `^ninja-(\d{4}-\d{2})\.json$`) and returns available months newest-first; `action=ninjaRptMonth&month=YYYY-MM` validates the month (`^\d{4}-\d{2}$`, blocks traversal) and streams that file.

**UI**: month dropdown, a **date-range input** accepting `MMDDYY-MMDDYY` (can cross months), a **By Organization / By Agent** toggle, four KPI cards (Tickets, Billable, Non-Billable, Total Hours), a sortable/filterable table, and a totals row. All rollups are computed **client-side from the day buckets**, so month view, agent view, and arbitrary ranges share one code path. A cross-month range fetches each spanning month file and sums only the days inside the window.

**Table styling**: uses a self-styled `.nr-table` (no Bootstrap `.table` classes — those forced a light theme and broke dark mode).

**Known data caveat**: agents are attributed via the log entry's `appUserContactId`, mapped to a technician name through the users endpoint. If names show as "Technician #NNNN", the contact-id space differs from the user-id space and a contacts-endpoint lookup is needed. Billable currently reads zero across the board — verify the `ticketTimeEntry.billing` value if that's unexpected. See [06](06-powershell-jobs.md).

---

## Cross-cutting features

### Global search
Top-bar search box (Ctrl/Cmd+K or `/`). Indexes the live DOM — tabs, cards, form labels, link tiles — plus **org chart people** (name, title, email, extension, manager). Selecting a person calls `ocFocusPerson` to jump to their card. Role-gated panes aren't indexed for users who can't see them. Re-indexes on `dynLinksLoaded`.

### Deep-linking
Panes have URL slugs (e.g. `?pane=ninjareporting`); shadowbox tools have `?sb=…` (e.g. `?sb=phonetic`). See [08-frontend.md](08-frontend.md).

### Diagnostics
`?diag=writetest` / `action=ccWriteTest` probes which directories the app-pool identity can write to and reports the identity. **Currently ungated and has a destructive bug** — see [09-security.md](09-security.md).
