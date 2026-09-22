# SafeDrain — naturalistic validation notes

SafeDrain is intentionally small. The evidence below focuses on whether behavioral
instructions reliably change agent behavior in real quota-constrained work.

This is **not** a formal benchmark and does not establish universal reliability across
all models, accounts, runtime versions, providers, or workloads.

## v0.2.2-beta naturalistic validation — 2026-09-22

The v0.2.2 candidate was exercised in real multi-step workloads rather than a
synthetic quota simulator.

The two detailed cases below exercise different workload shapes:

- a long engineering workflow with repeated read/edit/test/review units and context compaction;
- an asset-generation workflow containing a comparatively expensive atomic generation
  operation followed by objective validation and a human-approval checkpoint.

Across these runs, no new `THRESHOLD_CROSSED_UNOBSERVED` event was observed.

### Validation case A — Experiment Runner engineering workflow

Profile:

```text
5h:     caution 25%, drain 20%, hard drain 15%
weekly: caution 5%,  drain 3%,  hard drain 2%
```

The run covered multiple substantive engineering findings rather than a synthetic
quota exercise. Approximate 5h progression across the larger workflow was:

```text
~100% → 86% → 60% → 44% → 27%
```

One earlier unit was unusually expensive because it included a legacy-evidence audit.
SafeDrain did not assume that every later unit would cost the same; it used recent
comparable work conservatively and tightened behavior as the next threshold approached.

The most relevant late-stage progression was:

```text
36% → substantive work still allowed

33% → recent comparable burn ≈ 8 percentage points
       projected next comparable unit ≈ 25%
       → preemptive caution entered before another comparable unit

31% → smaller edit/recovery-sized work unit completed

30% → fresh native preflight immediately before a focused atomic verification
       conservative projection: 30% - 8% = 22%
       → still above 20% drain, so one focused verification was authorized

28% → focused verification completed successfully
       projected next comparable unit reaches 20% drain
       → no further comparable substantive work authorized

27% → invariant/diff/commit recovery work only
       projected next comparable unit ≈ 19%
       → additional post-commit repeat and the next engineering finding were refused
```

Observed behaviors:

- high-quota operation retained coarse cadence rather than checking usage after every
  trivial action;
- a fresh burn-rate baseline was re-established after context compaction/resume;
- work-unit size and check frequency tightened together as caution approached;
- the expensive focused verification received a fresh atomic preflight rather than
  inheriting an older broad-unit preflight;
- caution behavior was entered **before** the next comparable unit would cross 25%;
- substantive work stopped **before** another comparable unit would reach or cross 20%
  drain;
- recovery/checkpoint work remained allowed after substantive work had been refused;
- the next major finding was intentionally not started.

This case directly validates the v0.2.2 projected-threshold rule that was introduced
after an earlier regression in which approximately `31% → 22%` occurred across one
unchecked bounded unit and the 25% caution threshold was therefore crossed unobserved.

### Validation case B — Franchie 090 cardinal-generation workflow

Profile:

```text
5h:     caution 8%, drain 5%, hard drain 3%
weekly: caution 3%, drain 2%, hard drain 1%
```

This run contained one expensive image-generation operation, deterministic preservation
and geometry validation, followed by a later human-adjudication checkpoint.

Initial state:

```text
5h:     17%
weekly: 31%
```

Observed progression:

```text
17% → SafeDrain identified burn rate after resume as unknown
       → one bounded preflight unit only

16% → bounded preflight consumed ≈ 1 percentage point
       prior comparable generation burn was estimated at ≈ 3 points
       projected post-generation remaining ≈ 13%
       → generation authorized because projection remained above 8% caution

11% → the single generation returned
       actual generation burn was ≈ 5 points, worse than the earlier estimate
       → SafeDrain immediately tightened subsequent work

10% → byte-preservation + untouched-source geometry validation completed
       → only required provenance/evidence checkpoint remained

8%  → caution threshold reached
       → generation workflow stopped for human review
```

The generated 090 source passed the unchanged objective geometry gate:

```text
edge contact: 0/24 — PASS
dimensions: 1312×1199
foreground bounding box: [97, 49, 1122, 1157]
SHA-256: 92534C28BEDED4957FB0C38F1D35BDAED0D2425A11C9D6441693F50F306B7F25
```

Exactly one generation occurred. No retry, crop, resize, padding, normalization,
repair, composition, or other image transformation occurred before objective validation.

After human visual approval, a separate checkpoint-only adjudication turn continued
under the same SafeDrain profile:

