# Camera Health — MyGeotab Add-In

A custom MyGeotab page that reports the health of every camera in the fleet
(Surfsight by Lytx and Geotab GO Focus Plus), with online/offline status, last
recording date, and drill-down into the active diagnostics behind each problem.

## Files

| File | Purpose |
|---|---|
| `cameraHealth.html` | The add-in page. Self-contained (HTML + CSS + JS), wired to the live MyGeotab `api`. |
| `config.json` | The add-in manifest you paste into MyGeotab. |
| `icon.svg` | Menu icon. |

## How it works

When MyGeotab loads the page it injects an authenticated `api` object. On every
focus the page calls (one `multiCall`):

- `Get Device` → asset/vehicle name, VIN, serial
- `Get DeviceStatusInfo` → `isDeviceCommunicating` + last-contact time → **online / offline**
- `Get MediaFile` (lookback window) → **last recording** per device
- `Get FaultData` + `Get Diagnostic` → **active diagnostics**, grouped into problem types

Status is then computed: **Offline** (no contact > 24h) › **Not working** (a
critical fault) › **Needs attention** (a warning fault or no recording > 72h) ›
**Healthy**. All thresholds are in the `CONFIG` block at the top of the JS.

Opening the page outside MyGeotab (e.g. the GitHub Pages URL directly) renders
**sample data** in "Preview mode" so you can see the layout.

## Setup

### 1. Host the files over HTTPS
MyGeotab requires the files to be served over HTTPS (TLS 1.2+). Two easy options
for a GitHub repo:

**Option A — jsDelivr (no setup, recommended).** Push these files to your repo,
then the raw files are served at:
```
https://cdn.jsdelivr.net/gh/<USER>/<REPO>@<BRANCH>/cameraHealth.html
https://cdn.jsdelivr.net/gh/<USER>/<REPO>@<BRANCH>/icon.svg
```
> jsDelivr caches aggressively. While iterating, pin a commit hash instead of a
> branch (`@<commit>`), or purge via `https://purge.jsdelivr.net/gh/<USER>/<REPO>@<BRANCH>/cameraHealth.html`.

**Option B — GitHub Pages.** Enable Pages on the repo (Settings → Pages → deploy
from branch). Files are served at `https://<USER>.github.io/<REPO>/cameraHealth.html`.

### 2. Point `config.json` at your URLs
Edit `config.json` and replace `YOUR_GH_USER` / `YOUR_REPO` (and the branch if
not `main`) in both the `url` and `icon` fields.

### 3. Install in MyGeotab
1. In MyGeotab go to **Administration → System → System Settings → Add-Ins**.
2. Click **New Add-In**.
3. Paste the contents of `config.json` into the **Configuration** box.
4. **Save**, then **fully reload** MyGeotab.
5. The page appears under **Activity → Camera Health** (change `path` in
   `config.json` to move it — e.g. `EngineMaintenanceLink/` or
   `AdministrationLink/`).

You need a security clearance that can read Device, DeviceStatusInfo, FaultData
and MediaFile.

## Tuning it to your database (important)

The defaults work generically, but two things should be validated against your
actual data. Both live at the top of the `<script>` in `cameraHealth.html`:

**`CONFIG` — camera detection & thresholds.**
A device is treated as a camera if it is in `cameraGroupIds`, has a MediaFile in
the lookback window, matches `cameraNameRegex`, or has a camera-related fault.
If you keep your cameras in a MyGeotab group, paste its ID into `cameraGroupIds`
for the cleanest result.

**`NAME_RULES` — diagnostic mapping.**
Each rule matches a raw MyGeotab Diagnostic **name** (regex) and maps it to a
friendly problem type, a provider (`ss`/`gfp`/`other`), and a severity
(`crit`/`warn`). To discover the real diagnostic names your cameras report, open
the add-in, open the browser console, and look for the
`[CameraHealth] unmapped diagnostics:` log — add a rule for each one you want to
surface. The `code` shown in the UI is the actual diagnostic name from your DB.

## Notes / next steps

- The diagnostic names/codes in `NAME_RULES` are best-guess starting points.
  One pass against your live portal will lock them to the exact strings Surfsight
  and GO Focus Plus emit in your fleet.
- If you want scheduled email/export of the report, or the native MyGeotab
  "Camera Health" status surfaced alongside this, that can be layered on.
