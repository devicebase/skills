---
name: devicebase
description: "Devicebase CLI — cross-platform tool for remote device control across three platforms: mobile (Android/HarmonyOS/iOS), browser (Chrome/Chromium/Edge over CDP), and computer (macOS/Windows/Linux desktops). Supports tap, swipe, text input, DOM automation, mouse and keyboard control, app launching, shell access, screenshots, and UI hierarchy inspection."
---

# Devicebase CLI

A cross-platform tool for remote device control across three device platforms:

| Platform | Covers | Serial is |
|----------|--------|-----------|
| **mobile** | Android, HarmonyOS, iOS | the device serial (adb / hdc / ios) |
| **browser** | Chrome / Chromium / Edge over CDP | the registered browser device's `serialno` |
| **computer** | macOS / Windows / Linux desktops | the registered computer device's `serialno` |

Capabilities include tap, swipe, text input, DOM automation, mouse and keyboard control, app launching, shell access, screenshots, and UI hierarchy inspection.

## When to Activate

- Building AI agents or automation scripts that interact with mobile devices, browsers, or desktops
- Developing mobile app or web UI testing pipelines
- Creating remote device control workflows
- Implementing device farm or CI/CD integrations
- Any task requiring programmatic device interaction

## Quick Reference

### Command Tree

```text
devicebase
├── list-devices        # device discovery — no -s required
├── mobile ...          # Android / HarmonyOS / iOS  →  /v1/{action}/{serialno}
├── browser ...         # Chrome / Chromium / Edge   →  /api/browser/{serialno}/{action...}
└── computer ...        # Desktop                    →  /api/computer/{serialno}/{action}
```

Run `--help` at any level for full usage: `devicebase --help`, `devicebase browser --help`, `devicebase computer click --help`.

### Selecting a Device (`-s, --serialno`)

Every platform command requires `-s, --serialno`. There is no default. The root command and each platform group both declare the flag and bind the same value, so either position works:

```bash
devicebase mobile -s <serialno> tap 100,200     # group flag
devicebase -s <serialno> mobile tap 100,200     # root flag
```

The value means something different per platform — see the table above. A missing `-s` reports an error, and for browser/computer also a hint pointing at the matching query:

```text
Error: required flag(s) "--serialno" not set
HINT: find a browser device to control first: devicebase list-devices --type browser
```

### All Leaf Commands

| Group | Count | Commands |
|-------|-------|----------|
| root | 1 | `list-devices` |
| `mobile` | 18 | `tap` `double-tap` `long-press` `swipe` `back` `home` `launch-app` `stop-app` `stop-current-app` `bash` `input` `clear-text` `current-app` `dump-hierarchy` `device-info` `install-app` `install-status` `screenshot` |
| `browser` | 22 | `navigate` `refresh` `go-back` `go-forward` `input` `click` `fill` `select` `text` `attribute` `exists` `execute` `hotkey` `state` `tabs` `tab-open` `tab-close` `tab-close-all` `tab-switch` `launch` `close` `screenshot` |
| `computer` | 16 | `click` `double-click` `long-click` `move` `drag` `scroll` `type-text` `press` `hotkey` `position` `screen-size` `permissions` `launch-app` `wait` `bash` `screenshot` |

---

## Environment Setup

```bash
# Install Devicebase CLI on Linux/macOS (if not already installed)
which devicebase || curl -fsSL https://downloads.devicebase.cn/cli/install.sh | bash

# Install Devicebase CLI on Windows (if not already installed)
powershell -c "irm https://downloads.devicebase.cn/cli/install.ps1 | iex"

# Export API key (recommended for repeated use)
export DEVICEBASE_API_KEY="your_api_key"
```

On Windows:

```powershell
$env:DEVICEBASE_API_KEY = "your_api_key"
```

Or inline, for a one-off command:

```bash
DEVICEBASE_API_KEY=your_api_key devicebase list-devices
```

| Variable | Required | Description |
|----------|----------|-------------|
| `DEVICEBASE_API_KEY` | Yes | API key for authentication. Sent as `Authorization: Bearer <your_api_key>` on every request. Get one from https://www.devicebase.cn/ |
| `DEVICEBASE_BASE_URL` | No | API base URL. Defaults to `https://api.devicebase.cn`. |

A missing `DEVICEBASE_API_KEY` fails before any request is sent.

---

## Output and Exit Codes

