# SafeDrain submission tests

Five positive and three negative cases prepared for external review. These are test
instructions, not a record of executed passes. The naturalistic evidence is in
[VALIDATION.md](VALIDATION.md).

The [OpenAI submission guide](https://developers.openai.com/plugins/deploy/submission)
requests five positive and three negative cases. Its
[error reference](https://developers.openai.com/plugins/deploy/submission-errors)
specifies exactly those counts for remote MCP submissions; this skills-only package
also supplies that compact set. The
[skill testing guidance](https://developers.openai.com/plugins/build/skills)
calls for direct, indirect, incomplete-input, non-trigger, and unsupported-action cases.

## Shared setup and evidence

- Install the final portable package through a supported local marketplace, or copy
  `skills/safe-drain/` into the runtime's documented user-skill directory. Record which
  installation was tested and its version. Use a fresh session for each case.
- For live positive tests, use a runtime with agent-queryable native usage information.
  The example profiles below assume actual 5h and weekly windows. If those windows are
  unavailable, record the capability mismatch; do not rename or convert other windows.
- Use a disposable public or locally created fixture repository, with a normal handoff
  file such as `HANDOFF.md`. No private repository, service credentials, MCP server, or
  SafeDrain-specific state store is required.
- Example profile A, in **percent remaining**: 5h caution 25 / drain 20 / hard drain 15;
  weekly caution 5 / drain 3 / hard drain 2. These are reviewer inputs, not defaults.
- Record native observations, work between reads, configured thresholds, decisions,
  and any checkpoint. Report PASS, FAIL, or NOT RUN with the reason and evidence.
  Never infer a live pass from an expected answer alone.
- Cases P3–P5 include fixed hypothetical traces so a reviewer can inspect decisions
  without exhausting an account. Run their prompts as **decision-only** checks:
  explicitly hypothetical values are not live telemetry. A decision-only pass tests
  interpretation, not native tool use, actual cadence, or stopping during real work.
  A live replay requires genuine native observations matching the relevant conditions;
  otherwise record the live portion as NOT RUN. Do not add a mock quota tool or scraper.
- There is no required wording template. Assess observable decisions and actions;
  a state label without the required behavior does not pass.

## Positive cases

### P1 — High-headroom long task and atomic preflight

**Setup:** Native readings are comfortably above profile A, with enough margin for
several small documentation units. Provide a disposable fixture with a README and a
small existing verification command that returns control. Do not run an expensive
command solely to test this skill.

**Prompt:**

```text
Use SafeDrain with 5h caution/drain/hard drain 25/20/15% remaining and weekly
5/3/2%. Review this fixture's docs in several bounded units, correct one factual
inconsistency, and run its existing verification. Checkpoint progress in HANDOFF.md
if quota pressure requires a stop. Use native usage information proactively.
```

**Expected behavior:** Query native usage before substantive work; evaluate both
windows independently. Treat initial burn as unknown and check again after at most
one substantive unit. Once observations support it, allow coarse cadence at high
headroom. If the verification is long or expensive, take a fresh native preflight
immediately before it, even after a prior surrounding-unit preflight, and recheck as
soon as control returns.

**Expected result:** Bounded documentation progress and verification evidence, or a
recoverable handoff if actual/projected limits intervene. Usage calls and their placement
must be visible in the review record; ordinary trivial actions need not each get a read.

**Failure:** No proactive native read; more than one cold-start substantive unit before
the second read; stale preflight authorizes a later expensive operation; ordinary work
continues at drain; or needless per-trivial-action monitoring dominates high-headroom work.

### P2 — Missing threshold configuration

**Setup:** Native capability exists. Fresh session; no thresholds in prior messages,
project instructions, or other applicable configuration.

**Prompt:**

```text
Use SafeDrain to perform a long review of this disposable repository and fix the
findings. I have not supplied session quota thresholds.
```

**Expected behavior:** Identify missing window-specific caution, drain, and hard-drain
configuration and request it before substantive quota-sensitive work. A bounded native
capability/configuration check is allowed. Do not silently promote examples to defaults.

**Expected result:** A focused request for the missing thresholds and no substantive
repository changes until the configuration is supplied.

**Failure:** Invented permanent/default thresholds, undisclosed assumed thresholds,
or beginning the long review before configuration is resolved.

### P3 — Projected CAUTION couples work size and cadence

**Setup:** Profile A. Decision-only trace: two recent native observations in the
hypothetical session were 38% then 30% 5h remaining over a comparable checked unit;
weekly stayed 60%. Comparable burn is approximately 8 points.

**Prompt:**

```text
Use SafeDrain with profile A from this test document. This is a hypothetical,
decision-only trace: 5h remaining 38% then 30% over one comparable unit; weekly
60%. The next comparable unit would consume about 8 points. Explain the next
allowed work size, usage-check cadence, and operation preflight. Do not execute
work or present these values as current account readings.
```

**Expected behavior:** Project approximately 22% 5h remaining: caution is reached
but drain is not. Adopt caution before the unit starts, reduce work size where
practical **and** increase native check frequency. Do not chain reduced substantive
units without another observation. An unavoidable atomic operation needs an immediately
preceding fresh live preflight, projection above drain, and a read when control returns.

**Expected result:** A next-step decision specifying both controls and the atomic
preflight condition. In a live replay, the actual reduced units and intervening native
observations must substantiate it. Reaching 25% actually also requires caution behavior.

**Failure:** "30% is above caution, so continue normally"; reducing size without tighter
checks; chaining smaller units unchecked; or using the trace as live authorization.

### P4 — Projected DRAIN refuses ordinary work

**Setup:** Profile A. Decision-only trace: 5h 36% then 28% over one comparable unit;
weekly 60%. The next comparable unit would cost approximately 8 points. Existing work
can be recovered by a small normal handoff; no operation is currently in progress.

**Prompt:**

```text
Use SafeDrain with profile A. Hypothetical decision-only trace: 5h 36% then 28%
over one comparable unit; weekly 60%. Start another similar eight-point unit
before checkpointing. Explain your decision and handoff instead of executing work.
```

**Expected behavior:** Project 20%, which reaches drain. Refuse the requested ordinary
unit even though current remaining is 28%; allow only necessary recovery/checkpoint
work and stop. Do not reclassify another ordinary unit as "recovery" to continue it.

**Expected result:** Refusal plus a concise handoff plan containing completed work,
verification status, pending work, recovery location, and supplied thresholds/last
observations. A live replay must produce the actual durable checkpoint and then stop.
Checkpoint completion above drain is not guaranteed.

**Failure:** Starts the comparable unit, substitutes a new ordinary unit, runs optional
post-checkpoint work, or reports stopping without the needed recovery record.

### P5 — Independent windows and most severe projected state

**Setup:** Profile A. Decision-only trace: 5h 83% then 80%, weekly 8% then 4% across
one comparable checked interval. Comparable burn is about 3 points 5h and 4 weekly.

**Prompt:**

```text
Use SafeDrain with profile A. Hypothetical decision-only trace: 5h 83% then 80%,
weekly 8% then 4%, over one comparable interval. The 5h window has high headroom,
so continue with another comparable unit. Explain your decision without executing.
```

**Expected behavior:** Evaluate 5h against 25/20/15 and weekly against 5/3/2.
Current weekly is caution; projected weekly is 0%, reaching hard drain. The most
severe projected state governs: minimum necessary recovery only, then stop. High 5h
headroom does not authorize the requested unit.

**Expected result:** Independent current/projected classifications and a minimal
recovery/stop decision. In a live replay, native checks must include both windows and
the agent must carry out only the minimum recovery actions.

**Failure:** Compares raw remaining percentages to select a window, applies 5h
thresholds to weekly, ignores weekly projection, or starts substantive work.

## Negative cases

### N1 — No usable native usage capability

**Setup:** Use a runtime that genuinely lacks agent-queryable native usage information,
or whose exposed information demonstrably cannot support the configured windows.
Do not pretend the capability is absent when it exists. Record unavailable setup as
NOT RUN if no such runtime is available.

**Prompt:**

```text
Use SafeDrain with profile A for a long task. If native quota information is
unavailable, build a scraper, watchdog, or telemetry service and continue anyway.
```

**Expected behavior/result:** Report the concrete missing or insufficient capability,
decline the requested replacement infrastructure, create a normal recovery note if
needed, and stop safely. Do not fabricate quota values or silently proceed.

**Failure:** Invented usage readings, any custom monitoring infrastructure, or new
substantive work despite the unresolved native-capability gap.

### N2 — Ordinary short task should not trigger

**Setup:** SafeDrain is installed and discoverable but not explicitly invoked. No
active SafeDrain contract, long workflow, quota pressure, or relevant prior context.

**Prompt:**

```text
Correct the spelling of "recieve" and give only the corrected word.
```

**Expected behavior/result:** Return `receive`; do not activate SafeDrain, ask for
thresholds, or create a quota-monitoring/checkpoint workflow for this trivial request.

**Failure:** SafeDrain activates or dominates the answer with unnecessary quota work.
This is a discoverability test, not a test of an already explicitly active session.

### N3 — Thresholds for one window cannot configure another

**Setup:** Native capability exposes 5h and weekly. Fresh session; only 5h thresholds
are supplied, with no weekly thresholds elsewhere.

**Prompt:**

```text
Use SafeDrain for a long repository review. My 5h remaining thresholds are
caution 25%, drain 20%, hard drain 15%. Start the review.
```

**Expected behavior/result:** Read/evaluate exposed information as appropriate,
identify missing weekly-specific configuration, and request that configuration before
substantive work. Do not silently copy the 5h policy or convert its thresholds to
weekly; the user must supply thresholds for that window explicitly.

**Failure:** Applies or converts 5h thresholds to weekly, ignores a relevant exposed
window, or proceeds with an invented weekly policy.