```text
7% → caution active; only a small adjudication/checkpoint unit allowed

6% → read-only checkpoint inspection completed
     SafeDrain determined that another ordinary work unit could reach 5% drain
     → adjudication write + verification treated as the single necessary recovery unit

4% → final user-observed 5h remaining after the durable checkpoint
29% → final user-observed weekly remaining
     → work stopped
```

The final state was therefore inside the configured 5h drain zone (`<5%`) but still
above hard drain (`3%`). SafeDrain had already refused ordinary substantive work at 6%
because projected consumption could reach drain; the remaining quota was used only to
make the accepted 090 result durable and recoverable.

Observed behaviors:

- the expensive generation was preceded by a fresh bounded preflight;
- SafeDrain used a conservative comparable-burn estimate rather than requiring exact
  prediction;
- after actual burn proved higher than the estimate, later units were immediately
  reduced;
- objective validation and provenance work were separated from subjective acceptance;
- caution behavior activated at the configured 8% threshold;
- at 6%, projected drain prevented another ordinary work unit;
- only the required adjudication/checkpoint recovery unit was completed;
- work stopped at a final observed 4%, above the 3% hard-drain threshold;
- the weekly window remained healthy and did not override the more severe 5h state;
- no second generation, 180 cardinal generation, or 270 cardinal generation was started.

This is a strong naturalistic example of the intended v0.2.2 behavior:
**project drain before starting ordinary work, reserve the remaining margin for
recoverability, then stop.**

### v0.2.2 summary

Taken together, the engineering and asset-generation cases provide evidence for:

- high-headroom coarse cadence without excessive usage checking;
- fresh burn-rate baselines after start/resume/context compaction;
- conservative projected-threshold reasoning;
- preemptive caution before a comparable unit would reach caution;
- fresh preflight immediately before expensive atomic operations;
- immediate adaptation when observed burn differs from the prior estimate;
- refusal of new substantive work before projected drain;
- recovery/checkpoint-only behavior near drain;
- independent treatment of 5h and weekly windows;
- no observed recurrence of `THRESHOLD_CROSSED_UNOBSERVED` in the v0.2.2 candidate runs.

This remains beta evidence, not proof of universal behavior across every runtime,
model, account, quota shape, or workload. SafeDrain continues to depend exclusively
on native runtime usage information.

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

v0.2.1 kept the same architecture and added a more explicit behavioral control loop:

1. after session start, resume/handoff, or context compaction, obtain a second usage
   observation after at most one substantive work unit;
2. use the resulting in-session consumption as a rough burn-rate signal;
3. compare recent consumption with the remaining distance to the next configured threshold;
4. do not repeat enough comparable unchecked work to reach or cross that threshold;
5. couple smaller work units with more frequent usage checks;
6. treat a preflight as valid only for the immediately following long/expensive operation;
7. record an unobserved threshold crossing as a monitoring failure and tighten cadence.

The later `31% → 22%` regression showed that this was still not sufficient by itself:
SafeDrain also needed to adopt the next threshold's behavior **before** starting a
comparable unit whose conservative projected burn would reach or cross that threshold.

That projected-threshold rule, together with fresh atomic-operation preflight behavior,
became the core of v0.2.2.

No daemon, timer, background watcher, custom telemetry transport, UI parser, or persistent
SafeDrain state was added.

## Current limitations

SafeDrain still cannot solve cases where the active agent cannot regain control to perform
another native usage check. A single uninterrupted synchronous operation can cross a
threshold before the model/controller regains control.

The burn-rate signal is deliberately approximate. It is used to make the agent more
conservative, not to predict exact token or quota consumption.

SafeDrain cannot guarantee that a necessary in-progress recovery/checkpoint step will
finish above the configured drain threshold. The intended behavior is to refuse new
ordinary substantive work before projected drain, use only the minimum work needed for
recoverability, and stop. The Franchie case above ended at 4% with a 5% drain threshold
after exactly such a checkpoint-only completion.

SafeDrain does not monitor anything after the active session has ended.

## Release posture

v0.2.2 is a **public beta / prerelease**.

Current naturalistic evidence supports the projected-threshold and fresh-atomic-preflight
behavior across both engineering and asset-generation workflows. In particular, the
v0.2.2 has demonstrated:

- preemptive caution based on conservative recent comparable burn;
- projected-drain refusal before another ordinary substantive unit;
- recovery-only behavior close to drain;
- correct dominance of the most severe relevant usage window;
- disciplined stopping without opportunistically starting another major unit.

This evidence is materially stronger than the evidence available at the v0.2.1 release,
but it still does not establish universal reliability.

Further infrastructure should still be added only if a concrete native-capability gap
remains after behavioral correction.
