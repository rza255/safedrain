# Changelog

## 1.0.0 candidate — Unreleased

- Prepared a stable public-release candidate without changing the v0.2.2 behavioral instructions.
- Clarified the skill trigger for long-running or quota-sensitive tasks with native usage limits; excluded ordinary short tasks.
- Retained the portable root manifest and automatic root skills discovery; added supported OpenAI listing metadata and three starter prompts.
- Added five positive and three negative reviewer scenarios with setup, expected behavior, and failure criteria.
- Added user-reported real caution-crossing and explicit cross-thread policy-propagation evidence.
- Clarified that policy survival across automatic compactions does not establish an immediate fresh usage read after every compaction.
- Made runtime-capability limits, missing submission branding, and external review gates explicit. No submission, tag, or release has occurred.

## 0.2.2-beta — 2026-09-22

- Added projected-threshold behavior based on conservative recent comparable usage burn.
- Enters caution behavior before another comparable work unit would reach the caution threshold.
- Refuses new substantive work when projected remaining usage would reach or cross the drain threshold.
- Restricts hard-drain behavior to minimum recovery/checkpoint work.
- Added fresh native usage preflight immediately before expensive atomic operations inside larger compound work units.
- Rechecks native usage as soon as control returns from expensive atomic operations.
- Preserves coarse monitoring cadence while quota headroom is high and tightens cadence as projected threshold risk increases.
- Observed SafeDrain remain behaviorally active across context compaction/resume; this does not establish immediate reads after every compaction.
- Preserves Native Capability First: no daemon, watchdog, custom telemetry, polling loop, usage parser, or persistent SafeDrain state.
- Naturalistic validation includes engineering and asset-generation workloads, including successful preemptive caution and projected-drain refusal.

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
