# Guide

This page talks over Web Bluetooth to a Hitway H11 e-scooter that runs on the Tuya white-label app (`com.hitway.ytd`). Pick your model, then connect. It is a feasibility study and stays honest about what already works and what still needs the real device.

## What you need

- A Hitway H11 that connects through the Tuya app. It advertises the Tuya BLE service `0xfd50`.
- A browser with Web Bluetooth: Chrome or Edge on Android or desktop, or the Bluefy app on iPhone.
- Your scooter's per-device localKey (from the Tuya cloud) and the numeric dpIds of the functions you want to use (from the cloud schema or a BLE capture). Both are explained below.

## The honest situation in one paragraph

The Bluetooth transport is understood and implemented in this page. The H11 speaks the generic Tuya data-point (DP) protocol. But four things are cloud- or device-side and not in the app package: (1) the localKey, (2) the numeric dpIds and their ranges/scaling, (3) the exact session-key derivation (the Hitway SDK obfuscates the KDF math; only the inputs are confirmed), and (4) whether any DP write actually changes the speed at all. Until you supply the first two, every write and every telemetry scaling stays greyed out with a stated reason. Point four is the big one: **an over-BLE speed unlock is not proven on the H11.**

## Why a BLE unlock is not proven (read this)

The hard speed cap on the H11 does not live in a Tuya data point. It lives in the controller firmware as `MOTOR[0x3e]`, fed mainly from the display's 1200-baud UART frame (`SpeedLevel * 2400 + 12000`). A BLE write reaches the Tuya module's MCU, not automatically the display line, and the controller only has a QS-S4 display parser - no Tuya-frame handling. So:

- A BLE write can at most lift a percent-throttle limit, **not** the hard cap.
- Raising the real top speed over 22 km/h needs the controller-side cap raised (UART / programmer / SWD routes), not BLE alone.
- Whether setting `speed_limit_enum` or `max_speed` over BLE changes anything is neither proven nor disproven; the first write test after localKey extraction answers it.

This tool therefore shows telemetry and offers a gated write path, but it does not pretend to have an unlock button.

## Step 1: Pick your model

Use the Model dropdown at the top.

- **Auto / Generic Hitway (Tuya)** - the full proven DP surface, everything gated until confirmed.
- **Hitway H11 (dual-drive)** - the same set, labelled for the H11. The H11 has two controllers (B master, A slave); the app link is a single Tuya BLE connection.

## Step 2: Connect

1. Turn the scooter on.
2. Tap Connect, pick your scooter from the list. The page filters on the Tuya service `0xfd50` and the classic `0x1910` (the SDK ships both profiles and which one the H11 exposes is unproven offline).
3. After connecting, the raw notifications appear in the log. The plaintext device-info reply (opcode `0x00`) contains the `srand` (6 bytes) the page needs for the session key.

## Step 3: Enter the keys

1. Enter your scooter's localKey (16 chars or as hex).
2. Enter the `srand` you read from the device-info notification in the log.
3. The page then shows `secretKey5 = MD5(localKey concatenated with srand)`.

**Honest caveat on the key derivation:** in the decompiled Hitway SDK the exact key-derivation math is obfuscated. What is confirmed is only the *inputs*: the cloud localKey, the 6-byte srand, and a 32-byte authKey. The `secretKey5 = MD5(localKey || srand)` shown here is the Tuya-standard candidate; whether the H11 uses exactly that is unverified. If a correctly built, armed frame is rejected, the KDF or the DP-channel path is the likely reason, not a bug in the tool.

### Where the localKey comes from

The localKey is assigned at first pairing to your Tuya account and lives in the cloud. It never travels over Bluetooth, so you cannot read it from a BLE capture, and it is not in the firmware dump (the `0x2000` Base64 blocks there are provisioning keys, not the cloud localKey). It is account-bound: re-assigning the scooter to another account generates a new localKey.

Two routes get it:

1. **Frida on a rooted phone.** Hook `DeviceBean.getLocalKey`; the script dumps devId, localKey, productId and the schemaMap in one go. Needs root or a debuggable app.
2. **Tuya IoT platform (iot.tuya.com).** Create a free account, make a cloud project, under Devices link your app account by QR, then in the API Explorer call `GET /v1.0/devices/{device_id}` for `local_key` and `.../specifications` for the schema. OEM white-label accounts sometimes cannot be linked directly; then also pair the scooter in Tuya's generic Smart Life app and link that.

