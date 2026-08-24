# CIS Toolkit

Internal IT operations dashboard for the Baldwin County Commission's Central Information Services (CIS) department. A single self-contained Classic ASP application (`default.asp`) that runs on IIS behind Microsoft Entra ID via Azure AD Application Proxy, backed by a small set of scheduled PowerShell jobs that write JSON/CSV cache files the page reads.

This `docs/` folder is the complete rebuild reference. If the server were lost tomorrow, these files plus the source in the `toolkit` repo are enough to stand the whole thing back up.

---

## What it is, in one paragraph

`default.asp` is one large file (~16,000 lines) containing the entire application: the VBScript server logic (AJAX-style action handlers at the top), the HTML for every pane, the CSS theme, and the client-side JavaScript. There is no build step and no framework — it is served directly by IIS Classic ASP. Authentication and identity are handled entirely upstream by Entra/App Proxy; the app reads the signed-in user's roles from a request header. Anything that needs credentials or heavy processing (NinjaOne reporting, warranty data, AI usage logs, Cloudflare) is done by scheduled PowerShell that writes a cache file to disk or to a GitHub repo; the ASP page only ever reads those caches. Editable content (link tiles, scripts, org chart) lives as JSON/CSV either in a GitHub repo or in a server data directory.

## Read these in order

1. **[01-architecture.md](01-architecture.md)** — how the pieces fit, request flow, the single-file layout, the role model.
2. **[02-deployment.md](02-deployment.md)** — IIS, App Proxy, directories, permissions, DNS. The physical rebuild.
3. **[03-configuration.md](03-configuration.md)** — every constant, secret, GitHub repo, and JSON/CSV config file.
4. **[04-features.md](04-features.md)** — every pane and tool, what it does, where its data comes from.
5. **[05-ajax-handlers.md](05-ajax-handlers.md)** — reference for every server endpoint (`action=…`).
6. **[06-powershell-jobs.md](06-powershell-jobs.md)** — the scheduled scripts and their Task Scheduler setup.
7. **[07-external-services.md](07-external-services.md)** — NinjaOne, Cloudflare, Exchange, Entra, GitHub, Cloudinary.
8. **[08-frontend.md](08-frontend.md)** — theme tokens, pane/slug system, deep-linking, conventions.
9. **[09-security.md](09-security.md)** — the trust model and the known hardening backlog. Read before exposing anything.
10. **[10-rebuild-runbook.md](10-rebuild-runbook.md)** — the ordered checklist for a total rebuild.

## Repository layout (recommended)

The application is currently a single file. For GitHub storage, keep it at the repo root and keep scripts and docs beside it:

```
toolkit/                         (GitHub: BaldwinCountyCommission/toolkit)
├── default.asp                  the entire application
├── version.json                 { "version": "4.40" } — shown in the footer
├── links-tools.json             Tools pane tiles
├── links-quicklinks.json        Quick Links pane tiles
├── links-infographics.json      Infographics pane tiles
├── links-secops.json            SecOps pane tiles
├── links-sysadmin.json          SysAdmin pane tiles
├── scripts.json                 Scripts pane entries
├── ninja.json                   (legacy link list — see configuration doc)
├── scripts/
│   ├── Generate-NinjaReports-Backfill.ps1
│   ├── Generate-NinjaReports-Monthly.ps1
│   ├── Send-TicketFeedbackReport.ps1
│   └── Build-AiLogsCache.ps1
└── docs/                        this folder
```

Two other GitHub repos hold data the app reads: `BaldwinCountyCommission/aiwarninglogs` (AI usage logs) and `BaldwinCountyCommission/ticketfeedback` (ticket-feedback reports). See the configuration doc.

## The one thing to understand first

**The ASP page never talks to a vendor API that needs a secret at request time.** Every integration follows the same pattern: a scheduled PowerShell job authenticates, does the work, and writes a plain file (JSON/CSV) to `C:\inetpub\serv\`, `C:\inetpub\reports\`, or a GitHub repo. The page reads that file through a small role-gated handler and renders it. Get that pattern and the whole architecture follows. If a pane is "stale" or "empty," the first question is always *did the scheduled job that writes its cache run?* — not *is the API down?*

## Current version

Version is read at page load from `version.json` in the toolkit repo (`{"version":"4.40"}` at time of writing) and shown in the footer, falling back to `4.0` if the fetch fails.
