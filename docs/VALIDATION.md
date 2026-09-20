# SafeDrain v0.2 — naturalistic validation notes

SafeDrain v0.2 is intentionally small. The current evidence is therefore focused on whether the behavioral instructions reliably change agent behavior in real quota-constrained work.

This is **not** a formal benchmark and does not establish universal reliability across all models, accounts, runtime versions, or providers.

## Naturalistic observation 1 — engineering workflow

Profile:

```text
5h:    caution 25%, drain 20%, hard drain 15%
weekly: caution 5%, drain 3%, hard drain 2%
```

Observed behavior:

- the agent proactively checked native usage during the session;
- at 23% remaining in the 5h window it explicitly entered a caution posture;
- it avoided starting new substantive work while an already-running review completed;
- at 19% remaining it entered controlled drain;
- it wrote a durable recovery checkpoint and stopped.

A comparable earlier workflow without the persistent SafeDrain skill had previously observed a 23% reading and then did not check again until roughly 14% remained. The SafeDrain run closed that behavioral gap.

## Naturalistic observation 2 — bounded asset workflow

Profile:

```text
5h:    caution 8%, drain 5%, hard drain 3%
weekly: caution 3%, drain 2%, hard drain 1%
```

Observed behavior:

- at 5% remaining in the 5h window the configured drain threshold was reached;
- the agent did not launch the next expensive image-generation job;
- it updated the existing checkpoint and stopped.

## Naturalistic observation 3 — longer asset workflow

Same aggressive profile as observation 2.

Observed weekly-window progression:

```text
5% → reduced the next work wave from three jobs to two
5% → reduced further to one job
4% → took the smallest remaining bounded standard-row unit
3% → caution; stopped generation fan-out and used bounded checkpoint/QA steps
2% → drain; did not start the pending repair; wrote recovery checkpoint and stopped
```

The 5h window remained healthy enough that the weekly window was the controlling authority. This supports the v0.2 rule that windows must be evaluated independently and behavior must follow the most severe triggered state.

## Current limitations

SafeDrain does not solve cases where the active agent cannot regain control to perform another native usage check. For example, a long uninterrupted synchronous operation may run past a threshold before the model/controller can observe the new usage state.

SafeDrain also does not monitor anything after the active session has ended.

These are known capability boundaries, not reasons to add non-native infrastructure preemptively. Any future mechanism should be justified by a concrete observed failure.

## Release posture

v0.2 is suitable for public beta testing. The skill itself should remain unchanged until external or internal evidence demonstrates a specific failure mode that the current instructions do not handle.