Successful commands print the server's JSON envelope to **stdout** and exit `0`:

```json
{ "code": 200, "message": "success", "data": { "width": 1470, "height": 956 }, "timestamp": "2026-09-20 03:21:06" }
```

Failures print `Error: …` to **stderr** and exit `1`. There are two error layers, and the second one is easy to miss:

| Layer | Message shape | Example |
|-------|---------------|---------|
| Gateway (HTTP status is non-2xx) | `API error (HTTP <code>): <body>` | `API error (HTTP 404): {"code":404,"message":"设备不存在: nope"}` |
| Business (HTTP 200, non-2xx `code` inside the envelope) | `API error (code <n>): <body>` | `API error (code 502): {"code":502,"message":"Element not found: #x"}` |

The second layer matters for automation: the control API reports **action failures inside an otherwise successful response**, so guarding on the HTTP status alone would treat a failed action as a success. `devicebase … && next_step` is safe.

---

## Device Discovery

### list-devices

The only command that takes no `-s`, and the entry point for every other command — it is how you learn a device's `serialno`.

```bash
devicebase list-devices
devicebase list-devices --type browser
devicebase list-devices --state free
devicebase list-devices --type mobile --keyword "Samsung" --state busy
```

| Flag | Description |
|------|-------------|
| `--keyword <kw>` | Case-insensitive substring match across `name` / `alias_name` / `brand` / `model` / `serialno` / `device_sn` / `type` / `os_type` / `os_version` / `location` / `operator` |
| `--state <state>` | `busy` / `free` / `offline` |
| `--type <type>` | Category (`mobile` \| `browser` \| `computer`) or system type (`android` \| `harmonyos` \| `ios` \| `macos` \| `windows` \| `linux` \| `chrome` \| `chromium` \| `edge` \| `other`) |

`--type` accepts two kinds of value:

- **Category buckets** — `mobile` (adb / hdc / ios), `browser`, `computer`. Use these to find a device for a platform group.
- **System types** — resolved against the device's `os_type`, because a device row only carries the coarse type: a Chrome browser is `type=browser` with `os_type=Chrome`.

Two system types are defined by exclusion: `linux` is every computer that is neither macOS nor Windows (Deepin / UOS / Kylin and unknown systems included), and `other` is every browser that is not Chrome / Chromium / Edge. An unrecognised value falls back to an exact match on the device `type`.

**The `serialno` is the device's `serialno` field** (e.g. `db-mttul4i41di8`). The `device_sn` UUID also resolves — the gateway looks devices up with `WHERE (serialno = ? OR device_sn = ?)`.

**Common workflows:**

```bash
# Find a free device of any kind
devicebase list-devices --state free

# Find a browser to drive
devicebase list-devices --type browser

# Find a computer by platform
devicebase list-devices --type macos

# Find a specific device by model
devicebase list-devices --keyword "Pixel"
```

---

## Mobile Platform

Serial: a mobile device serial (adb / hdc / ios). Endpoints: `POST/GET /v1/{action}/{serialno}`. Coordinates keep the original CLI style — points as `x,y`, bounds as `x1,y1,x2,y2`.

### Touch

| Command | Description |
|---------|-------------|
| `tap <x,y>` | Single tap at coordinates |
| `double-tap <x,y>` | Double tap at coordinates |
| `long-press <x,y>` | Press and hold at coordinates |
| `swipe <x1,y1,x2,y2>` | Swipe from start point to end point |

```bash
devicebase mobile -s <serialno> tap 100,200
devicebase mobile -s <serialno> double-tap 540,960
devicebase mobile -s <serialno> long-press 200,400
devicebase mobile -s <serialno> swipe 540,1600,540,200   # pull to refresh
```

```bash
# Tap center-bottom of a 1080x1920 screen
devicebase mobile -s <serialno> tap 540,1800

# Long press to trigger a context menu
devicebase mobile -s <serialno> long-press 540,960
```

### Navigation

| Command | Description |
|---------|-------------|
| `back` | Press the back button |
| `home` | Press the home button |

```bash
devicebase mobile -s <serialno> back
devicebase mobile -s <serialno> home
devicebase mobile -s <serialno> launch-app com.android.settings
```

### Apps and Shell

| Command | Description |
|---------|-------------|
| `launch-app <app>` | Launch an app by package name (Android/HarmonyOS) or bundle ID (iOS) |
| `stop-app <app>` | Stop an app |
| `stop-current-app` | Stop the foreground app |
| `bash <command>` | Run a shell command on the device (adb / hdc only) |

