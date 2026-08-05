---
name: Monitor Point One devices and reference stations in real time
description: Use GraphQL subscriptions and station queries to watch device connectivity, solution quality and Polaris reference-station health.
api: point-one-navigation:graphql
operations:
- device
- devices
- stations
- stationsByUuids
generated: '2026-08-05'
method: generated
source: https://docs.pointonenav.com/docs/graphql-api/
---

# Monitor devices and reference stations

Point One's real-time surface is **GraphQL subscriptions**, not webhooks. There is no callback registration, no AsyncAPI document and no event catalog — you hold a subscription open against `https://graphql.pointonenav.com/graphql`.

A **read-only** Personal Access Token is enough; subscriptions do not need read/write.

## Watch devices

- `devices` (subscription) — fleet-wide stream.
- `device` (subscription) — one device.

Both are the live counterpart of the `device` / `myDevices` queries. `Device` implements the `PositionalEntity` interface, so position arrives on the same shape used elsewhere — with `PositionEcef`, `PositionLladd` (decimal degrees) and `PositionLladms` (deg/min/sec) projections available, and `PositionHistory` for the time series.

Read `ConnectionStatus` for link state and `SolutionType` for fix quality — a device that is *connected* is not necessarily *fixed*. Alert on `SolutionType` degradation, not just on disconnects.

Cross-check against `RtkService`: `currentDatum`, `currentCorrectionsMode` and `currentDataIntervalSeconds` tell you what the device is actually being sent, which is the first thing to check when accuracy drops.

## Watch reference stations

- `stations` / `stationsByUuids` — available as both **queries** and **subscriptions**.
- `StationStatus` for up/down, `StationQuality` for health metrics.
- `SkyPlot` and `SkyPlotByBand` for satellite visibility; `Signal` / `Signals` with `SignalFilter` and `SignalType` for per-signal tracking.
- Filter geographically with `StationFilter` and `PointFilter`.

Baseline length matters: RTK accuracy degrades with distance from the serving base station. When a device's fix quality falls, check which stations near it are healthy before treating it as a device fault.

## Rules and gotchas

- **No webhooks.** If you need push-into-your-infrastructure, you must run a subscription client and bridge it yourself.
- **No documented rate limits** on subscriptions or queries.
- Paged queries return `PageResult` implementations (`DevicePage`, `StationPage`, `TagPage`, `LicensePage`) — page through rather than assuming one response holds the fleet.
- Network-wide incidents show at <https://status.pointonenav.com/> (99.99% aggregate uptime target, `<<5s` corrections latency). Check it before paging an on-call engineer.

## Related

- Surface inventory: `graphql/point-one-navigation-graphql-surface.yml`
- Entity graph: `data-model/point-one-navigation-data-model.yml`
- Lifecycle + SLA: `lifecycle/point-one-navigation-lifecycle.yml`