The localKey is a secret of your device; do not share it.

## Step 4: Load the device schema (dpIds)

The numeric dpId of a function is not in the app and is hardcoded in no model - it lives in the Tuya cloud product schema. The known code names for the H11 include `speed_limit_enum`, `max_speed`, `work_mode`, `boost`, `zero_start`, `cruise_switch`, `assist`, and the telemetry codes `speed`, `battery_percentage`, `voltage`, `current`, `mileage_total`, `endurance_mileage`, `temp_value`, `fault`. Their numbers are unknown offline.

Enter what you know in the Device schema box as JSON, mapping code to dpId:

```
{ "speed_limit_enum": 104, "work_mode": 108, "battery_percentage": 106 }
```

The long form lets you also set the wire type and protocol version per entry:

```
{ "work_mode": { "dpId": 108, "type": "enum", "pv": 3 } }
```

Every row whose dpId you enter unlocks (once the keys are also in). Rows without a dpId stay greyed with a "schema needed" badge. Do not trust the doc's section 4.4 sample values (`max_speed` id 104, `speed_limit_enum` id 105) - those are explicit placeholders, not the real device schema.

## Step 5: Parse a Bluetooth log (btsnoop)

The Keys card has a Bluetooth-log upload. Load a btsnoop `.log`, a `.gz`, an Android bug-report `.zip`, or a hex-text dump. Everything is parsed locally (the CSP forbids any upload).

What a capture reveals and does not:

- **Reveals:** the plaintext device-info reply (opcode `0x00`), so the tool surfaces `srand` (6 bytes) and `authKey` (32 bytes) candidates, plus all frame metadata; and the encrypted DP frames as ciphertext (counted only).
- **Does not reveal:** the localKey (never sent over the air), the session AES key (obfuscated KDF over localKey + srand + authKey), and therefore the cleartext of any DP value. You can harvest srand and authKey, but without the localKey the DP payloads stay opaque.

So the log helps you fill `srand` and suggest `dpId` candidates; it can never give you the localKey. Writing stays gated afterwards.

## Step 6: Telemetry, settings and advanced settings

- **Live values** are filled from the DP report once a matching dpId is in the schema. Values are shown raw with an "unconfirmed" badge, because the scaling and unit are cloud-side and not confirmed here.
- **Settings** and **Advanced settings** show one labeled row per DP. Each row is greyed until its dpId is in the schema and the session is ready (localKey + srand + connected). Risky writes (speed limit, `work_mode`, locks) ask for an extra confirmation.
- **Write raw DP** is the escape hatch: you supply the dpId yourself, so it works under the session gate alone. Preview only computes and logs the frame; Armed actually sends it.

Honest caveat: even a correctly built armed frame may be rejected by the device (KDF/DP-channel), and even if accepted it may not change the speed (the hard cap is in the controller). That is not a bug in the tool.

## Step 7: Ride-mode sequencer

Unlike some other scooters, the H11 has **no documented boost trick**. This card only switches the ride-mode DP (`work_mode`) over BLE, with optional voice prompts so both hands stay on the bars. Whether a BLE mode change does anything on the H11 is unconfirmed, and it cannot lift the hard firmware cap. Treat it as an experiment, on private ground only.

## Diagnostics and self-test

- Public log (on by default) masks sensitive spans and long hex runs (localKey, serial, srand) before you copy or save, so a shared log leaks nothing.
- Diagnostics shows every decoded frame (reassembled, control byte, seq/cmd/flag, per-DP dpId/type/value) and taps extra notify channels. This is the capture you need to learn the dpIds.
- Diagnostics: all devices lists the GATT services of any picked device.
- Self-test checks the crypto building blocks against known test vectors. It must be green.
- Copy log / Save log give you the whole capture to keep or send.

## Legal and safety

Only on your own vehicle on private ground. Raising the speed removes the throttle limit, the road approval (ABE) lapses and riding on public roads is then not allowed. Riding a modified scooter in public traffic very likely constitutes criminal offences (driving without the required licence and a breach of the compulsory insurance act). Everything at your own risk.
