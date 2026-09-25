# Laufbursche Hitway Tool (hi-unlock)

**The page: https://laufbursche42.github.io/hi-unlock/**

A single-page Web Bluetooth tool for the Hitway H11 e-scooter, which runs on the Tuya white-label app (`com.hitway.ytd`). Connect, see the telemetry, settings and advanced settings, and parse an uploaded Bluetooth (btsnoop) capture. It speaks the Tuya BLE secure protocol and runs entirely in your browser. Nothing is sent to any server.

The H11 uses the generic Tuya data-point protocol over the advertised service `0xfd50`; the tool never invents a dpId, range or scaling.

> **This is a feasibility study.** It shows what the H11's Tuya BLE protocol makes possible from static analysis of the app and firmware, not a finished product. Error-free operation is not promised and there is no warranty of any kind.

> **Out of the box it does not tune your scooter.** It connects, reads and proves the Tuya BLE protocol; tuning and writing are gated behind the per-device localKey and the Tuya cloud dpId schema, which are cloud- and device-provisioned and not shipped in this tool. An over-BLE speed unlock is **not proven** on the H11 (see below).

## What it does

- Model selector: Auto / Generic Hitway, or Hitway H11 (dual-drive). Both show the full proven DP surface over BLE.
- Connect to the Tuya BLE service `0xfd50` (write `00000001-...07d0`, notify `00000002-...07d0`) or the classic profile `0x1910` (write `0x2b11`, notify `0x2b10`), and decode notifications: reassemble packets, verify CRC, and (with the session key) parse DP report entries.
- Derive the DP session-key candidate `secretKey5 = MD5(localKey || srand)` from the per-device localKey and the srand the device sends. **Honest note:** the Hitway SDK's exact key derivation is obfuscated; only the inputs (localKey, 6-byte srand, 32-byte authKey) are confirmed, so this MD5 form is the Tuya-standard candidate and is unverified for the H11.
- Telemetry tiles, settings rows and advanced settings rows, built from the proven Hitway DP code set. Values and writes stay greyed with a stated reason until you supply the per-device dpId (device schema) and the session keys.
- Device schema box: paste `{ "code": dpId }` (or the long form with type/pv) to unlock the matching rows. Nothing is hardcoded per model.
- **Bluetooth-log (btsnoop) upload + viewer:** load a local capture; the tool CRC-scans it for plaintext Tuya frames, surfaces srand/authKey candidates from the plaintext device-info reply, and suggests observed dpId+type pairs. Fully local; it **cannot** recover the localKey.
- Write raw DP escape hatch (you supply the dpId), a ride-mode sequencer, and a self-test of the crypto building blocks (MD5, AES-128, CRC16-MODBUS, varint, DP encoder).
- Full lb-tool-web-style protocol log: public-log anonymizer (on by default), diagnostics raw-frame decode, copy/clear/save.

## What it cannot do without the device or cloud

The static analysis proves the protocol, but these facts are cloud-provisioned or device-only and are gated, never invented:

1. The per-device **localKey** (Tuya cloud, issued at first pairing; extract via Frida on a rooted phone or the Tuya IoT platform - never from the MAC or a BLE capture).
2. The numeric **dpId**, enum **range** and **scaling** of every DP (Tuya cloud product schema).
3. The exact **session-key KDF** (obfuscated in the SDK; only the inputs are confirmed).
4. Whether any **DP write** actually overrides the QS-S4 controller cap `MOTOR[0x3e]` or the UI speed limit - **not proven** on the device. The hard cap lives in the controller firmware, not in a BLE DP, so a BLE unlock at most lifts the percent throttle, not the hard cap.

Because of point 4 the page shows telemetry but no unlock button: it stays gated until an unlock is proven on a paired device with the extracted localKey and DP schema.

## Run it

Web Bluetooth needs a secure context (localhost or https); `file://` will not work.

```
cd hi-unlock
py -3 -m http.server 8000
```

Then open `http://localhost:8000` in Chrome or Edge (desktop or Android). On iPhone use the Bluefy app.

## Files

- `index.html`, `app.js`, `i18n.js`, `styles.css` - the app (self-contained, strict CSP, no third-party code).
- `GUIDE.*.md`, `LICENSE*.md`, `PRIVACY*.md`, `TRADEMARKS*.md` - documentation.
- `scripts/` - `security-scan.py` (XSS/secret/structure scan) and `check-i18n.js` (de/en key parity).
- `.githooks/`, `.github/workflows/` - pre-commit/pre-push gates and CI (security scan + CodeQL).

## License

PolyForm Noncommercial 1.0.0 plus additional terms. See [LICENSE.md](LICENSE.md). Free for private research and tinkering, not for commercial use.

## Legal

Only on your own vehicle on private ground. Raising the top speed removes the throttle limit, the road approval (ABE) lapses and riding on public roads is then not allowed. Everything at your own risk.
