# SafeDrain

**Usage-aware graceful drain for long-running Codex / agent sessions.**

SafeDrain is a small behavioral skill that tells an agent to use the runtime's
**native usage/quota capability** proactively, adapt work-unit size and usage-check
cadence as quota pressure rises, and reserve margin for a recoverable stop.

It deliberately does **not** add a daemon, watchdog, custom telemetry transport, polling
script, usage parser, or persistent SafeDrain state. The design principle is simple:
use the capability the runtime already has, and build extra machinery only if a concrete
gap is observed.

> **Status:** SafeDrain v1.0.0 is the first stable public GitHub release. The behavioral
> instructions are unchanged from v0.2.2; the trigger description, listing metadata,
> evidence wording, and submission tests are prepared for OpenAI directory submission.
> Stable means a coherent documented contract with meaningful naturalistic evidence and
> explicit limitations, not universal reliability. No OpenAI directory submission or
> approval is claimed.

## What SafeDrain does

SafeDrain instructs the active agent to:

- check all relevant usage windows exposed by the runtime;
- evaluate each window independently against its own thresholds;
- avoid treating "above threshold" as sufficient evidence that the current cadence is safe;
- establish a fresh in-session burn-rate baseline after start, resume/handoff, or context compaction;
- compare recent quota consumption with distance to the next configured threshold;
- reduce work-unit size **and** increase usage-check frequency together under quota pressure;
- avoid chaining enough unchecked comparable work to cross the next threshold;
- preflight usage immediately before each long-running or expensive operation;
- stop starting substantive work at the configured drain threshold;
- finish only the current atomic step needed to reach a recoverable boundary;
- create/update the normal project checkpoint or handoff;
- stop immediately after a minimal recovery checkpoint at hard drain.

## Requirement

SafeDrain depends on the active runtime exposing a **native usage / rate-limit capability
that the agent can query**. If the runtime cannot expose current quota information to the
agent, SafeDrain does not invent an alternative telemetry stack.

Installing the skill or plugin does not supply that capability. Its availability varies
by product, account, and runtime; support is not assumed across all ChatGPT or Codex
surfaces. The workflow must also supply thresholds for the relevant windows and its
normal project recovery/checkpoint mechanism.

If native usage information itself proves insufficient, the skill instructs the agent to
record the concrete gap and stop rather than silently creating monitoring infrastructure.

## Install as a standalone skill

Current OpenAI Codex documentation supports user skills under:

```text
$HOME/.agents/skills
```

Copy this repository's skill directory to:

```text
$HOME/.agents/skills/safe-drain/
```

The resulting file should be:

```text
$HOME/.agents/skills/safe-drain/SKILL.md
```

On Windows PowerShell, from the cloned repository:

```powershell
New-Item -ItemType Directory -Force "$HOME\.agents\skills\safe-drain" | Out-Null
Copy-Item ".\skills\safe-drain\SKILL.md" "$HOME\.agents\skills\safe-drain\SKILL.md" -Force
```

Then invoke it explicitly in Codex with:

```text
$safe-drain
```

## Set the quota thresholds in the prompt

SafeDrain intentionally contains **no permanent universal thresholds**. Supply the
thresholds for the current workflow/session.

Use this prompt template:

```text
$safe-drain

SafeDrain session thresholds:

5h window:
- monitoring/caution: <N>% remaining
- drain: <N>% remaining
- hard drain: <N>% remaining

Weekly window:
- monitoring/caution: <N>% remaining
- drain: <N>% remaining
- hard drain: <N>% remaining

[Your task here]
```

Each usage window is evaluated against **its own thresholds**. SafeDrain must not convert
one window into another or apply one window's thresholds to a different window.

### Example profile: long engineering session

```text
$safe-drain

SafeDrain session thresholds:

5h window:
- monitoring/caution: 25% remaining
- drain: 20% remaining
- hard drain: 15% remaining

Weekly window:
- monitoring/caution: 5% remaining
- drain: 3% remaining
- hard drain: 2% remaining
```

### Example profile: intentionally aggressive low-quota finish

```text
$safe-drain

SafeDrain session thresholds:

5h window:
- monitoring/caution: 8% remaining
- drain: 5% remaining
- hard drain: 3% remaining

Weekly window:
- monitoring/caution: 3% remaining
- drain: 2% remaining
- hard drain: 1% remaining
```

These are **examples, not universal recommendations**. Choose thresholds based on the
cost and recoverability of the work you are about to start.

## Cadence and projected thresholds

Threshold classification and monitoring cadence are separate decisions.

Being above caution does **not** automatically mean that continuing with the current
unchecked work interval is safe.

After the first usage read in a session, after a resume/handoff, or after context
compaction, SafeDrain allows at most one substantive work unit before another native
usage read. Those two observations provide a rough in-session burn-rate baseline.

A substantive work unit is a bounded sequence that materially advances the task, such as
one test-fix cycle, one generation job, one review, one commit-sized change, or one
multi-step diagnostic batch.

Once a baseline exists, SafeDrain compares recent observed quota consumption with the
remaining distance to the next configured threshold. If repeating comparable unchecked
work could reach or cross that threshold, it adopts that threshold's behavior before the
unit starts. At projected caution, it reduces work size where practical and checks more
often. At projected drain, it refuses new substantive work, checkpoints, and stops.

Work-unit reduction and increased monitoring are complementary controls:

```text
quota pressure rises
→ smaller recoverable work units
AND
→ more frequent native usage checks
```

A usage preflight is single-use for the immediately following long-running or expensive
operation. Intervening work makes that preflight stale for any later expensive operation.

## Naturalistic evidence

Earlier v0.2 runs showed good adaptive behavior in several engineering and asset workflows.

