# Changelog

## 0.2.1-beta — 2026-09-21

- Tightened proactive usage-monitoring cadence after two reproduced naturalistic `THRESHOLD_CROSSED_UNOBSERVED` failures.
- Added a cold-start/resume/context-compaction rule: establish a fresh burn-rate baseline after at most one substantive work unit.
- Coupled work-unit reduction with increased usage-check frequency; they are no longer treated as alternative controls.
- Added recent-consumption / distance-to-next-threshold reasoning to prevent repeating an unsafe unchecked work interval.
- Made long-operation usage preflights single-use; a preflight for one expensive operation cannot authorize a later one after intervening work.
- Added explicit monitoring-failure handling when thresholds are crossed unobserved despite the runtime returning control.
- Preserved Native Capability First and the no-daemon/no-watchdog/no-custom-telemetry architecture.
- Updated validation notes to distinguish successful v0.2 observations from the two later cadence regressions that motivated this patch.

## 0.2.0-beta — 2026-09-20

- Added separate monitoring/caution, drain, and hard-drain thresholds for every relevant usage window.
- Explicitly requires independent evaluation of quota windows.
- Bases behavior on the most severe state triggered by any relevant window.
- Keeps adaptive monitoring cadence and work-unit granularity under quota pressure.
- Preserves Native Capability First: no daemon, watchdog, telemetry transport, polling script, usage parser, or persistent SafeDrain state.
- Naturalistic validation across engineering and asset-generation workflows.

## 0.1.0 — 2026-09-18

- Initial behavioral SafeDrain skill.
