# SafeDrain v0.2.1 — naturalistic validation notes

SafeDrain is intentionally small. The evidence below focuses on whether behavioral
instructions reliably change agent behavior in real quota-constrained work.

This is **not** a formal benchmark and does not establish universal reliability across
all models, accounts, runtime versions, providers, or workloads.

## Earlier successful observations under v0.2

### Observation 1 — engineering workflow

Profile:

```text
5h:     caution 25%, drain 20%, hard drain 15%
weekly: caution 5%,  drain 3%,  hard drain 2%
```

Observed behavior:

- the agent proactively checked native usage during the session;
- at 23% remaining in the 5h window it explicitly entered a caution posture;
- it avoided starting new substantive work while an already-running review completed;
- at 19% remaining it entered controlled drain;
- it wrote a durable recovery checkpoint and stopped.

A comparable earlier workflow without persistent SafeDrain behavior had observed a 23%
reading and then did not check again until roughly 14% remained. This v0.2 run appeared
to close that gap, but later regressions showed the cadence behavior was not yet reliable.

### Observation 2 — bounded asset workflow

Profile:

```text
5h:     caution 8%, drain 5%, hard drain 3%
weekly: caution 3%, drain 2%, hard drain 1%
```

Observed behavior:

- at 5% remaining in the 5h window the configured drain threshold was reached;
- the agent did not launch the next expensive image-generation job;
- it updated the existing checkpoint and stopped.

### Observation 3 — longer asset workflow

Same aggressive profile as observation 2.

Observed weekly-window progression:

```text
5% → reduced the next work wave from three jobs to two
5% → reduced further to one job
4% → took the smallest remaining bounded standard-row unit
3% → caution; stopped generation fan-out and used bounded checkpoint/QA steps
2% → drain; did not start the pending repair; wrote recovery checkpoint and stopped
```

The 5h window remained healthy enough that the weekly window was the controlling
authority. This supports independent evaluation of usage windows and following the
most severe triggered state.

## Reproduced cadence failure 1 — engineering workflow

Profile:

```text
5h:     caution 25%, drain 20%, hard drain 15%
weekly: caution 5%,  drain 3%,  hard drain 2%
```

Observed progression:

```text
42% → agent explicitly reduced work-unit size
28% → still reported "normal" because it was above caution
15% → hard-drain state became known only after the user surfaced the reading
```

Between 42% and 28%, several separate substantive RED→GREEN / documentation units were
completed without another native usage observation.

Between 28% and 15%, additional substantive work and an independent rereview launch
occurred without a fresh usage read. Context compaction occurred during this later
phase, so it was initially a plausible contributor.

Result:

```text
THRESHOLD_CROSSED_UNOBSERVED
```

The configured caution and drain thresholds were crossed without an observed SafeDrain
transition even though control had returned repeatedly between substantive operations.

## Reproduced cadence failure 2 — fresh standalone asset workflow

Profile:

```text
5h:     caution 8%, drain 5%, hard drain 3%
weekly: caution 3%, drain 2%, hard drain 1%
```

Test conditions intentionally differed from failure 1:

- fresh thread;
- explicit `$safe-drain`;
- standalone skill rather than implicit plugin activation;
- no observed context compaction before the failure.

Observed progression:

```text
12% → initial native usage read; agent classified the window as healthy
 4% → next native usage read immediately before the expensive generation step
```

Between those observations, the agent performed multiple read/diagnostic/image-inspection,
skill-loading, reasoning, and prompt-editing operations.

The 8% caution and 5% drain thresholds were therefore crossed unobserved.

Result:

```text
THRESHOLD_CROSSED_UNOBSERVED
```

This reproduced the same failure class without plugin packaging, implicit invocation,
or context compaction. Those factors may still affect salience, but they are not required
for the failure.

## v0.2.1 corrective hypothesis

The reproduced failures indicate that v0.2's instruction:

> As a window approaches its configured limits, increase the frequency of native usage checks.

was too subjective by itself.

v0.2.1 keeps the same architecture and adds a more explicit behavioral control loop:

1. after session start, resume/handoff, or context compaction, obtain a second usage
   observation after at most one substantive work unit;
2. use the resulting in-session consumption as a rough burn-rate signal;
3. compare recent consumption with the remaining distance to the next configured threshold;
4. do not repeat enough comparable unchecked work to reach or cross that threshold;
5. couple smaller work units with more frequent usage checks;
6. treat a preflight as valid only for the immediately following long/expensive operation;
7. record an unobserved threshold crossing as a monitoring failure and tighten cadence.

No daemon, timer, background watcher, custom telemetry transport, UI parser, or persistent
SafeDrain state was added.

## Current limitations

SafeDrain still cannot solve cases where the active agent cannot regain control to perform
another native usage check. A single uninterrupted synchronous operation can cross a
threshold before the model/controller regains control.

The burn-rate signal is deliberately approximate. It is used to make the agent more
conservative, not to predict exact token or quota consumption.

SafeDrain does not monitor anything after the active session has ended.

## Release posture

v0.2.1 is a **corrective public beta**.

The two cadence regressions are evidence for the patch, not evidence that the patch itself
is already validated. The next naturalistic runs should specifically test whether:

- a fresh session establishes a second usage observation after one substantive unit;
- cadence tightens before caution rather than only after crossing it;
- work-unit reduction and check-frequency increase occur together;
- a fresh preflight happens before each later long/expensive operation;
- `THRESHOLD_CROSSED_UNOBSERVED` does not recur when the runtime repeatedly returns control.

Further infrastructure should still be added only if a concrete native-capability gap
remains after behavioral correction.
