---
name: Manage a Point One device fleet over the GraphQL API
description: Register devices, tag them, apply seat licenses and attach RTK device profiles using the Point One platform GraphQL API.
api: point-one-navigation:graphql
operations:
- myDevices
- device
- createDevice
- createDevices
- updateDevice
- deleteDevices
- tags
- deviceTags
- setDeviceTag
- unsetDeviceTag
- licenses
- licenseTypes
- createLicenses
- createLicensesForDevices
- applyLicenseToDevice
- removeLicenseFromDevice
- updateLicenseAutoRenewal
- deviceProfiles
- deviceProfile
- createDeviceProfile
- updateDeviceProfile
- deleteDeviceProfile
- setDeviceProfile
- unsetDeviceProfile
generated: '2026-08-05'
method: generated
source: https://docs.pointonenav.com/docs/graphql-api/
---

# Manage a Point One device fleet

One endpoint: `https://graphql.pointonenav.com/graphql`.

## Authenticate

Create a **Personal Access Token** in the web console under *Account > Personal access tokens*. Give it a name, an expiration and an access level:

- **read-only** — queries and subscriptions.
- **read/write** — queries, subscriptions **and mutations**. Anything below that mutates needs this.

The token is shown once at creation and cannot be retrieved later. Treat it like a password.

> The exact header name and format are documented on the GraphQL quickstart page. That page currently does not render (see the caveat at the end), so confirm the header against the console's own network traffic rather than guessing.

An unauthenticated request returns HTTP 401 with:
```json
{"errors":[{"message":"Context creation failed: Unable to authenticate user","extensions":{"code":"UNAUTHENTICATED"}}]}
```

## Register devices

- `createDevice` for one, `createDevices` for a batch (both take `DeviceInput`).
- `updateDevice` to change metadata.
- `deleteDevices` to remove a batch — returns `DeleteDevicesResult`.

There is **no documented idempotency key**. Batch creates and deletes are not documented as retry-safe; make the caller's own request id the guard, and re-read with `myDevices` before retrying a failed batch.

## Find devices

- `myDevices` — your fleet, paged. It returns `DevicePage`, which implements the `PageResult` interface; every list surface here is paged the same way.
- `device` — one device by id.
- Narrow with the `DeviceFilter` input; string fields accept a `StringConditional`.

## Tag devices

- `tags` / `deviceTags` to read (returns `TagPage`).
- `setDeviceTag` / `unsetDeviceTag` to attach and detach.

Tags are the grouping primitive — use them before reaching for client-side bookkeeping.

## License devices

A device needs a seat license to consume corrections.

1. `licenseTypes` — what you can buy.
2. `createLicenses` — mint unattached licenses; or `createLicensesForDevices` to mint and attach in one call.
3. `applyLicenseToDevice` / `removeLicenseFromDevice` — move a license between devices.
4. `updateLicenseAutoRenewal` — control renewal.
5. `licenses` / `license` to read (`LicensePage`, with `Entitlement` describing what each grants).

## Configure corrections with device profiles

A `DeviceProfile` carries the correction settings — `dataIntervalSeconds`, `datumOverride`, `correctionsMode` — in one of two shapes:

- **`MaxTrueRtk`** — True RTK, the 1-3 cm class.
- **`AutoVirtualRtk`** — Virtual RTK, the sub-10 cm class.

Flow: `createDeviceProfile` → `setDeviceProfile` to bind it to a device → `updateDeviceProfile` to change it → `unsetDeviceProfile` / `deleteDeviceProfile` to unwind. Read back with `deviceProfiles` / `deviceProfile`.

Verify what the device is *actually* getting — not what you asked for — by reading the `RtkService` fields on the `Device`: `currentDatum`, `currentCorrectionsMode`, `currentDataIntervalSeconds`. These are the effective values on the live stream and were added in GQL Edge 1.18.1 (2026-07-08) precisely so the requested and effective settings can be compared.

`DatumType` covers `AUTO_LOCAL` and `ITRF2014` plus twelve regional realizations (NAD83, ETRS89, GDA2020, JGD2011, KGD2002, NZGD2000 variants).

## Rotate your own credentials

`viewPersonalAccessTokens`, `createPersonalAccessToken`, `revokePersonalAccessToken`, `deletePersonalAccessToken` — token management is in the API, so rotation can be automated. Revoke before delete if you want an audit trail.

## Caveat

`docs.pointonenav.com` is a Docusaurus SPA whose per-route JavaScript chunks are not deployed, so no reference page body renders and field arguments/return types cannot be read without credentials. Every operation name above is taken from the provider's own published reference page for that operation; **argument shapes are not asserted here.** Introspect with a valid token before writing queries.

## Related

- Surface inventory: `graphql/point-one-navigation-graphql-surface.yml`
- Entity graph: `data-model/point-one-navigation-data-model.yml`
- Auth: `authentication/point-one-navigation-authentication.yml`
