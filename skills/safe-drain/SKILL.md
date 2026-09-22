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

Do not classify a window as operationally safe merely because its current remaining
percentage is above the next configured threshold.
Monitoring cadence must also consider how quickly quota has been consumed and how
much unchecked work is about to start.

Treat work-unit size and usage-check cadence as complementary controls, not alternatives.
Whenever quota pressure causes you to reduce work-unit size, also increase native
usage-check frequency.
Do not execute several reduced substantive work units consecutively without another
usage observation.

A substantive work unit is a bounded sequence that materially advances the task,
for example:
- one test-fix cycle;
- one generation job;
- one review;
- one commit-sized change;
- one multi-step diagnostic or analysis batch.

A substantive work unit may itself contain a much more expensive atomic operation,
for example:
- running a long test target;
- launching an independent review;
- running a generation job;
- executing a long build or analysis;
- starting an operation that may not return control for a while.

Do not treat a preflight before the surrounding work unit as sufficient for a later
expensive atomic operation after intervening work.
Perform a fresh native usage preflight immediately before the expensive atomic operation.

After the first native usage observation in a session, after a resume/handoff, or
after context compaction, treat the recent burn rate as unknown.
Complete at most one substantive work unit before performing another native usage
check.
Use those two observations to establish a fresh in-session cadence baseline.
Do not create persistent SafeDrain state for this purpose.

Once two or more recent observations exist for the same window, use the observed
consumption between checks as a rough burn-rate signal.
For each window, compare the remaining distance to its next configured threshold
with the quota consumed over the preceding checked work interval.

Use recent comparable work-unit consumption conservatively.
Do not require an exact prediction.

If recent comparable consumption is greater than or equal to the remaining distance
to the next configured threshold, adopt that threshold's behavior before starting
another comparable unit.

Conceptually:

current remaining
- conservative recent comparable burn
= projected remaining after the next comparable unit

If the projected remaining reaches or crosses:
- caution: enter caution behavior before starting the unit;
- drain: do not start new substantive work; checkpoint and stop;
- hard drain: perform only minimum recovery work and stop.

Preemptive caution does not automatically forbid an unavoidable atomic operation.
If the operation cannot be split, a fresh native usage preflight is required
immediately before it.
Only proceed if the projected remaining stays above drain.
Check usage again as soon as control returns.

If another comparable unchecked work unit could reach or cross the next threshold:
- do not start that unit at the same size under the previous state;
- reduce the work unit if possible;
- increase usage-check frequency;
- if the expensive portion is atomic, preflight immediately before that atomic operation.

If multiple comparable work units could cumulatively reach the next threshold,
do not chain enough of them together unchecked to reach or cross it.
Check usage between smaller recoverable units instead.

As a window approaches its configured limits, increase the frequency of native usage checks.
When in doubt about whether the next bounded unit could cross a threshold, check sooner.

A low-quota observation must change behavior, not merely be reported.
Keep the more conservative monitoring cadence and work granularity while the relevant
quota remains constrained.

Before starting an operation that may run for a long time, consume substantial quota,
or not return control for a while, perform a fresh native usage preflight and consider
all relevant quota windows before starting it.

A preflight is valid only for the immediately following long-running or expensive
operation.
Do not reuse an earlier preflight to authorize a later long-running or expensive
operation after intervening work has occurred.

The monitoring objective is to observe configured threshold transitions before
substantially lower thresholds are reached whenever the runtime returns control often
enough to permit checks.

If a later native usage observation shows that one or more configured thresholds were
crossed unobserved even though control returned between substantive work units, treat
that as a SafeDrain monitoring failure:
- record the concrete failure;
- immediately adopt a more conservative cadence;
- perform native usage checks between substantive work units while work is still allowed;
- follow drain or hard-drain behavior immediately if either threshold has already been reached.

When a configured caution threshold is reached or preemptively projected:
- reduce work-unit size where practical;
- increase native usage-check frequency;
- preflight immediately before expensive atomic operations;
- do not let a prior broad work-unit preflight authorize a later expensive operation;
- continue only while projected remaining stays above drain.

When a configured drain threshold is reached or preemptively projected:
- do not start new substantive work;
- finish only an already in-progress atomic step needed to reach a recoverable boundary;
- create or update the normal project checkpoint or handoff needed for recovery;
- stop work.

When a configured hard-drain threshold is reached or preemptively projected:
- do no further substantive work;
- perform only the minimum actions required for a safe, recoverable checkpoint;
- stop immediately afterward.

Prefer native runtime capabilities first.

Do not create a daemon, watchdog, custom telemetry transport, polling script,
usage parser, persistent SafeDrain state, or other monitoring infrastructure
while this skill is active.

If native usage information itself is unavailable or proves insufficient, record
the concrete observed gap and stop.
Any non-native mechanism requires a separate design decision.
