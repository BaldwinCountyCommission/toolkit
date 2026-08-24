# 06 — PowerShell Scheduled Jobs

The app reads caches; these jobs write them. All are designed to run unattended (no GUI, no message boxes), log to `C:\inetpub\logs\`, and exit non-zero on failure so Task Scheduler surfaces the result. Keep the canonical copies in the `toolkit` repo under `scripts/` and deploy them to wherever Task Scheduler runs them (e.g. `C:\Scripts\`).

## Inventory

| Script | Cadence | Writes | Reads-secret |
|--------|---------|--------|--------------|
| `Generate-NinjaReports-Backfill.ps1` | Once (setup / schema change) | `C:\inetpub\reports\ninja\ninja-YYYY-MM.json` × trailing 12 months | `ninja-clientsecret.txt` |
| `Generate-NinjaReports-Monthly.ps1` | Daily | current month's `ninja-YYYY-MM.json` | `ninja-clientsecret.txt` |
| `Send-TicketFeedbackReport.ps1` | Weekly (Mondays) | emails a branded report; reads/writes `ticketfeedback` repo | `ticketfeedback.txt` |
| `Build-AiLogsCache.ps1` | Scheduled (e.g. hourly/daily) | `C:\inetpub\serv\ai-logs\ai-logs-index.json` + per-day files | `gh-ai-token.txt` |
| *(warranty export)* | Scheduled | `C:\inetpub\reports\warranty.json` | NinjaOne creds |

> The warranty exporter is referenced by the app but its script may live outside this repo; if you can't find it, any NinjaOne export that produces `warranty.json` in the expected shape works. Document it here when located.

---

## Ninja Reporting jobs

Both derive from the original all-orgs ticket report but are stripped for unattended use (no date-picker GUI, no PDF, no message boxes). They share an identical helper core and differ only in windowing.

### Common configuration (top of each script)
```
$NinjaOneInstance = 'baldwincountyal.rmmservices.net'
$NinjaOneClientID = '7wXGs97IGsBrqrRi5fy8k3s_0uI'
$SecretFile       = 'C:\inetpub\secrets\ninja-clientsecret.txt'   # preferred
$BoardID          = '2'
$ClosedStatuses   = @('Closed','Resolved','Completed')
```
Secret is read from `SecretFile` when present, else from an inline `$NinjaOneClientSecret` fallback.

### Shared helpers
- `ConvertFrom-UnixSeconds` — epoch seconds → local `DateTime` (via `DateTimeOffset`), used to bucket a ticket by its close **day**.
- `Resolve-TicketOrg` — best-effort org name from the many field names NinjaOne instances use (`organizationId`, `organization.id`, `clientId`, …), falling back through name fields to `(No Organization)`.
- `Resolve-LogTech` — best-effort technician name. **This instance carries the tech on a log entry as `appUserContactId` (int) plus `appUserContactUid` (guid)**, so those are tried first, compared as strings against `$TechMap`. If no name maps, returns a **distinct** `Technician #<id>` rather than collapsing everyone into one "Unknown" bucket.
- `Get-NinjaTicketsPage` — one page of the board's tickets (`ticketing/trigger/board/$BoardID/run`, sorted by `lastUpdated` desc, cursor paginated).
- `Add-TicketToDay` — accumulates one closed ticket into a day bucket: `+1` to its org's ticket count, and for each technician who logged time, `+1` to that agent's count plus their billable/non-billable seconds. `$TechMap` is built from the users endpoint keyed by **both** `id` and `uid` (as strings).
- `Write-MonthJson` — serializes a month's day buckets to schema-2 JSON, writing to a `.tmp` then `Move-Item`-renaming into place so the Report Center never reads a half-written file.

### Schema 2 (output)
See [04-features.md](04-features.md#ninja-reporting-detail--pane-ninjarpt). Per-day, per-org and per-agent `{t,b,n}`. Depth-8 JSON.

### Backfill
`Generate-NinjaReports-Backfill.ps1 [-Months 12] [-Diagnose]`
- Fetches all board tickets once, filters to closed tickets in the trailing-N-month window, fetches each ticket's detail + logs once, buckets by close day into the right month file. Writes N month files (default 12 = current month-to-date + 11 prior).
- **`-Diagnose`**: dumps the raw structure of the first few technician log entries (full JSON + candidate id/name fields + a `TechMap` sample) and **exits without writing files**. This is how the `appUserContactId` field was identified. Use it whenever agent attribution looks wrong.

### Monthly
`Generate-NinjaReports-Monthly.ps1`
- Same rollup scoped to the current month; overwrites only the current month's file. Run daily. Historical months are never touched.

### Task Scheduler (monthly job)
```
Program : powershell.exe
Args    : -NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\Generate-NinjaReports-Monthly.ps1"
Trigger : Daily, e.g. 05:30
Run as  : an account with READ on the secrets file and MODIFY on C:\inetpub\reports\ninja\
```

### First-time setup order
1. Create `C:\inetpub\reports\ninja\`, grant app-pool **Read** and task-account **Modify**.
2. Put the NinjaOne client secret in `C:\inetpub\secrets\ninja-clientsecret.txt`.
3. Run the backfill once (optionally `-Diagnose` first to sanity-check agent fields).
4. Register the daily monthly task.

### Known data caveats (verify after a run)
- **Agent names**: if the By-Agent view shows `Technician #NNNN`, the contact-id space differs from the user-id space; add a NinjaOne contacts-endpoint lookup to map contact id → name.
- **Billable = 0 everywhere**: check the actual `ticketTimeEntry.billing` value on a log entry (via `-Diagnose`); the code counts `BILLABLE` vs everything-else.
- **Outliers**: very large single entries (e.g. 50+ hours) are almost certainly timers left running. Consider an optional cap that flags/discards any single entry over N hours.

---

## Send-TicketFeedbackReport.ps1

Weekly (Mondays). Reads the `ticketfeedback` repo (PAT from `ticketfeedback.txt`), computes metrics and week-over-week deltas, and **direct-sends** a branded HTML report to the CIS leads. Sender `DSNinjaFeedback@baldwincountyal.gov` must be permitted on the Exchange connector.

> The known "GitHub token file not found" failure mode is almost always the task running on the wrong machine or under an account without read access to `C:\inetpub\secrets\ticketfeedback.txt`.

---

## Build-AiLogsCache.ps1

Pulls AI usage logs from the GitHub `aiwarninglogs` repo (PAT from `gh-ai-token.txt`) and writes the on-disk cache the AI Reporting pane reads: `ai-logs-index.json` plus per-day `ai-logs-YYYY-MM-DD.json` under `C:\inetpub\serv\ai-logs\`.

> **If AI Reporting stops updating, this job stopped.** The page reads the disk cache, not GitHub live. The 07/08 freeze was this job halting (expired PAT / disabled task). Check the task history and the PAT expiry first.

---

## Mail delivery note (applies to all emailing jobs + `tfShare`)

Delivery uses unauthenticated **direct send** to the Exchange relay (`baldwincountyal-gov.mail.protection.outlook.com`, port 25). Microsoft is moving to reject Direct Send by default; if reports silently stop arriving, check the tenant `RejectDirectSend` setting and consider migrating to **Graph `sendMail`** with app-only auth scoped by `New-ApplicationAccessPolicy` to the sender mailbox. See [07-external-services.md](07-external-services.md).
