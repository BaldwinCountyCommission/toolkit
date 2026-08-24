# 10 — Total Rebuild Runbook

The ordered checklist to stand the whole toolkit back up from nothing. Each step references the doc with the detail.

## Phase 0 — Gather

- [ ] Clone `BaldwinCountyCommission/toolkit` (has `default.asp`, config JSON, scripts, these docs).
- [ ] Confirm access to the other two repos: `aiwarninglogs`, `ticketfeedback`.
- [ ] Collect / re-issue secrets: NinjaOne client secret; GitHub PATs (AI logs, edit, ticket feedback); Cloudflare token. ([03](03-configuration.md), [07](07-external-services.md))
- [ ] Note the current `PROXY_SECRET` value (or plan to rotate it in both App Proxy and `default.asp`).

## Phase 1 — Server & IIS ([02](02-deployment.md))

- [ ] Windows Server with IIS + **Classic ASP** feature installed.
- [ ] Create the IIS site; physical path `C:\inetpub\wwwroot\`; `default.asp` as default document.
- [ ] ASP settings: **Send Errors To Browser = False** in prod.
- [ ] Note the **app-pool identity** (needs Read on cache dirs).
- [ ] Install **PowerShell 5.1+**; the NinjaOne jobs will install `NinjaOneDocs` on first run.

## Phase 2 — Directories & permissions ([02](02-deployment.md))

- [ ] Create: `C:\inetpub\serv\`, `C:\inetpub\serv\ai-logs\`, `C:\inetpub\reports\`, `C:\inetpub\reports\ninja\`, `C:\inetpub\secrets\`, `C:\inetpub\logs\`.
- [ ] ACLs: app-pool identity **Read** on `serv\` and `reports\`; scheduled-task account **Modify** on the same; `secrets\` readable **only** by task accounts/admins (never the app pool).
- [ ] Place data files: `serv\list.csv` (`Name,Type,IP`), `serv\poc-people.csv`, `wwwroot\printlist.csv`, `wwwroot\tools\orgchart.csv`. (Consider moving `orgchart.csv` to `serv\` per [09](09-security.md).)
- [ ] Drop secret files into `secrets\` ([03](03-configuration.md#secrets-token-files)).

## Phase 3 — Deploy the app

- [ ] Copy `default.asp` to the site root.
- [ ] Verify in-file constants match this environment: `PROXY_SECRET`, `CC_DATA_DIR`, `CC_ICS_URL` (both ICS URLs), `GH_OWNER_REPO`/`GH_BRANCH`, `AL_REPO`, `TF_REPO`, `CF_ACCOUNT`. ([03](03-configuration.md))
- [ ] Ensure `version.json` in the repo has the right version.

## Phase 4 — Entra ID + App Proxy ([02](02-deployment.md), [07](07-external-services.md))

- [ ] App Registration with roles `SecOps.View`, `CIS.Admin`, `Tech.Basic`.
- [ ] Assign users/groups to roles in the Enterprise App.
- [ ] App Proxy application → internal origin; pre-auth Entra ID.
- [ ] Forward **`X-MS-APP-ROLES`** to the origin.
- [ ] Inject **`X-BCC-PROXY-SECRET`** = `PROXY_SECRET`.
- [ ] **Network-layer**: restrict the IIS origin so only the App Proxy connector can reach it ([09](09-security.md)).

## Phase 5 — Scheduled jobs ([06](06-powershell-jobs.md))

- [ ] Deploy scripts (e.g. to `C:\Scripts\`).
- [ ] **Ninja backfill**: run `Generate-NinjaReports-Backfill.ps1` once (optionally `-Diagnose` first). Confirm `ninja-YYYY-MM.json` files appear.
- [ ] **Ninja monthly**: register the daily task (`-NoProfile -ExecutionPolicy Bypass -File …`), running as an account with Read on secrets and Modify on `reports\ninja\`.
- [ ] **AI logs**: schedule `Build-AiLogsCache.ps1` (PAT `gh-ai-token.txt`).
- [ ] **Ticket feedback**: schedule `Send-TicketFeedbackReport.ps1` weekly (Mondays; PAT `ticketfeedback.txt`).
- [ ] **Warranty export**: schedule the NinjaOne warranty export → `reports\warranty.json`.
- [ ] Confirm the Exchange connector permits `DSNinjaFeedback@baldwincountyal.gov`.

## Phase 6 — Verify ([02](02-deployment.md#verifying-a-deployment))

- [ ] Load as a **SecOps** user; footer version matches `version.json`.
- [ ] **At a Glance**: Systems card + Patch Tuesday row populate; Upcoming calendar loads with durations.
- [ ] **Report Center → Ninja Reporting**: months list; a month loads; By Organization/By Agent toggle works; a `MMDDYY-MMDDYY` range (crossing a month boundary) totals correctly.
- [ ] **AI Reporting**, **Warranty**, **Ticket Feedback**, **Cloudflare DNS/Analytics**, **Internal Systems Status** each load.
- [ ] **Org Chart** renders; **global search** finds a person and jumps to their card.
- [ ] **Tools**: `links-tools.json` tiles render; the **Phonetic Alphabet** shadowbox opens (and `?sb=phonetic` deep-links).
- [ ] Test with a **Tech.Basic-only** account: no SecOps/SysAdmin, no Report Center.
- [ ] Send a test **`tfShare`** email; confirm delivery (watch the Direct Send caveat in [07](07-external-services.md)).

## Phase 7 — Post-rebuild hardening (recommended) ([09](09-security.md))

- [ ] Hoist the `PROXY_SECRET` gate above the handlers.
- [ ] Fix or remove `ccWriteTest` / `?diag=writetest` (destructive + ungated).
- [ ] Gate `orgChart`; move `orgchart.csv` out of the web root.
- [ ] Add token-expiry alerting to each job (or move secrets to Key Vault).
- [ ] Plan the Direct Send → Graph `sendMail` migration.

## Common failure signatures (fast triage)

| Symptom | Likely cause |
|---------|--------------|
| A report pane is empty/stale | Its scheduled writer stopped (expired PAT / disabled task). Check the job log + PAT expiry — **not** the vendor API. |
| AI Reporting frozen at a date | `Build-AiLogsCache.ps1` stopped. |
| Ninja agents show "Technician #NNNN" | Contact-id ≠ user-id; add a contacts-endpoint lookup ([06](06-powershell-jobs.md)). |
| Ninja billable all zero | Check `ticketTimeEntry.billing` value via backfill `-Diagnose`. |
| Roles not applied | App Proxy not forwarding `X-MS-APP-ROLES`, or user unassigned in the Enterprise App. |
| Emails stopped arriving | Tenant enabled `RejectDirectSend`, or sender not permitted on the connector. |
| "token file not found" | Job on the wrong machine or account lacks Read on `C:\inetpub\secrets\`. |
| Data table is light-on-light | A Bootstrap `.table` class crept onto it; self-style instead ([08](08-frontend.md)). |