```bash
devicebase mobile -s <serialno> launch-app com.tencent.mm
devicebase mobile -s <serialno> stop-app com.tencent.mm
devicebase mobile -s <serialno> stop-current-app
devicebase mobile -s <serialno> bash "ls -la /sdcard"
```

An empty app name is rejected before any request is sent:

```text
Error: app_name cannot be empty
```

### Text Input

| Command | Description |
|---------|-------------|
| `input [text]` | Type text into the focused field |
| `clear-text` | Clear the focused text field |

`input` takes the text as an argument, or reads it from **stdin** when the argument is omitted — which keeps secrets out of shell history:

```bash
devicebase mobile -s <serialno> input "Hello World"
echo "hello 世界" | devicebase mobile -s <serialno> input
```

Quote text containing spaces or special characters:

```bash
devicebase mobile -s <serialno> input "user@example.com"
devicebase mobile -s <serialno> input "Password123!"
devicebase mobile -s <serialno> clear-text
```

The device must already have a text field focused — tap it first:

```bash
devicebase mobile -s <serialno> tap 540,100      # tap the search box
devicebase mobile -s <serialno> input "android"  # type the query
```

### Device State

| Command | Description |
|---------|-------------|
| `current-app` | Get the foreground app identifier |
| `device-info` | Device status, hardware, OS version, screen resolution, battery |
| `dump-hierarchy` | UI accessibility tree as JSON |

```bash
devicebase mobile -s <serialno> current-app
devicebase mobile -s <serialno> device-info
devicebase mobile -s <serialno> dump-hierarchy
```

`dump-hierarchy` returns nodes with attributes such as `resource-id`, `text`, `class`, `bounds`, `clickable`, `enabled` — use it to find coordinates instead of guessing:

```bash
devicebase mobile -s <serialno> dump-hierarchy | jq '.nodes[] | select(.text == "Submit") | .bounds'
```

### Installing Apps

`install-app` takes a path **on the agent host**, not a local file, and returns an `install_id` for a background task. Poll it with `install-status`:

```bash
devicebase mobile -s <serialno> install-app /tmp/app.apk
devicebase mobile -s <serialno> install-status <install_id>
```

---

## Browser Platform

Serial: the browser device's `serialno` from `devicebase list-devices --type browser`. Endpoints: `POST/GET /api/browser/{serialno}/{action...}`. Selectors are **CSS selectors**.

### Navigation

| Command | Description |
|---------|-------------|
| `navigate <url>` | Navigate the current tab to a URL |
| `refresh` | Reload the current page |
| `go-back` | Navigate back in history |
| `go-forward` | Navigate forward in history |

```bash
devicebase browser -s <serialno> navigate https://example.com
devicebase browser -s <serialno> refresh
devicebase browser -s <serialno> go-back
```

### DOM Operations

| Command | Description |
|---------|-------------|
| `click <selector>` | Click an element |
| `fill <selector> <text>` | Clear and type into an input |
| `select <selector> <value>` | Pick an option in a dropdown |
| `text <selector>` | Get an element's text content |
| `attribute <selector> <name>` | Get an element attribute |
| `exists <selector>` | Check whether an element exists |
| `execute <js>` | Evaluate JavaScript in the page |

```bash
devicebase browser -s <serialno> click "button#submit"
devicebase browser -s <serialno> fill "#search" "devicebase"
devicebase browser -s <serialno> select "#country" "CN"
devicebase browser -s <serialno> text "#search"
devicebase browser -s <serialno> attribute "a.logo" href
devicebase browser -s <serialno> exists ".modal"
devicebase browser -s <serialno> execute "document.title"
```

`execute` is **danger tier** — the same reach as shell access, because the script runs inside the page with the page's privileges.

### Text Input

| Command | Description |
|---------|-------------|
| `input [text]` | Insert text into the focused element |

Like the mobile `input`, the text may come from stdin:

```bash
devicebase browser -s <serialno> input "hello 世界"
echo "hello 世界" | devicebase browser -s <serialno> input
```

This uses CDP `Input.insertText`, which is reliable for CJK — unlike synthesised key events.

### Keyboard

| Command | Description |
|---------|-------------|
| `hotkey <keys...>` | Press keys together |

