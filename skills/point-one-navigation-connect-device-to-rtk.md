---
name: Connect a device to the Point One Polaris RTK network
description: Get a GNSS receiver receiving centimetre-accurate RTCM corrections from Polaris, choosing between the NTRIP interface and the native Polaris protocol.
api: point-one-navigation:polaris-ntrip
operations:
- POST /api/v1/auth/ntrip
- POST /api/v1/auth/token
generated: '2026-08-05'
method: generated
source: https://support.pointonenav.com/polaris-ntrip-api-docs
---

# Connect a device to Polaris RTK

Two interfaces reach the same corrections network. Pick one:

- **NTRIP** — use this if the receiver already speaks NTRIP (Septentrio, NovAtel, u-blox, Emlid, Bad Elf, SparkFun, Trimble). No Point One code required.
- **Native Polaris protocol** — use this when you want the C/C++ client, a non-NTRIP integration, or tighter control over the stream.

## Prerequisites

- A Polaris API key from <https://app.pointonenav.com>.
- A connection identifier that is **unique across every concurrent session on that key**. Two sessions sharing an identifier conflict and neither receives corrections. Alphanumerics plus `-` and `_`; max 32 chars for NTRIP, 36 for native.

## Path A — NTRIP

1. Choose the regional caster:
   - North America: `truertk.pointonenav.com:2102` (TLS). Port `2101` is plaintext and not recommended.
   - Europe: `truertk-eu.pointonenav.com:2102` (TLS).
2. Choose the mountpoint: `ITRF2014` or `LOCAL`.
3. Get credentials one of two ways:
   - Use the username/password Point One issued you, **or**
   - `POST https://api.pointonenav.com/api/v1/auth/ntrip` (no auth on the request) with your API key, grant type, token type and a unique device identifier. It returns a temporary username/password pair valid roughly 24 hours. An invalid key returns **403 FORBIDDEN**.
4. Connect with HTTP Basic auth. The caster answers `ICY 200 OK` for NTRIP 1.0 and standard chunked `HTTP/1.1` for NTRIP 2.0.
5. Send your position as **NMEA GGA** — in the initial request header, then as continuous GGA lines. Corrections do not flow until the caster knows where you are.
6. Consume the **RTCM v3.2 MSM4** stream. It carries GPS (L1C/A, L2P, L2C, L5), GLONASS (G1, G2), Galileo (E1, E5a, E5b), BeiDou (B1-I, B1-C, B2-I, B2a, B2b, B3I) and QZSS where applicable.

## Path B — native Polaris protocol

1. `POST https://api.pointonenav.com/api/v1/auth/token` (no auth on the request):
   ```json
   {"grant_type":"authorization_code","token_type":"bearer","authorization_code":"<API key>","unique_id":"<unique id>"}
   ```
   Response: `{"token_type":"bearer","access_token":"<token>","expires_in":"604800","issued_at":"<timestamp>"}`. Invalid key returns **403 FORBIDDEN**.
2. Open a TLS stream to `rtk.pointonenav.com:8090` (`8088` is plaintext, not recommended).
3. Send the **Authorization** system message — class `0xE0`, id `0x01`, payload the access token as ASCII.
4. Report position periodically as a system message: ECEF (`0xE0`/`0x03`, signed 32-bit X/Y/Z in centimetres) or LLA/WGS84 (`0xE0`/`0x04`, lat/lon as signed 32-bit scaled by 1e7, altitude in millimetres).
5. Read RTCM3 corrections off the socket. **They are not wrapped in the system-message envelope** — frame them separately.

Message envelope: `0xB5 0x62 <CLASS> <ID> <LEN_DATA:2> <DATA> <CHK:2>`, little-endian, checksum over everything after the first two bytes.

Use the first-party clients rather than writing the framer: <https://github.com/PointOneNav/polaris> (C and C++).

## Rules and gotchas

- **Token expiry does not drop an established stream.** A connected client keeps receiving data after `expires_in` elapses; expiry only blocks *new* connections. Do not treat a live stream as proof the token is still valid.
- Both token endpoints are **POST-only** — a GET returns 405.
- There is **no documented idempotency contract and no documented rate limit**. Back off on your own schedule.
- Network SLA is 99.99% aggregate with `<<5s` station-to-device latency; check <https://status.pointonenav.com/> before blaming your integration.

## Related

- Auth detail: `authentication/point-one-navigation-authentication.yml`
- Errors: `errors/point-one-navigation-problem-types.yml`
- Conventions: `conventions/point-one-navigation-conventions.yml`
