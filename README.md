# SafeDrain

**Usage-aware graceful drain for long-running Codex / agent sessions.**

SafeDrain is a small behavioral skill that tells an agent to use the runtime's
**native usage/quota capability** proactively, adapt work-unit size and usage-check
cadence as quota pressure rises, and checkpoint safely before a quota window is exhausted.

It deliberately does **not** add a daemon, watchdog, custom telemetry transport, polling
script, usage parser, or persistent SafeDrain state. The design principle is simple:
use the capability the runtime already has, and build extra machinery only if a concrete
gap is observed.

> **Status:** v0.2.1 beta. Corrective cadence patch prompted by two reproduced naturalistic
> `THRESHOLD_CROSSED_UNOBSERVED` failures. The patch itself still requires further
> naturalistic validation.

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

## Cadence model in v0.2.1

v0.2.1 makes one important correction: threshold classification and monitoring cadence
are separate decisions.

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
work could cross that threshold, it must check sooner and/or make the next unit smaller.

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

v0.2.1 is the behavioral patch for that evidence. See
[`docs/VALIDATION.md`](docs/VALIDATION.md) for the compact evidence record and the
remaining validation questions.

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
├── docs/
│   └── VALIDATION.md
└── skills/
    └── safe-drain/
        └── SKILL.md
```

The root `plugin.json` keeps the repository ready to package as a minimal skills-only
plugin. The plugin contains no MCP server or custom telemetry implementation.

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
