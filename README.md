# Baldwin County CIS Toolkit — Link Configuration

This repository contains the link definitions for the **Baldwin County CIS Toolkit** web application. Each pane in the toolkit pulls its buttons from a JSON file in this repo at page load — no changes to the ASP file are needed to add, remove, or update links.

---

## Files

| File | Pane |
|---|---|
| `links-tools.json` | Hardware & Connection Tools (visible to all users) |
| `links-quicklinks.json` | Quick Links (Tech.Basic and above) |
| `links-secops.json` | SecOps Portal (SecOps.View role) |
| `links-sysadmin.json` | SysAdmin (SecOps.View role) |

---

## JSON Format

Each file contains a JSON array of button objects. Every object requires three fields — `label`, `icon`, and `action` — plus additional fields depending on the action type.

```json
[
  { "label": "Button Name", "icon": "fa-solid fa-icon-name", "action": "newtab", "url": "https://example.com" }
]
```

---

## Action Types

### `newtab`
Opens a URL in a new browser tab. Use this for all standard external and internal links.

```json
{ "label": "Defender", "icon": "fa-solid fa-shield", "action": "newtab", "url": "https://security.microsoft.com/" }
```

---

### `protocol`
Opens a protocol-based URL (e.g. `toolkit://`, `mailto:`). Does **not** open in a new tab — use this only for custom protocol handlers.

```json
{ "label": "Launch Signing Tool", "icon": "fa-solid fa-file-signature", "action": "protocol", "url": "toolkit://sign-script" }
```

---

### `shadowbox`
Triggers one of the toolkit's built-in overlay panels. The `target` field must match the button ID exactly as listed in the table below.

```json
{ "label": "Warranty Report", "icon": "fa-solid fa-file-contract", "action": "shadowbox", "target": "btnWarrantyReport" }
```

**Available shadowbox targets:**

| Target ID | Opens |
|---|---|
| `btnScreenTest` | Screen color cycle test |
| `btnMicTest` | Microphone level meter |
| `btnCamTest` | Camera preview |
| `btnSpeedTest` | Speed test (Speedtest Custom) |
| `btnKeyTest` | Keyboard key tester |
| `btnIPLookup` | IP address geolocation lookup |
| `btnMHA` | Message Header Analyzer |
| `btnCountyMap` | Baldwin County Google Map |
| `btnKnoxEnroll` | Samsung Knox Enrollment QR code |
| `btnWarrantyReport` | Warranty Report iframe |
| `btnPrinterLookup` | Printer Lookup (search by name, IP, or MAC) |

---

## Icons

Icons use [Font Awesome 6](https://fontawesome.com/icons). The full icon class string goes in the `icon` field.

```json
"icon": "fa-solid fa-database"
"icon": "fa-brands fa-github"
"icon": "fa-regular fa-envelope"
```

- `fa-solid` — filled style (most common)
- `fa-regular` — outline style
- `fa-brands` — brand logos (Google, GitHub, Discord, etc.)

Browse available icons at **[fontawesome.com/icons](https://fontawesome.com/icons)** — search for what you need and copy the class string shown on the icon's page.

---

## Adding a Link

1. Open the appropriate JSON file for the pane you want to update
2. Add a new object to the array — order in the file = order on screen
3. Commit and push — the toolkit picks up changes on next page load (no cache)

**Example — adding a new SysAdmin link:**
```json
[
  { "label": "Cohesity 1", "icon": "fa-solid fa-database", "action": "newtab", "url": "https://bcccohesity1.internal.co.baldwin.al.us/" },
  { "label": "My New Tool", "icon": "fa-solid fa-wrench",   "action": "newtab", "url": "https://mytool.example.com/" }
]
```

---

## Removing a Link

Delete the object from the array and commit. The button will disappear on next page load.

---

## Updating a Link

Edit the `url`, `label`, or `icon` fields directly and commit. No ASP changes needed.

---

## Notes

- Button order on screen matches the order in the JSON array
- If a JSON file fails to load (network error, malformed JSON), the pane will show a warning message instead of the buttons
- The toolkit fetches with `cache: 'no-store'` so changes appear immediately on next page load without needing a hard refresh
- JSON must be valid — use [jsonlint.com](https://jsonlint.com) to check if buttons stop appearing after an edit