```bash
devicebase browser -s <serialno> hotkey Meta a
devicebase browser -s <serialno> hotkey Backspace
```

Editing shortcuts (select-all, cut, copy, undo, redo) act on the page. **Browser-chrome shortcuts such as `Control t` are not reachable** — CDP drives the page, not the browser UI, so keys aimed at the window itself do nothing.

### Tabs and State

| Command | Description |
|---------|-------------|
| `state` | URL / title / viewport / tab count |
| `tabs` | List open tabs |
| `tab-open <url>` | Open a new tab |
| `tab-close <id>` | Close a tab |
| `tab-close-all` | Close every tab |
| `tab-switch <id>` | Focus a tab |

```bash
devicebase browser -s <serialno> state
devicebase browser -s <serialno> tabs
devicebase browser -s <serialno> tab-open https://example.com
devicebase browser -s <serialno> tab-switch <tab_id>
devicebase browser -s <serialno> tab-close <tab_id>
devicebase browser -s <serialno> tab-close-all
```

### Lifecycle

| Command | Description |
|---------|-------------|
| `launch` | Start the browser / CDP endpoint |
| `close` | Stop the browser / CDP endpoint |

```bash
devicebase browser -s <serialno> launch
devicebase browser -s <serialno> close
```

---

## Computer Platform

Serial: the computer device's `serialno` from `devicebase list-devices --type computer`. Endpoints: `POST/GET /api/computer/{serialno}/{action}`. Coordinates are **absolute screen pixels**, points as `x,y` and bounds as `x1,y1,x2,y2` — same style as mobile.

### Mouse

| Command | Description |
|---------|-------------|
| `click <x,y> [--button left\|right\|middle]` | Click (button omitted → server default `left`) |
| `double-click <x,y>` | Double click (left button) |
| `long-click <x,y> [--seconds N]` | Press and hold for N seconds (1-60) |
| `move <x,y>` | Move the mouse without clicking |
| `drag <x1,y1,x2,y2>` | Press at the first point, move to the second, release |
| `scroll <up\|down\|left\|right> [--amount N]` | Mouse-wheel scroll |

```bash
devicebase computer -s <serialno> click 640,360
devicebase computer -s <serialno> click 640,360 --button right
devicebase computer -s <serialno> double-click 640,360
devicebase computer -s <serialno> long-click 640,360 --seconds 2
devicebase computer -s <serialno> move 100,100
devicebase computer -s <serialno> drag 100,100,800,600
devicebase computer -s <serialno> scroll down --amount 5
```

### Keyboard

| Command | Description |
|---------|-------------|
| `type-text <text>` | Type text at the current caret |
| `press <key>` | Press a single key (e.g. `Enter`, `F5`) |
| `hotkey <keys...>` | Press a key combination |

```bash
devicebase computer -s <serialno> type-text "hello"
devicebase computer -s <serialno> press Enter
devicebase computer -s <serialno> hotkey Control Shift Escape
```

### System and Apps

| Command | Description |
|---------|-------------|
| `position` | Current mouse position |
| `screen-size` | Primary screen size |
| `permissions` | Desktop-control permission status |
| `launch-app <app>` | Launch a desktop app |

```bash
devicebase computer -s <serialno> position
devicebase computer -s <serialno> screen-size
devicebase computer -s <serialno> permissions
devicebase computer -s <serialno> launch-app "Visual Studio Code"
```

### Wait

| Command | Description |
|---------|-------------|
| `wait <ms>` | Block for a duration in milliseconds (1-300000) |

Useful between steps in agent scripts.

```bash
devicebase computer -s <serialno> wait 2000
```

`wait` takes **milliseconds** — note that `bash --timeout` takes **seconds**.

### Shell

| Command | Description |
|---------|-------------|
| `bash <command> [--timeout N]` | Run a shell command on the host machine |

**Danger tier** — the same reach as shell access. The command runs as the desktop user, unsandboxed, under the platform default shell (`/bin/sh` on macOS/Linux, `cmd.exe` on Windows), so bash-only syntax such as `[[ ]]` may not work.

```bash
devicebase computer -s <serialno> bash "ls -la"
devicebase computer -s <serialno> bash "sleep 5 && echo done" --timeout 30
```

Quote the command so the CLI does not try to parse its flags.

`--timeout` is in **seconds** (0-600; 0 or omitted → server default 120). A non-zero command exit is reported in `data.exitCode`; the CLI still exits `0` because the API call itself succeeded — parse `data.exitCode` if you need the command's own status.

