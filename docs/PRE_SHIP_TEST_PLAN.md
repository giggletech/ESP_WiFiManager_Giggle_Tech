# Giggle Spark Board — Pre-Ship Test Plan

Use this checklist to verify each board before shipping. Tests cover WiFi, IP (static/DHCP), config portal, persistence, and device features.

---

## Prerequisites

- **Test network**: Router with DHCP (e.g. 192.168.1.x), and known free static IPs in the same subnet (e.g. 192.168.1.69, .70, .71).
- **Tools**: Serial monitor (115200 baud), phone or PC for WiFi, browser, optional OSC client (e.g. TouchOSC, OSCulator).
- **Known values**: Gateway (e.g. 192.168.1.254), subnet 255.255.255.0, DNS (e.g. 8.8.8.8) to match your network.

---

## 1. First Boot / No Stored Config

**Goal**: Board behaves correctly when no WiFi credentials are stored (e.g. fresh flash or config deleted).

| Step | Action | Expected |
|------|--------|----------|
| 1.1 | Erase flash or delete `/wifi_cred.dat` (e.g. via LittleFS/SPIFFS upload or factory reset if you add one). | N/A |
| 1.2 | Power on board. | Serial: "Open Config Portal without Timeout: No stored Credentials." |
| 1.3 | AP appears: `GiggleTech_Haptics`. | Phone/PC sees SSID. |
| 1.4 | Connect to AP, open captive portal (or 192.168.232.1). | Config portal loads. |
| 1.5 | Enter router SSID + password, leave or set static IP fields as desired, save. | Board connects to WiFi. |
| 1.6 | Serial shows ">>> IP Address: x.x.x.x <<<". | IP is in expected range (DHCP or static). |
| 1.7 | Power cycle. | Board reconnects with same IP (config persisted). |

**Pass**: Config portal opens on first boot, credentials and IP save, board connects and survives reboot.

---

## 2. Double Reset → Config Portal

**Goal**: Double reset reliably opens config portal even when credentials exist.

| Step | Action | Expected |
|------|--------|----------|
| 2.1 | Board has stored WiFi (already configured once). | Connected to your network. |
| 2.2 | Double-tap reset within the DRD window (~10 s). | Serial: "Open Config Portal without Timeout: Double Reset Detected". |
| 2.3 | AP `GiggleTech_Haptics` appears. | Config portal reachable. |
| 2.4 | Connect, open portal; verify SSID/password and IP fields show current or default. | No blank or 0.0.0.0 crash. |
| 2.5 | Save (with or without changes). | Board saves and reconnects. |

**Pass**: Double reset always opens portal; no crash; save works.

---

## 3. Adding / Setting Static IP

**Goal**: Board accepts and uses a static IP from the config portal and persists it.

| Step | Action | Expected |
|------|--------|----------|
| 3.1 | Open config portal (double reset or first boot). | Portal with IP section visible. |
| 3.2 | Enter a **static IP** in your subnet (e.g. 192.168.1.69), gateway, subnet, DNS. | Fields accept values. |
| 3.3 | Save and let board reconnect. | Serial: ">>> IP Address: 192.168.1.69 <<<" (or your IP). |
| 3.4 | From PC: `ping 192.168.1.69`. | Replies. |
| 3.5 | Browser: `http://192.168.1.69/` or `http://giggletech.local/`. | Page shows "Giggletech Device IP: 192.168.1.69". |
| 3.6 | Power cycle. | Board comes back with same static IP. |

**Note**: Your firmware uses `user_dhcp_select` (from EEPROM) to decide DHCP vs static. If the board is built with `user_dhcp_select = true` (DHCP), it may ignore static IP until EEPROM is set to static (or you use a build with static default). For this test, use a build with `user_dhcp_select = false`, or ensure EEPROM byte at 0 is 0 so static is used.

**Pass**: Static IP set in portal is used, ping and HTTP work, and config survives reboot.

---

## 4. Switching to DHCP (from static)

**Goal**: Board can switch to DHCP and get an address from the router.

| Step | Action | Expected |
|------|--------|----------|
| 4.1 | Start from a board using **static** IP (e.g. 192.168.1.69). | Confirmed via Serial/HTTP. |
| 4.2 | Open config portal (double reset). | Portal opens. |
| 4.3 | In portal: clear static IP or set to 0.0.0.0 / use "Use DHCP" if the library shows it. Save. | Config saves. |
| 4.4 | If your firmware stores DHCP/static in EEPROM: ensure board uses DHCP (e.g. flash build with `user_dhcp_select = true` or add code to set EEPROM and reboot). | After reboot, board uses DHCP. |
| 4.5 | Reboot/power cycle. | Serial shows an IP in DHCP range (e.g. 192.168.1.x from router). |
| 4.6 | Ping and HTTP that new IP (or giggletech.local). | Reachable. |

**Pass**: After switching to DHCP, board gets a DHCP lease and is reachable; setting persists if your code persists it.

---

## 5. Switching Back to Static (from DHCP)

**Goal**: Board can switch from DHCP back to a chosen static IP.

| Step | Action | Expected |
|------|--------|----------|
| 5.1 | Start from board on **DHCP**. | Note current DHCP IP. |
| 5.2 | Double reset → config portal. | Portal opens. |
| 5.3 | Enter static IP (e.g. 192.168.1.70), gateway, subnet, DNS. Save. | Saved. |
| 5.4 | Ensure EEPROM/behavior is set to use static (e.g. build with static default or EEPROM byte = 0). Reboot. | Board uses 192.168.1.70. |
| 5.5 | Ping and HTTP to that static IP. | Reachable. |
| 5.6 | Power cycle. | Same static IP after reboot. |