Two later naturalistic runs reproduced the same cadence failure class:

```text
THRESHOLD_CROSSED_UNOBSERVED
```

One engineering run went from 28% to 15% in the 5h window without observing the configured
25% caution or 20% drain transition. A fresh standalone asset run later went from 12% to
4% without observing the configured 8% caution or 5% drain transition.

The second reproduction used explicit `$safe-drain`, a standalone skill, a fresh thread,
and no observed context compaction before failure. This makes plugin packaging, implicit
invocation, and context compaction insufficient explanations for the failure class.

v0.2.1 tightened cadence; a later regression motivated v0.2.2's projected-threshold
behavior and fresh atomic-operation preflights. Subsequent naturalistic engineering
and asset-generation runs demonstrated preemptive caution, projected-drain refusal,
recovery-only work near drain, and adaptation when actual burn exceeded an estimate.

Additional engineering evidence observed a real 5h caution transition:
approximately `26% → 25% → 23% → 21%`, with smaller work and tighter cadence at 25%,
then a fresh preflight for one required local verification while projected remaining
stayed above 20% drain. No unobserved drain crossing was observed in that sequence.

Multiple automatic context compactions left the policy behaviorally active. Some
post-compaction usage reads were immediate; others followed a bounded read-only or
small interval. This does not establish immediate detection and a fresh read after
every compaction.

One parent explicitly carried the SafeDrain contract into a child thread; the child
reloaded the skill and took its own fresh native usage reading. That is
`CROSS_THREAD_POLICY_PROPAGATION_PASS`, not evidence of implicit activation.

See [`docs/VALIDATION.md`](docs/VALIDATION.md) for evidence provenance, historical
failures, and limitations. These observations are naturalistic validation, not a
formal benchmark or a guarantee. A necessary recovery step can still finish inside
the drain zone, and an uninterrupted operation can cross a threshold before control
returns.

## What SafeDrain is not

SafeDrain is not:

- a background daemon;
- a quota scraper;
- a UI parser;
- an event subscription service;
- an external telemetry server;
- a guarantee that the agent can be interrupted during an operation that never returns control;
- a replacement for project-specific recovery/checkpoint logic.

SafeDrain changes **agent behavior around native quota information**. Your existing project
checkpoint/handoff mechanism remains the recovery authority.

## Native Capability First

SafeDrain follows a narrow design rule:

```text
need capability
→ try the native capability first
→ characterize reliability and conditions
→ identify the actual gap
→ build only the gap
```

Capability existence and reliable capability utilization are different problems.
SafeDrain addresses the latter: the runtime may already know its usage state, but the
agent may fail to check it proactively or fail to change behavior after a low-quota
observation.

## Repository layout

```text
.
├── plugin.json
├── README.md
├── LICENSE
├── CHANGELOG.md
├── assets/
│   └── safedrain.png
├── docs/
│   ├── VALIDATION.md
│   └── SUBMISSION_TESTS.md
└── skills/
    └── safe-drain/
        └── SKILL.md
```

The portable root `plugin.json` uses the Agent Plugins 1.0.0 schema and keeps OpenAI
listing metadata in `extensions.com.openai.interface`. Skills are discovered from
root `skills/` automatically; no legacy skills declaration or compatibility overlay
is needed. The plugin contains no MCP server or custom telemetry implementation.
This follows the current [OpenAI packaging requirements](https://developers.openai.com/plugins/build/plugins).

## Public submission preparation

The repository is prepared for later submission through the **Skills only** path.
The [submission guide](https://developers.openai.com/plugins/deploy/submission) calls
for local testing, realistic starter prompts, five positive and three negative cases,
verified publisher identity, availability choices, and release notes. Reviewer scenarios
and setup are in [`docs/SUBMISSION_TESTS.md`](docs/SUBMISSION_TESTS.md); preparing them
does not mean they have passed portal review.

The [submission error reference](https://developers.openai.com/plugins/deploy/submission-errors)
sets the stricter final listing limits: display name and short description up to 30
characters, developer name up to 80, long description up to 4,000, and at most three
unique, single-line starter prompts of up to 128 characters. Listing URLs are optional
for skills-only ZIP uploads, despite the general preparation checklist naming them.
The repository URL is supplied as the website; separate support, privacy, and terms
URLs are not supplied.

The approved production branding image at [`assets/safedrain.png`](assets/safedrain.png)
is used unchanged for both `logo` and `composerIcon`, each referencing
`./assets/safedrain.png`. The decoded PNG is square (1,254 × 1,254 pixels), is
1,023,137 bytes, and meets the current directory image format, dimension, and size limits.
Screenshots are not applicable to this skills-only package.

Remaining external gates are matching verified developer identity and submission
write access, final package testing/upload, portal test-case entry where
requested, availability and release notes, policy attestations, automated skill scans,
and OpenAI review. This package has not been submitted to or approved by OpenAI.

Local package installation was checked with Codex CLI 0.153.4 using a temporary
marketplace and isolated configuration: the portable package was recognized, installed,
and listed as version 1.0.0, with the skill contents preserved. This checks packaging,
not implicit activation or portal certification. The directory's automated safety,
security, image, and identity checks still require the submission infrastructure.

## Contributing / feedback

The most useful feedback is a **concrete failure case**:

- runtime/model used;
- quota windows exposed;
- thresholds supplied;
- sequence of native usage observations;
- substantive work performed between those observations;
- whether context compaction/resume occurred;
- whether control returned between operations;
- what SafeDrain did;
- what you expected it to do.

Please avoid proposing a daemon or custom telemetry layer unless a reproducible
native-capability gap requires one.

## License

MIT. See [`LICENSE`](LICENSE).
