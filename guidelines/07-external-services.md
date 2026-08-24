# 07 — External Services

Every third-party the toolkit touches, how it authenticates, and what breaks if it changes.

## Microsoft Entra ID + Azure AD Application Proxy
**Role**: identity and publishing. See [02-deployment.md](02-deployment.md) for setup.
- App Registration defines the three roles; Enterprise App assigns them to users/groups.
- App Proxy pre-authenticates, forwards `X-MS-APP-ROLES` (authorization) and injects `X-BCC-PROXY-SECRET` (origin protection).
- Hosts involved: `login.microsoftonline.com`, `*.msappproxy.net`, `launcher.myapps.microsoft.com`.
- **If roles stop working**: confirm App Proxy still forwards `X-MS-APP-ROLES` and that the user is assigned a role in the Enterprise App.

## NinjaOne (RMM)
**Instance**: `baldwincountyal.rmmservices.net`. **Client ID**: `7wXGs97IGsBrqrRi5fy8k3s_0uI`. **Board**: `2`.
- Used by the Ninja Reporting jobs (ticket rollups), the warranty export (→ `warranty.json`), and the Tools-pane device lookup (`ccNinja`).
- Auth: OAuth client credentials; secret in `C:\inetpub\secrets\ninja-clientsecret.txt`. PowerShell uses the `NinjaOneDocs` module (`Connect-NinjaOne`, `Invoke-NinjaOneRequest`, `Get-NinjaOneTime`).
- Ticket URL base for links: `https://baldwincountyal.rmmservices.net/#/ticketing/ticket/<id>`.
- **Field quirks** (see [06](06-powershell-jobs.md)): org id lives under one of several field names; technician identity is `appUserContactId`. Use `-Diagnose` on the backfill to inspect live payloads.

## Cloudflare
**Account**: `15f12352613f2eb0f9d4c975da0653c8`. **Token**: `C:\inetpub\secrets\clouddnsviewapi.txt` (read scope for DNS + analytics).
- Handlers `cloudflareDns` (list zones + records) and `cloudflareAnalytics`. Both SecOps-gated.
- API host: `api.cloudflare.com`.
- **If DNS/analytics panes error**: token expired or lost the required read scopes.

## Exchange Online (mail)
**Relay**: `baldwincountyal-gov.mail.protection.outlook.com`, port **25**, no SSL, via `CDO.Message`.
- Used by `tfShare` and the emailing PowerShell jobs. Sender: `DSNinjaFeedback@baldwincountyal.gov` — must be permitted on the Exchange connector.
- **Direct Send is being deprecated by Microsoft.** Watch the tenant `RejectDirectSend` setting; when enabled, direct send returns NDR 550 5.7.68 and mail silently fails. **Recommended migration**: Graph `sendMail` with app-only auth, scoped by `New-ApplicationAccessPolicy` so the app can only send as the one report mailbox. This also removes the "is the sender permitted on the connector?" fragility.

## GitHub
Three repos (see [03-configuration.md](03-configuration.md)). Hosts: `api.github.com`, `raw.githubusercontent.com`.
- `toolkit` — app source + config JSON. Read via raw URLs; edited via the contents API by the `ghEdit` handler (PAT in `gh-edit-token.txt`, file must be in the `ghAllowed` whitelist).
- `aiwarninglogs` — AI usage logs, cached by `Build-AiLogsCache.ps1` (PAT in `gh-ai-token.txt`).
- `ticketfeedback` — feedback JSON, read by the report and written by `Send-TicketFeedbackReport.ps1` (PAT in `ticketfeedback.txt`).
- Also a status banner from `raw.githubusercontent.com/ericbcc/cisstatus/…/banner.txt`.
- **PAT expiry is the #1 recurring failure.** Track expirations; prefer fine-grained PATs scoped to the single repo each token needs.

## Microsoft 365 Calendar (ICS)
Two published Outlook calendar `.ics` URLs (`outlook.office365.com/owa/calendar/…/calendar.ics`) feed the At-a-Glance Upcoming list and a secondary calendar view (constants `CC_ICS_URL` and the one near line ~3466).
- **If a URL is regenerated** (calendar re-shared), update the constant in `default.asp`.

## Cloudinary (image hosting)
County logo and seal for branded emails/UI:
- Logo: `res.cloudinary.com/baldwincounty/image/upload/v1781701739/Full_White_2x_gm5n2g.png`
- Seal: `res.cloudinary.com/baldwincounty/image/upload/…/Seal_na8i8w.png`

## CDNs (front-end assets)
Bootstrap, Font Awesome, and fonts load from `cdnjs.cloudflare.com`, `cdn.jsdelivr.net`, `fonts.googleapis.com`. Also `static.cloudflareinsights.com` (Cloudflare browser insights) and `get.geojs.io` (client IP/geo lookup). No auth. If locked-down/offline operation is ever required, these would need to be self-hosted.

## Internal hosts referenced
- `toolkit.internal.co.baldwin.al.us`, `bccpbirs.internal.co.baldwin.al.us` — internal origin/related hosts.
- `baldwin.speedtestcustom.com` — a linked speed-test.
- Microsoft security portals (`security.microsoft.com`) — Defender links in the SecOps pane.

---

## Recommended integrations not yet built (Entra/Graph)
Captured here so the rebuild can consider them. All fit the existing cache pattern (a scheduled PS job writes JSON to `serv\`; the page reads it) — **do not call Graph from VBScript**; use a scheduled PowerShell job with app-only auth and a **separate app registration from the mail sender**, certificate auth preferred.

- **Generate the org chart from Entra** (`displayName`, `jobTitle`, `manager`, `businessPhones`, `mobilePhone`, `mail`) instead of hand-maintaining `orgchart.csv`.
- **Intune compliance × NinjaOne warranty** join → one "out of warranty **and** non-compliant" list (the highest-value item; the portal can't do this because warranty lives in NinjaOne).
- **Privileged-role-change deltas** (what changed since last run), and **stale accounts × org chart** to distinguish service accounts from missed offboardings.
- Risky sign-ins require **Entra ID P2**; `signInActivity` (stale accounts) requires **P1**. Confirm licensing (G3 vs G5) before designing.
- Move secrets to **Azure Key Vault** with a managed identity to end the token-file expiry problem.
