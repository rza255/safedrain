---
name: safe-drain
description: Use for long-running or usage-sensitive agent engineering sessions where runtime quota must be monitored and work drained safely before exhaustion.
---

# SafeDrain

Use the runtime's native usage capability proactively during work.
Do not wait for the user to request a usage check.

Use the monitoring, drain, and hard-drain thresholds supplied for each relevant usage window.
Do not invent permanent SafeDrain thresholds.

Check all relevant usage windows exposed by the runtime.
Evaluate each window independently against its own configured thresholds.
Never apply one window's thresholds to another window or compare raw remaining percentages across windows.
Base behavior on the most severe state triggered by any relevant window.

As a window approaches its configured limits, increase the frequency of native usage checks.

As quota pressure increases, reduce the size of newly started work units.
Prefer smaller, recoverable steps over large batches when quota is constrained.

A low-quota observation must change behavior, not merely be reported.
Keep the more conservative monitoring cadence and work granularity while the relevant quota remains constrained.

Before starting an operation that may run for a long time without returning control,
perform a native usage preflight and consider all relevant quota windows before starting it.

When a configured drain threshold is reached:
- do not start new substantive work;
- finish only an already in-progress atomic step needed to reach a recoverable boundary;
- create or update the normal project checkpoint or handoff needed for recovery;
- stop work.

When a configured hard-drain threshold is reached:
- do no further substantive work;
- perform only the minimum actions required for a safe, recoverable checkpoint;
- stop immediately afterward.

Prefer native runtime capabilities first.

Do not create a daemon, watchdog, custom telemetry transport, polling script,
usage parser, persistent SafeDrain state, or other monitoring infrastructure
while this skill is active.

If native behavior proves insufficient, record the concrete observed gap and stop.
Any non-native mechanism requires a separate design decision.