---

## Screenshot (Cross-Family)

`screenshot` is the one **cross-family** command: it does not live under `/api/browser/*` or `/api/computer/*`. The server dispatches `POST /v1/screen/{serialno}` by device type — computer → full-desktop capture, browser → CDP capture, otherwise the device image queue — so one command serves every platform, and it is registered in each group so `--help` surfaces it.

```bash
devicebase mobile   -s <serialno> screenshot -o screen.jpg
devicebase browser  -s <serialno> screenshot -o screen.jpg
devicebase computer -s <serialno> screenshot -o screen.jpg
```

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--output` | `-o` | stdout | Output file path |

The image goes to **stdout** by default, which suits piping:

```bash
devicebase mobile -s <serialno> screenshot > screen.jpg
devicebase mobile -s <serialno> screenshot | base64
```

The **server** decides the format (JPEG), independently of the output file name — a `.png` target still receives JPEG bytes. That contradiction is reported on stderr rather than silently producing a mislabelled file:

```text
Screenshot saved to shot.png
Warning: shot.png has a .png extension but the server returned JPEG data
```

Full inspection pipeline:

```bash
devicebase mobile -s <serialno> screenshot -o /tmp/screen.jpg
devicebase mobile -s <serialno> dump-hierarchy > /tmp/hierarchy.json
devicebase mobile -s <serialno> device-info > /tmp/device.json
```

---

## AI Agent Integration

### ⚠️ Coordinate Normalization (CRITICAL)

When using LLM-based vision models to analyze screenshots and determine tap/interaction coordinates:

1. **LLM returns normalized coordinates** in the range `[0, 1000]` (both x and y).
2. **NEVER use normalized coordinates directly** — they are NOT pixel coordinates.
3. **You MUST convert to actual pixel coordinates** using the device's screen resolution:

```
actual_x = normalized_x * screen_width / 1000
actual_y = normalized_y * screen_height / 1000
```

**Example workflow:**

```bash
# Step 1: Get screen resolution
devicebase mobile -s <serialno> device-info
# Output includes: Screen: 1080x1920

# Step 2: Take a screenshot and send it to the LLM
devicebase mobile -s <serialno> screenshot -o /tmp/screen.jpg
# LLM analyzes and returns: tap at (500, 300) [normalized]

# Step 3: Convert normalized coordinates to actual pixels
# actual_x = 500 * 1080 / 1000 = 540
# actual_y = 300 * 1920 / 1000 = 576

# Step 4: Use the converted coordinates
devicebase mobile -s <serialno> tap 540,576
```

On the computer platform, `screen-size` gives the resolution to convert against instead of `device-info`.

**Wrong approach (will tap the wrong location):**

```bash
# LLM returns normalized (500, 300)
devicebase mobile -s <serialno> tap 500,300  # ❌ WRONG — this is NOT the correct position
```

**Correct approach:**

```bash
# LLM returns normalized (500, 300), screen is 1080x1920
devicebase mobile -s <serialno> tap 540,576  # ✅ CORRECT — converted to actual pixels
```

### Sequential Actions

```bash
devicebase mobile -s <serialno> launch-app com.android.settings
sleep 2
devicebase mobile -s <serialno> tap 540,500
sleep 1
devicebase mobile -s <serialno> input "Wi-Fi"
```

On the computer platform, prefer `wait` over a local `sleep` when the delay must be observed by the device side:

```bash
devicebase computer -s <serialno> wait 2000
```

### Conditional Execution

Exit codes are reliable for this, because business errors also exit non-zero:

```bash
if devicebase mobile -s <serialno> launch-app com.example.app; then
    echo "App launched successfully"
else
    echo "Failed to launch app"
    devicebase mobile -s <serialno> screenshot -o /tmp/error.jpg
    exit 1
fi
```

```bash
CURRENT=$(devicebase mobile -s <serialno> current-app)
if [ "$CURRENT" = "com.android.settings" ]; then
    devicebase mobile -s <serialno> back
else
    devicebase mobile -s <serialno> launch-app com.android.settings
fi
```

### Hierarchy-Driven Navigation

```bash
# Find a button's bounds, then tap its center
devicebase mobile -s <serialno> dump-hierarchy | jq -r '
  .nodes[]
  | select(.text == "Continue" and .enabled == true)
  | .bounds
