# 09 — Security Model & Hardening Backlog

Read this before exposing the app or making auth changes. It documents the trust model and the known issues found in review, so a rebuild doesn't silently reintroduce them.

## Trust model (how it's *supposed* to work)

1. **Entra ID authenticates** every user upstream; unauthenticated requests never reach the app.
2. **App Proxy adds `X-MS-APP-ROLES`** (authorization data) and **injects `X-BCC-PROXY-SECRET`** (proof the request came through the proxy).
3. The app **re-checks the role in every privileged handler** — server-side markup gating is UX only.
4. The **`PROXY_SECRET` gate** rejects any request lacking the shared secret, so direct hits to the IIS origin are refused.

The whole model rests on one assumption: **the IIS origin is not reachable except through App Proxy.** If the origin is reachable on the network, a client could send a forged `X-MS-APP-ROLES: secops.view` header and — depending on handler ordering — reach privileged handlers. Enforce origin isolation at the **network layer** (firewall/NSG so only the App Proxy connector can reach the origin), not just via the header secret.

## Known issues & hardening backlog

Ordered by priority. None were auto-"fixed" in a way that changes behavior without review; treat these as the backlog.

### 1. `PROXY_SECRET` gate runs *after* the AJAX handlers — HIGH
The action handlers appear above the `PROXY_SECRET` check and each calls `Response.End`, so an AJAX request that matches a handler is served **before** the secret gate runs. Combined with a reachable origin, this means the header-based role check is the only thing standing between a direct caller and privileged data.
**Fix**: hoist the `PROXY_SECRET` check (and ideally the role-header parsing) to the very top of the file, before any handler dispatch, so nothing is served without the proxy secret. Pair with network-layer origin isolation.

### 2. `ccWriteTest` / `?diag=writetest` — ungated **and destructive** — HIGH
The diagnostic is reachable by GET and POST with **no role check**, and it discloses the app-pool identity. Worse, its first probe path opens `CC_DATA_DIR & "poc.txt"` for writing (truncating it) and its cleanup skips index 0, so **it destroys/overwrites POC data** — visible on the At-a-Glance board as corrupted content. Any authenticated user hitting the URL triggers it.
**Fix** (offered, not yet applied): probe a scratch file (e.g. `poc-probe.tmp`) instead of `poc.txt` and delete it after; require `SecOps.View`; make it POST-only (drop the QueryString trigger). Or remove the endpoint entirely if not needed.

### 3. `orgChart` handler ungated; `orgchart.csv` inside the web root — MEDIUM
`action=orgChart` has no role check, so any authenticated user can pull the full roster including **cell numbers and emails**. Additionally `orgchart.csv` lives at `C:\inetpub\wwwroot\tools\orgchart.csv` — **inside** the web root — so it's very likely directly fetchable at `/tools/orgchart.csv`, bypassing the handler entirely. The global people-search makes this data much easier to reach.
**Fix**: move `orgchart.csv` out of the web root (into `C:\inetpub\serv\`), serve it only through the handler, and decide whether the roster should be gated (at least strip cell/personal fields for non-privileged users).

### 4. Secrets as plaintext token files — MEDIUM (operational + exposure)
All API tokens sit as plaintext under `C:\inetpub\secrets\`. Expiry has caused real outages (AI logs, ticket feedback). Exposure risk if that directory's ACLs drift.
**Fix**: migrate to **Azure Key Vault** with a managed identity (rotation, expiry alerts, access audit). Minimum interim step: an expiry check in each job that emails a warning ~14 days out; use fine-grained, single-repo PATs.

### 5. Direct Send mail path is deprecating — MEDIUM (availability)
`tfShare` and the emailing jobs use unauthenticated Direct Send to the Exchange relay. Microsoft is enabling `RejectDirectSend` by default; when your tenant flips it, mail **silently fails** (NDR 550 5.7.68).
**Fix**: migrate to Graph `sendMail` (app-only), scoped with `New-ApplicationAccessPolicy` to the single sender mailbox. Removes both the deprecation risk and the connector-permission fragility.

## Practices that are already correct (keep them)

- Privileged handlers re-read `X-MS-APP-ROLES` and return `403` on mismatch — don't remove these even though App Proxy also gates access.
- Path inputs are regex-validated before building a file path (`ninjaRptMonth` requires `^\d{4}-\d{2}$`). **Apply this to every new handler that touches the filesystem.**
- `tfShare` validates recipient emails, strips CR/LF (mail-header-injection defense), caps recipients, and sets ReplyTo to the sender. Mirror this rigor in any new mail feature.
- Cache writers write to a `.tmp` then rename into place, so the app never reads a half-written file.

## If you change the role model
- Roles are defined on the App Registration and delivered in `X-MS-APP-ROLES`. Adding/renaming a role means updating: the App Registration, the two booleans (`canSeeSecOps`, `canSeeAll`), the server-side markup gates, and the per-handler `InStr(... )` checks. There is intentionally **no** separate SysAdmin role — SysAdmin rides on `SecOps.View`. Don't split it without updating all of the above.