**Pass**: Board reliably switches back to static and keeps it across reboot.

---

## 6. Removing / Clearing Static IP (revert to DHCP)

**Goal**: "Removing" the static IP results in DHCP behavior (no stale static).

| Step | Action | Expected |
|------|--------|----------|
| 6.1 | Board currently on **static** IP. | Known static IP. |
| 6.2 | Open config portal. Set static IP to 0.0.0.0 or clear the field (if UI allows), or use "Use DHCP". Save. | Portal saves; library may store 0.0.0.0. |
| 6.3 | Set firmware to use DHCP (EEPROM/build) and reboot. | Board requests DHCP. |
| 6.4 | Serial: IP is in router’s DHCP range. | No old static IP in use. |
| 6.5 | Optional: delete `/wifi_cred.dat` and reboot. | Board goes to "no credentials" and opens config portal again. |

**Pass**: Clearing static results in DHCP; no leftover static; optional file delete resets to portal.

---

## 7. Persistence and Power Cycle

**Goal**: Stored WiFi and IP survive power loss and multiple reboots.

| Step | Action | Expected |
|------|--------|----------|
| 7.1 | Configure board: WiFi + static IP. Note IP. | Connected. |
| 7.2 | Power off, wait 10 s, power on. | Same IP, connected. |
| 7.3 | Power off, wait 10 s, power on. Repeat 2–3 times. | Same behavior every time. |
| 7.4 | Repeat 7.1–7.3 with **DHCP** (if supported). | DHCP lease renewed or same IP as expected. |

**Pass**: No loss of WiFi or IP after multiple power cycles.

---

## 8. mDNS and HTTP

**Goal**: Device is discoverable and web page works.

| Step | Action | Expected |
|------|--------|----------|
| 8.1 | Board connected (any IP). | Serial shows IP. |
| 8.2 | On PC on same network: `http://giggletech.local/`. | Page loads. |
| 8.3 | Page body shows "Giggletech Device IP: x.x.x.x". | Matches Serial/localIP. |
| 8.4 | Try `http://<board-ip>/` (no path). | 200 OK, same content. |
| 8.5 | Try `http://<board-ip>/other`. | 404 Not Found. |

**Note**: **Windows often does not resolve `giggletech.local`** (mDNS/.local support is poor). Use the device **IP address** in the browser (e.g. `http://192.168.1.69/`) or install [Bonjour for Windows](https://support.apple.com/kb/DL999) — still not guaranteed. macOS/Linux and many phones resolve `.local` fine.

**Pass**: mDNS resolves on supported OS (or use IP); root URL returns correct IP; invalid path returns 404.

---

## 9. OSC (UDP 8888)

**Goal**: Haptics/LED respond to OSC.

| Step | Action | Expected |
|------|--------|----------|
| 9.1 | Send OSC to board IP, port **8888**: `/led` (int 0–255). | Serial: "OSC_0 rx: <value>"; external LED brightness changes. |
| 9.2 | Send `/motor` (int 0–255). | Serial: "OSC_1 rx: <value>"; motor/LED respond. |
| 9.3 | Send invalid or wrong format. | Serial may show error; no crash. |

**Pass**: `/led` and `/motor` control hardware; device stays stable.

---

## 10. Hardware Smoke Test (in loop)

**Goal**: LED and vibe output work.

| Step | Action | Expected |
|------|--------|----------|
| 10.1 | Power on. | Serial: "Full IO Test..."; LED and motor briefly pulse 3×. |
| 10.2 | Trigger via OSC (steps 9.1–9.2). | Visible/audible response. |

**Pass**: Boot IO test and OSC-driven outputs work.

---

## 11. Per-Board Summary Checklist

Use one row per board; note serial/ID if needed.

| # | Board ID / Serial | 1 First boot | 2 Double reset | 3 Static IP | 4 →DHCP | 5 →Static | 6 Remove IP | 7 Persist | 8 mDNS/HTTP | 9 OSC | 10 IO | Notes |
|---|-------------------|--------------|---------------|-------------|---------|-----------|-------------|----------|-------------|------|------|--------|
| 1 |                   | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |      |
| 2 |                   | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ | ☐ |      |
| … |                   |   |   |   |   |   |   |   |   |   |   |      |

---

## 12. Failure Notes

- **Config portal doesn’t open**: Check double-reset timing; ensure no flash corruption (reflash if needed).
- **Static IP not applied**: Confirm `user_dhcp_select`/EEPROM and that `configWiFi(WM_STA_IPconfig)` is used when static is selected; confirm gateway/subnet match router.
- **IP not persisted**: Confirm `saveConfigData()` is called after `getSTAStaticIPConfig()` when leaving config portal; check LittleFS/SPIFFS.
- **mDNS not resolving**: Same subnet required; try `<ip>/` directly to confirm HTTP works first.

---

## Quick Full Regression (one board)

Run in order on a single board to validate full flow:

1. **Factory**: Delete config → power on → config portal → set WiFi + static IP → save → verify IP and reboot.
2. **Static**: Confirm static IP in Serial/HTTP/ping; power cycle twice.
3. **Portal again**: Double reset → change static IP to another → save → verify new IP.
4. **DHCP**: In portal clear/disable static (or set DHCP), set EEPROM/build for DHCP, reboot → verify DHCP IP.
5. **Back to static**: Double reset → set static again → reboot → verify.
6. **Remove IP**: Clear static in portal, use DHCP, reboot → verify no static.
7. **mDNS**: `http://giggletech.local/` and `http://<ip>/`.
8. **OSC**: `/led` and `/motor`; confirm Serial and hardware.
9. **Persistence**: 3× power cycle with static, then with DHCP if used.

When all pass, use the per-board checklist for the rest of the batch.