'
```

### Web Automation

Prefer semantic selectors over coordinates — they survive layout changes:

```bash
devicebase browser -s <serialno> navigate https://example.com/login
devicebase browser -s <serialno> fill "#username" "myuser"
echo "$PASSWORD" | devicebase browser -s <serialno> input
devicebase browser -s <serialno> click "button[type=submit]"
devicebase browser -s <serialno> exists ".dashboard"
```

### Desktop Automation

```bash
devicebase computer -s <serialno> launch-app "Visual Studio Code"
devicebase computer -s <serialno> wait 3000
devicebase computer -s <serialno> hotkey Meta s
devicebase computer -s <serialno> screenshot -o /tmp/editor.jpg
```

### Multi-Device Orchestration

```bash
devicebase mobile -s device1_serial tap 100,200 &
devicebase mobile -s device2_serial tap 300,400 &
devicebase browser -s browser_serial navigate https://example.com &
wait
```

---

## Error Reference

### CLI Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `required flag(s) "--serialno" not set` | Missing `-s` | Add `-s <serialno>` |
| `invalid point format "x y", expected x,y` | Wrong separator | Use a comma: `100,200` |
| `invalid bounds format "…", expected x1,y1,x2,y2` | Wrong format | Use four comma-separated integers |
| `invalid x coordinate: …` | Non-numeric coordinate | Use integers only |
| `app_name cannot be empty` | Empty string to `launch-app` | Provide a valid package / bundle ID |
| `invalid button "…", expected one of: left, right, middle` | Bad `--button` | Use `left`, `right`, or `middle` |
| `invalid direction "…", expected one of: up, down, left, right` | Bad scroll direction | Use `up`, `down`, `left`, or `right` |
| `--seconds must be between 1 and 60` | Out-of-range `long-click` | Use 1-60 |
| `wait duration must be between 1 and 300000 ms` | Out-of-range `wait` | Use 1-300000 |
| `--timeout must be between 0 and 600 seconds …` | Out-of-range `bash --timeout` | Use 0-600 |
| `no text received on stdin — …` | `input` with no argument and empty stdin | Pass the text as an argument or pipe it |
| `unknown command` | Typo in a subcommand | Check the command name |

### API Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `API error (HTTP 401): …` | Invalid or missing API key | Check `DEVICEBASE_API_KEY` |
| `API error (HTTP 404): …` | Device not found | Verify the serialno, and that the device is connected |
| `API error (code 502): …` | The action failed in the driver (HTTP 200 with a non-2xx code) | Read the `message` — usually a missing element or an unsupported action |
| `API error (HTTP 500): …` | Server error | Check the Devicebase server logs |
| `request failed: connection refused` | Server unreachable | Verify `DEVICEBASE_BASE_URL` and that the server is running |

---

## Critical Rules

1. **Always pass `-s <serialno>`** — every platform command requires it, and there is no default. The value is platform-specific: a mobile device serial for `mobile`, a registered device `serialno` for `browser` and `computer`.
2. **Discover the serialno with `list-devices`** — `devicebase list-devices --type browser` before driving a browser, and likewise for `computer` and `mobile`.
3. **Set `DEVICEBASE_API_KEY`** before running. `DEVICEBASE_BASE_URL` is optional and defaults to `https://api.devicebase.cn`.
4. **Use comma-separated coordinates** — `100,200`, NOT `100 200`. Points are `x,y`; bounds are `x1,y1,x2,y2`.
5. **Quote text with spaces** — `input "Hello World"`.
6. **Business errors also exit non-zero** — an HTTP 200 response carrying a non-2xx `code` is reported as `API error (code N): …` and exits `1`. Do not treat HTTP 200 as success.
7. **Screenshot defaults to stdout** — use `-o file` to save to a file. The server decides the format (JPEG) regardless of the extension you choose.
8. **`bash` is danger tier** — on `mobile` it runs on the device; on `computer` it runs on the host machine as the desktop user, unsandboxed. Treat both as shell access.
9. **`wait` takes milliseconds; `bash --timeout` takes seconds** — the two commands on `computer` use different units.
10. **Convert LLM normalized coordinates** — an LLM returns `[0, 1000]` normalized coordinates, NOT pixel coordinates. You MUST convert them: `actual_x = normalized_x * screen_width / 1000`, `actual_y = normalized_y * screen_height / 1000`. Using normalized coordinates directly will tap the wrong location.
