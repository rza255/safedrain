# Changelog

## 0.2.0-beta — 2026-09-20

- Added separate monitoring/caution, drain, and hard-drain thresholds for every relevant usage window.
- Explicitly requires independent evaluation of quota windows.
- Bases behavior on the most severe state triggered by any relevant window.
- Keeps adaptive monitoring cadence and work-unit granularity under quota pressure.
- Preserves Native Capability First: no daemon, watchdog, telemetry transport, polling script, usage parser, or persistent SafeDrain state.
- Naturalistic validation across engineering and asset-generation workflows.

## 0.1.0 — 2026-09-18

- Initial behavioral SafeDrain skill.
