# 01 — Architecture

## The big picture

```
        Browser (staff)
             │  HTTPS
             ▼
   Microsoft Entra ID  ──────────────┐  authenticates the user,
   (login.microsoftonline.com)       │  attaches app-role claims
             │                        │
             ▼                        │
   Azure AD Application Proxy         │  ...msappproxy.net
   (adds X-MS-APP-ROLES header,       │
    forwards to the connector)        │
             │                        │
             ▼                        │
   IIS + Classic ASP  ◄───────────────┘
   default.asp
     ├─ reads X-MS-APP-ROLES for authorization
     ├─ AJAX action handlers (VBScript, top of file)
     ├─ renders panes (HTML/CSS/JS, same file)
     └─ reads cache files written by scheduled jobs
             │
             ▼
   Cache files on disk / GitHub
     ├─ C:\inetpub\serv\        (device list, POC roster, AI logs)
     ├─ C:\inetpub\reports\     (warranty.json, ninja\ninja-YYYY-MM.json)
     └─ GitHub repos            (link JSONs, AI logs, ticket feedback)
             ▲
             │  written by
   Scheduled PowerShell jobs (Task Scheduler)
     authenticate to NinjaOne / Cloudflare / Graph / GitHub,
     do the heavy work, drop plain files.
```

## Single-file application

`default.asp` contains, in order:

1. **VBScript action handlers** (roughly the first ~3,600 lines). Each is an `If Request.Form("action") = "…" Then … Response.End End If` block that returns JSON. A few use `Request.QueryString` instead. These run *before* any HTML is emitted and terminate the response, so the page HTML is never sent for an AJAX call.
2. **The Entra/proxy auth gate** and role variable setup (see below).
3. **The HTML** for the top bar, sidebar, and every pane.
4. **The CSS** — a large `<style>` block driven by CSS custom properties (theme tokens), with light/dark modes.
5. **The client JavaScript** — one `<script>` region near the bottom, organized as a series of IIFEs (one per feature), plus the navigation/deep-link core.

There is **no build step, no bundler, no external framework beyond Bootstrap + Font Awesome loaded from CDNs.** Editing the app means editing this file. Keep it in the `toolkit` GitHub repo; deployment is copying it to the IIS site root.

## Request flow

**A normal page load:** the browser requests `/` (or `/default.asp`). App Proxy has already authenticated the user and added the `X-MS-APP-ROLES` header. The ASP top-of-file logic reads that header once, sets three role booleans, then renders the HTML — panes the user isn't entitled to are simply not written into the page (server-side `If canSee… Then … End If` around the markup).

**An AJAX call:** the client `fetch()`s the same URL with a POST body like `action=ninjaRptMonth&month=2026-08`. The matching handler re-reads `X-MS-APP-ROLES`, checks the required role, does its work (usually reading a cache file), writes JSON, and calls `Response.End`. Because handlers live above the HTML and end the response, one file serves both the page and its API.

## The role model

There are exactly **three app roles**, defined on the Entra App Registration and delivered in the `X-MS-APP-ROLES` request header:

| Role | Meaning |
|------|---------|
| `SecOps.View` | Security/admin. Unlocks the SecOps and SysAdmin panes **and** the entire Report Center. |
| `CIS.Admin` | General CIS admin. Unlocks admin-flavored content but **not** the Report Center. |
| `Tech.Basic` | Baseline technician access. |

In VBScript the header is lower-cased and tested with `InStr`. The two derived booleans that gate everything are:

```vbscript
canSeeSecOps = (InStr(rolesHeader, "secops.view") > 0)
canSeeAll    = (InStr(rolesHeader, "cis.admin")   > 0) Or canSeeSecOps
```

- `canSeeSecOps` gates the **SecOps** pane, the **SysAdmin** pane, and the **Report Center** and all its sub-panes (Ticket Feedback, Cloudflare DNS, Cloudflare Analytics, Internal Systems Status, Computer Warranty, Ninja Reporting).
- `canSeeAll` gates general admin content that CIS.Admin users may see.
- There is **no** separate SysAdmin or SecOpsAdmin role — SysAdmin visibility rides on `canSeeSecOps`.

Every server handler that returns privileged data re-reads the header and re-checks the role independently — the server-side markup gating is for UX, the per-handler check is the actual control. Example pattern used throughout:

```vbscript
Dim xxRoles : xxRoles = LCase(Trim(Request.ServerVariables("HTTP_X_MS_APP_ROLES")))
If InStr(xxRoles, "secops.view") = 0 Then
  Response.Status = "403 Forbidden"
  Response.Write "{""ok"":false,""msg"":""Not authorized""}"
  Response.End
End If
```

## The proxy shared-secret gate

Near line ~3717 there is a shared-secret check:

```vbscript
Const PROXY_SECRET = "…GUID…"
incomingSecret = Request.ServerVariables("HTTP_X_BCC_PROXY_SECRET")
If incomingSecret <> PROXY_SECRET Then …
```

App Proxy is configured to inject `X-BCC-PROXY-SECRET` on every forwarded request; the gate rejects anything without it, so requests that reach the IIS origin directly (bypassing App Proxy) are refused. **Important ordering caveat:** this gate currently runs *after* the AJAX action handlers (which `Response.End` first). See [09-security.md](09-security.md) — the recommended hardening is to hoist this gate to the very top so it protects the handlers too, and to restrict the origin at the network layer.

## The cache pattern (the core idea)

No vendor API is called from ASP at request time. Instead:

1. A **scheduled PowerShell job** authenticates to the vendor, does the work, and writes a plain file:
   - NinjaOne ticket rollups → `C:\inetpub\reports\ninja\ninja-YYYY-MM.json`
   - NinjaOne warranty export → `C:\inetpub\reports\warranty.json`
   - AI usage logs → GitHub `aiwarninglogs` repo → mirrored to `C:\inetpub\serv\ai-logs\`
   - Ticket feedback → GitHub `ticketfeedback` repo
2. A **role-gated ASP handler** reads that file and streams it to the browser (`StreamCacheFile`), or fetches from GitHub server-side.
3. The **client JS** renders it.

Consequences to internalize:
- A stale or empty pane almost always means *the scheduled writer stopped* (expired token, disabled task), not an API outage. This is exactly what happened when AI Reporting froze at 07/08 — the cache builder had stopped.
- Adding a new "report" is: write a PS job that emits JSON to a server dir, add a `StreamCacheFile`-style handler with a role check and input validation, add a pane and a bit of JS. The Ninja Reporting feature is the canonical example.

## Helper functions worth knowing

Defined in the VBScript region and reused across handlers:

- `StreamCacheFile(pathStr)` — streams a file from disk to the response, UTF-8/BOM aware; returns `True`/`False`. The workhorse for cache endpoints.
- `JsonStr(s)`, `IIfStr(cond,a,b)`, `HtmlEsc(s)` — string/JSON/HTML escaping helpers.
- `ParseCSV` and `tsDetailRow` — CSV parsing and table-row rendering used by the device/roster features.

## Client-side structure

The JavaScript is a set of IIFEs, one per feature (At-a-Glance board, global search, org chart, each report sub-pane, the phonetic tool, etc.), plus a **navigation core** that handles pane switching, URL slugs, and deep-linking. Features communicate loosely through DOM events — e.g. `dynLinksLoaded` fires when the GitHub-driven link tiles finish loading, and several features re-bind on it; `ocDataReloaded` fires when the org chart roster is refreshed. See [08-frontend.md](08-frontend.md).
